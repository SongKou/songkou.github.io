+++
title = 'SaltStack_Guide'
date = 2026-07-21T03:00:00+08:00
draft = false
categories = ['Infrastructure']
tags = ['SaltStack', 'Linux', 'Automation', 'Ubuntu']
+++

*Target platform: Debian / Ubuntu. Current LTS at time of writing: **Salt 3008 LTS**. Packages are now hosted on Broadcom's Artifactory (the old `repo.saltproject.io` has been retired).*

---

## 1. What Salt is

Salt (formerly SaltStack) is an open-source automation and infrastructure management framework used for:

- Configuration management
- Remote command execution
- Software deployment
- Server orchestration
- Compliance enforcement
- Infrastructure automation
- Network device automation

It runs in two common modes:

- **Master / minion (agent) mode** — a central `salt-master` sends commands to many `salt-minion` agents over an encrypted ZeroMQ message bus. This is the classic, scalable setup:

![Salt master/minion architecture: one Salt Master (control server) connected over ZeroMQ TCP 4505/4506 to three minions - two Linux and one Windows; commands and states flow down, results and grains flow up](/posts/salt-guide/master-minion-topology.svg)

  The Salt Master defines the desired state. Salt Minions execute commands and enforce configurations. Salt is designed to manage thousands of systems and provides both remote execution and declarative configuration management through Salt States.

- **Masterless (`salt-call --local`) mode** — a single machine applies its own configuration with no master. Good for testing, small setups, or immutable-image workflows.

Core building blocks:

| Concept | What it is |
|---|---|
| **Execution modules** | Ad-hoc commands you run right now (e.g. install a package, restart a service). |
| **States (SLS files)** | Declarative YAML describing the *desired* end state of a system. |
| **Grains** | Static facts about a minion (OS, CPU, IP, custom labels). |
| **Pillar** | Secure, per-minion data (secrets, variables) defined on the master. |
| **Top file** | Maps which states/pillars apply to which minions. |
| **Targeting** | Selecting which minions a command hits (by ID, grain, regex, etc.). |

### 1.1 Main components in more detail

**Salt Master** — the controller node. Responsibilities:

- Stores configurations
- Sends commands
- Manages authentication keys
- Executes orchestration
- Maintains state files

Example:

```text
salt-master01
IP: 10.10.80.99
OS: Ubuntu 24.04
```

**Salt Minion** — the managed node. Responsibilities:

- Receives commands
- Executes modules
- Applies states
- Reports results

Example:

```text
web01
IP: 10.10.80.91

db01
IP: 10.10.80.92
```

**Salt States** — a declarative configuration language. Instead of the imperative sequence:

```text
Install nginx, copy config, start service
```

you describe the outcome:

```text
This server should have nginx installed and running.
```

Example:

```yaml
nginx:
  pkg.installed

nginx-service:
  service.running:
    - enable: True
```

Salt ensures the system reaches this state — and keeps it there on every subsequent run (see Section 7).

**Salt Pillar** — stores sensitive or environment-specific data, such as:

- database passwords
- API keys
- SSL certificates
- environment variables

Example:

```text
pillar/
 |
 └── database.sls
```

```yaml
# database.sls
mysql_password: MySecret123
```

(Pillar is covered in detail in Section 8.)

**Salt Grains** — static information collected from minions. Examples:

```text
OS:        Ubuntu
CPU:       Intel Xeon
Hostname:  web01
IP:        10.10.80.91
```

Query them with:

```bash
salt '*' grains.items
```

### 1.2 Communication model

Salt's transport stack uses:

- **ZeroMQ** over **TCP**
- **AES encryption** for the message bus
- **Public key authentication** for minion identity (the `salt-key` handshake, Section 4)

| Port | Purpose                                               |
|------|-------------------------------------------------------|
| 4505 | Publisher (master pushes commands to all minions)     |
| 4506 | Request server (minions return results, fetch files)  |

```text
Salt Master
   |
   |  TCP 4505/4506
   |
Minions
```

Connections are initiated **outbound from the minions** to the master, so only the master needs these two ports opened (firewall commands in Section 3.2).

---

## 2. Installation (Debian / Ubuntu, APT)

### 2.1 Decide the topology

- **One control node** → install `salt-master` (and usually `salt-minion` on the same box so it can manage itself).
- **Each managed node** → install `salt-minion`.
- **Single/standalone box** → install `salt-minion` only and run masterless.

### 2.2 Add the Salt repository and GPG key

Run on every node (master and minions):

```bash
# Create the keyring directory
sudo mkdir -m 0755 -p /etc/apt/keyrings

# Import the Salt Project signing key
curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp > /dev/null

# Add the repo definition (points at packages.broadcom.com/artifactory/saltproject-deb/)
curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources \
  | sudo tee /etc/apt/sources.list.d/salt.sources
```

### 2.3 (Recommended) Pin the major version

Pinning stops an unintended jump to a new major release during a routine `apt upgrade`:

```bash
sudo tee /etc/apt/preferences.d/salt-pin-1001 > /dev/null <<'EOF'
Package: salt-*
Pin: version 3008.*
Pin-Priority: 1001
EOF
```

### 2.4 Refresh metadata and install

```bash
sudo apt update

# On the control node:
sudo apt install -y salt-master salt-minion

# On each managed node:
sudo apt install -y salt-minion
```

`salt-common` is pulled in automatically as a dependency (it contains the shared Salt libraries and the `salt-call` binary).

To install an exact point release instead of the newest in the pinned series:

```bash
sudo apt install -y salt-minion=3008.2 salt-common=3008.2
```

> **Note on packaging:** Modern Salt ships as **onedir** — a self-contained bundle with its own Python. It no longer depends on the system Python, so upgrades are cleaner and there are no `pip`-versus-system conflicts.

### 2.5 Alternative: the bootstrap script (cross-distro, quick)

If you want a single command that detects the OS and installs for you (handy for provisioning scripts, or on non-Debian systems):

```bash
curl -fsSL https://github.com/saltstack/salt-bootstrap/releases/latest/download/bootstrap-salt.sh -o bootstrap-salt.sh

# Install a minion for the 3008 (Argon) LTS series:
sudo sh bootstrap-salt.sh -X stable 3008

# Install BOTH master and minion (-M adds the master):
sudo sh bootstrap-salt.sh -M -X stable 3008
```

Flags worth knowing: `-M` also install the master, `-N` install *no* minion, `-X` don't start services immediately, `-P` allow pip-based deps.

Always review a bootstrap script before piping it to a shell as root.

---

## 3. Configuration

Config lives under `/etc/salt/`. Files ending in `.conf` inside `/etc/salt/master.d/` and `/etc/salt/minion.d/` are merged in — that's the tidy way to add settings without editing the big default file.

### 3.1 Point the minion at the master

Edit `/etc/salt/minion` (or drop a file in `/etc/salt/minion.d/`):

```yaml
# /etc/salt/minion.d/master.conf
master: 192.0.2.10          # IP or DNS name of your salt-master
id: web01.example.com       # optional; defaults to the machine's FQDN
```

If you use the DNS name `salt` for your master, minions find it automatically with zero config — a common convention.

### 3.2 Open the firewall on the master

The master listens on **TCP 4505** (publish/command bus) and **4506** (return/file server). Allow these from your minions, e.g.:

```bash
sudo ufw allow 4505:4506/tcp
```

Note that ufw requires the /tcp (or /udp) suffix whenever you use a port range with the colon — it won't accept a bare range — so that short form is exactly what you want here.
If you'd rather restrict it to only your minion subnet (better practice than opening it to the world), use a from clause:

```bash
sudo ufw allow from 10.10.80.0/24 to any port 4505:4506 proto tcp
```

Verify it landed:

```bash
sudo ufw status numbered
```

Check firewall status:

```bash
sudo ufw status
```

Enable firewall:

```bash
sudo ufw enable
```

That shows whether the firewall is active and lists every rule (To / Action / From). Two useful variants:

```bash
sudo ufw status numbered   # adds an index number to each rule
sudo ufw status verbose    # adds default policies, logging level, and profile info
```

The numbered view is the one you'll use most, because deleting a rule is done by its number:

```bash
sudo ufw delete 3          # removes rule #3 from the numbered list
```
A couple of things worth knowing:

If ufw status just prints Status: inactive, no rules are being enforced at all (even if you've added some — they're staged but not active until sudo ufw enable).
ufw is only the friendly front-end. If you want to see what's actually loaded in the kernel — including rules from Docker, Salt, or other tools that bypass ufw — look at the underlying tables:

```bash
sudo iptables -L -n -v        # legacy view
sudo nft list ruleset         # nftables (modern Ubuntu default backend)
```

### 3.3 Start and enable the services

```bash
# On the master:
sudo systemctl enable --now salt-master

# On each minion:
sudo systemctl enable --now salt-minion
```

Check Status:

Master:

```text
ekou@saltmaster:~$ sudo systemctl status salt-master
● salt-master.service - The Salt Master Server
     Loaded: loaded (/usr/lib/systemd/system/salt-master.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-07-21 09:38:28 UTC; 34min ago
       Docs: man:salt-master(1)
             file:///usr/share/doc/salt/html/contents.html
             https://docs.saltproject.io/en/latest/contents.html
   Main PID: 821 (/opt/saltstack/)
      Tasks: 33 (limit: 4543)
     Memory: 297.7M (peak: 304.2M)
        CPU: 17.392s
     CGroup: /system.slice/salt-master.service
             ├─ 821 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master MainProcess"
             ├─1253 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master EventPublisher"
             ├─1327 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master PubServerChannel._publish_daemon"
             ├─1330 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master EventMonitor"
             ├─1331 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master Maintenance"
             ├─1332 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master BatchManager"
             ├─1334 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master RequestServer ReqServer_ProcessManager"
             ├─1335 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master RequestServer MWorkerQueue"
             ├─1336 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master FileServerUpdate"
             ├─1376 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master RequestServer MWorker-default-0"
             ├─1377 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master RequestServer MWorker-default-1"
             ├─1378 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master RequestServer MWorker-default-2"
             ├─1383 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master RequestServer MWorker-default-3"
             └─1384 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-master RequestServer MWorker-default-4"

Jul 21 09:38:26 saltmaster systemd[1]: Starting salt-master.service - The Salt Master Server...
Jul 21 09:38:28 saltmaster systemd[1]: Started salt-master.service - The Salt Master Server.
```

On Minion: one thing worth to notice is that the Minion may start automatically after you install it. So the new conf will take effect only after restart the service. 

Verification command:

```bash
sudo salt-call --local config.get master
ekou@ubuntu1:~$ sudo salt-call --local config.get master
local:
    saltmaster
```
Monitor log:

```bash
sudo journalctl -u salt-minion -f
ekou@ubuntu2:~$ sudo journalctl -u salt-minion -f
Jul 21 10:18:37 ubuntu2 salt-minion[2234]: [ERROR   ] DNS lookup or connection check of 'salt' failed.
Jul 21 10:18:37 ubuntu2 salt-minion[2234]: [ERROR   ] Master hostname: 'salt' not found or not responsive. Retrying in 30 seconds
Jul 21 10:19:07 ubuntu2 salt-minion[2234]: [ERROR   ] DNS lookup or connection check of 'salt' failed.
Jul 21 10:19:07 ubuntu2 salt-minion[2234]: [ERROR   ] Master hostname: 'salt' not found or not responsive. Retrying in 30 seconds
Jul 21 10:19:37 ubuntu2 salt-minion[2234]: [ERROR   ] DNS lookup or connection check of 'salt' failed.
Jul 21 10:19:37 ubuntu2 salt-minion[2234]: [ERROR   ] Master hostname: 'salt' not found or not responsive. Retrying in 30 seconds
Jul 21 10:20:07 ubuntu2 salt-minion[2234]: [ERROR   ] DNS lookup or connection check of 'salt' failed.
Jul 21 10:20:07 ubuntu2 salt-minion[2234]: [ERROR   ] Master hostname: 'salt' not found or not responsive. Retrying in 30 seconds
Jul 21 10:20:37 ubuntu2 salt-minion[2234]: [ERROR   ] DNS lookup or connection check of 'salt' failed.
Jul 21 10:20:37 ubuntu2 salt-minion[2234]: [ERROR   ] Master hostname: 'salt' not found or not responsive. Retrying in 30 seconds
  q^C
ekou@ubuntu2:~$ 
```
Verify keys on salt master: 

```bash
ekou@saltmaster:~$ sudo salt-key -L
Accepted Keys:
Denied Keys:
Unaccepted Keys:
skou_test
Rejected Keys:
```
> **Note:** deleting the key here is purely for testing — normally you would simply accept it, since it is sitting in the Unaccepted list.
Delete a key:
```bash

ekou@saltmaster:~$ sudo salt-key -d skou_test
The following keys are going to be deleted:
Unaccepted Keys:
skou_test
Proceed? [N/y] y
Key for minion skou_test deleted.
```

> **Note:** take note of how the minion ID is determined (here `id:` is set in the config file, so that value wins — the full logic is below):
```text
id: in the config — if present, always wins (what you're doing now).
/etc/salt/minion_id — a cache file. Once the ID is determined the first time, Salt writes it here and reuses it forever after.
Auto-detected FQDN — if neither of the above exists, Salt computes the ID from the machine's fully-qualified domain name (essentially socket.getfqdn()), deliberately avoiding localhost.

So the auto value is the FQDN when the box has a domain — e.g. ubuntu1.example.com. If there's no domain (which is my case — my /etc/hosts only has 127.0.1.1 ubuntu1 with no domain suffix), the FQDN collapses to just the short hostname, so the ID would come out as ubuntu1. If it can't resolve any usable name at all, it falls back to an IP address as a last resort.
Two things that trip people up here:
The /etc/salt/minion_id cache means the ID sticks. If you let it auto-detect as ubuntu1, then later rename the host to web01, the minion will still report as ubuntu1 — because it reads the cached file, not the current hostname. To force re-detection you delete that file (and the minion's key), then restart:

```
to remove the id cache:

```bash
sudo rm /etc/salt/minion_id
sudo systemctl restart salt-minion
```
And because the ID is what the key and all your targeting/states are tied to, changing it later is disruptive — the minion presents a new ID, generates/uses a key under that name, and you'd re-accept it on the master (salt-key) and update any state/pillar top-file matches. That's exactly why setting id: explicitly, like you did with skou_test, is the cleaner approach: it's stable and predictable rather than dependent on whatever DNS/hostname happens to resolve to at first boot.



Restart service on Minion
```text
sudo systemctl restart salt-minion

ekou@ubuntu1:~$ sudo systemctl restart salt-minion
ekou@ubuntu1:~$ sudo systemctl status salt-minion
● salt-minion.service - The Salt Minion
     Loaded: loaded (/usr/lib/systemd/system/salt-minion.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-07-21 10:29:53 UTC; 1min 8s ago
       Docs: man:salt-minion(1)
             file:///usr/share/doc/salt/html/contents.html
             https://docs.saltproject.io/en/latest/contents.html
   Main PID: 3101 (python3.14)
      Tasks: 7 (limit: 4543)
     Memory: 88.7M (peak: 92.3M)
        CPU: 2.198s
     CGroup: /system.slice/salt-minion.service
             ├─3101 /opt/saltstack/salt/bin/python3.14 /usr/bin/salt-minion
             └─3109 "/opt/saltstack/salt/bin/python3.14 /usr/bin/salt-minion MultiMinionProcessManager MinionProcessManager"

Jul 21 10:29:53 ubuntu1 systemd[1]: Starting salt-minion.service - The Salt Minion...
Jul 21 10:29:53 ubuntu1 systemd[1]: Started salt-minion.service - The Salt Minion.
```


---

## 4. Key management (the trust handshake)

Salt is secure by default: a minion generates a key on first start and the master must **accept** it before any command will reach that minion.

```bash
# On the master, list pending/accepted keys:
sudo salt-key -L

# Accept one minion by ID:
sudo salt-key -a web01.example.com

# Accept everything currently pending (fine for a lab, cautious in prod):
sudo salt-key -A

# Reject or delete a key:
sudo salt-key -r <id>
sudo salt-key -d <id>
```

For production, verify the key fingerprint matches on both ends (`salt-key -f <id>` on the master, `salt-call --local key.finger` on the minion) before accepting.

---

## 5. First contact — execution modules

Once keys are accepted, test connectivity:

```bash
# Ping all minions (this is Salt's test.ping, not ICMP):
sudo salt '*' test.ping

# Run a shell command everywhere:
sudo salt '*' cmd.run 'uptime'

# Gather system facts:
sudo salt '*' grains.items
sudo salt '*' grains.get os

# Package + service management:
sudo salt 'web*' pkg.install nginx
sudo salt 'web*' service.restart nginx
sudo salt '*' disk.usage
```

Test output:

```text

ekou@saltmaster:~$ sudo salt '*' test.ping
skou_test1:
    True
skou_test:
    True
salt_master:
    True
ekou@saltmaster:~$ sudo salt '*' cmd.run 'uptime'
salt_master:
     10:39:51 up  1:01,  3 users,  load average: 0.18, 0.05, 0.02
skou_test1:
     10:39:51 up  1:04,  2 users,  load average: 0.41, 0.52, 0.30
skou_test:
     10:39:51 up  1:04,  2 users,  load average: 0.01, 0.02, 0.06

ekou@saltmaster:~$ sudo salt '*' grains.get os
skou_test1:
    Ubuntu
skou_test:
    Ubuntu
salt_master:
    Ubuntu

ekou@saltmaster:~$ sudo salt -E 'skou_[a-z]+'  grains.get os
skou_test1:
    Ubuntu
skou_test:
    Ubuntu
ekou@saltmaster:~$ sudo salt -E 'skou_[a-z]+$' grains.get os
skou_test:
    Ubuntu
ekou@saltmaster:~$ sudo salt '*' cmd.run 'ip a'
[sudo] password for ekou: 
skou_test:
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host noprefixroute 
           valid_lft forever preferred_lft forever
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 00:0c:29:a0:4a:ce brd ff:ff:ff:ff:ff:ff
        altname enp2s1
        inet 10.10.80.91/24 brd 10.10.80.255 scope global ens33
           valid_lft forever preferred_lft forever
        inet6 fe80::20c:29ff:fea0:4ace/64 scope link 
           valid_lft forever preferred_lft forever
salt_master:
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host noprefixroute 
           valid_lft forever preferred_lft forever
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 00:0c:29:d1:fe:ea brd ff:ff:ff:ff:ff:ff
        altname enp2s1
        inet 10.10.80.99/24 brd 10.10.80.255 scope global ens33
           valid_lft forever preferred_lft forever
        inet6 fe80::20c:29ff:fed1:feea/64 scope link 
           valid_lft forever preferred_lft forever
skou_test1:
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host noprefixroute 
           valid_lft forever preferred_lft forever
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 00:0c:29:1f:8a:61 brd ff:ff:ff:ff:ff:ff
        altname enp2s1
        inet 10.10.80.92/24 brd 10.10.80.255 scope global ens33
           valid_lft forever preferred_lft forever
        inet6 fe80::20c:29ff:fe1f:8a61/64 scope link 
           valid_lft forever preferred_lft forever
ekou@saltmaster:~$ sudo salt '*' cmd.run 'netstat -rn'
skou_test:
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    0.0.0.0         10.10.80.2      0.0.0.0         UG        0 0          0 ens33
    10.10.80.0      0.0.0.0         255.255.255.0   U         0 0          0 ens33
skou_test1:
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    0.0.0.0         10.10.80.2      0.0.0.0         UG        0 0          0 ens33
    10.10.80.0      0.0.0.0         255.255.255.0   U         0 0          0 ens33
salt_master:
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    0.0.0.0         10.10.80.2      0.0.0.0         UG        0 0          0 ens33
    10.10.80.0      0.0.0.0         255.255.255.0   U         0 0          0 ens33
```

### Use Regex to filter devices:

Salt targets by minion ID with regex using the -E (or --pcre) flag. The pattern is standard Python/PCRE regex, matched against each minion's ID:

```bash
sudo salt -E 'skou_.*' test.ping          # skou_test and any skou_ siblings
sudo salt -E 'skou_[a-z]+$' grains.get os
sudo salt -E 'web.*' test.ping           # any ID starting with "web"
sudo salt -E 'web0[1-3]' test.ping        # web01, web02, web03
sudo salt -E '(web|db)\d+' test.ping      # web or db followed by digits
sudo salt -E '.*\.example\.com$' test.ping  # IDs ending in .example.com
```

Salt's ID regex is anchored at the start of the ID but **not** at the end. So `web` matches `web01`, `web-prod`, and `website` alike. If you need an exact end, anchor with `$` — e.g. `-E 'web01$'` to match only `web01` and not `web011`.

To target by a grain with regex instead of the ID, use `-P` (grain PCRE):

```bash
sudo salt -P 'os:Ubuntu.*' test.ping           # os grain matching regex
sudo salt -P 'kernelrelease:5\.15.*' test.ping  # kernel version pattern
```

Inside a top file, set the match type to pcre (the top file itself is explained in detail in section 7.2):

```yaml
base:
  'web0[1-9]':
    - match: pcre
    - nginx
```
And in compound targeting (mixing matchers with and/or/not), regex uses the E@ prefix for IDs and P@ for grains:

```bash
sudo salt -C 'E@web.* and G@os:Ubuntu' test.ping
sudo salt -C 'E@db.* and not E@db99' test.ping
```

Command anatomy: `salt <target> <module.function> [arguments]`.

### Masterless equivalent (no master needed)

Run the same functions locally on a single box:

```bash
sudo salt-call --local test.ping
sudo salt-call --local pkg.install nginx
```

---

## 6. Targeting minions

The `<target>` field selects which minions run the command:

```bash
sudo salt '*' test.ping                       # all minions (glob)
sudo salt 'web*' test.ping                     # glob on minion ID
sudo salt -E 'web(1|2)\.example\.com' test.ping  # regex (-E)
sudo salt -L 'web01,db01' test.ping            # explicit list (-L)
sudo salt -G 'os:Ubuntu' test.ping             # by grain (-G)
sudo salt -I 'role:webserver' test.ping        # by pillar (-I)
sudo salt -C 'web* and G@os:Ubuntu' test.ping  # compound (-C)
```

---

## 7. States — declarative configuration

States describe the desired end state and are stored on the master, by default under `/srv/salt/`.

### 7.1 A simple state file

Create `/srv/salt/nginx.sls`:

```yaml
install_nginx:
  pkg.installed:
    - name: nginx

nginx_running:
  service.running:
    - name: nginx
    - enable: True
    - require:
      - pkg: install_nginx      # ordering: install before starting
```

Apply it directly to a target:

```bash
sudo salt 'web*' state.apply nginx
```

### 7.2 The top file — mapping states to minions

#### What the top file is for

So far every command targeted minions ad hoc, one command at a time — `salt -E 'skou_[a-z]+$' grains.get os`. The top file is how you make those assignments **permanent and declarative** instead. It is a mapping that says "these minions should always have these states applied to them." When you run a highstate, Salt reads the top file, works out which states each minion is assigned, and applies them.

#### Where it lives

On the master, at the root of your state directory:

```text
/srv/salt/top.sls
```

(`/srv/salt/` is the default `file_roots` — the same place your `.sls` state files live. The pillar system has its own separate top file at `/srv/pillar/top.sls`, covered in Section 8.)

To verify where your server actualy uses: 

```bash

ekou@saltmaster:~$ sudo salt-run config.get file_roots
base:
    - /srv/salt
    - /srv/spm/salt
```

#### Its structure

Three levels of indentation, each meaning something specific:

```yaml
base:                          # 1. the environment (usually "base")
  '<target>':                  # 2. which minions
    - <state to apply>         # 3. what to apply to them
```

By default the `<target>` is interpreted as a **glob** (shell-style wildcards like `web*`). To make Salt read it as a regex instead, you add one special line — `- match: pcre` — as the first item in the list under that target. That line isn't a state; it's an instruction telling Salt how to interpret the target string above it. The same mechanism selects any other matcher: `- match: grain`, `- match: list`, `- match: compound`, and so on.

#### A concrete example

Say you have a state file `/srv/salt/ntp.sls` and you want it applied to `skou_test` but **not** `skou_test1` — the exact case solved on the CLI in Section 5 with `-E 'skou_[a-z]+$'`. Your `/srv/salt/top.sls` would be:

```yaml
base:
  'skou_[a-z]+$':
    - match: pcre
    - ntp
```

Reading that top-to-bottom: in the `base` environment, for every minion whose ID matches the regex `skou_[a-z]+$` (interpret it as PCRE, not a glob), apply the state called `ntp`. Because of the `$` anchor, `skou_test` matches and `skou_test1` doesn't — same logic as the command line.

You can stack multiple target blocks, mixing match types freely. A minion that matches several blocks gets the union of all of them:

```yaml
base:
  '*':                         # glob (default) — everyone
    - common
  'skou_[a-z]+$':              # regex — only skou_test
    - match: pcre
    - ntp
  'web0[1-9]':                 # regex — web01 through web09
    - match: pcre
    - nginx
  'os:Ubuntu':                 # grain match
    - match: grain
    - ubuntu_tweaks
```

One practical note: only put `- match: pcre` under targets that are actually regex. A plain `'*'` or `'web*'` should stay as glob (no match line), since those are wildcard patterns, not regex.

#### What `- common`, `- ntp`, `- nginx` actually are

Those entries are the **states to apply** — each one is the name of a `.sls` file containing the actual configuration instructions. In the top file you're just referencing them by name; the real content lives in separate files. Salt resolves each name to a file under `/srv/salt/` in one of two ways:

```text
- common   →  /srv/salt/common.sls
              (or /srv/salt/common/init.sls  if you organize it as a folder)
- ntp      →  /srv/salt/ntp.sls
- nginx    →  /srv/salt/nginx.sls
```

So `- nginx` means "apply everything defined in `/srv/salt/nginx.sls` to the matched minions." The dash makes it a YAML list item, because a minion can be assigned several states at once:

```yaml
  'web0[1-9]':
    - match: pcre
    - common      # apply common.sls
    - ntp         # AND ntp.sls
    - nginx       # AND nginx.sls
```

The names are just labels you choose — `common`, `ntp`, `nginx` aren't built-in keywords; they're whatever you named your files (the `nginx.sls` from Section 7.1 is exactly such a file). The convention is that `common` is the state you apply to everything (base packages, users, standard config), while `ntp` and `nginx` are role-specific — only certain minions need them. That's exactly why the top file exists: it's the map that decides which of those state files lands on which minions.

#### The full picture, end to end

```yaml
# /srv/salt/top.sls  — the MAP (who gets what)
base:
  '*':
    - common          # every minion gets common.sls
  'web0[1-9]':
    - match: pcre
    - nginx           # only web minions also get nginx.sls
```

```yaml
# /srv/salt/common.sls  — the actual work for "common"
install_vim:
  pkg.installed:
    - name: vim
```

```yaml
# /srv/salt/nginx.sls  — the actual work for "nginx"
install_nginx:
  pkg.installed:
    - name: nginx
```

When you run a highstate, Salt reads `top.sls`, sees that every minion gets `common` and web minions additionally get `nginx`, then opens those `.sls` files and enforces what's inside them.

#### Running it — the highstate

Once the top file is saved on the master, trigger it with a highstate. This tells each targeted minion to apply everything the top file assigns to it:

```bash
sudo salt '*' state.highstate                 # apply to all minions
sudo salt '*' state.apply                     # equivalent: state.apply with NO state name runs the highstate
sudo salt '*' state.highstate test=True       # DRY RUN first — see what would change
sudo salt skou_test state.highstate           # just one minion
```

Note the distinction: `state.apply nginx` applies one named state directly, ignoring the top file's assignments; `state.apply` with no argument is the highstate.

A subtlety that trips people up: **the CLI target and the top-file target are two different layers, and they compose.** The CLI target answers *"who should run a highstate right now?"*; the top file answers *"what does each minion get when it runs one?"*. `state.highstate` never means "apply the top file's states to whatever I typed" — each targeted minion consults the top file and applies only what matches **its own** ID. So `sudo salt '*' state.highstate` is always safe in the sense that minions matched nowhere in the top file simply report *"No Top file or master_tops data matches found"* and change nothing. The reason to narrow the CLI target anyway is blast radius: in a large fleet you don't want every minion re-converging just because you are iterating on one box. In fully automated setups (a scheduled highstate, or `startup_states: highstate` in the minion config) there is no CLI target at all — the top file alone decides everything.

#### The mental model: `-E` vs `match: pcre`

The CLI `-E` flag and the `- match: pcre` line do the **exact same regex matching** — they're just used in two different places. `-E` is for one-off ad-hoc commands you type; `match: pcre` is for making that same targeting a saved, repeatable rule inside the top file. The reason the top file needs the explicit `match:` line is that, unlike the command line where you chose the `-E` flag, the top file defaults to glob matching — so you have to tell it when a target should be treated as a regex.

### 7.3 Test before you commit

Always dry-run with `test=True` — it reports what *would* change without changing anything:

```bash
sudo salt 'web*' state.apply nginx test=True
```

### 7.4 Worked example — install MySQL on skou_test via the top file

Everything from 7.1 to 7.3 in one small, real exercise: make `skou_test` a MySQL server, declaratively.

**Step 1 — the state file.** On the master, create `/srv/salt/mysql.sls`:

```yaml
install_mysql:
  pkg.installed:
    - name: mysql-server

mysql_running:
  service.running:
    - name: mysql
    - enable: True
    - require:
      - pkg: install_mysql
```

Same shape as the nginx state in 7.1: install the package, then keep the service running and enabled at boot, with `require` guaranteeing the install happens first. On Ubuntu the package `mysql-server` provides a service named `mysql`; on Debian, which ships MariaDB instead, you would use `mariadb-server` and service `mariadb`.

**Step 2 — the top file.** Add the assignment to `/srv/salt/top.sls`, reusing the regex targeting from 7.2:

```yaml
base:
  'skou_[a-z]+$':
    - match: pcre
    - mysql
```

The `- match: pcre` line tells Salt to treat the target as a regex, and the `$` anchor makes `skou_[a-z]+$` match `skou_test` but **not** `skou_test1` — exactly the selection built on the CLI in Section 5. If this target block already exists in your top file (e.g. with `- ntp` from 7.2), just append `- mysql` to its list rather than creating a duplicate block. The simpler alternative — targeting the literal ID `'skou_test'` with no `match:` line, since a glob without wildcards matches exactly one minion — also works; the regex form is used here to practice the pattern you'll actually need once minions multiply.

**Step 3 — dry-run, apply, verify:**

```bash
sudo salt skou_test state.highstate test=True   # dry run: both IDs show as "would be installed/started"
sudo salt skou_test state.highstate             # actually apply
```

Why name `skou_test` on the CLI when the top file already targets it? Because these are two different layers (see the note at the end of 7.2): the CLI ID only scopes *who runs a highstate now*; the top file still decides *what they get*. `sudo salt '*' state.highstate` would produce the identical result on `skou_test` — with every unmatched minion reporting "no matches found" — but narrowing the CLI target keeps the run fast and quiet while you iterate on one box.

Test output:
```text
ekou@saltmaster:~$ sudo salt '*' state.highstate
skou_test1:
----------
          ID: states
    Function: no.None
      Result: False
     Comment: No Top file or master_tops data matches found. Please see master log for details.
     Changes:   

Summary for skou_test1
------------
Succeeded: 0
Failed:    1
------------
Total states run:     1
Total run time:   0.000 ms
salt_master:
----------
          ID: states
    Function: no.None
      Result: False
     Comment: No Top file or master_tops data matches found. Please see master log for details.
     Changes:   

Summary for salt_master
------------
Succeeded: 0
Failed:    1
------------
Total states run:     1
Total run time:   0.000 ms
skou_test:
----------
          ID: install_mysql
    Function: pkg.installed
        Name: mysql-server
      Result: True
     Comment: All specified packages are already installed
     Started: 12:26:46.271866
    Duration: 44.855 ms
     Changes:   
----------
          ID: mysql_running
    Function: service.running
        Name: mysql
      Result: True
     Comment: The service mysql is already running
     Started: 12:26:46.362049
    Duration: 129.403 ms
     Changes:   

Summary for skou_test
------------
Succeeded: 2
Failed:    0
------------
Total states run:     2
Total run time: 174.258 ms
ERROR: Minions returned with non-zero exit code
```

The summary at the bottom should report `Succeeded: 2 (changed=2)` on the first run. Re-run the same highstate and it reports `Succeeded: 2` with no changes — that's idempotence, the whole point of states: Salt enforces the end state rather than re-executing installation commands.

Confirm from the master:

```bash
sudo salt skou_test pkg.version mysql-server    # e.g. 8.0.x
sudo salt skou_test service.status mysql        # True
sudo salt skou_test cmd.run 'mysql --version'
```

The full first-run capture from this lab — the top file and state exactly as created, the dry run, the real install (42 s; the apt dependency list is trimmed for readability), and the verification:

```text
ekou@saltmaster:~$ cat /srv/salt/top.sls 
base:
  'skou_[a-z]+$':
    - match: pcre
    - mysql
ekou@saltmaster:~$ cat /srv/salt/mysql.sls
install_mysql:
  pkg.installed:
    - name: mysql-server

mysql_running:
  service.running:
    - name: mysql
    - enable: True
    - require:
      - pkg: install_mysql
ekou@saltmaster:~$ sudo salt skou_test state.highstate test=True
skou_test:
----------
          ID: install_mysql
    Function: pkg.installed
        Name: mysql-server
      Result: None
     Comment: The following packages would be installed/updated: mysql-server
     Started: 12:12:12.699933
    Duration: 3171.076 ms
     Changes:   
              ----------
              mysql-server:
                  ----------
                  new:
                      8.0.46-0ubuntu0.24.04.3
                  old:
----------
          ID: mysql_running
    Function: service.running
        Name: mysql
      Result: None
     Comment: Service mysql not present; if created in this state run, it would have been started
     Started: 12:12:15.892350
    Duration: 40.825 ms
     Changes:   

Summary for skou_test
------------
Succeeded: 2 (unchanged=2, changed=1)
Failed:    0
------------
Total states run:     2
Total run time:   3.212 s
ekou@saltmaster:~$ sudo salt skou_test state.highstate
skou_test:
----------
          ID: install_mysql
    Function: pkg.installed
        Name: mysql-server
      Result: True
     Comment: The following packages were installed/updated: mysql-server
     Started: 12:12:26.515181
    Duration: 42540.467 ms
     Changes:   
              ----------
              ...(22 non-MySQL dependency packages trimmed)...
              mysql-client-8.0:
                  ----------
                  new:
                      8.0.46-0ubuntu0.24.04.3
                  old:
              mysql-client-core-8.0:
                  ----------
                  new:
                      8.0.46-0ubuntu0.24.04.3
                  old:
              mysql-common:
                  ----------
                  new:
                      5.8+1.1.0build1
                  old:
              mysql-server:
                  ----------
                  new:
                      8.0.46-0ubuntu0.24.04.3
                  old:
              mysql-server-8.0:
                  ----------
                  new:
                      8.0.46-0ubuntu0.24.04.3
                  old:
              mysql-server-core-8.0:
                  ----------
                  new:
                      8.0.46-0ubuntu0.24.04.3
                  old:
----------
          ID: mysql_running
    Function: service.running
        Name: mysql
      Result: True
     Comment: The service mysql is already running
     Started: 12:13:09.073003
    Duration: 49.261 ms
     Changes:   

Summary for skou_test
------------
Succeeded: 2 (changed=1)
Failed:    0
------------
Total states run:     2
Total run time:  42.590 s
ekou@saltmaster:~$ sudo salt skou_test pkg.version mysql-server 
skou_test:
    8.0.46-0ubuntu0.24.04.3
ekou@saltmaster:~$ sudo salt skou_test service.status mysql 
skou_test:
    True
ekou@saltmaster:~$ sudo salt skou_test cmd.run 'mysql --version'
skou_test:
    mysql  Ver 8.0.46-0ubuntu0.24.04.3 for Linux on x86_64 ((Ubuntu))
ekou@saltmaster:~$ 
```

Because the assignment lives in the top file, it is now permanent: every future highstate re-asserts that `skou_test` has MySQL installed and running — if someone stops the service or removes the package, the next highstate puts it back.

### 7.5 Worked example continued — database, user and password via pillar

A real MySQL deployment doesn't stop at "installed and running": it needs a database, a table, an application user, and a password — and the password must **not** be hardcoded in the state file, because states are not secret (any minion assigned the state can render it, and it usually ends up in git). This is exactly the split Salt is designed around: the *what* lives in the state, the *secrets* live in pillar (introduced formally in Section 8 — peek ahead if needed).

**Step 1 — pillar data.** Pillar has its own top file, with the same matching rules as the state top file. On the master, create `/srv/pillar/top.sls`:

```yaml
base:
  'skou_[a-z]+$':
    - match: pcre
    - mysql
```

And `/srv/pillar/mysql.sls`:

```yaml
# Connection default for Salt's mysql modules: talk to MySQL over the unix
# socket, where Ubuntu's auth_socket lets root in without a password.
mysql.unix_socket: /var/run/mysqld/mysqld.sock

# Application database definition — names plus the actual secret.
appdb:
  name: ekou_test_db
  table: ekou_test_table
  user: ekoumysql
  password: Cisco12345
```

Only minions matched in the pillar top file ever see this data — that per-minion scoping is the point of pillar. (It is still plaintext on the master; for production-grade secrets Salt supports encrypting pillar values with the GPG renderer.)

**Step 2 — extend the state.** Four Ubuntu/Salt realities shape it:

- Salt's `mysql_*` modules need the **PyMySQL** Python library — and modern Salt is *onedir*, bundling its own Python, so installing `python3-pymysql` with apt is invisible to Salt. It has to go into Salt's own Python via `salt-pip`.
- **On Salt 3008, the `mysql_*` state modules are no longer in core.** Salt's "great module migration" moved them to the [saltext-mysql](https://pypi.org/project/saltext-mysql/) extension — core 3008 still ships the *execution* module (`mysql.db_list` works), but `mysql_database.present` and friends do not exist until the extension is installed. Section 7.6 shows how this was discovered the hard way.
- A fresh Ubuntu `mysql-server` sets root to `auth_socket`: no password, authenticated by the OS user over the unix socket. Since `salt-minion` runs as root, Salt can administer MySQL through the socket with no credentials — which is what makes the first run work at all.
- Salt has states for databases, users and grants, but **no state for tables** — schema normally belongs to the application (migrations), not the config-management layer. For a lab table, `mysql_query.run` executes raw SQL, with an `unless` guard so it stays idempotent.

Replace `/srv/salt/mysql.sls` with:

```yaml
{% set db = pillar['appdb'] %}

install_mysql:
  pkg.installed:
    - name: mysql-server

mysql_running:
  service.running:
    - name: mysql
    - enable: True
    - require:
      - pkg: install_mysql

# Salt is onedir: both libraries must go into Salt's bundled Python, not
# the system Python. pymysql is the MySQL client; saltext-mysql provides
# the mysql_* STATE modules that Salt 3008 removed from core.
salt_mysql_deps:
  cmd.run:
    - name: salt-pip install pymysql saltext-mysql
    - unless: /opt/saltstack/salt/bin/python3 -c "import pymysql, saltext.mysql"
    - reload_modules: True
    - require:
      - service: mysql_running

appdb_database:
  mysql_database.present:
    - name: {{ db.name }}
    - require:
      - cmd: salt_mysql_deps

appdb_user:
  mysql_user.present:
    - name: {{ db.user }}
    - host: localhost
    - password: '{{ db.password }}'
    - require:
      - mysql_database: appdb_database

appdb_grants:
  mysql_grants.present:
    - grant: ALL PRIVILEGES
    - database: {{ db.name }}.*
    - user: {{ db.user }}
    - host: localhost
    - require:
      - mysql_user: appdb_user

# No mysql_table state exists - create the table with raw SQL. The unless
# guard keeps the state idempotent: once the table exists, nothing runs.
appdb_table:
  mysql_query.run:
    - database: {{ db.name }}
    - query: |
        CREATE TABLE IF NOT EXISTS {{ db.table }} (
          id INT AUTO_INCREMENT PRIMARY KEY,
          name VARCHAR(64) NOT NULL,
          created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    - unless: mysql -N -e "SHOW TABLES IN {{ db.name }} LIKE '{{ db.table }}'" | grep -q {{ db.table }}
    - require:
      - mysql_database: appdb_database
```

The `{{ ... }}` expressions are Jinja (Section 9): the state file is a *template* that pulls its values from pillar at render time, so the same state serves any minion — each renders with its own pillar data, and no secret ever appears in the state file. With the pillar above, this run produces database `ekou_test_db` containing table `ekou_test_table`, owned (via grants) by `appuser`.

**Step 3 — refresh pillar, then apply:**

```bash
sudo salt skou_test saltutil.refresh_pillar          # push the new pillar data
sudo salt skou_test pillar.items                     # confirm appdb is visible
sudo salt skou_test state.highstate test=True
sudo salt skou_test state.highstate
```

Expect the **first converge to take two runs plus a restart**, and don't let the intermediate errors scare you:

- The `test=True` dry run reports `'mysql_database.present' is not available` with a cascade of requisite failures. Expected: test mode doesn't actually run the `salt-pip` bootstrap, so the modules it would provide can't exist yet.
- The first real run installs `pymysql` and `saltext-mysql`, but the `mysql_*` states still fail with "not available" **in that same run** — newly installed state modules don't become loadable mid-highstate, even with `reload_modules`.
- Restart the minion so it builds its loader in a world where the packages exist, then run again:

```bash
sudo salt skou_test service.restart salt-minion      # "Minion did not return" is expected
sudo salt skou_test test.ping                        # wait a few seconds, confirm it is back
sudo salt skou_test state.highstate                  # converges: Succeeded: 7 (changed=4)
```

Every run after that is a clean no-op single pass. Section 7.6 walks through how each of those first-run errors was diagnosed in this lab.

**Troubleshooting: `pillar.items` comes back empty.** This is what it looks like when it goes wrong — a real capture from this lab:

```text
ekou@saltmaster:~$ sudo salt skou_test saltutil.refresh_pillar
skou_test:
    True
ekou@saltmaster:~$ sudo salt skou_test pillar.items
skou_test:
    ----------
```

That bare `----------` is **not** normal: it means the master rendered *zero* pillar data for this minion. The `True` from `refresh_pillar` is misleading — it only confirms the minion asked for a refresh, not that anything matched. With the files from Step 1 in place you should see `mysql.unix_socket` and the whole `appdb` block here.

The most common cause is the pillar top file: either `/srv/pillar/top.sls` doesn't exist, or the assignment was added to `/srv/salt/top.sls` instead — the classic mix-up, because pillar has its **own** top file and a `mysql.sls` sitting in `/srv/pillar/` is assigned to nobody until the *pillar* top file maps it. Runner-up causes: a tab instead of spaces in the YAML (fails silently as a no-match), a misspelled filename, or a changed `pillar_roots`.

Diagnose from the master side before touching the minion:

```bash
sudo salt-run pillar.show_top minion=skou_test   # what does the pillar top file assign?
sudo salt-run pillar.show_pillar skou_test       # what does it render to?
```

If `pillar.show_top` is empty, fix the pillar top file; if it shows `- mysql` but the minion still gets nothing, re-run `saltutil.refresh_pillar` and check the minion again. The full set of pillar-debugging commands, and how they mirror the state-side ones, is in Section 8.1.

**Step 4 — verify:**

```bash
sudo salt skou_test mysql.db_list                    # ekou_test_db listed
sudo salt skou_test mysql.db_tables ekou_test_db     # ekou_test_table listed
sudo salt skou_test mysql.user_list                  # appuser@localhost listed
sudo salt skou_test cmd.run "mysql -u appuser -pMySecret123 -e 'SHOW TABLES IN ekou_test_db;'"
```

The last command proves the whole chain end to end: the password that only exists in pillar on the master now authenticates a real client on the minion, and that client can see the table the state created.

**Why this example does not set the root password.** It could — `mysql_user.present` for `root@localhost` with a pillar password — but think through the second run: once root has a password, every later Salt connection needs it too (`mysql.pass` in pillar), and if the state half-applies you can lock Salt out of its own database. On Ubuntu the practical convention is to leave root on `auth_socket` (root shell access already implies database admin) and manage application users the pillar way shown here. If you do password root, set `mysql.pass` in the same pillar *before* the run that changes it.

### 7.6 Troubleshooting the first 7.5 run — a real debugging walkthrough

The first time 7.5 was applied in this lab, it failed — repeatedly, with the same error — and the diagnosis took five distinct stages. They are reproduced here with the real outputs, because the *method* transfers to any `State 'X' was not found in SLS` error, and because the root cause (a Salt 3008 packaging change) will hit anyone following this guide on the current LTS.

**Stage 1 — the dry run fails, and that's expected.** The first `state.highstate test=True` reported:

```text
          ID: appdb_database
    Function: mysql_database.present
        Name: ekou_test_db
      Result: False
     Comment: State 'mysql_database.present' was not found in SLS 'mysql'
              Reason: 'mysql_database.present' is not available.
```

with the user, grants and table states failing as `One or more requisite failed` dominoes. This is a test-mode artifact: `test=True` never actually ran the `salt-pip` bootstrap (its `Result: None` / "would have been executed"), so the modules that install provides can't exist yet. A dry run cannot fully validate a state that bootstraps its own dependencies.

**Stage 2 — the real run installs the library, and *still* fails.** The first real highstate showed the bootstrap genuinely succeed…

```text
ekou@saltmaster:~$ sudo salt skou_test state.highstate


skou_test:
----------
          ID: install_mysql
    Function: pkg.installed
        Name: mysql-server
      Result: True
     Comment: All specified packages are already installed
     Started: 12:50:41.326848
    Duration: 38.183 ms
     Changes:   
----------
          ID: mysql_running
    Function: service.running
        Name: mysql
      Result: True
     Comment: The service mysql is already running
     Started: 12:50:41.385162
    Duration: 107.177 ms
     Changes:   
----------
          ID: salt_pymysql
    Function: cmd.run
        Name: salt-pip install pymysql
      Result: True
     Comment: unless condition is true
     Started: 12:50:41.498519
    Duration: 2175.542 ms
     Changes:   
----------
          ID: appdb_database
    Function: mysql_database.present
        Name: ekou_test_db
      Result: False
     Comment: State 'mysql_database.present' was not found in SLS 'mysql'
              Reason: 'mysql_database.present' is not available.
     Changes:   
----------
          ID: appdb_user
    Function: mysql_user.present
        Name: ekoumysql
      Result: False
     Comment: One or more requisite failed: mysql.appdb_database
     Started: 12:50:43.687603
    Duration: 0.021 ms
     Changes:   
----------
          ID: appdb_grants
    Function: mysql_grants.present
      Result: False
     Comment: One or more requisite failed: mysql.appdb_user
     Started: 12:50:43.691035
    Duration: 0.014 ms
     Changes:   
----------
          ID: appdb_table
    Function: mysql_query.run
      Result: False
     Comment: One or more requisite failed: mysql.appdb_database
     Started: 12:50:43.693716
    Duration: 0.011 ms
     Changes:   

Summary for skou_test
------------
Succeeded: 3
Failed:    4
------------
Total states run:     7
Total run time:   2.321 s
ERROR: Minions returned with non-zero exit code
```

…followed by the *same* `not available` failure in the same run. Newly installed modules did not become loadable mid-highstate, despite `reload_modules: True`. And a second run was no better — this time `salt_pymysql` reported `unless condition is true` (proving PyMySQL was importable) while `mysql_database.present` stayed unavailable. Package present, loader blind.

**Stage 3 — probe the loader directly, and eliminate the cache theory.** Instead of re-running the whole highstate, ask the minion what its loader is offering. An empty reply under the minion ID means "that state module does not exist as far as my loader is concerned":

```text
ekou@saltmaster:~$ sudo salt skou_test sys.list_state_functions mysql_database
skou_test:
ekou@saltmaster:~$
```

The obvious suspect was the long-running minion daemon caching a stale module list. So: restart the daemon on the minion box and probe again —

```text
ekou@ubuntu1:~$ sudo systemctl restart salt-minion
ekou@ubuntu1:~$ sudo systemctl status salt-minion
● salt-minion.service - The Salt Minion
     Loaded: loaded (/usr/lib/systemd/system/salt-minion.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-07-21 13:00:15 UTC; 1s ago
```

```text
ekou@saltmaster:~$ sudo salt skou_test sys.list_state_functions mysql_database
skou_test:
ekou@saltmaster:~$
```

Empty before, empty after a fresh daemon. Whatever this was, it was not a cache.

**Stage 4 — fresh-process test, and a misleading error worth understanding.** `salt-call --local` builds a brand-new loader in a brand-new process, bypassing the daemon entirely. Two lessons came out of it. First, run it **on the right machine** — running it on the master probes the *master's* local minion, where PyMySQL was never installed:

```text
ekou@saltmaster:~$ sudo salt-call --local mysql.db_list -l debug 2>&1 | grep -i mysql
'mysql' __virtual__ returned False: No python mysql client installed.   <-- the MASTER, expected
```

On the actual minion, the module loaded fine — and promptly failed with a *different* error:

```text
ekou@ubuntu1:~$ sudo salt-call --local mysql.db_list -l debug 2>&1 | grep -iE 'mysql|virtual'
[DEBUG   ] LazyLoaded mysql.db_list
[ERROR   ] MySQL Error 1698: Access denied for user 'root'@'localhost'
```

That 1698 is an artifact of the diagnostic itself, not a real problem: `--local` means masterless, so **no pillar** was delivered, so `mysql.unix_socket` was absent, so PyMySQL connected over TCP — and `auth_socket` cannot authenticate root over TCP (no OS peer credentials to check). Confirmation came from running the same probe through the master, pillar included:

```text
ekou@saltmaster:~$ sudo salt skou_test mysql.db_list
skou_test:
    - information_schema
    - mysql
    - performance_schema
    - sys
```

**Stage 5 — the contradiction that named the culprit.** At this point the evidence was: execution module loads and connects everywhere (`mysql.db_list` works), PyMySQL is confirmed in the right onedir location —

```text
ekou@saltmaster:~$ sudo salt skou_test cmd.run '/opt/saltstack/salt/bin/python3 -c "import pymysql; print(pymysql.__file__)"'
skou_test:
    /opt/saltstack/salt/extras-3.14/pymysql/__init__.py
```

— yet the *state* module lists empty even in a fresh process on the minion:

```text
ekou@saltmaster:~$ sudo salt skou_test cmd.run 'salt-call --local sys.list_state_functions mysql_database'
skou_test:
    local:
```

An execution module without its sibling state modules points at packaging, not configuration. Check the disk:

```text
ekou@saltmaster:~$ sudo salt skou_test cmd.run 'ls /opt/saltstack/salt/lib/python3.14/site-packages/salt/states/ | grep -i mysql'
skou_test:
                                     <-- EMPTY: no mysql state files exist
ekou@saltmaster:~$ sudo salt skou_test cmd.run 'ls /opt/saltstack/salt/lib/python3.14/site-packages/salt/modules/ | grep -i mysql'
skou_test:
    mysql.py                         <-- the execution module is there
```

Root cause: **Salt's "great module migration."** Salt 3008 moved the `mysql_*` state modules out of core into the `saltext-mysql` extension, while the execution module survived in core — which is exactly why every "is mysql working?" probe half-succeeded. The fix:

```bash
sudo salt skou_test cmd.run 'salt-pip install saltext-mysql'
sudo salt skou_test service.restart salt-minion
sudo salt skou_test test.ping
```
Command output:

```text

ekou@saltmaster:~$ sudo salt skou_test cmd.run 'salt-pip install saltext-mysql'
skou_test:
    Requirement already satisfied: saltext-mysql in /opt/saltstack/salt/extras-3.14 (1.1.0)
    Requirement already satisfied: salt>=3006 in /opt/saltstack/salt/lib/python3.14/site-packages (from saltext-mysql) (3008.2)
    Requirement already satisfied: sqlparse in /opt/saltstack/salt/extras-3.14 (from saltext-mysql) (0.5.5)
    Requirement already satisfied: aiohappyeyeballs==2.6.1 in /opt/saltstack/salt/lib/python3.14/site-packages (from salt>=3006->saltext-mysql) (2.6.1)
    ....... output ommitted
    Requirement already satisfied: zipp==4.1.0 in /opt/saltstack/salt/lib/python3.14/site-packages (from salt>=3006->saltext-mysql) (4.1.0)
    WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
    
    [notice] A new release of pip is available: 25.2 -> 26.1.2
    [notice] To update, run: /opt/saltstack/salt/bin/python3.14 -m pip install --upgrade pip
ekou@saltmaster:~$ sudo salt skou_test service.restart salt-minion
skou_test:
    True

ekou@saltmaster:~$ sudo salt skou_test test.ping
skou_test:
    True
ekou@saltmaster:~$ sudo salt skou_test sys.list_state_functions mysql_database
skou_test:
    - mysql_database.absent
    - mysql_database.present
```
And the proof, immediately after:

```text

ekou@saltmaster:~$ sudo salt skou_test test.ping
skou_test:
    True
ekou@saltmaster:~$ sudo salt skou_test sys.list_state_functions mysql_database
skou_test:
    - mysql_database.absent
    - mysql_database.present
ekou@saltmaster:~$ sudo salt skou_test state.highstate

skou_test:
----------
          ID: install_mysql
    Function: pkg.installed
        Name: mysql-server
      Result: True
     Comment: All specified packages are already installed
     Started: 13:07:45.122998
    Duration: 35.414 ms
     Changes:   
----------
          ID: mysql_running
    Function: service.running
        Name: mysql
      Result: True
     Comment: The service mysql is already running
     Started: 13:07:45.175226
    Duration: 48.966 ms
     Changes:   
----------
          ID: salt_pymysql
    Function: cmd.run
        Name: salt-pip install pymysql
      Result: True
     Comment: unless condition is true
     Started: 13:07:45.225424
    Duration: 1902.711 ms
     Changes:   
----------
          ID: appdb_database
    Function: mysql_database.present
        Name: ekou_test_db
      Result: True
     Comment: The database ekou_test_db has been created
     Started: 13:07:47.128440
    Duration: 107.082 ms
     Changes:   
              ----------
              ekou_test_db:
                  Present
----------
          ID: appdb_user
    Function: mysql_user.present
        Name: ekoumysql
      Result: True
     Comment: The user ekoumysql@localhost has been added
     Started: 13:07:47.235733
    Duration: 385.417 ms
     Changes:   
              ----------
              ekoumysql:
                  Present
----------
          ID: appdb_grants
    Function: mysql_grants.present
      Result: True
     Comment: Grant ALL PRIVILEGES on ekou_test_db.* to ekoumysql@localhost has been added
     Started: 13:07:47.621563
    Duration: 299.971 ms
     Changes:   
              ----------
              appdb_grants:
                  Present
----------
          ID: appdb_table
    Function: mysql_query.run
      Result: True
     Comment: {'query time': {'human': '57.1ms', 'raw': '0.05713'}, 'rows affected': 0}
     Started: 13:07:47.921771
    Duration: 207.329 ms
     Changes:   
              ----------
              query:
                  Executed

Summary for skou_test
------------
Succeeded: 7 (changed=4)
Failed:    0
------------
Total states run:     7
Total run time:   2.987 s
```

Final confirmation from inside MySQL on the minion:

```text
ekou@ubuntu1:~$ sudo mysql -u root
mysql> show databases;
+--------------------+
| ekou_test_db       |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
mysql> use ekou_test_db;
mysql> show tables;
+------------------------+
| Tables_in_ekou_test_db |
+------------------------+
| ekou_test_table        |
+------------------------+
```

(The state file in 7.5 already incorporates the lesson — its bootstrap installs `pymysql` **and** `saltext-mysql` together.)

**The reusable debugging ladder.** Each rung discriminates between two possible worlds, which is what made the diagnosis converge instead of guessing:

| Rung | Command | Question it answers |
|---|---|---|
| 1 | `state.highstate test=True` vs real run | Is this just a dry-run artifact of a self-bootstrapping state? |
| 2 | the `unless` result in the state output | Is the dependency actually installed? |
| 3 | `sys.list_state_functions <module>` | Does the minion's loader offer the state module *right now*? |
| 4 | restart `salt-minion`, probe again | Stale daemon cache, or something deeper? |
| 5 | `salt-call --local <fn> -l debug` **on the minion** | Does a fresh loader agree? And *why* does it refuse? (`__virtual__ returned False: <reason>`) |
| 6 | exec module vs state module behavior | Loads half-way = suspect packaging, not configuration |
| 7 | `ls .../salt/states/ \| grep <name>` | Does the module even exist on disk? |

Two traps worth remembering from stage 4: `salt-call --local` probes the machine you type it on, not your target minion — and it runs **without pillar**, so any behavior that depends on pillar (like the `mysql.unix_socket` connection setting) will differ from a master-driven run. An error seen only under `salt-call --local` is not necessarily a real error.

---

## 8. Pillar — secure per-minion data

Pillar holds variables and secrets, rendered per-minion so machines only see their own data. It lives under `/srv/pillar/` by default.

`/srv/pillar/top.sls`:

```yaml
base:
  'web*':
    - webserver
```

`/srv/pillar/webserver.sls`:

```yaml
nginx_worker_processes: 4
tls_cert_password: s3cr3t
```

Reference pillar data inside a state or template with Jinja:

```yaml
# in an SLS file
worker_processes {{ pillar['nginx_worker_processes'] }};
```

Refresh pillar data on minions after changes:

```bash
sudo salt '*' saltutil.refresh_pillar
sudo salt 'web01' pillar.items          # inspect what a minion sees
```

### 8.1 Debugging pillar — which files is Salt actually using?

When a minion's pillar looks wrong (or comes back empty, as in the 7.5 troubleshooting note), work down this funnel — from "where does Salt even look" to "what did the minion actually get." All four commands run on the master:

**1. Which directory Salt searches for pillar files** — `pillar_roots` is a *master* config setting:

```bash
sudo salt-run config.get pillar_roots
```

Expected output on a default install:

```text
base:
    - /srv/pillar
```

Use the runner form here — `salt-call --local config.get pillar_roots` would read the **minion** config on that box instead, which is not what decides where your pillar files must live.

**2. Which pillar SLS files the pillar top file assigns to a given minion:**

```bash
sudo salt-run pillar.show_top minion=skou_test
```

Expected output with the 7.5 pillar top file in place:

```text
base:
    - mysql
```

This is usually the one-shot answer: empty output means the pillar top file isn't matching (missing file, wrong target, tab in the YAML); `- mysql` means the mapping is fine and any problem is further down.

**3. What that assignment renders into** — the full pillar computed master-side, no minion involved:

```bash
sudo salt-run pillar.show_pillar skou_test
```

Expected output — the rendered result of the 7.5 `mysql.sls`:

```text
mysql.unix_socket:
    /var/run/mysqld/mysqld.sock
appdb:
    ----------
    name:
        ekou_test_db
    table:
        ekou_test_table
    user:
        appuser
    password:
        MySecret123
```

Note that the secret is printed in clear text — this command shows exactly what the matched minion would receive, which is the point, but it also means treat master shell history and scrollback as sensitive.

**4. What the minion actually holds** — its cached copy, updated by `saltutil.refresh_pillar`:

```bash
sudo salt skou_test pillar.items
```

Healthy, it returns the same data as step 3 under the minion's ID. Broken, it returns the bare `----------` shown in the real capture in the 7.5 troubleshooting note — the master rendered zero pillar for this minion, and steps 1–3 above tell you which link in the chain dropped it. If step 3 shows the data but the minion still returns nothing, re-run `saltutil.refresh_pillar` and check again.

The state and pillar systems mirror each other, but the commands differ:

| Question | States | Pillar |
|---|---|---|
| Where are files searched? | `salt-run config.get file_roots` | `salt-run config.get pillar_roots` |
| What does the top file assign this minion? | `salt skou_test state.show_top` | `salt-run pillar.show_top minion=skou_test` |
| Fully rendered result? | `salt skou_test state.show_highstate` | `salt-run pillar.show_pillar skou_test` |
| What is live on the minion? | `state.highstate test=True` | `salt skou_test pillar.items` |

The asymmetry is deliberate: the state commands are execution modules (the **minion** fetches and renders state files from the master's file server), while the pillar ones are **runners** (`salt-run`) that execute purely on the master — pillar is rendered on the master and only the finished, per-minion result is ever sent down. That is also why secrets in pillar never exist as files on the minion.

---

## 9. Templating with Jinja

SLS and managed config files support Jinja, so states adapt to each minion's grains/pillar:

```yaml
# /srv/salt/motd.sls
/etc/motd:
  file.managed:
    - contents: |
        Welcome to {{ grains['id'] }}
        OS: {{ grains['os'] }} {{ grains['osrelease'] }}
        Managed by Salt — do not edit by hand.
```

For larger files, use `file.managed` with a `- source: salt://path/to/template.jinja` and `- template: jinja`.

---

## 10. Day-to-day operations & troubleshooting

```bash
# Version / health
salt --versions-report
sudo systemctl status salt-master salt-minion

# Run master/minion in the foreground with debug logging:
sudo salt-master -l debug
sudo salt-minion -l debug

# Logs:
sudo tail -f /var/log/salt/master
sudo tail -f /var/log/salt/minion

# Force a minion to re-sync custom modules/states from the master:
sudo salt '*' saltutil.sync_all

# Job management:
sudo salt-run jobs.active           # currently running jobs
sudo salt-run manage.up             # which minions are responsive
sudo salt-run manage.down           # which are not answering
```

Common gotchas:

- **Minion not showing in `salt-key -L`** → check it can reach the master on 4505/4506 (firewall/DNS), and that `master:` is set correctly.
- **`No response` from a minion** → the `salt-minion` service may be stopped, or the key was deleted after acceptance (delete on the minion side too and re-handshake).
- **Time skew** → master and minions should have synced clocks (NTP/chrony); large skew breaks the crypto handshake.
- **State applies on master but not minions** → confirm `/srv/salt` is on the *master* and minions were told to `state.apply` (the file server serves from the master).

---

## 11. Impact of changing an IP address

The short version: Salt identifies machines by their **minion ID**, not their IP, and the cryptographic keys are tied to that ID — so changing an IP does not invalidate keys or force you to re-accept anything with `salt-key`. That's the part people usually worry about, and it's fine. The impact depends entirely on **whose** IP changes.

### 11.1 If a minion's IP changes

This is mostly transparent. Minions initiate the connection outbound to the master (on 4505/4506), so the master doesn't care what address a minion comes from; it matches on the accepted key/ID. The minion just reconnects from its new address. The things that do shift:

- Its IP-related grains (`ipv4`, `ip_interfaces`, `fqdn_ip4`, etc.) update after the minion restarts or you run `salt '<id>' saltutil.refresh_grains`. If any of your states, pillar top files, or targeting use `-G 'ipv4:...'`, those matches move with it.
- If you use the Salt mine to share IPs between minions (a common pattern for load-balancer or `/etc/hosts` configs), refresh it with `salt '*' mine.update` so consumers pick up the new value.
- Any firewall rule on the master that whitelists the minion by source IP needs updating, or the minion will be silently blocked on 4505/4506.
- Config files your states rendered with the old IP won't self-correct until the next `state.apply`/highstate.

Best practice after the change:

```bash
sudo systemctl restart salt-minion
salt '<id>' saltutil.refresh_grains
salt '*' mine.update        # only if the Salt mine is used
```

### 11.2 If the master's IP changes

This is the disruptive case, because every minion is configured to reach the master. What happens next depends on how you pointed them at it:

- If minions reference the master **by IP** (`master: 192.0.2.10` in `/etc/salt/minion`), they'll all stop connecting until you update that value on each one and restart `salt-minion`. Chicken-and-egg problem: you can't easily push that change through Salt once they're disconnected, so you'd need config management from your provisioning layer, SSH, or `salt-ssh`.
- If minions reference it **by DNS name** (or the conventional `salt` hostname, as in Section 3.1), you just update the DNS record and the minions reconnect on their own — no per-minion edits. This is exactly why pointing minions at a name rather than a raw IP is the recommended setup.
- The master's own key pair is not IP-bound, so no keys regenerate and all previously accepted minion keys stay accepted.
- Update the master's firewall for the new address, and if you use TLS certs or a published `salt://` endpoint bound to the old IP, refresh those.

### 11.3 Takeaway

Minion IP changes are low-risk (restart + refresh grains/mine). Master IP changes are only painful if your minions hardcode the IP — and the fix for that going forward is to address the master by DNS name, so a future IP change is a one-line DNS update instead of a fleet-wide reconfiguration.

---

## 12. Upgrading

Because the major version is pinned (Section 2.3), routine patching stays within the series:

```bash
sudo apt update && sudo apt upgrade salt-master salt-minion salt-common
```

To move to a new major LTS, edit the pin file to the new series (e.g. `Pin: version 3009.*`), `apt update`, then upgrade the **master first**, followed by minions. A master must be the same or newer version than its minions.

---

## 13. Quick reference card

```
salt-key -L / -a / -A / -d        # manage minion trust; -L alone = full inventory
                                  # (every Accepted key = a managed node, online or not)
salt-run manage.status            # that inventory split into up / down in one output
salt-run manage.up                # only responsive minions
salt-run manage.down              # only unresponsive minions
salt '<target>' test.ping         # connectivity (message-bus ping, not ICMP)
salt '<target>' cmd.run '<cmd>'   # ad-hoc shell
salt '<target>' grains.items      # facts about a host
salt '<target>' pkg.install <p>   # install a package
salt '<target>' state.apply <s>   # apply a state
salt '<target>' state.apply test=True   # dry run
salt '<target>' state.highstate   # apply everything from top.sls
salt-call --local <fn>            # masterless / local run
```

---

*Docs: Salt install guide — https://docs.saltproject.io/salt/install-guide/en/latest/ · Salt user guide — https://docs.saltproject.io/salt/user-guide/en/latest/*
