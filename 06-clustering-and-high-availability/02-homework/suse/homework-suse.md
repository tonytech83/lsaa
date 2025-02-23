## Tasks
Implement the following:

- Research and implement two node failover cluster that hosts a web site served by LVM volume group managed by the cluster. The volume group must reside on a separate iSCSI target server
## Solution
```plain
Step 1 - Setup iSCSI Target server.
Step 2 - Initiator setup. Discover and log in to the iSCSI target.
Step 3 - Set up Pacemaker & Corosync for failover on nodes.
Step 4 - Create a Cluster Resource for LVM & Filesystem management.
Step 5 - Install Nginx and setup Cluster resource for it.
Step 6 - Test failover.
```
### Step 1. Setup iSCSI Target server
- Install iSCSI package
  ```sh
  sudo zypper refresh && sudo zypper install targetcli-fb
  ```
- Start iSCSI administrative tool
  ```sh
  sudo targetcli
  ```
- Create block storage device
  ```sh
  /backstores/block create name=iscsi_disk dev=/dev/sdb
  ```
- Create an IQN
  ```
  /iscsi create iqn.2025-02.lab.homework:iscsi-srv.target
  ```
- Attach `/dev/sdb` as LUN
  ```
  /iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/luns create /backstores/block/iscsi_disk
  ```
- Register initiators
  ```sh
  /iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls create iqn.2025-02.lab.homework.fo-node-1.init

  /iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls create iqn.2025-02.lab.homework.fo-node-2.init
  ```
- Set username and password for initiator
  ```sh
  # set username and password for first initiator
  /iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-1.init/ set auth userid=web-app
  /iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-1.init/ set auth password=New_123123

  # set username and password for second initiator
  /iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-2.init/ set auth userid=web-app
  /iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/acls/iqn.2025-02.lab.homework.fo-node-2.init/ set auth password=New_123123
  ```
- Set authentication flag on for the target portal group (tpg1)
  ```sh
  /iscsi/iqn.2025-02.lab.homework:iscsi-srv.target/tpg1/ set attribute authentication=1
  ```
- Setup after save and exit
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
- Setup firewall
  ```sh
  sudo firewall-cmd --add-service iscsi-target --permanent
  sudo firewall-cmd --reload
  ```
- Start and enable target service
  ```sh
  sudo systemctl enable --now targetcli.service
  ```

### Step 2. Initiator setup. Discover and log in to the iSCSI target.
This step should be executed on both nodes.

- Install the initiator package
  ```sh
  sudo zypper install open-iscsi
  ```
- Reboot the system and login again.
- Open the initiator configuration file for editing (/etc/iscsi/initiatorname.iscsi). Replace `fo-node-1` with `fo-node-2` for second node.
  ```sh
  iqn.2025-02.lab.homework.fo-node-1.init
  ```
- Adjust the authentication settings in `/etc/iscsi/iscsid.conf` file
  ```sh
  # change mode from manual to automatic (line 54)
  node.startup = automatic

  # uncomment CHAP auth (line 67)
  node.session.auth.authmethod = CHAP

  # uncomment and set iscsi username and password (line 79 and 80)
  node.session.auth.username = web-app
  node.session.auth.password = New_123123
  ```
- Initiate a target discovery
  ```sh
  sudo iscsiadm -m discovery -t sendtargets -p iscsi-srv
  ```
- Login to the target server
  ```sh
  sudo iscsiadm -m node --login
  ```
- Confirm the established session
  ```sh
  sudo iscsiadm -m session -o show
  ```