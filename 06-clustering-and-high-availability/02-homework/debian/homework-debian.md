# Tasks
Implement the following:

- Research and implement two node failover cluster that hosts a web site served by LVM volume group managed by the cluster. The volume group must reside on a separate iSCSI target server
# Solution
### Setup iSCSI Target server
```sh
lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0   19G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0    5G  0 disk 
sr0     11:0    1 1024M  0 rom
```
1. Install iSCSI package
```sh
sudo apt update && sudo apt install targetcli-fb -y
```
4. Start iSCSI administrative tool
```sh
sudo targetcli
```
5. Create block storage device
```sh
/backstores/block> /backstores/block create name=iscsi_disk dev=/dev/sdb
```
6. Create an iSCSI target
```sh
/iscsi create iqn.2025-02.lab.homework:iscsi-srv.target
```
7. Attach `/dev/sdb` as LUN
```sh
/iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/luns create /backstores/block/iscsi_disk
```
8. Register initiators
```sh
iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls create iqn.2025-02.lab.homework.fo-node-1.init

/iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls create iqn.2025-02.lab.homework.fo-node-2.init
```
9. Set **username** and **password** for initiator
```sh
# set username and password for first initiator
/iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-1.init/ set auth userid=web-app
iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-1.init/ set auth password=New_123123

# set username and password for second initiator
/iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-2.init/ set auth userid=web-app
iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-2.init/ set auth password=New_123123
```
10. Set authentication flag on for the target portal group (tpg1)
```sh
/iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/ set attribute authentication=1
```
11. Setup after save and exit
```sh
sudo targetcli ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 1]
  | | o- iscsi_disk ....................................................................... [/dev/sdb (5.0GiB) write-thru activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2025-02.lab.homework:iscsi-srv.target ......................................................................... [TPGs: 1]
  |   o- tpg1 .......................................................................................... [no-gen-acls, auth per-acl]
  |     o- acls .......................................................................................................... [ACLs: 2]
  |     | o- iqn.2025-02.lab.homework.fo-node-1.init .................................................. [1-way auth, Mapped LUNs: 1]
  |     | | o- mapped_lun0 ............................................................................ [lun0 block/iscsi_disk (rw)]
  |     | o- iqn.2025-02.lab.homework.fo-node-2.init .................................................. [1-way auth, Mapped LUNs: 1]
  |     |   o- mapped_lun0 ............................................................................ [lun0 block/iscsi_disk (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ................................................................. [block/iscsi_disk (/dev/sdb) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]
  o- vhost ............................................................................................................ [Targets: 0]
  o- xen-pvscsi ....................................................................................................... [Targets: 0]
```
12. Start and enable service
```sh
sudo systemctl restart rtslib-fb-targetctl
sudo systemctl enable rtslib-fb-targetctl
```
### Discover and log in to the iSCSI target on `fo-node-1` and `fo-node-2`. Steps are similar for both nodes.
1. Install the iSCSI initiator package
```sh
sudo apt install -y open-iscsi
```
2. Add initiator name into `/etc/iscsi/initiatorname.iscsi` (replace fo-node-1 with fo-node-2 for other node)
```plain
InitiatorName=iqn.2025-02.lab.homework.fo-node-1.init
```
3. Adjust `/etc/iscsi/iscsid.conf` file
```conf
node.session.auth.authmethod = CHAP # uncomment
node.session.auth.username # uncomment and set iscsi username
node.session.auth.passwor # uncomment ant set iscsi password
```
4. Initiate a target discovery (on both nodes)
```sh
sudo iscsiadm -m discovery -t sendtargets -p iscsi-srv
```
5. Login to the target (on both nodes)
```sh
sudo iscsiadm -m node --login
```
6. Confirm the established session (on both nodes)
```sh
sudo iscsiadm -m session -o show
```
### Set up Pacemaker & Corosync for failover on both `fo-node-1` and `fo-node-2.`
1. Install Hing Availability packages
```sh
sudo apt update && sudo apt install -y pacemaker pcs
```
2. Start and enable `pcsd` service
```sh
sudo systemctl enable --now pcsd
```
3. Set password for `hacluster` user
```sh
sudo passwd hacluster
```
4. Remove corosync config file (on both nodes)
```sh
sudo rm /etc/corosync/corosync.conf
```
5. Authenticate both nodes (execute on one node)
```sh
sudo pcs host auth fo-node-1.homework.lab fo-node-2.homework.lab
```
6. Create and start the cluster (execute on one node)
```sh
sudo pcs cluster setup cluster-1 fo-node-1.homework.lab fo-node-2.homework.lab --start --enable
```
7. Check the cluster status
```sh
sudo pcs cluster status
```
8. Disable STONITH, we don’t have hardware fencing.
```sh
sudo pcs property set stonith-enabled=false
```
### Create a Cluster Resource for LVM & Filesystem management.
1. Create a virtual IP address for the cluster
```sh
sudo pcs resource create cluster-virtual-ip ocf:heartbeat:IPaddr2 \
    ip=192.168.99.200 cidr_netmask=24 \
    op monitor interval=30s \
    --group web-application
```
2. Create the isCSI resource
```sh
sudo pcs resource create iscsi_storage systemd:iscsi \
    op start interval=0 timeout=30 \
    op stop interval=0 timeout=30 \
    op monitor interval=10 timeout=30 \
    --group web-application
```
3. install LVM package
```sh
sudo apt install -y lvm2
```
4. Modify LVM configuration `/etc/lvm/lvm.conf`. Ensures LVM is correctly handled in a clustered environment and avoids conflicts when both fo-node-1 and fo-node-2 access the same LVM resources (on both nodes)
```sh
system_id_source = "uname" # uncomment and set to "uname"
auto_activation_volume_list = [ "system_vg" ] # uncomment ans set
```
### Creating the volume group `iscsi_vg` on `fo-node-1.homework.lab`
1. On `fo-node-1`, create a physical volume using the iSCSI disk
```sh
sudo pvcreate /dev/sdb
```
2. Create the Volume group `iscsi_vg`
```sh
sudo vgcreate iscsi_vg /dev/sdb
```
3. Remove system Id for multi-node access (execute where volume group is attached)
```sh
sudo vgexport iscsi_vg
```
4. On node where volume group is not attached, import and activate the volume group.
```sh
# import
sudo vgimport iscsi_vg

# activate
sudo vgscan --cache
sudo vgchange -ay iscsi_vg
```
### Create the Logical volume `web_lv` on `fo-node-1.homework.lab`
1. On `fo-node-1 `create logical volume
```sh
sudo lvcreate -L 4.9G -n web_lv iscsi_vg
```
1. Format `web_lv` on `fo-node-1`
```sh
sudo mkfs.ext4 /dev/iscsi_vg/web_lv
```
1. Turn off automounting
```sh
sudo vgs --noheadings -o vg_name
```
1. Create mounting point
```sh
sudo mkdir -p /var/www/html
```
1. Set permissions
```sh
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```
### Configure Pacemaker for Failover
1. Create the iSCSI resource
```sh
sudo pcs resource create lvm_storage ocf:heartbeat:LVM-activate \
    vgname=iscsi_vg \
    vg_access_mode=system_id \
    activation_mode=exclusive \
    op monitor interval=10 timeout=30 \
    --group web-application
```
- `vgname=iscsi_vg` → Specifies the Volume Group to activate.
- `vg_access_mode=system_id` → Ensures LVM only activates on one node (as we already configured system_id_source in lvm.conf).
- `activation_mode=exclusive `→ Ensures only one node can use the VG at a time.
- `op monitor interval=10 timeout=30` → Monitors the LVM status every 10 seconds.
1. Add Filesystem resource
```sh
sudo pcs resource create web_mount ocf:heartbeat:Filesystem \
    device="/dev/iscsi_vg/web_lv" \
    directory="/var/www/html" \
    fstype="ext4" \
    op monitor interval=10 timeout=30 \
    --group web-application
```
- `device="/dev/iscsi_vg/web_lv"` → The Logical Volume to mount.
- `directory="/var/www/html"` → The mount point for the website.
- `fstype="ext4"` → The filesystem type (you formatted it as XFS).
- `op monitor interval=10 timeout=30` → Checks every 10 seconds to ensure it's mounted.
1. Test failover
```
