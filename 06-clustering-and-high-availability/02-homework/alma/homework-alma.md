# Tasks

Implement the following:

-  Research and implement two node failover cluster that hosts a web site served by LVM volume group managed by the cluster. The volume group must reside on a separate iSCSI target server

# Solution

## iscsi-srv.homework.lab (192.168.99.103)

```sh
lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                       8:0    0   20G  0 disk 
├─sda1                    8:1    0    1G  0 part /boot
└─sda2                    8:2    0   19G  0 part 
  ├─almalinux_vbox-root 253:0    0   17G  0 lvm  /
  └─almalinux_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                       8:16   0    5G  0 disk 
sr0                      11:0    1 1024M  0 rom 
```

1. Install iSCSI required package
```sh
sudo dnf install targetcli
```
2. Setup LVM
```sh
# create LVM on the sdb
sudo pvcreate /dev/sdb

# create a Volume group - iscsi_vg
sudo vgcreate iscsi_vg /dev/sdb

# create a Logical volume  - web_lv
sudo lvcreate -L 4.9G -n web_lv iscsi_vg

# format the Logical volume
sudo mkfs.xfs /dev/iscsi_vg/web_lv
```
3. Present LVM through iSCSI using Target administration tool
```sh
sudo targetcli
```
4. Create block device
```sh
/backstores/block create name=iscsi_store dev=/dev/iscsi_vg/web_lv
```
5. Create target IQN (define new Target)
```sh
/iscsi create iqn.2025-02.lab.homework:iscsi-srv:target
```
6. Create a LUN
```sh
/iscsi/iqn.2025-02.lab.homework:iscsi-srv:target/tpg1/luns create /backstores/block/iscsi_store 
```
7. Register initiators
```sh
iscsi/iqn.2025-02.lab.homework:iscsi-srv:target/tpg1/acls create iqn.2025-02.lab.homework.fo-node-1.init

iscsi/iqn.2025-02.lab.homework:iscsi-srv:target/tpg1/acls create iqn.2025-02.lab.homework.fo-node-2.init
```
8. Set **username** and **password** for initiator
```sh
# set username and password for first initiator
iscsi/iqn.2025-02.lab.homework:iscsi-srv:target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-1.init/ set auth userid=web-app
iscsi/iqn.2025-02.lab.homework:iscsi-srv:target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-1.init/ set auth password=New_123123

# set username and password for second initiator
iscsi/iqn.2025-02.lab.homework:iscsi-srv:target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-2.init/ set auth userid=web-app
iscsi/iqn.2025-02.lab.homework:iscsi-srv:target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-2.init/ set auth password=New_123123
```
9. Set authentication flag on for the target portal group (tpg1)
```sh
/iscsi/iqn.2025-02.lab.homework:iscsi-srv:target/tpg1/ set attribute authentication=1
```
10. Setup after save and exit.
```sh
sudo targetcli ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 1]
  | | o- iscsi_store .......................................................... [/dev/iscsi_vg/web_lv (4.9GiB) write-thru activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2025-02.lab.homework:iscsi-srv:target ......................................................................... [TPGs: 1]
  |   o- tpg1 .......................................................................................... [no-gen-acls, auth per-acl]
  |     o- acls .......................................................................................................... [ACLs: 2]
  |     | o- iqn.2025-02.lab.homework.fo-node-1.init .................................................. [1-way auth, Mapped LUNs: 1]
  |     | | o- mapped_lun0 ........................................................................... [lun0 block/iscsi_store (rw)]
  |     | o- iqn.2025-02.lab.homework.fo-node-2.init .................................................. [1-way auth, Mapped LUNs: 1]
  |     |   o- mapped_lun0 ........................................................................... [lun0 block/iscsi_store (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 .................................................... [block/iscsi_store (/dev/iscsi_vg/web_lv) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]
```
11. Setup firewall
```sh
sudo firewall-cmd --add-service iscsi-target --permanent
sudo firewall-cmd --reload
```
12. Start and enable Target service
```sh
sudo systemctl enable --now target
```

## Discover and log in to the iSCSI target on `fo-node-1` and `fo-node-2`. Steps are similar for both nodes.

1. Setup iSCSI initiator
```sh
# install initiator package
sudo dnf install iscsi-initiator-utils

# add initiator name into /etc/iscsi/initiatorname.iscsi (replace fo-node-1 with fo-node-2 for other node)
InitiatorName=iqn.2025-02.lab.homework.fo-node-1.init

# adjust /etc/iscsi/iscsid.conf file
node.session.auth.authmethod = CHAP # uncomment
node.session.auth.username # uncomment and set iscsi username
node.session.auth.passwor # uncomment ant set iscsi password
```
2. Initiate a target discovery
```sh
sudo iscsiadm -m discovery -t sendtargets -p iscsi-srv
```
3. Show more from discovered
```sh
sudo iscsiadm -m node -o show
```
4. Login to the target
```sh
sudo iscsiadm -m node --login
```
5. Confirm the established session
```sh
sudo iscsiadm -m session -o show
```
## Activate the existing Volume Group (iscsi_vg) on the initiators



----

 Install the required packages
```sh
# Enable high availability repository
sudo dnf config-manager --set-enabled highavailability

# Install of the main packages
sudo dnf install pacemaker pcs
```