## PREPARATION

### Router

1. Check IP addresses on `router`

```sh
$ ip -c addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:75:bb:3f brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 86296sec preferred_lft 86296sec
    inet6 fd00::a00:27ff:fe75:bb3f/64 scope global dynamic mngtmpaddr
       valid_lft 86290sec preferred_lft 14290sec
    inet6 fe80::a00:27ff:fe75:bb3f/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:90:a7:40 brd ff:ff:ff:ff:ff:ff
    inet 192.168.88.109/24 brd 192.168.88.255 scope global dynamic enp0s8
       valid_lft 86304sec preferred_lft 86304sec
    inet6 fe80::a00:27ff:fe90:a740/64 scope link
       valid_lft forever preferred_lft forever
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:62:d6:27 brd ff:ff:ff:ff:ff:ff
    inet 192.168.99.101/24 brd 192.168.99.255 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe62:d627/64 scope link
       valid_lft forever preferred_lft forever
```

2. Disable VirtualBox own NAT interfaces (connected via enp0s3). Connection via vagrant ssh will be lost.

```sh
$ sudo ip link set enp0s3 down
```

3. Make ssh to `router` via `enp0s8`

```sh
ssh vagrant@192.168.88.109
```

4. Add the Default Gateway

```sh
$ sudo ip route add default via 192.168.88.1 dev enp0s8

```

5. Check the access to `client`

```sh
$ ping -c 5 192.168.99.102
PING 192.168.99.102 (192.168.99.102) 56(84) bytes of data.
64 bytes from 192.168.99.102: icmp_seq=1 ttl=64 time=1.76 ms
64 bytes from 192.168.99.102: icmp_seq=2 ttl=64 time=1.43 ms
64 bytes from 192.168.99.102: icmp_seq=3 ttl=64 time=1.85 ms
64 bytes from 192.168.99.102: icmp_seq=4 ttl=64 time=1.83 ms
64 bytes from 192.168.99.102: icmp_seq=5 ttl=64 time=2.00 ms

--- 192.168.99.102 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4008ms
rtt min/avg/max/mdev = 1.427/1.774/2.001/0.190 ms
```

### Client

1. Disable VirtualBox own NAT interfaces (connected via enp0s3). Connection via vagrant ssh will be lost.

```sh
$ sudo nmcli dev disconnect enp0s3
```

2. Test Connectivity: Ping an external IP (google dns) from `client`

```sh
$ ping 8.8.8.8
ping: connect: Network is unreachable
```

## Research and implement two-node solution (one machine with two NICs and the second with one) with NAT capabilities in the first VM using either firewalld or ufw (depending on the selected distribution)

1. Install UFW (Uncomplicated Firewall)

```sh
$ sudo apt update
$ sudo apt install ufw
```

2. Enable and start ufw

```sh
$ sudo systemctl enable --now ufw
Synchronizing state of ufw.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable ufw
```

3. Enable ip forwarding permanently (edit the`/etc/sysctl.conf` file). Uncomment following line

```plain
net.ipv4.ip_forward=1
```

4. Apply the changes

```sh
$ sudo sysctl -p
net.ipv4.ip_forward = 1
```

5. Enable NAT with **ufw**. Edit the `/etc/ufw/before.rules` file, add following lines at the top of the file , before the `filter` line.

```plain
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.99.0/24 -o enp0s8 -j MASQUERADE
COMMIT
```

6. Allow forwarding . Edit `/etc/default/ufw`, change `DEFAULT_FORWARD_POLICY` form DROP to ACCEPT

```plain
DEFAULT_FORWARD_POLICY="ACCEPT"
```

7.  Allow traffic on both NICs

```sh
$ sudo ufw allow in on enp0s9
Rules updated
Rules updated (v6)

$ sudo ufw allow out on enp0s8
Rules updated
Rules updated (v6)
```

8. Enable ufw

```sh
$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

10. Add a Default Route to VM1

```sh
$ sudo ip route add default via 192.168.99.101
```

11. Test Connectivity: Ping an external IP (google dns)

```sh
$ ping -c 5 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=57 time=3.76 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=57 time=4.63 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=57 time=3.97 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=57 time=4.78 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=57 time=5.92 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4009ms
rtt min/avg/max/mdev = 3.763/4.613/5.924/0.759 ms
```
## Research and implement two-node solution (one machine with two NICs and the second with one) with NAT capabilities in the first VM using nftables

Same as AlmaLinux