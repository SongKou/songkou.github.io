+++
title = 'Linux_name_space'
date = 2021-01-01T14:52:01+08:00
draft = false
categories = ['Operating System']
tags = ['Linux','Virtualization']
+++
# Linux NameSpace

Docker denpends on Linux NameSpace, Thus this is a brief explaination on linux name space.

We can use unshare command to manipulate/playaround with linux name spaces.

## How to check what kind of linux namespace is supported

We can use “man unshare” to check what kind of linux namespace is supported. There are 8 types of name spaces:

```
   mount namespace
          Mounting  and  unmounting  filesystems will not affect the rest of the system (CLONE_NEWNS flag), except for filesystems which are explicitly marked as shared (with mount --make-shared; see /proc/self/mountinfo or
          findmnt -o+PROPAGATION for the shared flags).

          unshare automatically sets propagation to private in the new mount namespace to make sure that the new namespace is really unshared. This feature is possible to disable by  option  --propagation  unchanged.   Note
          that private is the kernel default.

   UTS namespace
          Setting hostname or domainname will not affect the rest of the system.  (CLONE_NEWUTS flag)

   IPC namespace
          The process will have an independent namespace for System V message queues, semaphore sets and shared memory segments.  (CLONE_NEWIPC flag)

   network namespace
          The process will have independent IPv4 and IPv6 stacks, IP routing tables, firewall rules, the /proc/net and /sys/class/net directory trees, sockets, etc.  (CLONE_NEWNET flag)

   pid namespace
          Children will have a distinct set of PID to process mappings from their parent.  (CLONE_NEWPID flag)

   user namespace
          The process will have a distinct set of UIDs, GIDs and capabilities.  (CLONE_NEWUSER flag)

   See clone(2) for the exact semantics of the flags.

```

1.  Mount（mnt） isolate mounting space
2.  Process ID (pid) isolate process ID.
3.  Network (net) isolate network , port ID, etc
4.  Interprocess Communication (ipc) isolate System V IPC & POSIX message queues
5.  UTS Namespace(uts) Isolate hostname and domain name
6.  User Namespace (user) isolate user group and users.

Additionally, Linux 4.6 and 5.6 introduced cgroups and time two types. in total it’s 8.

So, normally we will begin with a process ID name space as a start since it’s more close to the original env and easier to understand.

```
[ekou@localhost ~]$ sudo unshare --pid --fork --mount-proc /bin/bash
[root@localhost ekou]# ps -ef 
UID         PID   PPID  C STIME TTY          TIME CMD
root          1      0  0 03:46 pts/4    00:00:00 /bin/bash
root         31      1  0 03:46 pts/4    00:00:00 ps -ef

```

now we can see that we got only one process running and bash runs with PID 1, and other processes are disappreared. Which means we already created a namespace for this bash shell. execute sleep 1000 on the namespace and execute pstree on hostmachine, we can see below out put:

```
************************on bash namespace**************************
    [root@localhost ekou]# sleep 1000
************************on hostmachine**************************
[ekou@localhost ~]$ sudo pstree
[sudo] password for ekou: 
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─NetworkManager─┬─dhclient
        │                └─2*[{NetworkManager}
output omitted. 
        ├─sshd─┬─sshd───sshd───2*[bash]
        │      ├─sshd───sshd───bash───bash───sudo───unshare───bash───sleep --> this is the one
        │      ├─sshd───sshd───bash
        │      └─sshd───sshd───bash───sudo───pstree

```

You can check all name spaces running on your hostmachine by executing below command:

```
[ekou@localhost ~]$ sudo lsns
        NS TYPE  NPROCS   PID USER   COMMAND
4026531836 pid      224     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531837 user     225     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531838 uts      225     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531839 ipc      225     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531840 mnt      214     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531856 mnt        1    13 root   kdevtmpfs
4026531956 net      224     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532500 net        1   695 rtkit  /usr/libexec/rtkit-daemon
4026532563 mnt        2 42815 root   unshare --pid --fork --mount-proc /bin/bash  -->this is the Namespace we created. 
4026532564 pid        1 42816 root   /bin/bash -->this is the bashshell under Namespace we created.
4026532585 mnt        1   695 rtkit  /usr/libexec/rtkit-daemon
4026532627 mnt        1   735 chrony /usr/sbin/chronyd
4026532628 mnt        2   810 root   /usr/sbin/NetworkManager --no-daemon
4026532724 mnt        1  1165 root   /usr/sbin/cupsd -f
4026532797 mnt        1  1738 root   /usr/libexec/boltd
4026532868 mnt        1  1844 colord /usr/libexec/colord
4026532869 mnt        1  2650 root   /usr/libexec/fwupd/fwupd

```

Then we can use blow command to verify

```
[ekou@localhost ~]$ sudo ls -l /proc/self/ns/
total 0
lrwxrwxrwx. 1 root root 0 Jan 28 04:49 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 root root 0 Jan 28 04:49 mnt -> mnt:[4026531840]
lrwxrwxrwx. 1 root root 0 Jan 28 04:49 net -> net:[4026531956]
lrwxrwxrwx. 1 root root 0 Jan 28 04:49 pid -> pid:[4026531836]
lrwxrwxrwx. 1 root root 0 Jan 28 04:49 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 Jan 28 04:49 uts -> uts:[4026531838]
[ekou@localhost ~]$ sudo ls -l /proc/42900/ns/
total 0
lrwxrwxrwx. 1 root root 0 Jan 28 04:47 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 root root 0 Jan 28 04:47 mnt -> mnt:[4026532563]
lrwxrwxrwx. 1 root root 0 Jan 28 04:47 net -> net:[4026531956]
lrwxrwxrwx. 1 root root 0 Jan 28 04:47 pid -> pid:[4026532564]
lrwxrwxrwx. 1 root root 0 Jan 28 04:47 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 Jan 28 04:47 uts -> uts:[4026531838]

```

as per the command output above, we can see that pid and mnt ID are different with the hostmachine. which means these are running in a different namespace.

## Trial on network namespace

Please you need either root account or you have the sudoer privilege on your linux box.

```
[ekou@localhost Python_Scripts]$ sudo ip netns add red
[ekou@localhost Python_Scripts]$ sudo ip netns add test
[ekou@localhost Python_Scripts]$ sudo ip netns add blue
[ekou@localhost Python_Scripts]$ ip netns list
test
red
blue

```

then you need to create a veth pair to connect a network namespace to the outside world via the “default” or “global” namespace where physical interfaces exist.

```
[ekou@localhost Python_Scripts]$ sudo ip link add veth0 type veth peer name veth1 
[ekou@localhost Python_Scripts]$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eno16777736: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:9e:d7:91 brd ff:ff:ff:ff:ff:ff
3: virbr0: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:f2:57:77 brd ff:ff:ff:ff:ff:ff
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:f2:57:77 brd ff:ff:ff:ff:ff:ff
***5: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether f2:e0:f7:be:ae:ae brd ff:ff:ff:ff:ff:ff
***6: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 8e:b8:a6:c2:ac:63 brd ff:ff:ff:ff:ff:ff******

```

Then we need to associate the veth1 interface with the namespace that we created before, eg: we link it to blue namespace.

```
**[ekou@localhost Python_Scripts]$ sudo ip link set veth1 netns blue**
[sudo] password for ekou: 
**[ekou@localhost Python_Scripts]$ ip link**
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eno16777736: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:9e:d7:91 brd ff:ff:ff:ff:ff:ff
3: virbr0: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:f2:57:77 brd ff:ff:ff:ff:ff:ff
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:f2:57:77 brd ff:ff:ff:ff:ff:ff
6: veth0@if5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 8e:b8:a6:c2:ac:63 brd ff:ff:ff:ff:ff:ff link-netnsid 0
**[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ip link list**
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth1@if6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether f2:e0:f7:be:ae:ae brd ff:ff:ff:ff:ff:ff link-netnsid 0

```

Configure IP address for this veth interface :

```
**[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ip addr add 192.168.10.10/24 dev veth1**
[sudo] password for ekou: 
**[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ip link set dev veth1 up**
**[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ifconfig**
veth1: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.10.10  netmask 255.255.255.0  broadcast 0.0.0.0
        ether f2:e0:f7:be:ae:ae  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

**[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ip add**
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth1@if6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether f2:e0:f7:be:ae:ae brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.10.10/24 scope global veth1
       valid_lft forever preferred_lft forever

```

To execute commands in the namespace, the format is : ip netns exec [network namespace] [command to run against that namespace]

However, when you try to ping the IP address in it’s namespace, it won’t work.

```
[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ping 192.168.10.10
PING 192.168.10.10 (192.168.10.10) 56(84) bytes of data.
^C
--- 192.168.10.10 ping statistics ---
16 packets transmitted, 0 received, 100% packet loss, time 14999ms

```

The reason is packets destined for 192.168.10.0/24(like ping) goes through the “local” route table. Thus we need to bring up it’s loopback interface as well.

```
**[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ip link set dev lo up**
**[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ip link** 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth1@if6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN mode DEFAULT group default qlen 1000
    link/ether f2:e0:f7:be:ae:ae brd ff:ff:ff:ff:ff:ff link-netnsid 0
**[ekou@localhost Python_Scripts]$ sudo ip netns exec blue ping 192.168.10.10**        
PING 192.168.10.10 (192.168.10.10) 56(84) bytes of data.
64 bytes from 192.168.10.10: icmp_seq=1 ttl=64 time=0.026 ms
64 bytes from 192.168.10.10: icmp_seq=2 ttl=64 time=0.045 ms
^C
--- 192.168.10.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.026/0.035/0.045/0.011 ms

```

### Link the namespace IP to the outside world

When this configuration is done, internal communication in the network namespace is working, but from host machine we can’t reach the namespace.

1st method is we use a linux bridge to link veth0-1 pair to the host machine.

```
**[ekou@centos_server network-scripts]$ sudo brctl addbr bridge0**
**[ekou@centos_server network-scripts]$ sudo ip addr add 192.168.10.1 dev bridge0**
**[ekou@centos_server network-scripts]$ sudo ip link set dev bridge0 up**
**[ekou@centos_server network-scripts]$ sudo brctl addif bridge0 veth0**
[ekou@centos_server network-scripts]$ ip add
9: bridge0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether ce:2d:1c:2b:9c:8a brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.1/32 scope global bridge0
       valid_lft forever preferred_lft forever
    inet6 fe80::bce3:48ff:fe7c:de43/64 scope link 
       valid_lft forever preferred_lft forever
 **[ekou@centos_client network-scripts]$ brctl show bridge0**
bridge name     bridge id               STP enabled     interfaces
bridge0         8000.c27f6e9bf240       no              veth0
[ekou@centos_client ~]$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.10.80.2      0.0.0.0         UG    100    0        0 eno16777736
10.10.80.0      0.0.0.0         255.255.255.0   U     100    0        0 eno16777736
[ekou@centos_client sysconfig]$ sudo route add -net 192.168.10.0/24 dev bridge0
[ekou@centos_client sysconfig]$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.10.80.2      0.0.0.0         UG    100    0        0 eno16777736
10.10.80.0      0.0.0.0         255.255.255.0   U     100    0        0 eno16777736
192.168.10.0    0.0.0.0         255.255.255.0   U     0      0        0 bridge0
[ekou@centos_client network-scripts]$ sudo ip netns exec blue route add -net 0.0.0.0/0 dev veth1
[ekou@centos_client network-scripts]$ sudo ip netns exec blue route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         0.0.0.0         0.0.0.0         U     0      0        0 veth1
192.168.10.0    0.0.0.0         255.255.255.0   U     0      0        0 veth1
[ekou@centos_client sysconfig]$ sudo ip -all netns exec ip link
[sudo] password for ekou: 

netns: test
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

netns: blue
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth1@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ea:38:3d:5e:bc:f9 brd ff:ff:ff:ff:ff:ff link-netnsid 0

netns: red
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
9: veth22@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether c2:c9:6e:9c:db:bc brd ff:ff:ff:ff:ff:ff link-netnsid 0
[ekou@centos_client sysconfig]$ sudo ip -all netns exec ip addr

netns: test
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

netns: blue
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
5: veth1@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ea:38:3d:5e:bc:f9 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.10.10/24 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::e838:3dff:fe5e:bcf9/64 scope link 
       valid_lft forever preferred_lft forever

netns: red
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
9: veth22@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether c2:c9:6e:9c:db:bc brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.10.11/24 scope global veth22
       valid_lft forever preferred_lft forever
    inet6 fe80::c0c9:6eff:fe9c:dbbc/64 scope link 
       valid_lft forever preferred_lft forever
[ekou@centos_client sysconfig]$ sudo ip netns exec red ping 192.168.10.2
PING 192.168.10.2 (192.168.10.2) 56(84) bytes of data.
64 bytes from 192.168.10.2: icmp_seq=1 ttl=64 time=0.095 ms
64 bytes from 192.168.10.2: icmp_seq=2 ttl=64 time=0.065 ms
^C
--- 192.168.10.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.065/0.080/0.095/0.015 ms
[ekou@centos_client sysconfig]$ sudo ip netns exec red ping 192.168.10.10
PING 192.168.10.10 (192.168.10.10) 56(84) bytes of data.
64 bytes from 192.168.10.10: icmp_seq=1 ttl=64 time=0.066 ms
64 bytes from 192.168.10.10: icmp_seq=2 ttl=64 time=0.076 ms
^C
--- 192.168.10.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.066/0.071/0.076/0.005 ms

```

Then the communication between namespaces will be working.

### Other Ways to do this:

There are may ways to make this happen. below is the procedure that I copied from internet, he use mac-vlan to make it happen.

```

    $ sudo ip netns add ns0
    $ sudo ip netns exec ns0 ip link set lo up
    $ sudo ip link add macvlan0 link eth0 type macvlan mode bridge
    $ sudo ip link set macvlan0 netns ns0
    $ sudo ip netns exec ns0 ip link set macvlan0 up
    $ sudo ip netns exec ns0 ip addr add 172.29.6.123/21 dev macvlan0
    $ sudo ip netns exec ns0 ping 172.29.0.1
    PING 172.29.0.1 (172.29.0.1) 56(84) bytes of data.
    64 bytes from 172.29.0.1: icmp_seq=1 ttl=64 time=0.360 ms
    64 bytes from 172.29.0.1: icmp_seq=2 ttl=64 time=0.412 ms


```
