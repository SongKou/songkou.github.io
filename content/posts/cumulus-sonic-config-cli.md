+++
title = 'Configuration Standardization and Automation on Cumulus and SONiC Switches'
date = 2026-08-18T17:30:00+08:00
draft = false
categories = ['Network', 'Python']
tags = ['Cumulus Linux', 'NVUE', 'SONiC', 'Jinja2', 'Automation', 'asyncio', 'REST API', 'config_db']
+++

> **Note:** this post documents `cumulus_sonic_cli.py`, a standalone Python script I wrote to manage **Cumulus Linux 5.x (NVUE)** and **SONiC** switches from a single vendor-neutral YAML per device. The full source is at the bottom of the post.

## 1. What the tool does

The idea is the classic intent-based loop, kept deliberately small:

```text
device_vars/<hostname>.yaml     one YAML per device = the single source of truth
        |  rendered through
j2_template/<vendor>.j2         per-vendor Jinja2 template, chosen by VENDOR in the yaml
        v
vendor-native config dict       NVUE tree (cumulus) / config_db (sonic)
        |  --diff               compare against the running config
        |  --commit             push it, with confirmation and rollback paths
```

One script, five capabilities:

| Flag | What it does | Needs a connection? |
|---|---|---|
| `--get_config` | export the full running config as native JSON | yes |
| `--to_yaml` | same, converted to YAML | yes |
| `--generate_config` | render the device YAML through the vendor template | no |
| `--generate_bootstrap` | render the day-0 bootstrap commands for a fresh box | no |
| `--diff` (+`--dry_run`, `--commit`) | diff running vs intended, optionally apply | yes |

Everything it produces lands on disk too: generated configs and exports in `configs/`, diffs in `diffs/`.

```bash
python3 cumulus_sonic_cli.py -d leaf1 --generate_config     # offline render
python3 cumulus_sonic_cli.py -d "leaf.*" --get_config --parallel
python3 cumulus_sonic_cli.py -d leaf1 --diff --commit       # the full loop
```

Both platforms are Linux-based whitebox NOSes, both represent their entire configuration as a single JSON/YAML tree, and yet almost every interesting design decision in the script comes from where the two **differ**. That contrast is what this post is really about.

## 2. No inventory file: the device YAML *is* the inventory

There is no `hosts.ini`, no `inventory.yaml`, and no `-vendor` parameter. Each file in `device_vars/` carries its own identity:

```yaml
# device_vars/leaf1.yaml (head of the file)
GROUP: PRIMARY
ROLE: LEAF
VENDOR: CUMULUS
MANAGEMENT:
  HOSTNAME: leaf1
  IP_MGMT_VIP: 192.168.0.21
  IP_DEFAULT_GATEWAY: 192.168.0.1
  MASK: 24
  API_PORT: 8765
```

`-d leaf1` (or a regex like `-d "leaf.*"`, or `-f device_list.txt`) is matched against the **filenames** in `device_vars/`. Every matching file is parsed, `VENDOR` decides which template and which transport will be used, and `MANAGEMENT` provides the address and ports. Adding a device to the system is literally "drop one YAML file in a folder" — there is no second place to keep in sync.

The connection address follows a preference order: an explicit `CONNECT_ADDRESS` wins, then the loopback IP if the device is flagged as in-band managed, then the management VIP.

## 3. Rendering: vendor-neutral variables in, vendor-native tree out

The same *kind* of variables (`VLANS`, `INTERFACES`, `BONDS`, `SVIS`, `BGP`, …) render through two very different templates:

- `cumulus.j2` produces the **NVUE config tree** — the same shape you get from `GET /nvue_v1/?rev=applied`, e.g. `interface: → swp1: → type: swp`.
- `sonic.j2` produces **config_db tables** — the same shape as `show runningconfiguration all`, e.g. `PORT: → Ethernet0: → admin_status: up`.

A small excerpt of each, fed from the same idea ("this port is an access port in VLAN X"):

```jinja
{# cumulus.j2 #}
  {{ intf.NAME }}:
    type: swp
{% if intf.UNTAGGED_VLAN is defined %}
    bridge:
      domain:
        {{ device_vars.BRIDGE.DOMAIN | default('br_default') }}:
          access: {{ intf.UNTAGGED_VLAN }}
{% endif %}
```

```jinja
{# sonic.j2 #}
VLAN_MEMBER:
{% for intf in live_interfaces %}
{% if intf.UNTAGGED_VLAN is defined %}
  Vlan{{ intf.UNTAGGED_VLAN }}|{{ intf.NAME }}:
    tagging_mode: untagged
{% endif %}
{% endfor %}
```

The templates emit **YAML text**, and the script immediately parses that text back into a Python dict (`yaml.safe_load(rendered)`). From that point on everything downstream — diffing, merging, pruning, uploading — works on dicts, never on strings. Two template subtleties worth calling out:

1. **The quoted `'on'`.** NVUE wants the literal strings `on`/`off`, but unquoted `on` in YAML parses as the boolean `True`. Every `enable: 'on'` in the template is quoted for that reason. This is the kind of bug that only shows up as a mysterious diff ("`enable: true` → `enable: on`") if you get it wrong.
2. **The cumulus template renders "set config only, no defaults"** — the same view as `?rev=applied&filled=false`. If the template emitted defaults that the device doesn't store explicitly, every diff would be polluted with phantom changes.

The rendered output for leaf1 (a full EVPN/MLAG leaf: VNIs, MLAG bond, peerlink, VRR SVIs, tenant VRF with L3VNI, BGP unnumbered underlay) is saved as `configs/leaf1_gen_config.yaml`, and `--generate_config` needs no credentials at all — rendering is pure local computation.

## 4. Two vendors, two transports

| | Cumulus | SONiC |
|---|---|---|
| read running config | NVUE **REST API** `GET /nvue_v1/?rev=applied` (https :8765) | **SSH**: `show runningconfiguration all` |
| diff / dry run | **SSH**: `nv config patch` + `nv config diff` | computed **locally**, optional `config replace --dry-run` |
| apply | **SSH**: `nv config apply` | **SSH**: `config replace` |

Cumulus Linux has **no NETCONF**, but NVUE ships a clean REST API, so config export goes over REST — the script keeps one `requests.Session` per device so the TLS connection is reused across calls. The REST API has no session concept, so "connect" is simply a cheap `GET /system?rev=applied` to verify reachability and credentials up front, before any real work starts.

The diff and apply on Cumulus deliberately go over **SSH** instead, because the whole point (next section) is to use the device's own candidate-revision workflow — `nv config patch / diff / apply / detach` — which is a CLI flow.

SONiC does everything over SSH via `asyncssh`. One gotcha handled early: `show runningconfiguration all` appends the raw FRR configuration as a display-only `bgpraw` key. That key is **not** a config_db table, and writing it back on commit would be wrong — so it is popped immediately after the JSON is parsed.

## 5. The diff: let Cumulus do it, do it yourself for SONiC

This is the most interesting split in the script. The two platforms get two completely different diff strategies, each playing to what the platform is good at.

### 5.1 Cumulus: the diff *is* a device-side dry run

NVUE has a first-class candidate-revision workflow, so the script doesn't compute any diff locally. Instead:

1. `nv config detach` — start from a clean candidate, discarding leftovers from earlier runs
2. SFTP the rendered config to `/tmp/cs_cli_candidate.yaml`
3. `nv config patch /tmp/cs_cli_candidate.yaml` — the device **validates** the config while merging it into the candidate revision
4. `nv config diff -o commands` — the device prints exactly what would change, as `nv set` / `nv unset` commands

The output for the leaf1 example (removing a decommissioned management address, see section 6):

```text
nv unset interface eth0 ip address 10.10.80.21/24
nv unset interface eth0 ip gateway 10.10.80.2
```

Every `--diff` on Cumulus is therefore already a device-validated dry run — invalid config fails at the `patch` step, before anything is applied. The flip side of parking a candidate revision on the device is that you must never leave it behind: a `finally` block detaches any pending candidate on every path that doesn't apply it (no `--commit`, user answers "n", empty diff, or an exception halfway through patching — the pending flag is set *before* the patch call precisely so a half-completed patch still gets cleaned up).

One quirk: the unset tree is patched as a **separate file before** the set tree, because the NVUE patch converter cannot handle the same path appearing in both `unset:` and `set:` within a single file. Both patches land in the same candidate revision, so the eventual apply is still one atomic change.

### 5.1.1 A real run: `--diff` against a factory-fresh leaf

Here is the complete capture of a `--diff` run against leaf2, a freshly bootstrapped device with no configuration beyond the management interface (leaf2's device YAML is a lab clone of leaf1's, with only the management identity changed — hence the identical loopback and router-id):

```text
ekou@saltmaster:~/cumulus_test$ python3 cumulus_sonic_cli.py -d leaf2 -u cumulus --diff
Enter device password: 
CS_CLI: get_filtered_devices

CS_CLI: Found 1 device(s)

CS_CLI: ['leaf2']

CS_CLI: Process these devices? (y/n): y

CS_CLI: Connecting to SSH leaf2
CS_CLI: Connected to SSH leaf2
CS_CLI: Generating config for leaf2 from /home/ekou/cumulus_test/device_vars/leaf2.yaml
CS_CLI: Saved generated config to /home/ekou/cumulus_test/configs/leaf2_gen_config.yaml
CS_CLI: Loaded intended config from /home/ekou/cumulus_test/device_vars/leaf2.yaml
CS_CLI: Device-side dry run for leaf2
CS_CLI: Uploaded candidate config to leaf2:/tmp/cs_cli_candidate.yaml
CS_CLI: Patched candidate revision on leaf2
CS_CLI: Diff for leaf2


nv set bridge domain br_default vlan 100 vni 10100
nv set bridge domain br_default vlan 121 vni 10121
nv set evpn enable on
nv set interface bond1 bond member swp2
nv set interface bond1 bond mlag enable on
nv set interface bond1 bond mlag id 100
nv set interface bond1 bridge domain br_default access 100
nv set interface bond1 type bond
nv set interface lo ip address 10.255.0.11/32
nv set interface lo type loopback
nv set interface peerlink bond member swp7
nv set interface peerlink type peerlink
nv set interface peerlink.4094 base-interface peerlink
nv set interface peerlink.4094 type sub
nv set interface peerlink.4094 vlan 4094
nv set interface swp1 ip address
nv set interface swp1-3,7 type swp
nv set interface swp3 bridge domain br_default access 121
nv set interface vlan100 ip address 192.168.100.2/24
nv set interface vlan100 ip vrr address 192.168.100.1/24
nv set interface vlan100 vlan 100
nv set interface vlan100,121 ip vrf TENANT1
nv set interface vlan100,121 ip vrr enable on
nv set interface vlan100,121 ip vrr state up
nv set interface vlan100,121 type svi
nv set interface vlan121 ip address 192.168.121.2/24
nv set interface vlan121 ip vrr address 192.168.121.1/24
nv set interface vlan121 vlan 121
nv set mlag backup 10.255.0.12
nv set mlag enable on
nv set mlag mac-address 44:38:39:be:ef:aa
nv set mlag peer-ip linklocal
nv set nve vxlan arp-nd-suppress on
nv set nve vxlan enable on
nv set nve vxlan flooding enable on
nv set nve vxlan flooding head-end-replication evpn
nv set nve vxlan mlag shared-address 10.255.0.10
nv set nve vxlan source address 10.255.0.11
nv set router bgp autonomous-system 65101
nv set router bgp enable on
nv set router bgp router-id 10.255.0.11
nv set router vrr enable on
nv set system global anycast-mac 44:38:39:ff:00:10
nv set system global fabric-mac 00:00:5e:00:01:01
nv set vrf TENANT1 evpn enable on
nv set vrf TENANT1 evpn vni 50000
nv set vrf TENANT1 router bgp address-family ipv4-unicast enable on
nv set vrf TENANT1 router bgp address-family ipv4-unicast redistribute connected enable on
nv set vrf TENANT1 router bgp address-family ipv4-unicast route-export to-evpn enable on
nv set vrf TENANT1 router bgp autonomous-system 65101
nv set vrf TENANT1 router bgp enable on
nv set vrf TENANT1 router bgp router-id 10.255.0.11
nv set vrf default router bgp address-family ipv4-unicast enable on
nv set vrf default router bgp address-family ipv4-unicast network 10.255.0.11/32
nv set vrf default router bgp address-family ipv4-unicast redistribute connected enable on
nv set vrf default router bgp address-family l2vpn-evpn enable on
nv set vrf default router bgp enable on
nv set vrf default router bgp neighbor peerlink.4094 address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor peerlink.4094 remote-as external
nv set vrf default router bgp neighbor peerlink.4094 type unnumbered
nv set vrf default router bgp neighbor swp1 address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor swp1 remote-as external
nv set vrf default router bgp neighbor swp1 type unnumbered



CS_CLI: Saved diff to /home/ekou/cumulus_test/diffs/leaf2_diff.txt
CS_CLI: Discarded candidate revision on leaf2, device is clean
CS_CLI: Disconnected from SSH leaf2
```

Reading the capture top to bottom:

- The password is prompted interactively (`getpass`, no echo) because `--diff` needs a connection and no `-p` / `DEVICE_PASSWORD` was given.
- The device is empty, so the "diff" is the **entire intended config** appearing as `nv set` lines — the full EVPN/MLAG leaf (VNIs, MLAG bond, peerlink, VRR SVIs, tenant VRF with L3VNI, BGP unnumbered) in 63 commands, and not a single `nv unset`.
- Lines like `nv set interface swp1-3,7 type swp` and `nv set interface vlan100,121 ip vrr enable on` are the giveaway that this text was produced **by NVUE itself**, not by the script: the device compacts identical config across consecutive interfaces into range syntax. A locally computed diff would never invent `swp1-3,7`.
- The last lines are the safety path from section 5.1 doing its job: the run had no `--commit`, so the `finally` block detaches the candidate — `Discarded candidate revision on leaf2, device is clean` — and the 63 pending commands evaporate without touching the device.

### 5.2 SONiC: build the candidate locally, diff it locally

SONiC's `config replace` expects a **complete** config_db as input — it replaces, not merges. So the script builds the effective candidate itself:

```text
candidate = deep_merge(running_config, rendered_set_tree)
prune UNSET paths from candidate
diff = unified_diff(sorted_yaml(running), sorted_yaml(candidate))
```

Three details make this diff trustworthy:

- **`deep_merge` returns a deep copy.** Nested dicts merge recursively, everything else is replaced — and the result never aliases the running config, so pruning the candidate can't accidentally mutate the "before" side and silently hide removals from the diff.
- **Type normalisation.** config_db stores numbers as strings (`vlanid: '100'`), while rendered YAML has real ints. `normalise_for_sonic_diff` converts every scalar on both sides to strings so the diff never reports `100` vs `'100'` as a change.
- **Stable ordering.** Both sides are dumped as *sorted* YAML before `difflib.unified_diff`, so key order can never produce a phantom diff.

The result reads like any unified diff:

```diff
--- sonic_running
+++ sonic_intended
@@ -1314,6 +1314,8 @@
 VLAN:
   Vlan100:
     vlanid: '100'
+  Vlan300:
+    vlanid: '300'
 VLAN_MEMBER:
   Vlan100|Ethernet0:
     tagging_mode: untagged
```

The local diff is instant but not device-validated — that's what `--dry_run` adds: upload the candidate and run `sudo config replace <file> --dry-run`, which walks the full YANG validation on the device and reports the changes as JSON patch operations without touching anything. Slow, but it's the device's own opinion.

## 6. Deleting config: the merge trap and the `UNSET` tree

Here is the trap every merge-based config system has: **removing a line from the device YAML does not remove it from the device.** `nv config patch` merges; the SONiC candidate is a deep-merge too. Deleting the `swp4` block from your YAML merely means "I stopped managing swp4" — whatever is configured there stays.

The script's answer is an explicit delete marker in the device YAML:

```yaml
INTERFACES:
  - NAME: swp4
    DELETE: true
MANAGEMENT:
  ADDITIONAL_IPS:
    - IP: 10.10.80.21/24
      GATEWAY: 10.10.80.2
      DELETE: true          # decommission this eth0 address
```

The templates split every list into live entries (`rejectattr('DELETE')`) and delete entries (`selectattr('DELETE')`). Live entries render normally; delete entries render into a special **`UNSET`** tree at the bottom of the generated config, where `null` means "remove the whole subtree at this key":

```yaml
UNSET:
  interface:
    eth0:
      ip:
        address:
          10.10.80.21/24: null
        gateway:
          10.10.80.2: null
```

The script then translates `UNSET` per vendor:

- **cumulus** — it becomes the `- unset:` section of the `nv config patch` (applied before `- set:`), so the deletion shows up in the device-side diff as real `nv unset` commands.
- **sonic** — the paths are pruned from the merged candidate before `config replace`. The pruning knows a SONiC-specific idiom: tables key related entries as `Name` and `Name|extra` (e.g. `INTERFACE` holds both `Ethernet0` and `Ethernet0|10.0.0.0/31`), so deleting a key also deletes all of its `Key|...` derived entries.

The SONiC template even uses `UNSET` for a rule it enforces on itself: an access port must not stay a router interface, so any live interface with `UNTAGGED_VLAN` gets its `INTERFACE` table entries (IP assignments) added to `UNSET` automatically — otherwise the candidate fails YANG validation with "Port is a router interface".

### 6.1 The full lifecycle on a live device: add an eth0 address, then delete it

Here is the round-trip on leaf1, end to end. First, **add** an additional management address by putting it in the device YAML (the `DELETE` marker commented out):

```yaml
MANAGEMENT:
  HOSTNAME: leaf1
  IP_MGMT_VIP: 192.168.0.21
  IP_DEFAULT_GATEWAY: 192.168.0.1
  MASK: 24
  API_PORT: 8765
  ADDITIONAL_IPS:
    - IP: 10.10.80.21/24
      #DELETE: true
      GATEWAY: 10.10.80.2
```

Then run `--diff --commit`:

```text
ekou@saltmaster:~/cumulus_test$ python3 cumulus_sonic_cli.py -d leaf1 --diff --commit
Enter device username: cumulus
Enter device password: 
CS_CLI: get_filtered_devices

CS_CLI: Found 1 device(s)

CS_CLI: ['leaf1']

CS_CLI: Process these devices? (y/n): y

CS_CLI: Connecting to SSH leaf1
CS_CLI: Connected to SSH leaf1
CS_CLI: Generating config for leaf1 from /home/ekou/cumulus_test/device_vars/leaf1.yaml
CS_CLI: Saved generated config to /home/ekou/cumulus_test/configs/leaf1_gen_config.yaml
CS_CLI: Loaded intended config from /home/ekou/cumulus_test/device_vars/leaf1.yaml
CS_CLI: Device-side dry run for leaf1
CS_CLI: Uploaded candidate config to leaf1:/tmp/cs_cli_candidate.yaml
CS_CLI: Patched candidate revision on leaf1
CS_CLI: Diff for leaf1


nv set interface eth0 ip address 10.10.80.21/24
nv set interface eth0 ip gateway 10.10.80.2



CS_CLI: Saved diff to /home/ekou/cumulus_test/diffs/leaf1_diff.txt

CS_CLI: WARNING: this change modifies the MANAGEMENT interface on leaf1!
CS_CLI: If the new address is wrong you will LOSE ACCESS to the device.

Running for leaf1... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   0% -:--:--
CS_CLI: Commit the management interface change anyway? leaf1 (y/n): y
Running for leaf1... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   0% -:--:--
CS_CLI: Do you want to apply this config? leaf1 (y/n): y
CS_CLI: Applying candidate config on leaf1
CS_CLI: Applied config on leaf1 verifying[1000D[Jreadying[1000D[Jreloading[1000D[Japplied 
Running for leaf1... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   0% -:--:--
CS_CLI: Save config to startup (nv config save)? leaf1 (y/n): y
CS_CLI: Saved config to startup on leaf1
CS_CLI: Commit success leaf1: True
CS_CLI: Disconnected from SSH leaf1
Running for leaf1... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```

The device only knows what changed — two `nv set` lines — and because both touch `eth0`, the management tripwire from section 7 fires **before** the normal apply prompt: the red warning plus an extra explicit confirmation. The `Applied config` line echoes NVUE's own apply progress stream (`verifying → readying → reloading → applied` — the `[1000D[J` fragments are the device's cursor-control animation captured raw).

Now the **delete**: nothing is removed from the YAML — the same entry just gets its `DELETE: true` uncommented:

```yaml
  ADDITIONAL_IPS:
    - IP: 10.10.80.21/24
      DELETE: true
      GATEWAY: 10.10.80.2
```

Same command, opposite diff:

```text
ekou@saltmaster:~/cumulus_test$ python3 cumulus_sonic_cli.py -d leaf1 --diff --commit -u cumulus
Enter device password: 
CS_CLI: get_filtered_devices

CS_CLI: Found 1 device(s)

CS_CLI: ['leaf1']

CS_CLI: Process these devices? (y/n): y

CS_CLI: Connecting to SSH leaf1
CS_CLI: Connected to SSH leaf1
CS_CLI: Generating config for leaf1 from /home/ekou/cumulus_test/device_vars/leaf1.yaml
CS_CLI: Saved generated config to /home/ekou/cumulus_test/configs/leaf1_gen_config.yaml
CS_CLI: Loaded intended config from /home/ekou/cumulus_test/device_vars/leaf1.yaml
CS_CLI: Device-side dry run for leaf1
CS_CLI: Patched unset section into candidate revision on leaf1
CS_CLI: Uploaded candidate config to leaf1:/tmp/cs_cli_candidate.yaml
CS_CLI: Patched candidate revision on leaf1
CS_CLI: Diff for leaf1


nv unset interface eth0 ip address 10.10.80.21/24
nv unset interface eth0 ip gateway 10.10.80.2



CS_CLI: Saved diff to /home/ekou/cumulus_test/diffs/leaf1_diff.txt

CS_CLI: WARNING: this change modifies the MANAGEMENT interface on leaf1!
CS_CLI: If the new address is wrong you will LOSE ACCESS to the device.

Running for leaf1... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   0% -:--:--
CS_CLI: Commit the management interface change anyway? leaf1 (y/n): y
Running for leaf1... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   0% -:--:--
CS_CLI: Do you want to apply this config? leaf1 (y/n): y
CS_CLI: Applying candidate config on leaf1
CS_CLI: Applied config on leaf1 verifying[1000D[Jverified[1000D[Jreadying[1000D[Jready[1000D[Jreloading[1000D[Japplied 
Running for leaf1... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   0% -:--:--
CS_CLI: Save config to startup (nv config save)? leaf1 (y/n): y
CS_CLI: Saved config to startup on leaf1
CS_CLI: Commit success leaf1: True
CS_CLI: Disconnected from SSH leaf1
Running for leaf1... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```

Two details in the delete run are worth pausing on:

- `CS_CLI: Patched unset section into candidate revision on leaf1` — a log line the add run didn't have. This is the two-file patch quirk from section 5.1 happening for real: the `UNSET` tree goes to the device as a separate `- unset:` patch file **before** the `- set:` file, because NVUE can't take the same path in both within one file. Both land in the same candidate revision, so the apply is still atomic.
- The management tripwire fires just the same on a **removal** — deleting an eth0 address is every bit as capable of locking you out as adding a wrong one, and the `eth0`-in-diff check doesn't care about the direction of the change.

Together the two runs are the section 6 story in miniature: commenting the entry out of the YAML would have changed *nothing* on the device — only the explicit `DELETE: true` produces the `nv unset` lines.

## 7. Commit: guardrails everywhere

`--commit` only ever runs after `--diff`, only when the diff is non-empty, and every destructive step is confirmed interactively.

### 7.1 The management-interface tripwire

Before any commit prompt, the script checks whether the change touches the management path — the one change that can lock you out:

- **cumulus**: any diff line mentioning `eth0`
- **sonic**: compare the `MGMT_INTERFACE` / `MGMT_PORT` / `MGMT_VRF_CONFIG` tables between running and candidate

If it does, you get a red warning and an extra explicit confirmation before the normal commit prompt:

```text
CS_CLI: WARNING: this change modifies the MANAGEMENT interface on leaf1!
CS_CLI: If the new address is wrong you will LOSE ACCESS to the device.
```

### 7.2 Cumulus commit

The candidate is already patched and validated on the device from the `--diff` step, so commit is short:

- **y** → `nv config apply --assume-yes`, then an optional `nv config save` to persist to startup
- **n** → `nv config detach`, the candidate is discarded, device untouched
- **apply fails** (device rejects the config) → the script detaches the candidate itself, so nothing half-finished stays behind

### 7.3 SONiC commit — checkpoint, replace, rollback

SONiC has no candidate revision, but it has **checkpoints**, and `config replace` (Generic Config Updater) applies only the changed sections instead of restarting all services like `config reload` would:

1. SFTP the full merged candidate to `/tmp/cs_cli_config_db.json`
2. confirm → `sudo config checkpoint cs_cli_backup` (a rollback point, old one deleted first)
3. `sudo config replace /tmp/cs_cli_config_db.json` — validates first, then applies with minimum disruption; a failure here means the device rejected the candidate and the config is unchanged
4. prompt **keep / rollback** — answering "n" runs `sudo config rollback cs_cli_backup` and you're back where you started
5. prompt to persist → `sudo config save -y`

So both vendors end up with the same shape of safety story — validate before touching anything, apply atomically / minimally, and keep an escape hatch (`detach` / `rollback`) armed until the human says "keep it".

## 8. Day-0 bootstrap

A fresh switch isn't reachable by the script yet — no management IP, and on Cumulus the REST API is off by default. `--generate_bootstrap` renders `j2_template/<vendor>_bootstrap.j2` into the commands you paste into the console session, offline:

- **cumulus**: hostname + eth0 address/gateway via `nv set`, then enable the NVUE REST API — symlink the shipped nginx site (`/etc/nginx/sites-available/nvue.conf` → `sites-enabled`, the symlink *is* the on/off switch), `sed` the listener from `localhost:8765` to all addresses, `nginx -t`, enable and restart nginx, and a `curl -k -u user:pass https://<mgmt-ip>:8765/nvue_v1/system?rev=applied` to verify from the management station.
- **sonic**: `sudo config hostname`, `sudo config interface ip add eth0 <ip> <gw>`, `sudo config save -y`.

After that, the same YAML that generated the bootstrap is the one the main loop manages the device with.

The full capture for leaf2:

{{< details "leaf2 --generate_bootstrap — full run output" >}}
```text
ekou@saltmaster:~/cumulus_test$ python3 cumulus_sonic_cli.py -d leaf2 -u cumulus --generate_bootstrap
CS_CLI: get_filtered_devices

CS_CLI: Found 1 device(s)

CS_CLI: ['leaf2']

CS_CLI: Process these devices? (y/n): y

CS_CLI: Generating bootstrap config for leaf2
CS_CLI: Saved bootstrap config to /home/ekou/cumulus_test/configs/leaf2_bootstrap_config.txt
Running for leaf2... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   0% -:--:--# ============================================================
# Bootstrap leaf2 (Cumulus Linux)
# ============================================================

# ---- 1. management interface ----
sudo nv set system hostname leaf2
sudo nv set interface eth0 ip address 192.168.0.103/24
sudo nv set interface eth0 ip gateway 192.168.0.1
sudo nv config diff
sudo nv config apply --assume-yes
sudo nv config save

# ---- 2. enable the NVUE REST API (nginx, port 8765) ----
# enable the shipped nginx site (this symlink is the on/off switch):
sudo ln -sf /etc/nginx/sites-available/nvue.conf /etc/nginx/sites-enabled/nvue.conf

# listen on all addresses instead of localhost only:
sudo sed -i 's/listen localhost:8765 ssl;/listen [::]:8765 ipv6only=off ssl;/g' /etc/nginx/sites-available/nvue.conf

# validate, enable, restart, confirm the listener:
sudo nginx -t
sudo systemctl enable --now nginx
sudo systemctl restart nginx
sudo ss -lntp | grep 8765

# ---- 3. verify from the management station ----
# curl -k -u <user>:<password> https://192.168.0.103:8765/nvue_v1/system?rev=applied
```
{{< /details >}}

Two things to notice in this capture, compared with the `--diff` run in section 5.1.1:

- **No password prompt**, even though `-u cumulus` was passed: `--generate_bootstrap` is in the "no connection needed" group, so the credential prompting is skipped entirely — you can render bootstrap commands for a device that doesn't exist on the network yet.
- The Rich progress bar (`Running for leaf2... ━━━ 0%`) is painted on the same line the rendered output starts on — a cosmetic artifact of the progress display and the `Syntax`-highlighted output sharing the terminal. The file saved to `configs/leaf2_bootstrap_config.txt` is clean.

## 9. Async plumbing notes

A few implementation details that keep the parallel mode (`--parallel`, max 10 concurrent via an `asyncio.Semaphore`) sane:

- **Everything is asyncio-native where possible**: `asyncssh` for SSH/SFTP, `aiofiles` for file I/O. The two blocking libraries — `requests` for REST and Rich's `Prompt.ask` — are pushed into worker threads with `asyncio.to_thread`, so one device waiting on an HTTP response never stalls the others.
- **One global `prompt_lock`** serialises interactive prompts, so ten parallel commit tasks can never paint two "apply this config? (y/n)" questions over each other. The Rich progress display is stopped around each prompt and restarted after, for the same reason.
- **Credentials** come from `DEVICE_USERNAME` / `DEVICE_PASSWORD` env vars by preference (keeps the password out of shell history and process lists), falling back to `-u`/`-p` or an interactive prompt — and the password prompt uses `getpass`, so it never echoes.
- **Error hygiene**: `str(asyncio.TimeoutError())` is an empty string, so connection errors fall back to the exception type name; SSH command failures raise with the last 500 chars of stderr so the actual device error surfaces in the log.

## 10. Full source

The complete script — a single file with no private library dependencies; everything it imports is pip-installable (`requirements.txt`: PyYAML, Jinja2, requests, rich, aiofiles, asyncssh).

{{< details "cumulus_sonic_cli.py — full source (~1250 lines)" >}}
```python
"""
CUMULUS_SONIC_CLI

Independent CLI script for Cumulus Linux (NVUE) and SONiC switches.
Follows the structure and logic of n2c_cli.py, but standalone (no n2clib).

Capabilities:
    --get_config       export & display the full running config in its native
                       JSON format (both vendors return the config as JSON)
    --to_yaml          export & display the full running config as YAML
    --generate_config  render device_vars/<hostname>.yaml through the vendor
                       j2 template into the vendor native intended config
    --generate_bootstrap
                       generate the day-0 bootstrap commands for a fresh
                       device (management interface + enable the REST API),
                       rendered from j2_template/<vendor>_bootstrap.j2
    --diff             diff the running config against the intended config
    --dry_run          sonic only: with --diff, also let the device validate
                       the candidate (config replace --dry-run)
    --commit           after --diff, push / apply the intended config (with confirm)

Device yaml files (same logic as n2c_cli):
    device_vars/<hostname>.yaml is the single source of truth per device.
    It holds VENDOR, ROLE and MANAGEMENT (HOSTNAME, CONNECT_ADDRESS,
    API_PORT / PORT) plus the unified vendor neutral config variables,
    which are rendered through j2_template/<vendor>.j2 into the vendor
    native config. The device list is built by matching -d / -f against
    these files, no separate inventory file is needed.
    Fallback: intended/<hostname>.yaml with a raw vendor native config tree
    (used when no j2 template exists for the vendor).

Transport:
    cumulus  -> NVUE REST API (https, default port 8765) for config export,
                SSH for the device-side dry run (--diff) and apply (--commit):
                nv config patch -> nv config diff -o commands -> nv config apply / detach
                (no netconf support on Cumulus)
    sonic    -> SSH (show runningconfiguration all / config replace with
                checkpoint + rollback, applies with minimum disruption)

Outputs are written to configs/ and diffs/ (created automatically).

Credentials: DEVICE_USERNAME / DEVICE_PASSWORD environment variables
(preferred, keeps the password out of shell history and process lists),
or -u / -p, or interactive prompt.

Usage examples:
    python cumulus_sonic_cli.py -d leaf1 -u cumulus --get_config
    python cumulus_sonic_cli.py -d ".*" --get_config --to_yaml
    python cumulus_sonic_cli.py -d leaf1 --generate_config
    python cumulus_sonic_cli.py -f device_list.txt --diff
    python cumulus_sonic_cli.py -d leaf1 --diff --commit
"""

import traceback
import argparse
from argparse import RawTextHelpFormatter
import copy
import json
import getpass
import re
import difflib
import os
import errno
import asyncio

import yaml
import requests
import urllib3
import rich
from rich.progress import Progress
from rich.prompt import Prompt
from rich.console import Console
from rich.syntax import Syntax
from jinja2 import Environment, FileSystemLoader
import aiofiles
import asyncssh

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

console = Console()

# serialise interactive prompts so parallel device tasks never prompt at once
prompt_lock = asyncio.Lock()

SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
INTENDED_DIR = os.path.join(SCRIPT_DIR, "intended")
DEVICE_VARS_DIR = os.path.join(SCRIPT_DIR, "device_vars")
J2_TEMPLATE_DIR = os.path.join(SCRIPT_DIR, "j2_template")
CONFIGS_DIR = os.path.join(SCRIPT_DIR, "configs")
DIFFS_DIR = os.path.join(SCRIPT_DIR, "diffs")

SUPPORTED_VENDORS = ["cumulus", "sonic"]

# sonic fetches the full running config as JSON with this cli command,
# cumulus uses the NVUE REST API instead of a cli command
SONIC_RUNNING_CONFIG_COMMAND = "show runningconfiguration all"

NVUE_API_DEFAULT_PORT = 8765


class ConsoleLogger:
    """
    Same console logger pattern as n2c_cli.py ConsoleLogger,
    without the N2cLogger base class dependency.
    """

    def __init__(self, args):
        self.args = args

    async def debug(self, message: str):
        if self.args.silent:
            return
        elif self.args.no_color and self.args.debug:
            print(f"{message}")
        elif self.args.debug:
            rich.print(f"[light_cyan1]{message}")

    async def warn(self, message: str):
        if self.args.silent:
            return
        elif self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[dark_orange]{message}")

    async def error(self, message: str):
        if self.args.silent:
            return
        elif self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[bright_red]{message}")

    async def info(self, message: str):
        if self.args.silent:
            if 'connected' in message.lower():
                rich.print(f"[grey3]{message}")
            else:
                return
        elif self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[yellow3]{message}")

    async def console(self, message: str):
        if self.args.silent:
            return
        elif self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[cyan]{message}")

    async def green_diff(self, message: str):
        if self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[bright_green]{message}")

    async def red_diff(self, message: str):
        if self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[bright_red]{message}")

    async def grey_diff(self, message: str):
        if self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[grey54]{message}")

    async def change_diff(self, message: str):
        if self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[dark_orange]{message}")

    async def white(self, message: str):
        if self.args.no_color:
            print(f"{message}")
        else:
            rich.print(f"[white]{message}")


async def prompt(message, args):
    """
    Prompts run in a worker thread so they do not block the event loop
    (other device tasks keep running), and are serialised through a lock
    so parallel runs never show two prompts at the same time.
    """
    async with prompt_lock:
        if args.no_color:
            return await asyncio.to_thread(Prompt.ask, message)
        else:
            return await asyncio.to_thread(Prompt.ask, f"[dark_orange]{message}")


async def prepare_output_file(log_file):
    """
    PREPARE AN OUTPUT FILE PATH FOR WRITING
    CREATES THE PARENT FOLDER IF MISSING, REMOVES A LEFTOVER OLD FILE
    """

    try:
        log_dir = os.path.dirname(log_file)
        if log_dir != "" and not os.path.exists(log_dir):
            try:
                os.makedirs(log_dir)
            except OSError as exc:
                if exc.errno != errno.EEXIST:
                    raise
        elif os.path.exists(log_file):
            os.remove(log_file)
        else:
            pass
    except Exception as error:
        pass


async def process_args():
    """
    This function processes the arguments passed to the script from the cli
    """

    parser = argparse.ArgumentParser(
        description="Please see the arguments and usage examples below:",
        formatter_class=RawTextHelpFormatter,
    )
    parser.add_argument("-d", type=str, default=False, required=False,
                        help="Device hostname or regex to match against the inventory")
    parser.add_argument("-f", type=str, default=False, required=False,
                        help="File containing a list of device hostnames / regex, one per line")

    parser.add_argument("-u", type=str, required=False, default=False,
                        help="Device username (or set DEVICE_USERNAME env var)")
    parser.add_argument("-p", type=str, required=False, default=False,
                        help="Device password (prefer DEVICE_PASSWORD env var or the interactive\n"
                             "prompt, -p is visible in shell history and process lists)")

    parser.add_argument("--get_config", default=False, action="store_true",
                        help="Export & display the full running config in native JSON format")
    parser.add_argument("--to_yaml", default=False, action="store_true",
                        help="Export & display the full running config as YAML")
    parser.add_argument("--generate_config", default=False, action="store_true",
                        help="Render device_vars/<hostname>.yaml through j2_template/<vendor>.j2\n"
                             "and save the vendor native config (no device connection needed)")
    parser.add_argument("--generate_bootstrap", default=False, action="store_true",
                        help="Generate the day-0 bootstrap commands (management interface +\n"
                             "enable REST API) from j2_template/<vendor>_bootstrap.j2\n"
                             "(no device connection needed)")

    parser.add_argument("--diff", default=False, action="store_true",
                        help="Diff the running config against the generated intended config")
    parser.add_argument("--dry_run", default=False, action="store_true",
                        help="SONiC + --diff: additionally run the device-side dry run\n"
                             "(sudo config replace --dry-run, slow but device validated).\n"
                             "The cumulus --diff is already a device-side dry run")
    parser.add_argument("--commit", default=False, action="store_true",
                        help="Apply the intended config after --diff (with confirmation)")

    parser.add_argument("--parallel", default=False, action="store_true",
                        help="Process devices in parallel (max 10 concurrent)")
    parser.add_argument("--no_color", default=False, action="store_true")
    parser.add_argument("--debug", default=False, action="store_true",
                        help="Print debug messages")
    parser.add_argument("--silent", default=False, action="store_true",
                        help="Do not print any messages")

    args = parser.parse_args()
    return args


##############################################################################################################################
# conversion helpers
##############################################################################################################################

ANSI_ESCAPE_RE = re.compile(r'\x1b\[[0-9;]*m')


##############################################################################################################################
# Device inventory
##############################################################################################################################

async def add_devices_to_list(yaml_file, device_names, filtered_devices, args):
    """
    Load one device yaml file (n2c_cli device yaml format) and add it to the
    device list. VENDOR, ROLE and MANAGEMENT come from the file itself, so no
    separate inventory file or -vendor parameter is needed.
    """
    logger = ConsoleLogger(args)

    async with aiofiles.open(yaml_file, "r") as device_vars:
        try:
            device_vars = await device_vars.read()
            device_vars = yaml.safe_load(device_vars)

            vendor = str(device_vars.get("VENDOR", "")).lower()
            if vendor not in SUPPORTED_VENDORS:
                await logger.debug(
                    f"CS_CLI: Skipping {yaml_file}, unsupported vendor '{vendor}' "
                    f"(supported: {SUPPORTED_VENDORS})"
                )
                return

            device_dict = {}
            device_dict["hostname"] = device_vars["MANAGEMENT"]["HOSTNAME"].lower()
            device_dict["vendor"] = vendor
            device_dict["role"] = device_vars.get("ROLE", "UNKNOWN")
            device_dict["group"] = device_vars.get("GROUP", "UNKNOWN")

            # connection address, same order of preference as n2c_cli
            device_dict["inband"] = False
            if "CONNECT_ADDRESS" in device_vars["MANAGEMENT"]:
                device_dict["ip"] = device_vars["MANAGEMENT"]["CONNECT_ADDRESS"]
            elif device_vars["MANAGEMENT"].get("INBAND_MGMT") is True or device_vars["MANAGEMENT"].get("INBAND") is True:
                device_dict["ip"] = device_vars["LOOPBACK"]["LOOPBACK_IP"]
                device_dict["inband"] = True
            else:
                device_dict["ip"] = device_vars["MANAGEMENT"]["IP_MGMT_VIP"]

            device_dict["port"] = device_vars["MANAGEMENT"].get("PORT", 22)
            device_dict["api_port"] = device_vars["MANAGEMENT"].get("API_PORT", NVUE_API_DEFAULT_PORT)
            device_dict["yaml_file"] = yaml_file

            filtered_devices.append(device_dict)
            device_names.append(device_dict["hostname"])

        except Exception as error:
            traceback.print_exc()
            pass


async def get_filtered_devices(args):
    """
    INPUT IS THE REGEX TO FILTER THE DEVICE YML FILES IN device_vars/
    CREATES A LIST OF BASIC INFORMATION FOR EACH MATCHING DEVICE
    SUCH AS HOSTNAME, IP, VENDOR, ROLE, ETC
    RETURNS THIS LIST OF BASIC INFORMATION TO THE CLIENT
    """
    device_names = []
    filtered_devices = []
    logger = ConsoleLogger(args)

    if not os.path.isdir(DEVICE_VARS_DIR):
        await logger.error(f"CS_CLI: Device vars folder not found: {DEVICE_VARS_DIR}")
        return []

    dev_list = []
    if args.d != False:
        dev_list.append(args.d)
    elif args.f:
        if os.path.isfile(args.f):
            with open(args.f, "r") as ifile:
                for line in ifile:
                    line = line.strip()
                    if line == "":
                        continue
                    dev_list.append(line.lower())
        else:
            await logger.warn("CS_CLI: Please provide the valid file")

    for dev_name in dev_list:
        for name in sorted(os.listdir(DEVICE_VARS_DIR)):
            if name.endswith(".yaml"):
                file_name = name[:-len(".yaml")]
            elif name.endswith(".yml"):
                file_name = name[:-len(".yml")]
            else:
                continue
            if re.match(dev_name, file_name) and file_name not in device_names:
                await add_devices_to_list(
                    os.path.join(DEVICE_VARS_DIR, name), device_names, filtered_devices, args
                )

    await logger.info("CS_CLI: get_filtered_devices")

    return filtered_devices


##############################################################################################################################
# SSH device wrapper (asyncssh based, Cumulus and SONiC are both Linux)
##############################################################################################################################

class SSHDevice:

    def __init__(self, filtered_device, username, password, logger):
        self.filtered_device = filtered_device
        self.username = username
        self.password = password
        self.logger = logger
        self.conn = None

    async def connect(self):
        try:
            self.conn = await asyncio.wait_for(
                asyncssh.connect(
                    self.filtered_device["ip"],
                    port=self.filtered_device.get("port", 22),
                    username=self.username,
                    password=self.password,
                    known_hosts=None,
                ),
                timeout=60,
            )
            return True
        except Exception as e:
            # str() of e.g. asyncio.TimeoutError is empty, fall back to the type name
            return {"error": str(e) or type(e).__name__}

    async def send_command(self, command):
        result = await self.conn.run(command)
        if result.exit_status != 0:
            output = (result.stderr or result.stdout or "").strip()
            raise RuntimeError(
                f"command '{command}' failed (rc={result.exit_status}): {output[-500:]}"
            )
        return result.stdout

    async def transfer_file(self, content, remote_path):
        async with self.conn.start_sftp_client() as sftp:
            async with sftp.open(remote_path, "w") as remote_file:
                await remote_file.write(content)
        return True

    async def disconnect(self):
        if self.conn:
            self.conn.close()
            await self.conn.wait_closed()
        return True


##############################################################################################################################
# NVUE REST API device wrapper (Cumulus Linux 5.x, no netconf support)
##############################################################################################################################

class NvueRestDevice:

    def __init__(self, filtered_device, username, password, logger):
        self.filtered_device = filtered_device
        self.logger = logger
        port = filtered_device.get("api_port", NVUE_API_DEFAULT_PORT)
        self.base_url = f"https://{filtered_device['ip']}:{port}/nvue_v1"
        # one session reuses the TLS connection across all calls
        self.session = requests.Session()
        self.session.auth = (username, password)
        self.session.verify = False

    async def _request(self, method, path, payload=None):
        url = f"{self.base_url}{path}"

        def do_request():
            return self.session.request(method, url, json=payload, timeout=30)

        response = await asyncio.to_thread(do_request)
        if response.status_code not in (200, 201, 204):
            raise RuntimeError(f"{method} {path} failed ({response.status_code}): {response.text[:500]}")
        if not response.content:
            return {}
        try:
            return response.json()
        except ValueError:
            return {"text": response.text}

    async def connect(self):
        """
        There is no session with the REST API, so 'connect' just verifies
        reachability and credentials with a small GET request.
        """
        try:
            await self._request("GET", "/system?rev=applied")
            return True
        except Exception as e:
            return {"error": str(e) or type(e).__name__}

    async def get_config(self, rev="applied"):
        """
        Fetch the full running (applied) config tree as a dict.
        """
        config = await self._request("GET", f"/?rev={rev}")
        config.pop("header", None)
        return config

    async def disconnect(self):
        await asyncio.to_thread(self.session.close)
        return True


##############################################################################################################################
# Running config / diff / commit logic
##############################################################################################################################

async def get_running_config_dict(device, filtered_device, logger):
    """
    Fetch the full running config from the device as a python dict.
    cumulus -> NVUE REST API GET /nvue_v1/?rev=applied
    sonic   -> ssh: show runningconfiguration all
    """
    vendor = filtered_device["vendor"]
    if vendor == "cumulus":
        await logger.info(f"CS_CLI: Getting running config from {filtered_device['hostname']} (REST /nvue_v1/?rev=applied)")
        return await device.get_config("applied")
    else:
        command = SONIC_RUNNING_CONFIG_COMMAND
        await logger.info(f"CS_CLI: Getting running config from {filtered_device['hostname']} ({command})")
        output = await device.send_command(command)
        config = json.loads(output)
        # 'show runningconfiguration all' appends the raw FRR config as a
        # display-only 'bgpraw' key, it is not a config_db table and must not
        # be written back to the device on commit
        config.pop("bgpraw", None)
        return config


def find_yaml_file(directory, hostname):
    for extension in (".yaml", ".yml"):
        candidate = os.path.join(directory, f"{hostname}{extension}")
        if os.path.isfile(candidate):
            return candidate
    return None


async def render_device_config(filtered_device, device_vars_file, logger):
    """
    Render the unified device vars through the vendor j2 template to get the
    vendor native config, same logic as n2c_cli:
        device_vars/<hostname>.yaml + j2_template/<vendor>.j2 -> config dict
    """
    vendor = filtered_device["vendor"]
    j2_file = os.path.join(J2_TEMPLATE_DIR, f"{vendor}.j2")
    if not os.path.isfile(j2_file):
        await logger.error(f"CS_CLI: Template not found: {j2_file}")
        return None

    async with aiofiles.open(device_vars_file, "r") as f:
        device_vars = yaml.safe_load(await f.read())
    await logger.debug(f"CS_CLI: Loaded device vars for {filtered_device['hostname']}")

    async with aiofiles.open(j2_file, "r") as f:
        j2_content = await f.read()

    env = Environment(loader=FileSystemLoader(J2_TEMPLATE_DIR), trim_blocks=True, lstrip_blocks=True)
    template = env.from_string(j2_content)
    rendered = template.render(device_vars=device_vars)
    await logger.debug(f"CS_CLI: Rendered j2 template {vendor}.j2 for {filtered_device['hostname']}")

    config_dict = yaml.safe_load(rendered)

    # save the generated vendor native config, like n2c_cli *_gen_config files
    path = os.path.join(CONFIGS_DIR, f"{filtered_device['hostname']}_gen_config.yaml")
    await prepare_output_file(path)
    async with aiofiles.open(path, "w") as f:
        await f.write(yaml.safe_dump(config_dict, default_flow_style=False, sort_keys=False))
    await logger.info(f"CS_CLI: Saved generated config to {path}")

    return config_dict


async def render_bootstrap_config(filtered_device, logger):
    """
    Render the day-0 bootstrap commands for a fresh device from
    j2_template/<vendor>_bootstrap.j2 (management interface + REST API).
    Returns the rendered command text, or None if the template is missing.
    """
    vendor = filtered_device["vendor"]
    hostname = filtered_device["hostname"]
    j2_file = os.path.join(J2_TEMPLATE_DIR, f"{vendor}_bootstrap.j2")
    if not os.path.isfile(j2_file):
        await logger.error(f"CS_CLI: Bootstrap template not found: {j2_file}")
        return None

    async with aiofiles.open(filtered_device["yaml_file"], "r") as f:
        device_vars = yaml.safe_load(await f.read())

    async with aiofiles.open(j2_file, "r") as f:
        j2_content = await f.read()

    env = Environment(loader=FileSystemLoader(J2_TEMPLATE_DIR), trim_blocks=True, lstrip_blocks=True)
    bootstrap_text = env.from_string(j2_content).render(device_vars=device_vars)
    await logger.debug(f"CS_CLI: Rendered bootstrap template {vendor}_bootstrap.j2 for {hostname}")

    path = os.path.join(CONFIGS_DIR, f"{hostname}_bootstrap_config.txt")
    await prepare_output_file(path)
    async with aiofiles.open(path, "w") as f:
        await f.write(bootstrap_text)
    await logger.info(f"CS_CLI: Saved bootstrap config to {path}")

    return bootstrap_text


async def load_intended_config(filtered_device, logger):
    """
    Load the intended config for the device:
        1. the device yaml file rendered through j2_template/<vendor>.j2
        2. fallback: raw vendor native config from intended/<hostname>.yaml
           (used when no j2 template exists for the vendor)
    """
    hostname = filtered_device["hostname"]
    vendor = filtered_device["vendor"]

    j2_file = os.path.join(J2_TEMPLATE_DIR, f"{vendor}.j2")
    if os.path.isfile(j2_file):
        await logger.info(f"CS_CLI: Generating config for {hostname} from {filtered_device['yaml_file']}")
        config_dict = await render_device_config(filtered_device, filtered_device["yaml_file"], logger)
        return config_dict, filtered_device["yaml_file"]

    intended_file = find_yaml_file(INTENDED_DIR, hostname)
    if intended_file:
        await logger.warn(f"CS_CLI: No template {j2_file}, using raw config {intended_file}")
        async with aiofiles.open(intended_file, "r") as f:
            content = await f.read()
        return yaml.safe_load(content), intended_file

    await logger.warn(
        f"CS_CLI: No config source found for {hostname} "
        f"(expected template {j2_file} "
        f"or {os.path.join(INTENDED_DIR, hostname + '.yaml')})"
    )
    return None, None


def normalise_for_sonic_diff(data):
    """
    Convert all scalars to strings so the diff does not report type only
    differences (SONiC config_db stores numbers as strings, while rendered
    yaml has real ints, e.g. "100" vs 100).
    """
    if isinstance(data, dict):
        return {str(key): normalise_for_sonic_diff(value) for key, value in data.items()}
    if isinstance(data, list):
        return [normalise_for_sonic_diff(item) for item in data]
    if isinstance(data, bool):
        return str(data).lower()
    if data is None:
        return None
    return str(data)


def deep_merge(base, override):
    """
    Recursively merge override into a deep copy of base. Nested dicts are
    merged, any other value in override replaces the value in base. The
    result never aliases base, so pruning the result cannot mutate base
    (which would silently hide removals from the diff).
    """
    merged = copy.deepcopy(base)
    for key, value in override.items():
        if key in merged and isinstance(merged[key], dict) and isinstance(value, dict):
            merged[key] = deep_merge(merged[key], value)
        else:
            merged[key] = value
    return merged


def prune_sonic_config(config, unset_tree):
    """
    Remove the UNSET paths from a config dict (in place). A null value in the
    unset tree deletes the whole subtree at that key, a nested dict recurses.
    SONiC tables key related entries as 'Name' and 'Name|extra' (for example
    INTERFACE has 'Ethernet0' and 'Ethernet0|10.0.0.0/31'), deleting a key
    also deletes those derived entries.
    """
    for key, value in unset_tree.items():
        if isinstance(value, dict) and value and isinstance(config.get(key), dict):
            prune_sonic_config(config[key], value)
        else:
            for existing_key in list(config):
                if existing_key == key or existing_key.startswith(f"{key}|"):
                    del config[existing_key]


def build_sonic_diff(running_dict, intended_dict, hostname):
    """
    Build a unified diff between the SONiC running config and the candidate
    config (the running config deep-merged with the intended set config,
    minus the UNSET paths - both sides are complete configs). The cumulus
    diff is device-side (nv config diff) and does not come through here.
    Both sides are normalised to sorted YAML so the diff is stable.
    """
    running_dict = normalise_for_sonic_diff(running_dict)
    intended_dict = normalise_for_sonic_diff(intended_dict)
    running_yaml = yaml.safe_dump(running_dict, default_flow_style=False, sort_keys=True)
    intended_yaml = yaml.safe_dump(intended_dict, default_flow_style=False, sort_keys=True)
    diff_lines = difflib.unified_diff(
        running_yaml.splitlines(),
        intended_yaml.splitlines(),
        fromfile=f"{hostname}_running",
        tofile=f"{hostname}_intended",
        lineterm="",
    )
    return "\n".join(diff_lines)


async def display_sonic_diff(diff, logger):
    for line in diff.splitlines():
        if line.startswith("+"):
            line = line.replace("[", "\\[")
            await logger.green_diff(f"{line}")
        elif line.startswith("-"):
            line = line.replace("[", "\\[")
            await logger.red_diff(f"{line}")
        elif line.startswith("@@"):
            line = line.replace("[", "\\[")
            await logger.change_diff(f"{line}")
        else:
            line = line.replace("[", "\\[")
            await logger.grey_diff(f"{line}")


async def cumulus_candidate_diff(ssh_device, intended_dict, hostname, logger):
    """
    Device-side dry run: upload the rendered config, merge it into the NVUE
    candidate revision with `nv config patch`, and return the human readable
    `nv config diff -o commands` output. Nothing is applied here, the caller must either
    apply the candidate or discard it with `nv config detach`.

    The special UNSET key in the rendered config (entries marked DELETE: true
    in the device yaml, plus template-owned subtrees like the eth0 addresses)
    is patched as a separate `- unset:` file BEFORE the `- set:` file: the
    NVUE patch converter cannot handle the same path in unset and set within
    one file. Both patches land in the same candidate revision, so the final
    apply is still one atomic change.
    """
    set_tree = {key: value for key, value in intended_dict.items() if key != "UNSET"}
    unset_tree = intended_dict.get("UNSET")

    # start from a clean candidate, discard any leftover from earlier runs
    await ssh_device.send_command("nv config detach")

    if unset_tree:
        remote_unset_path = "/tmp/cs_cli_unset.yaml"
        unset_yaml = yaml.safe_dump([{"unset": unset_tree}], default_flow_style=False, sort_keys=False)
        await ssh_device.transfer_file(unset_yaml, remote_unset_path)
        await ssh_device.send_command(f"nv config patch {remote_unset_path}")
        await logger.info(f"CS_CLI: Patched unset section into candidate revision on {hostname}")

    remote_path = "/tmp/cs_cli_candidate.yaml"
    candidate_yaml = yaml.safe_dump([{"set": set_tree}], default_flow_style=False, sort_keys=False)
    await ssh_device.transfer_file(candidate_yaml, remote_path)
    await logger.info(f"CS_CLI: Uploaded candidate config to {hostname}:{remote_path}")

    await ssh_device.send_command(f"nv config patch {remote_path}")
    await logger.info(f"CS_CLI: Patched candidate revision on {hostname}")

    diff = await ssh_device.send_command("nv config diff -o commands")
    return ANSI_ESCAPE_RE.sub("", diff)


async def display_cumulus_diff(diff, logger):
    """
    Colorise `nv config diff -o commands` output: `nv set` lines are green
    (config being added / changed), `nv unset` lines red (config being
    removed). Also handles the yaml style set: / unset: section output.
    """
    mode = None
    for line in diff.splitlines():
        stripped = line.strip()
        if stripped.startswith("- set:") or stripped == "set:":
            mode = "set"
        elif stripped.startswith("- unset:") or stripped == "unset:":
            mode = "unset"
        line = line.replace("[", "\\[")
        if stripped.startswith("nv set"):
            await logger.green_diff(line)
        elif stripped.startswith("nv unset"):
            await logger.red_diff(line)
        elif mode == "set":
            await logger.green_diff(line)
        elif mode == "unset":
            await logger.red_diff(line)
        else:
            await logger.grey_diff(line)


async def commit_cumulus(ssh_device, filtered_device, args, logger, progress):
    """
    Apply the NVUE candidate revision that the device-side dry run (--diff)
    already patched and validated:
        y -> nv config apply --assume-yes, optional nv config save
        n -> nv config detach (discard the candidate, device stays clean)
    """
    hostname = filtered_device["hostname"]

    progress.stop()
    commit_confirmed = await prompt(f"CS_CLI: Do you want to apply this config? {hostname} (y/n)", args)
    progress.start()
    if commit_confirmed.lower() != "y":
        await ssh_device.send_command("nv config detach")
        await logger.info(f"CS_CLI: Discarded candidate revision on {hostname}, device is clean")
        return False

    await logger.info(f"CS_CLI: Applying candidate config on {hostname}")
    try:
        apply_result = await ssh_device.send_command("nv config apply --assume-yes")
    except Exception as e:
        # apply failed (e.g. the device rejected the config), discard the
        # candidate so nothing half-finished stays behind
        await logger.error(f"CS_CLI: Apply failed on {hostname}, {e}")
        try:
            await ssh_device.send_command("nv config detach")
            await logger.info(f"CS_CLI: Discarded candidate revision on {hostname}, device is clean")
        except Exception as detach_error:
            await logger.warn(f"CS_CLI: Could not discard candidate on {hostname}, {detach_error}")
        return False
    await logger.info(f"CS_CLI: Applied config on {hostname} {apply_result.strip()}")

    progress.stop()
    save = await prompt(f"CS_CLI: Save config to startup (nv config save)? {hostname} (y/n)", args)
    progress.start()
    if save.lower() == "y":
        await ssh_device.send_command("nv config save")
        await logger.info(f"CS_CLI: Saved config to startup on {hostname}")
    else:
        await logger.info(f"CS_CLI: Config applied but not saved to startup on {hostname}")
    return True


SONIC_CHECKPOINT = "cs_cli_backup"


async def commit_sonic(ssh_device, filtered_device, candidate_config, args, logger, progress):
    """
    SONiC commit flow (GCU based, minimum disruption):
        1. upload the candidate config_db json (already computed by the diff
           step: running config deep-merged with the intended set config,
           minus the UNSET / DELETE paths - the file must always be the
           complete config, 'config replace' expects the whole config)
        2. prompt user, take a rollback checkpoint (config checkpoint)
        3. sudo config replace <file>  -> device validates, then applies only
           the changed sections (no full service restart like config reload)
        4. prompt keep / rollback (config rollback restores the checkpoint)
        5. prompt to persist, sudo config save -y
    """
    hostname = filtered_device["hostname"]
    remote_path = "/tmp/cs_cli_config_db.json"
    candidate_json = json.dumps(candidate_config, indent=2)

    await logger.info(f"CS_CLI: Copying candidate config to {hostname}:{remote_path}")
    await ssh_device.transfer_file(candidate_json, remote_path)

    progress.stop()
    commit_confirmed = await prompt(
        f"CS_CLI: Apply config with config replace (minimum disruption)? {hostname} (y/n)", args
    )
    progress.start()
    if commit_confirmed.lower() != "y":
        await logger.info(f"CS_CLI: Skipping commit on {hostname}")
        return False

    await logger.info(f"CS_CLI: Taking rollback checkpoint '{SONIC_CHECKPOINT}' on {hostname}")
    await ssh_device.send_command(
        f"sudo config delete-checkpoint {SONIC_CHECKPOINT} > /dev/null 2>&1; "
        f"sudo config checkpoint {SONIC_CHECKPOINT}"
    )

    await logger.info(f"CS_CLI: Applying config on {hostname} (config replace). This might take a few minutes..")
    try:
        await ssh_device.send_command(f"sudo config replace {remote_path}")
    except Exception as e:
        # config replace validates before touching anything, a failure here
        # means the device rejected the candidate and the config is unchanged
        await logger.error(f"CS_CLI: Config replace failed on {hostname} (config unchanged), {e}")
        return False
    await logger.info(f"CS_CLI: Applied config on {hostname}")

    progress.stop()
    keep = await prompt(
        f"CS_CLI: Keep the applied config? {hostname} (y/n, n rolls back to checkpoint '{SONIC_CHECKPOINT}')", args
    )
    progress.start()
    if keep.lower() != "y":
        await logger.info(f"CS_CLI: Rolling back {hostname} to checkpoint '{SONIC_CHECKPOINT}'..")
        await ssh_device.send_command(f"sudo config rollback {SONIC_CHECKPOINT}")
        await logger.info(f"CS_CLI: Rolled back config on {hostname}")
        return False

    progress.stop()
    save = await prompt(f"CS_CLI: Save config to startup (config save)? {hostname} (y/n)", args)
    progress.start()
    if save.lower() == "y":
        await ssh_device.send_command("sudo config save -y")
        await logger.info(f"CS_CLI: Saved config to startup on {hostname}")
    else:
        await logger.info(f"CS_CLI: Config applied but not saved to startup on {hostname}")
    return True


##############################################################################################################################
# Per device worker
##############################################################################################################################

async def connect_device(device, transport, hostname, logger):
    """
    Connect a device wrapper, returns True on success, False on failure
    (the failure is already logged).
    """
    try:
        connected = await device.connect()
        if type(connected) == dict:
            if "error" in connected:
                await logger.error(f"CS_CLI: Error connecting to {transport} {hostname}, {connected['error']}")
                return False
    except Exception as e:
        await logger.error(f"CS_CLI: Error connecting to {transport} {hostname}, {e}")
        return False
    await logger.info(f"CS_CLI: Connected to {transport} {hostname}")
    return True

async def concurrent_device(filtered_device, args, progress, dev_count):

    vendor = filtered_device["vendor"]
    hostname = filtered_device["hostname"]
    logger = ConsoleLogger(args)

    progress_task = progress.add_task(f"[cyan]Running for {hostname}...", total=1)

    ##############################################################################################################################
    # generate the day-0 bootstrap commands (management interface + REST API)
    # no device connection needed, the device is usually not reachable yet
    ##############################################################################################################################
    if args.generate_bootstrap:
        await logger.info(f"CS_CLI: Generating bootstrap config for {hostname}")
        bootstrap_text = await render_bootstrap_config(filtered_device, logger)
        if bootstrap_text is not None:
            if args.no_color:
                print(bootstrap_text)
            else:
                console.print(Syntax(bootstrap_text, "bash", background_color="default"))
        if not (args.get_config or args.to_yaml or args.diff or args.generate_config):
            progress.update(progress_task, advance=1)
            return

    ##############################################################################################################################
    # generate the vendor native config from the unified device vars + j2 template
    # no device connection needed, same idea as n2c_cli --generate_config
    ##############################################################################################################################
    intended_dict = None
    intended_file = None
    if args.generate_config:
        await logger.info(f"CS_CLI: Generating config for {hostname}")
        intended_dict, intended_file = await load_intended_config(filtered_device, logger)
        if intended_dict is None:
            progress.update(progress_task, advance=1)
            return
        generated_yaml = yaml.safe_dump(intended_dict, default_flow_style=False, sort_keys=False)
        if args.no_color:
            print(generated_yaml)
        else:
            console.print(Syntax(generated_yaml, "yaml", background_color="default"))

        if not (args.get_config or args.to_yaml or args.diff):
            progress.update(progress_task, advance=1)
            return

    ##############################################################################################################################
    # connect to the device
    # cumulus -> REST API for config export, SSH for dry run (--diff) / apply
    # sonic   -> ssh for everything
    ##############################################################################################################################
    rest_device = None
    ssh_device = None
    connected_rest = False
    connected_ssh = False
    cumulus_candidate_pending = False

    if vendor == "cumulus":
        need_rest = args.get_config or args.to_yaml
        need_ssh = args.diff
    else:
        need_rest = False
        need_ssh = args.get_config or args.to_yaml or args.diff

    try:
        if need_rest:
            rest_device = NvueRestDevice(filtered_device, args.u, args.p, logger)
            await logger.info(f"CS_CLI: Connecting to REST API {hostname}")
            connected_rest = await connect_device(rest_device, "REST API", hostname, logger)
            if not connected_rest:
                return

        if need_ssh:
            ssh_device = SSHDevice(filtered_device, args.u, args.p, logger)
            await logger.info(f"CS_CLI: Connecting to SSH {hostname}")
            connected_ssh = await connect_device(ssh_device, "SSH", hostname, logger)
            if not connected_ssh:
                return
        ##############################################################################################################################
        # get the current config from the device
        # needed for --get_config / --to_yaml, and for the sonic diff / commit
        # (the cumulus diff is a device-side dry run, no local compare needed)
        ##############################################################################################################################
        running_config_dict = None
        if args.get_config or args.to_yaml or (vendor == "sonic" and args.diff):
            source_device = rest_device if vendor == "cumulus" else ssh_device
            try:
                running_config_dict = await get_running_config_dict(source_device, filtered_device, logger)
            except Exception as e:
                await logger.error(f"CS_CLI: Error getting running config from {hostname}, {e}")
                return
            await logger.info(f"CS_CLI: Got running config for {hostname}")

        ##############################################################################################################################
        # export & display the running config in its native json format
        ##############################################################################################################################
        if args.get_config:
            running_config_json = json.dumps(running_config_dict, indent=2, ensure_ascii=False)

            path = os.path.join(CONFIGS_DIR, f"{hostname}_run_config.json")
            await prepare_output_file(path)
            async with aiofiles.open(path, "w") as f:
                await f.write(running_config_json)
            await logger.info(f"CS_CLI: Saved running config JSON to {path}")

            await logger.info(f"CS_CLI: Running config JSON for {hostname}\n")
            if args.no_color:
                print(running_config_json)
            else:
                console.print(Syntax(running_config_json, "json", background_color="default"))

        ##############################################################################################################################
        # export the running config dict as yaml
        ##############################################################################################################################
        if args.to_yaml:
            await logger.info(f"CS_CLI: Exporting running config as YAML for {hostname}")
            config_yaml = yaml.safe_dump(running_config_dict, default_flow_style=False, sort_keys=False)

            path = os.path.join(CONFIGS_DIR, f"{hostname}_run_config.yaml")
            await prepare_output_file(path)
            async with aiofiles.open(path, "w") as f:
                await f.write(config_yaml)
            await logger.info(f"CS_CLI: Saved running config YAML to {path}\n")

            if args.no_color:
                print(config_yaml)
            else:
                console.print(Syntax(config_yaml, "yaml", background_color="default"))

        ##############################################################################################################################
        # diff the running config against the intended config
        ##############################################################################################################################
        no_diff = False
        sonic_candidate = None
        mgmt_change = False
        if args.diff:
            # reuse the config already generated by --generate_config if present
            if intended_dict is None:
                intended_dict, intended_file = await load_intended_config(filtered_device, logger)
            if intended_dict is None:
                return
            await logger.info(f"CS_CLI: Loaded intended config from {intended_file}")

            if vendor == "cumulus":
                # device-side dry run: the candidate revision on the device is
                # patched and validated, `nv config diff -o commands` is the diff output
                await logger.info(f"CS_CLI: Device-side dry run for {hostname}")
                # flag set before the call: if patching fails half-way, the
                # finally-block detach still cleans the partial candidate
                cumulus_candidate_pending = True
                diff = await cumulus_candidate_diff(ssh_device, intended_dict, hostname, logger)
            else:
                # effective candidate: running config with the intended set
                # config merged in and the UNSET / DELETE paths removed
                set_tree = {key: value for key, value in intended_dict.items() if key != "UNSET"}
                sonic_candidate = deep_merge(running_config_dict, set_tree)
                if "UNSET" in intended_dict:
                    prune_sonic_config(sonic_candidate, intended_dict["UNSET"])
                diff = build_sonic_diff(running_config_dict, sonic_candidate, hostname)

            if diff.strip() != "":
                await logger.info(f"CS_CLI: Diff for {hostname}\n\n")
                if vendor == "cumulus":
                    await display_cumulus_diff(diff, logger)
                else:
                    await display_sonic_diff(diff, logger)
                await logger.info("\n\n")

                path = os.path.join(DIFFS_DIR, f"{hostname}_diff.txt")
                await prepare_output_file(path)
                async with aiofiles.open(path, "w") as f:
                    await f.write(diff)
                await logger.info(f"CS_CLI: Saved diff to {path}")
            else:
                await logger.green_diff(f"\n\nCS_CLI: No diff {hostname}\n\n")
                no_diff = True

            ##############################################################################################################################
            # optional device-side dry run for sonic: the device validates the
            # candidate and reports the changes as json patch operations,
            # without changing any config (slow: full YANG validation)
            ##############################################################################################################################
            if args.dry_run and vendor == "sonic" and no_diff is False \
                    and sonic_candidate is not None and ssh_device is not None:
                await logger.info(
                    f"CS_CLI: Device-side dry run for {hostname} (config replace --dry-run), "
                    f"this might take a few minutes.."
                )
                remote_path = "/tmp/cs_cli_config_db.json"
                await ssh_device.transfer_file(json.dumps(sonic_candidate, indent=2), remote_path)
                try:
                    dry_run_output = str(await ssh_device.send_command(f"sudo config replace {remote_path} --dry-run"))
                except Exception as e:
                    await logger.error(f"CS_CLI: Device-side dry run failed on {hostname}, {e}")
                    dry_run_output = ""
                for line in ANSI_ESCAPE_RE.sub("", dry_run_output).splitlines():
                    line = line.replace("[", "\\[")
                    if "DryRun" in line:
                        await logger.change_diff(line)
                    else:
                        await logger.grey_diff(line)

            ##############################################################################################################################
            # detect management interface changes: applying a wrong management
            # address makes the device unreachable, so commits require an
            # extra explicit confirmation for these
            ##############################################################################################################################
            if no_diff is False:
                if vendor == "cumulus":
                    mgmt_change = any("eth0" in line for line in diff.splitlines())
                elif sonic_candidate is not None and running_config_dict is not None:
                    for mgmt_table in ("MGMT_INTERFACE", "MGMT_PORT", "MGMT_VRF_CONFIG"):
                        if running_config_dict.get(mgmt_table) != sonic_candidate.get(mgmt_table):
                            mgmt_change = True

        ##############################################################################################################################
        # commit the intended config to the device
        ##############################################################################################################################
        if args.commit and args.diff and no_diff is False and intended_dict is not None:
            if mgmt_change:
                await logger.red_diff(f"\nCS_CLI: WARNING: this change modifies the MANAGEMENT interface on {hostname}!")
                await logger.red_diff("CS_CLI: If the new address is wrong you will LOSE ACCESS to the device.\n")
                progress.stop()
                mgmt_confirmed = await prompt(
                    f"CS_CLI: Commit the management interface change anyway? {hostname} (y/n)", args
                )
                progress.start()
                if mgmt_confirmed.lower() != "y":
                    await logger.info(f"CS_CLI: Commit cancelled for {hostname} (management interface change not confirmed)")
                    return

            commit_success = False
            if vendor == "cumulus":
                commit_success = await commit_cumulus(
                    ssh_device, filtered_device, args, logger, progress
                )
                # commit_cumulus either applied or detached the candidate
                cumulus_candidate_pending = False
            elif vendor == "sonic":
                commit_success = await commit_sonic(
                    ssh_device, filtered_device, sonic_candidate, args, logger, progress
                )
            await logger.info(f"CS_CLI: Commit success {hostname}: {commit_success}")

    except Exception as e:
        traceback.print_exception(type(e), e, e.__traceback__)
        await logger.error(f"CS_CLI: Error processing {hostname}, {e}")

    finally:
        ##############################################################################################################################
        # discard any leftover candidate revision, then disconnect
        # (covers --diff without --commit, no-diff runs and error paths, so
        # the device is always left clean unless the config was applied)
        ##############################################################################################################################
        if cumulus_candidate_pending and connected_ssh and ssh_device is not None:
            try:
                await ssh_device.send_command("nv config detach")
                await logger.info(f"CS_CLI: Discarded candidate revision on {hostname}, device is clean")
            except Exception as e:
                await logger.warn(f"CS_CLI: Could not discard candidate on {hostname}, {e}")

        if connected_rest and rest_device is not None:
            await rest_device.disconnect()
            await logger.info(f"CS_CLI: Disconnected from REST API {hostname}")
        if connected_ssh and ssh_device is not None:
            await ssh_device.disconnect()
            await logger.info(f"CS_CLI: Disconnected from SSH {hostname}")

        progress.update(progress_task, advance=1)
        if dev_count > 10:
            progress.remove_task(progress_task)


##############################################################################################################################
# main
##############################################################################################################################

async def main():
    args = await process_args()

    needs_connection = args.get_config or args.to_yaml or args.diff
    if os.getenv("DEVICE_USERNAME"):
        args.u = os.getenv("DEVICE_USERNAME")
        args.p = os.getenv("DEVICE_PASSWORD") or args.p
    if needs_connection:
        if not args.u:
            args.u = input("Enter device username: ")
        if not args.p:
            args.p = getpass.getpass("Enter device password: ")

    logger = ConsoleLogger(args)

    if not (args.get_config or args.to_yaml or args.generate_config or args.generate_bootstrap or args.diff):
        await logger.warn(
            "CS_CLI: Nothing to do, use --get_config, --to_yaml, --generate_config, "
            "--generate_bootstrap and/or --diff (see --help)"
        )
        return

    filtered_devices = await get_filtered_devices(args)

    if len(filtered_devices) == 0:
        await logger.info(f"\nCS_CLI: No devices found")
        return
    await logger.info(f"\nCS_CLI: Found {len(filtered_devices)} device(s)")
    device_list = [filtered_device["hostname"] for filtered_device in filtered_devices]
    await logger.info(f"\nCS_CLI: {device_list}")

    question = await prompt(f"\nCS_CLI: Process these devices? (y/n)", args)

    if question.lower() != "y":
        return
    print()

    with Progress() as progress:
        if args.parallel:
            # Limit concurrent tasks to 10
            semaphore = asyncio.Semaphore(10)

            async def concurrent_device_with_limit(filtered_device, args, progress, total_devices):
                async with semaphore:
                    return await concurrent_device(filtered_device, args, progress, total_devices)

            tasks = [
                concurrent_device_with_limit(filtered_device, args, progress, len(filtered_devices))
                for filtered_device in filtered_devices
            ]

            # Use asyncio.gather to run all tasks, but controlled by the semaphore limit
            results = await asyncio.gather(*tasks, return_exceptions=True)
            return results
        else:
            for filtered_device in filtered_devices:
                task = concurrent_device(filtered_device, args, progress, len(filtered_devices))
                await task


if __name__ == "__main__":
    asyncio.run(main())
```
{{< /details >}}
