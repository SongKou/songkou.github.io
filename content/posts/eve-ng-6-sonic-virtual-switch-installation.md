+++
title = 'EVE-NG_6_SONiC_Virtual_Switch_Installation'
date = 2026-07-23T20:00:00+08:00
draft = false
categories = ['Network']
tags = ['SONiC', 'EVE-NG', 'Virtual Switch', 'Lab', 'Network']
+++

SONiC (Software for Open Networking in the Cloud) publishes a virtual-switch build (SONiC-VS) that boots as a QEMU VM, which makes it a perfect lab guest for EVE-NG. This post walks through installing SONiC-VS on EVE-NG 6 using the built-in **Sonic Switch** template, validating the first boot with real console captures, and finishes with a command reference for the SONiC CLI — the commands you actually reach for once the switch is up: interfaces, routing, logs, config, and user management.

**Use the right template.** EVE-NG 6 ships a dedicated Sonic Switch template; do not install the image under the generic Linux template (I did this first, and the recovery transcript is included below). The correct layout is:

```text
Template:  Sonic Switch
Directory: /opt/unetlab/addons/qemu/sonicsw-<version>/
Disk:      virtioa.qcow2
```

This guide uses `202607` as the version label, so the final path is:

```text
/opt/unetlab/addons/qemu/sonicsw-202607/virtioa.qcow2
```

## 1. Download the SONiC-VS image

Grab the image from the official [SONiC Latest Successful Builds](https://sonic-net.github.io/SONiC/sonic_latest_images.html) page:

1. In the **Latest Successful Builds** table, find the row labeled **VS**.
2. In that row, click the image link under **MASTER**. The displayed name follows the pattern `vs.img.gz-<build-number>` — at the time of this install it was:

   ```text
   VS → MASTER → vs.img.gz-1169751
   ```

   The numeric suffix changes with every successful build.

3. Download the **VS** image. Do **not** grab `docker-sonic-vs.gz` — that is a container artifact, not the bootable VM disk EVE-NG needs.
4. Rename the download so it matches the commands in this guide. On Windows PowerShell:

   ```powershell
   Rename-Item .\vs.img.gz-1169751 sonic-vs.img.gz
   ```

Keep a note of the branch and build number you downloaded — it becomes the version label on the EVE-NG directory, and `show version` on the running switch will echo it back (`SONiC.master.1169751-...` below). If you ever need to build your own image, the [sonic-buildimage repository](https://github.com/sonic-net/sonic-buildimage) supports `PLATFORM=vs`.

## 2. Upload the image to EVE-NG

Copy `sonic-vs.img.gz` to a temporary location on the EVE-NG server, then connect and confirm:

```powershell
scp .\sonic-vs.img.gz root@<eve-ng-ip>:/root/
```

```bash
ssh root@<eve-ng-ip>
ls -lh /root/sonic-vs.img.gz
```

## 3. Create the Sonic Switch image directory

The directory name must start with the `sonicsw-` prefix — that is what binds it to the Sonic Switch template. The suffix is a free-form label shown as the image version in the EVE-NG node dialog:

```bash
mkdir -p /opt/unetlab/addons/qemu/sonicsw-202607
```

## 4. Decompress and inspect the disk

Decompress while keeping the `.gz` archive (`-k`), then check what format the disk actually is:

```bash
cd /root
gzip -dk sonic-vs.img.gz
qemu-img info /root/sonic-vs.img
```

```text
root@eve-ng6:/opt/unetlab/addons/qemu/sonicsw-202607# qemu-img info sonic-vs.img
image: sonic-vs.img
file format: qcow2
virtual size: 16 GiB (17179869184 bytes)
disk size: 6.73 GiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
```

If `qemu-img` is not in your shell path, use `/opt/qemu/bin/qemu-img` instead.

**If the image is already QCOW2** (as above — recent VS builds ship as qcow2 despite the `.img` name), just move it into place with the filename the template expects:

```bash
mv /root/sonic-vs.img \
  /opt/unetlab/addons/qemu/sonicsw-202607/virtioa.qcow2
```

**If the image is raw**, convert it instead of merely renaming it:

```bash
qemu-img convert -p -f raw -O qcow2 \
  /root/sonic-vs.img \
  /opt/unetlab/addons/qemu/sonicsw-202607/virtioa.qcow2
```

## 5. Fix EVE-NG permissions and verify

Run the EVE-NG permissions repair, then confirm the installed disk:

```bash
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions

qemu-img info /opt/unetlab/addons/qemu/sonicsw-202607/virtioa.qcow2
ls -lh /opt/unetlab/addons/qemu/sonicsw-202607/
```

The directory should contain exactly one file, `virtioa.qcow2`, and `qemu-img info` should report `file format: qcow2`.

## 6. Verify the built-in EVE-NG template

Confirm the Sonic Switch template exists and check its defaults:

```bash
find /opt/unetlab/html/templates -type f -iname '*sonic*' -print

grep -Ei 'name:|image:|qemu_arch|qemu_nic|ethernet|ram|cpu|console' \
  /opt/unetlab/html/templates/*/sonicsw.yml
```

On EVE-NG 6 both the AMD and Intel template trees carry `sonicsw.yml` with identical defaults:

| Setting | Template value |
|---|---:|
| Name | `S-SW` |
| CPU | `2` |
| CPU limit | `1` |
| RAM | `4096` MB |
| Ethernet interfaces | `10` |
| Interface format | `Ethernet{0}` |
| Console | `telnet` |
| QEMU architecture | `x86_64` |
| QEMU NIC | `e1000` |

Use these defaults for the first boot. If the SONiC containers crash or restart under memory pressure, shut the node down and bump it to 4 vCPUs / 8192 MB RAM.

## 7. Add and start the node

In the EVE-NG web interface:

1. Open or create a lab.
2. Select **Add an object → Node**.
3. Choose **Sonic Switch**.
4. Select the image/version corresponding to `sonicsw-202607`.
5. Keep the template defaults for the first attempt (CPU 2, RAM 4096 MB, 10 Ethernet, e1000 NIC, Telnet console).
6. Add the node, start it, and open its Telnet console.

The first boot takes several minutes — SONiC has to initialize its Redis databases and start a dozen service containers before the CLI is fully functional. Be patient before declaring it broken.

## 8. First login and validation

The default credentials for the VS development image are:

```text
Username: admin
Password: YourPaSsWoRd
```

(Credentials can vary between builds — if these fail, check the documentation for the artifact you downloaded. And change the password immediately; see the command reference below.)

Run a quick validation pass:

```bash
show version
show platform summary
show interfaces status
docker ps
systemctl --failed
```

What a healthy result looks like: `show version` reports the build you downloaded and `ASIC: vs`; the front-panel `Ethernet` interfaces exist and are admin-up; the core containers (`database`, `swss`, `syncd`, `bgp`, `teamd`, `lldp`, `pmon`) are all running; and nothing critical is failed. The real capture from this install:

```text
sonic login: admin
Password:
Linux sonic 6.12.41+deb13-sonic-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.41-1 (2026-07-20) x86_64
You are on
  ____   ___  _   _ _  ____
 / ___| / _ \| \ | (_)/ ___|
 \___ \| | | |  \| | | |
  ___) | |_| | |\  | | |___
 |____/ \___/|_| \_|_|\____|

-- Software for Open Networking in the Cloud --

admin@sonic:~$ show version

SONiC Software Version: SONiC.master.1169751-d5b2072d3
SONiC OS Version: 13
Distribution: Debian 13.5
Kernel: 6.12.41+deb13-sonic-amd64
Build commit: d5b2072d3
Build date: Mon Jul 20 17:07:03 UTC 2026

Platform: x86_64-kvm_x86_64-r0
HwSKU: Force10-S6000
ASIC: vs
ASIC Count: 1
Serial Number: N/A
Model Number: N/A
Hardware Revision: N/A
Uptime: 11:34:40 up 10 min,  1 user,  load average: 0.41, 0.73, 0.65
Date: Thu 23 Jul 2026 11:34:40

admin@sonic:~$ show platform summary
Platform: x86_64-kvm_x86_64-r0
HwSKU: Force10-S6000
ASIC: vs
ASIC Count: 1
```

Two things worth noticing: the virtual platform emulates a **Force10-S6000** HwSKU, which is why 32 40G front-panel ports appear even though the EVE-NG node only has 10 NICs; and `ASIC: vs` confirms you are on the software data plane.

```text
admin@sonic:~$ show interfaces status
  Interface            Lanes    Speed    MTU    FEC           Alias    Vlan    Oper    Admin    Type    Asym PFC
-----------  ---------------  -------  -----  -----  --------------  ------  ------  -------  ------  ----------
  Ethernet0      25,26,27,28      40G   9100    N/A    fortyGigE0/0  routed      up       up     N/A         N/A
  Ethernet4      29,30,31,32      40G   9100    N/A    fortyGigE0/4  routed      up       up     N/A         N/A
  Ethernet8      33,34,35,36      40G   9100    N/A    fortyGigE0/8  routed      up       up     N/A         N/A
 Ethernet12      37,38,39,40      40G   9100    N/A   fortyGigE0/12  routed      up       up     N/A         N/A
 Ethernet16      45,46,47,48      40G   9100    N/A   fortyGigE0/16  routed      up       up     N/A         N/A
 Ethernet20      41,42,43,44      40G   9100    N/A   fortyGigE0/20  routed      up       up     N/A         N/A
 Ethernet24          1,2,3,4      40G   9100    N/A   fortyGigE0/24  routed      up       up     N/A         N/A
 Ethernet28          5,6,7,8      40G   9100    N/A   fortyGigE0/28  routed    down       up     N/A         N/A
 Ethernet32      13,14,15,16      40G   9100    N/A   fortyGigE0/32  routed    down       up     N/A         N/A
 ...
Ethernet124     97,98,99,100      40G   9100    N/A  fortyGigE0/124  routed    down       up     N/A         N/A
```

The interfaces that map to connected EVE-NG NICs come up `Oper: up`; the rest of the emulated S6000 ports stay `down`. Container and service health:

```text
admin@sonic:~$ docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS     NAMES
c3a371f5d9e4   docker-snmp:latest                   "/usr/bin/docker-snm…"   9 minutes ago    Up 9 minutes              snmp
470a71e1abbb   docker-sonic-mgmt-framework:latest   "/usr/local/bin/supe…"   9 minutes ago    Up 9 minutes              mgmt-framework
1c478aaa90fd   docker-lldp:latest                   "/usr/bin/docker-lld…"   9 minutes ago    Up 9 minutes              lldp
93557dfbe86c   docker-router-advertiser:latest      "/usr/bin/docker-ini…"   11 minutes ago   Up 11 minutes             radv
b5ac92c184fc   docker-sonic-gnmi:latest             "/usr/local/bin/supe…"   11 minutes ago   Up 11 minutes             gnmi
f2499e555651   docker-gbsyncd-vs:latest             "/usr/local/bin/supe…"   11 minutes ago   Up 11 minutes             gbsyncd
0ba83ee41071   docker-eventd:latest                 "/usr/local/bin/supe…"   11 minutes ago   Up 11 minutes             eventd
628d51a8389f   docker-syncd-vs:latest               "/usr/local/bin/supe…"   11 minutes ago   Up 11 minutes             syncd
89bb6197cbe2   docker-fpm-frr:latest                "/usr/bin/docker_ini…"   11 minutes ago   Up 11 minutes             bgp
9d48846dba5a   docker-teamd:latest                  "/usr/bin/docker-tea…"   11 minutes ago   Up 11 minutes             teamd
0339b86ee16b   docker-sysmgr:latest                 "/usr/local/bin/supe…"   11 minutes ago   Up 11 minutes             sysmgr
be616a66b28c   docker-orchagent:latest              "/usr/bin/docker-ini…"   11 minutes ago   Up 11 minutes             swss
3489c45c68ea   docker-platform-monitor:latest       "/usr/bin/docker_ini…"   11 minutes ago   Up 11 minutes             pmon
31b5aa7ea510   docker-database:latest               "/usr/local/bin/dock…"   11 minutes ago   Up 11 minutes             database

admin@sonic:~$ systemctl --failed
  UNIT                     LOAD   ACTIVE SUB    DESCRIPTION
● system-health.service    loaded failed failed SONiC system health monitor
● watchdog-control.service loaded failed failed watchdog control service

2 loaded units listed.
```

`system-health` and `watchdog-control` fail on the virtual platform because there is no real hardware to monitor — this is expected on VS and safe to ignore. Anything else in `systemctl --failed` deserves investigation.

You can also confirm the data-plane wiring from the Linux side — the front-panel `Ethernet` interfaces are ordinary kernel netdevs attached to a bridge, alongside the `eth0`-`eth7` NICs that EVE-NG provides:

```text
admin@sonic:~$ ip -br link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
eth0             UP             50:00:00:0c:00:00 <BROADCAST,MULTICAST,UP,LOWER_UP>
eth1             UP             50:00:00:0c:00:01 <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP>
eth2             UP             50:00:00:0c:00:02 <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP>
...
Ethernet0        UNKNOWN        22:4c:e8:1f:af:44 <BROADCAST,MULTICAST,UP,LOWER_UP>
Ethernet4        UNKNOWN        22:4c:e8:1f:af:44 <BROADCAST,MULTICAST,UP,LOWER_UP>
Bridge           UP             22:4c:e8:1f:af:44 <BROADCAST,MULTICAST,UP,LOWER_UP>
Loopback0        UNKNOWN        3a:05:85:08:0c:6b <BROADCAST,NOARP,UP,LOWER_UP>
```

Finally, persist the running configuration so it survives a reboot:

```text
admin@sonic:~$ sudo config save -y
Running command: /usr/local/bin/sonic-cfggen -d --print-data > /etc/sonic/config_db.json
```

## 9. Recovering from the generic-Linux template

I originally installed this image under the generic Linux template (`linux-sonic-vs/hda.qcow2`), which boots but gives you wrong interface naming and template settings. The fix is simply moving the disk into the correct `sonicsw-` directory — the transcript from that recovery, kept as captured:

```console
root@eve-ng6:/opt/unetlab/addons/qemu/sonicsw-202607# ls -l /opt/unetlab/addons/qemu/linux-sonic-vs
total 14102476
-rw-r--r-- 1 root root 7217807360 Jul 22 15:32 hda.qcow2
-rw-r--r-- 1 root root 7223181312 Jul 20 17:18 sonic-vs.img
root@eve-ng6:/opt/unetlab/addons/qemu/sonicsw-202607# mv /opt/unetlab/addons/qemu/linux-sonic-vs/sonic-vs.img ./
root@eve-ng6:/opt/unetlab/addons/qemu/sonicsw-202607# qemu-img info sonic-vs.img
image: sonic-vs.img
file format: qcow2
virtual size: 16 GiB (17179869184 bytes)
disk size: 6.73 GiB
root@eve-ng6:/opt/unetlab/addons/qemu/sonicsw-202607# mv ./sonic-vs.img virtioa.qcow2
root@eve-ng6:/opt/unetlab/addons/qemu/sonicsw-202607# /opt/unetlab/wrappers/unl_wrapper -a fixpermissions

root@eve-ng6:/opt/unetlab/addons/qemu/sonicsw-202607# ls -lh
total 6.8G
-rw-r--r-- 1 root root 6.8G Jul 20 17:18 virtioa.qcow2
```

Once the Sonic Switch node has booted and been validated, clean up the old directory — but move it aside first rather than deleting it, so the cleanup is recoverable:

```bash
test -f /opt/unetlab/addons/qemu/sonicsw-202607/virtioa.qcow2 \
  && qemu-img info /opt/unetlab/addons/qemu/sonicsw-202607/virtioa.qcow2

mv /opt/unetlab/addons/qemu/linux-sonic-vs /root/linux-sonic-vs-backup
```

Confirm no existing lab still references the old generic Linux image, retest the Sonic Switch node, and delete the backup only when everything checks out.

## Final expected state

```text
EVE-NG template: Sonic Switch
Image prefix:    sonicsw-
Image directory: /opt/unetlab/addons/qemu/sonicsw-202607/
Disk filename:   virtioa.qcow2
Disk format:     qcow2
NIC model:       e1000
Console:         Telnet
```

## 10. SONiC vswitch command reference

The commands you actually use day-to-day once the switch is up. SONiC's CLI is a thin layer over a Debian host: `show` reads state, `sudo config` writes it into the Redis CONFIG_DB, and everything is also reachable through plain Linux tools (`ip`, `journalctl`, `docker`). Most `show` commands accept `-h` for help.

### 10.1 Interfaces and IP addresses

- `show interfaces status` — the go-to overview: per-port speed, MTU, alias, VLAN mode (`routed`/`trunk`), and **Oper/Admin** state. Oper `down` with Admin `up` means no link (or no cable in the lab topology).
- `show interfaces description` — port-to-description/alias mapping; `show interfaces status Ethernet0` narrows any of these to one port.
- `show ip interfaces` / `show ipv6 interfaces` — every L3 interface with its address, admin/oper state, and BGP neighbor if one is bound. This is also where you see the management IP on `eth0`.
- `show interfaces counters` — RX/TX packets, errors, and drops per port; `sonic-clear counters` zeroes them so you can watch a fresh delta.
- `sudo config interface startup Ethernet0` / `sudo config interface shutdown Ethernet0` — admin up/down a port.
- `sudo config interface ip add Ethernet0 10.0.0.1/24` / `sudo config interface ip remove Ethernet0 10.0.0.1/24` — add/remove an L3 address on a front-panel port.
- **Configure the management interface (`eth0`)** — by default eth0 runs DHCP. To set a static management address, use the same `config interface ip` syntax but append the default gateway (required for eth0, unlike front-panel ports):

  ```bash
  sudo config interface ip add eth0 192.168.1.10/24 192.168.1.1
  sudo config save -y
  ```

  This writes a `MGMT_INTERFACE` entry into CONFIG_DB and installs the gateway in the separate management VRF/routing table, so management traffic does not leak into the front-panel routing table. To go back to DHCP, remove the static address: `sudo config interface ip remove eth0 192.168.1.10/24`. In EVE-NG, connect eth0 (the first NIC) to a management network/cloud object for this to be reachable.
- `show management_interface address` — the configured management address and gateway; empty output means eth0 is still on DHCP/unconfigured (as in the first-boot capture above).
- `ip -br link` / `ip -br addr` — the Linux view of the same interfaces; useful for confirming what the kernel actually has versus what CONFIG_DB says.

### 10.2 VLANs and MAC table

- `show vlan brief` — VLANs, their members with tagging mode, IP addresses, and DHCP helper config in one table.
- `sudo config vlan add 100` then `sudo config vlan member add -u 100 Ethernet0` — create a VLAN and add an untagged member (drop `-u` for tagged).
- `show mac` — the MAC address table (FDB): which MAC was learned on which port/VLAN.

### 10.3 Routing

- `show ip route` / `show ipv6 route` — the kernel/FRR routing table with protocol codes; `show ip route 10.0.0.0/24` for a single prefix.
- `show ip bgp summary` — BGP neighbors, states, and prefix counts; the first stop for any BGP problem. `show ip bgp neighbors <ip>` for the full per-peer detail.
- `vtysh` — drops into the FRR shell, where the full Cisco-like suite lives: `show running-config`, `show bgp summary`, `configure terminal`. SONiC's BGP runs inside the `bgp` container but `vtysh` on the host connects straight to it.
- `sudo config route add prefix 10.99.0.0/24 nexthop 10.0.0.2` — add a static route through the SONiC config layer (persisted by `config save`).
- `show arp` / `show ndp` — ARP and IPv6 neighbor tables, for verifying next-hop resolution.

### 10.4 Neighbors (LLDP)

- `show lldp table` — one line per neighbor: local port, remote device, remote port. The fastest way to verify your EVE-NG cabling matches the intended topology.
- `show lldp neighbors` — the verbose per-neighbor detail (capabilities, management address, TTL).

### 10.5 System health and services

- `show version` — SONiC build, kernel, platform, HwSKU, uptime, and the full docker image list.
- `show platform summary` — platform/HwSKU/ASIC in a compact form (`ASIC: vs` on the virtual switch).
- `show services` — which SONiC containers are running and the processes inside each.
- `show feature status` — every feature (bgp, lldp, snmp, teamd, …) with its enabled/disabled state and auto-restart policy.
- `show processes cpu` / `show system-memory` — top-style CPU view and memory summary; the first check when a VS node with default 4 GB RAM starts misbehaving.
- `docker ps` — the container view; a core container (`database`, `swss`, `syncd`, `bgp`) missing or restarting is the classic sick-SONiC symptom.
- `systemctl --failed` — failed systemd units. On VS, `system-health` and `watchdog-control` are expected failures (no hardware); anything else is real.

### 10.6 Configuration management

First, the model: **SONiC's classic CLI has no candidate/commit workflow.** Unlike Junos or Cumulus NVUE (`nv config diff` / `nv config apply`), every `sudo config ...` command writes CONFIG_DB and takes effect immediately — `config save` is persistence, not a commit. There is no literal `show config diff` or `commit` command, but the pieces to build the same workflow exist: a diff comes from comparing the running config against the saved file, and commit/rollback semantics come from the Generic Config Updater's checkpoints (available since SONiC 202111, present on master VS builds).

- `show runningconfiguration all` — the entire running config as JSON, straight from CONFIG_DB. Pipe through `head` or query specific sections (`show runningconfiguration interfaces`).
- `sudo config save -y` — persist the running config to `/etc/sonic/config_db.json`. **Nothing survives a reboot until you run this.**
- `sudo sonic-cfggen -d --print-data > /tmp/running.json && diff /tmp/running.json /etc/sonic/config_db.json` — the closest thing to `show config diff`: running config versus saved startup config. `config save` writes the file with the same generator, so an empty diff means everything is saved; any output is exactly your unsaved changes.
- `sudo config checkpoint <name>` / `sudo config rollback <name>` — Generic Config Updater checkpoints: snapshot the current CONFIG_DB under a name, and roll the running config back to it later. This is your safety net before a risky change — the SONiC equivalent of commit/revert. `sudo config list-checkpoints` and `sudo config delete-checkpoint <name>` manage them.
- `sudo config apply-patch <patch.json>` — apply a JSON Patch (RFC 6902) to CONFIG_DB; `sudo config replace <file.json>` replaces the whole running config, computing and applying only the minimal set of changes. These are the declarative, automation-friendly alternatives to a series of imperative `config` commands.
- `sudo config reload -y` — reload `/etc/sonic/config_db.json` into the running system (restarts services; disruptive).
- `sudo config load -y /path/to/config_db.json` — load an alternate config file.
- `sonic-db-cli CONFIG_DB keys '*'` — poke the Redis database directly; `sonic-db-cli CONFIG_DB hgetall 'PORT|Ethernet0'` shows one object. Great for understanding what a `config` command actually wrote.

### 10.7 Logs

- `show logging` — the syslog; `show logging | tail -50` for the recent tail, `show logging bgp` filtered by process.
- `sudo tail -f /var/log/syslog` — live-follow everything; SONiC aggregates container logs here too.
- `sudo journalctl -u swss -f` — follow one service's journal (`database`, `bgp`, `syncd`, …); pairs with `sudo systemctl status swss` when a container will not stay up.
- `docker logs bgp` — stdout/stderr from inside a container, for when the journal only says the container died.
- `show techsupport` — collects a full support bundle (logs, configs, core dumps) into a tarball under `/var/dump/`; overkill for a lab but the standard artifact for reporting SONiC bugs.

### 10.8 Users and passwords

- `passwd` — change your own password (do this on first login; the default `admin`/`YourPaSsWoRd` is public knowledge).
- `sudo passwd admin` — reset another user's password.
- `sudo useradd -m -s /bin/bash -G sudo,docker <name>` — add a user; SONiC user management is plain Debian, and membership in `sudo` and `docker` is what makes the SONiC CLI fully usable.
- `sudo userdel -r <name>` — remove a user and their home directory.

## References

- [SONiC Command Reference](https://github.com/sonic-net/sonic-utilities/blob/master/doc/Command-Reference.md) — the official, exhaustive reference for every `show` and `config` command, including the Generic Config Updater (checkpoint/rollback/apply-patch) section.
- [SONiC documentation portal](https://sonic-net.github.io/SONiC/) — landing page for user guides, tutorials, and architecture documents.
- [SONiC Latest Successful Builds](https://sonic-net.github.io/SONiC/sonic_latest_images.html) — where the VS image in this guide comes from.
- [sonic-buildimage](https://github.com/sonic-net/sonic-buildimage) — the image build system, if you need to build your own VS image (`PLATFORM=vs`).
- [Generic Config Updater and rollback design docs](https://github.com/sonic-net/SONiC/tree/master/doc/config-generic-update-rollback) — the design behind `config apply-patch`, `config checkpoint`, and `config rollback`.
- [FRRouting documentation](https://docs.frrouting.org/) — reference for everything inside `vtysh`.
- [EVE-NG documentation](https://www.eve-ng.net/index.php/documentation/) — image installation how-tos and template details.
