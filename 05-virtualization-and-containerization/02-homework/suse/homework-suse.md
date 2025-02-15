# Tasks
Chose and implement one or more of the following

- Research and create own LXC template (a distribution of your choice with web server)

- Create own Docker image based on CentOS or openSUSE that includes Apache web server and custom index page with some text (for example your SoftUni username) and a picture (of a cat, a dog, or whatever you like)

---
## Research and create own LXC template (a distribution of your choice with web server)

1. Install **Incus** and **Incus** tools
```sh
$ sudo zypper install incus incus-tools
```
2. Add user to **incus-admins** group to grant admin access
```sh
$ sudo usermod -aG incus-admin vagrant
```
3. Start and enable **Incus** service
```sh
$ sudo systemctl enable --now incus
Created symlink /etc/systemd/system/multi-user.target.wants/incus.service → /usr/lib/systemd/system/incus.service.
Created symlink /etc/systemd/system/multi-user.target.wants/incus-startup.service → /usr/lib/systemd/system/incus-startup.service.
Created symlink /etc/systemd/system/sockets.target.wants/incus.socket → /usr/lib/systemd/system/incus.socket.
```
4. Initialize **Incus**
```sh
$ sudo incus admin init
Would you like to use clustering? (yes/no) [default=no]: no
Do you want to configure a new storage pool? (yes/no) [default=yes]: yes
Name of the new storage pool [default=default]: suse-storage
Name of the storage backend to use (dir, lvm, lvmcluster, btrfs) [default=btrfs]: dir
Where should this storage pool store its data? [default=/var/lib/incus/storage-pools/suse-storage]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: yes
What should the new bridge be called? [default=incusbr0]: 
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: none
Would you like the server to be available over the network? (yes/no) [default=no]: yes
Address to bind to (not including port) [default=all]: 
Port to bind to [default=8443]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: yes
Would you like a YAML "init" preseed to be printed? (yes/no) [default=no]: yes
config:
  core.https_address: '[::]:8443'
networks:
- config:
    ipv4.address: auto
    ipv6.address: none
  description: ""
  name: incusbr0
  type: ""
  project: default
storage_pools:
- config: {}
  description: ""
  name: suse-storage
  driver: dir
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      network: incusbr0
      type: nic
    root:
      path: /
      pool: suse-storage
      type: disk
  name: default
projects: []
cluster: null
```
5. Setup firewall if needed
```sh
$ sudo firewall-cmd --zone=trusted --change-interface=incusbr0 --permanen
success
$ sudo firewall-cmd --reload
success

# restart Incus service
$ sudo systemctl restart incus
```
6. Search for **base** image (lets chose archlinux)
```sh
$ incus image list images: | grep archlinux
```
7. Create a container from chosen image
```sh
$ incus launch images:archlinux arch-template
Launching arch-template
```
8. Check running containers
```sh
incus list
+---------------+---------+-----------------------+------+-----------+-----------+
|     NAME      |  STATE  |         IPV4          | IPV6 |   TYPE    | SNAPSHOTS |
+---------------+---------+-----------------------+------+-----------+-----------+
| arch-template | RUNNING | 10.112.131.113 (eth0) |      | CONTAINER | 0         |
+---------------+---------+-----------------------+------+-----------+-----------+
```
9. Install **nginx** from **Incus** host
```sh
$ incus exec arch-template -- pacman -Sy --noconfirm nginx
```
10. Start and enable **Incus** service
```sh
$ incus exec arch-template -- systemctl enable --now nginx
```
11. Verify **nginx** is running
```sh
$ incus exec arch-template -- systemctl status nginx
● nginx.service - nginx web server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
    Drop-In: /run/systemd/system/service.d
             └─zzz-lxc-service.conf
     Active: active (running) since Sat 2025-02-15 15:13:34 UTC; 1min 19s ago
 Invocation: 8fe2fa41c653432488bddf0bc501d6ad
    Process: 431 ExecStart=/usr/bin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 432 (nginx)
      Tasks: 2 (limit: 4671)
     Memory: 2.7M (peak: 3M)
        CPU: 17ms
     CGroup: /system.slice/nginx.service
             ├─432 "nginx: master process /usr/bin/nginx"
             └─433 "nginx: worker process"
```
12. Check web service from Incus host (192.168.99.101)
```sh
$ curl -I http://10.112.131.113
HTTP/1.1 200 OK
Server: nginx/1.26.3
Date: Sat, 15 Feb 2025 15:16:41 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 05 Feb 2025 22:00:23 GMT
Connection: keep-alive
ETag: "67a3df77-267"
Accept-Ranges: bytes
```
13. Stop `arch-template` container
```sh
$ incus stop arch-template
```
14. Crete image from `arch-template` container
```sh
$ incus publish arch-template --alias arch-nginx
Instance published with fingerprint: b6b47aa798b4566faf5b273f27d35536bbcbdb54f2059ef60d01d1a7592e565d
```
15. Change the description of newly created image
```sh
$ incus image edit arch-nginx
```
16. Check images after creation of the new.
```sh
$ incus image list
+------------+--------------+--------+------------------------------------------+--------------+-----------+-----------+----------------------+
|   ALIAS    | FINGERPRINT  | PUBLIC |               DESCRIPTION                | ARCHITECTURE |   TYPE    |   SIZE    |     UPLOAD DATE      |
+------------+--------------+--------+------------------------------------------+--------------+-----------+-----------+----------------------+
| arch-nginx | b6b47aa798b4 | no     | Archlinux with Nginx                     | x86_64       | CONTAINER | 261.76MiB | 2025/02/15 17:19 EET |
+------------+--------------+--------+------------------------------------------+--------------+-----------+-----------+----------------------+
|            | fdbf0cb698da | no     | Archlinux current amd64 (20250215_04:18) | x86_64       | CONTAINER | 196.43MiB | 2025/02/15 17:03 EET |
+------------+--------------+--------+------------------------------------------+--------------+-----------+-----------+----------------------+
```
17. Create a new container from our image
```sh
$ incus launch arch-nginx arch-nginx-container

# verify running containers
$ incus list
+----------------------+---------+-----------------------+------+-----------+-----------+
|         NAME         |  STATE  |         IPV4          | IPV6 |   TYPE    | SNAPSHOTS |
+----------------------+---------+-----------------------+------+-----------+-----------+
| arch-nginx-container | RUNNING | 10.112.131.131 (eth0) |      | CONTAINER | 0         |
+----------------------+---------+-----------------------+------+-----------+-----------+
| arch-template        | STOPPED |                       |      | CONTAINER | 0         |
+----------------------+---------+-----------------------+------+-----------+-----------+
```
18. Verify that nginx web server is running inside container
```sh
$  incus exec arch-nginx-container -- systemctl status nginx
● nginx.service - nginx web server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
    Drop-In: /run/systemd/system/service.d
             └─zzz-lxc-service.conf
     Active: active (running) since Sat 2025-02-15 15:23:11 UTC; 2min 38s ago
 Invocation: f5915f9ca64040acbf2e242ad3ac4553
    Process: 198 ExecStart=/usr/bin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 199 (nginx)
      Tasks: 2 (limit: 4671)
     Memory: 2.6M (peak: 3.4M)
        CPU: 15ms
     CGroup: /system.slice/nginx.service
             ├─199 "nginx: master process /usr/bin/nginx"
             └─200 "nginx: worker process"
```
19. Test nginx serving page from Incus host (192.168.99.101)
```sh
$ curl -I http://10.112.131.131
HTTP/1.1 200 OK
Server: nginx/1.26.3
Date: Sat, 15 Feb 2025 15:27:20 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 05 Feb 2025 22:00:23 GMT
Connection: keep-alive
ETag: "67a3df77-267"
Accept-Ranges: bytes
```
20. Make container service visible outside of Incus host
```sh
# add net.ipv4.ip_forward=1 to /etc/sysctl.conf
$ sudo sysctl -p
net.ipv4.ip_forward = 1

# Forward port 80 from Incus host( 192.168.99.101) to container( 10.112.131.131)
$ sudo iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 -j DNAT --to-destination 10.112.131.131:80

# Add masquerade 
$ sudo iptables -t nat -A POSTROUTING -o incusbr0 -p tcp --dport 80 -j MASQUERADE
```
21.  Allow Traffic Through the Firewall
```sh
$ sudo firewall-cmd --zone=trusted --add-masquerade --permanent
$ sudo firewall-cmd --reload
```