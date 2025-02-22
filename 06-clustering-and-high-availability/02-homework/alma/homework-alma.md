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
2. Present LVM through iSCSI using Target administration tool
```sh
sudo targetcli
```
3. Create block device
```sh
/backstores/block create name=iscsi_store dev=/dev/sdb
```
4. Create target IQN (define new Target)
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
  | | o- iscsi_store ...................................................................... [/dev/sdb (5.0GiB) write-thru activated]
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
  |     | o- lun0 ................................................................ [block/iscsi_store (/dev/sdb) (default_tg_pt_gp)]
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
##  Set up Pacemaker & Corosync for failover on both `fo-node-1` and `fo-node-2`.

1. Install the required packages
```sh
# Enable high availability repository
sudo dnf config-manager --set-enabled highavailability

# Install of the main packages
sudo dnf install pacemaker pcs
```
2. Start and enable Pacemaker service
```sh
sudo systemctl enable --now pcsd
```
3. Set password for **hacluster** user
```sh
sudo passwd hacluster
```
4. Setup firewall
```sh
sudo firewall-cmd --add-service=high-availability --permanent
sudo firewall-cmd --reload
```
5. Authenticate both nodes (execute only on fo-node-1)
```sh
sudo pcs host auth fo-node-1.homework.lab fo-node-2.homework.lab
```
6. Create and start the cluster
```sh
sudo pcs cluster setup cluster-1 fo-node-1.homework.lab fo-node-2.homework.lab --start --enable
```
7. Check the cluster status
```sh
sudo pcs cluster status
```
8. Disable **STONITH**, we don’t have hardware fencing.
```sh
sudo pcs property set stonith-enabled=false
sudo pcs property set no-quorum-policy=ignore
```
## Create a Cluster Resource for LVM & Filesystem management.
1. Create a virtual IP address for the cluster
```sh
sudo pcs resource create cl-1-vip ocf:heartbeat:IPaddr2 \
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
3. Modify LVM configuration

Ensures LVM is correctly handled in a clustered environment and avoids conflicts when both fo-node-1 and fo-node-2 access the same LVM resources.
```sh
# edit LVM configuration /etc/lvm/lvm.conf file on both nodes
use_devicesfile = 0 # uncomment and set to 0
system_id_source = "uname" # uncomment and set to "uname"
```
4. Check configuration and if node name matches
```sh
# check configuration
sudo lvm lvmconfig

# lvm node name
sudo lvm systemid

# system name
uname -n
```
## Creating the Volume group `iscsi_vg` on `fo-node-1.homework.lab`
1. On `fo-node-1`, create a physical volume using the iSCSI disk
```sh
sudo pvcreate /dev/sdb
```
2. Create the Volume group `iscsi_vg`
```sh
sudo vgcreate iscsi_vg /dev/sdb
```
3. Remove system Id for multi-node access
```sh
sudo vgexport iscsi_vg
```
4. On `fo-node-2`, import and activate the volume group.
```sh
# import
sudo vgimport iscsi_vg

# activate
sudo vgscan --cache
sudo vgchange -ay iscsi_vg
```
## Create the Logical volume `web_lv` on `fo-node-1`
1. On `fo-node-1` create logical volume
```sh
sudo lvcreate -L 4.9G -n web_lv iscsi_vg
```
2. Format `web_lv` on `fo-node-1`.
```sh
sudo mkfs.xfs /dev/iscsi_vg/web_lv
```
3. Create mounting point
```sh
sudo mkdir -p /var/www/html
```
4. Mount `web_lv` on `var/wwww/html`
```sh
sudo mount /dev/iscsi_vg/web_lv /var/www/html
```
## Configure Pacemaker for Failover
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
- `activation_mode=exclusive` → Ensures only one node can use the VG at a time.
`op monitor interval=10 timeout=30` → Monitors the LVM status every 10 seconds.
2. Add Filesystem resource
```sh
sudo pcs resource create web_mount ocf:heartbeat:Filesystem \
    device="/dev/iscsi_vg/web_lv" \
    directory="/var/www/html" \
    fstype="xfs" \
    op monitor interval=10 timeout=30 \
    --group web-application
```
- `device="/dev/iscsi_vg/web_lv"` → The Logical Volume to mount.
- `directory="/var/www/html"` → The mount point for the website.
- `fstype="xfs"` → The filesystem type (you formatted it as XFS).
- `op monitor interval=10 timeout=30` → Checks every 10 seconds to ensure it's mounted.
3. Test failover
```sh
# move the resources to fo-node-2
sudo pcs resource move web-application fo-node-2.homework.lab

# check is mounted
df -hT
```
## Setup Nginx on both nodes
1. Install nginx package
```sh
sudo dnf install -y nginx
```
2. Create Nginx configuration file `/etc/nginx/conf.d/web-app.conf`
```conf
server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.html;
    
    location / {
        try_files \$uri \$uri/ =404;
    }
}
```
3. Restart and enable Nginx service
```sh
sudo systemctl restart nginx
sudo systemctl enable nginx
```
4. Create custom webpage (this should be done on active node, where /var/www/html is mounted)
```sh
echo "<h1>Hello from the Clustered Nginx!</h1> <p>You was served by $(hostname)</p>" | sudo tee /var/www/html/index.html
```
5. Setup firewall
```sh
sudo firewall-cmd --add-service http --permanent
sudo firewall-cmd --reload
```
