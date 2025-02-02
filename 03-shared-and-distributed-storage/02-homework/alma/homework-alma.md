# Tasks

Choose and implement one or more of the following:

### Create a Samba group share (one folder, two users, one group, accessible only by the group)

1. Install required packages

```sh
$ sudo dnf install policycoreutils-python-utils
```

2. Install **Samba** packages

```sh
$ sudo dnf install samba samba-client samba-common
```

3. Enable and start on boot.

```sh
$ sudo systemctl enable smb nmb
Created symlink /etc/systemd/system/multi-user.target.wants/smb.service → /usr/lib/systemd/system/smb.service.
Created symlink /etc/systemd/system/multi-user.target.wants/nmb.service → /usr/lib/systemd/system/nmb.service.

$ sudo systemctl start smb nmb
```

4. Add service to firewall and reload configuration

```sh
$ sudo firewall-cmd --add-service samba --permanent
success

$ sudo firewall-cmd --reload
success
```

5. Create share folder

```sh
$ sudo mkdir -p /homework/samba/department
```

6. Adjust the **SELinux** content

```sh
$ sudo chcon -R -t samba_share_t /homework/samba/department/

$ sudo semanage fcontext -at samba_share_t "/homework/samba/department(/.*)?"
```

7. Create group `department`

```sh
$ sudo groupadd department
```

8. Change the group ownership and permissions of the directory

```sh
$ sudo chgrp department /homework/samba/department/

$ sudo chmod -R 770 /homework/samba/department/

$ ls -al /homework/samba/
total 0
drwxr-xr-x. 3 root root       24 Feb  2 10:43 .
drwxr-xr-x. 3 root root       19 Feb  2 10:43 ..
drwxrwx---. 2 root department  6 Feb  2 10:43 department
```

9. Create two users on Samba server and add users in `department` group

```sh
$ sudo useradd user-1
$ sudo useradd user-2

$ sudo usermod -aG department user-1
$ sudo usermod -aG department user-2
```

10. Add the users to Samba server

```sh
$ sudo smbpasswd -a user-1
New SMB password: # type password
Retype new SMB password: # re-type password
Added user user-1.

$ sudo smbpasswd -a user-2
New SMB password: # type password
Retype new SMB password: # re-type password
Added user user-2.

# Check users in Samba
$ sudo pdbedit -Lv
---------------
Unix username:        user-1
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-3746210456-531547969-3149378222-1000
Primary Group SID:    S-1-5-21-3746210456-531547969-3149378222-513
Full Name:
Home Directory:       \\ALMA-1\user-1
HomeDir Drive:
Logon Script:
Profile Path:         \\ALMA-1\user-1\profile
Domain:               ALMA-1
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          Wed, 06 Feb 2036 17:06:39 EET
Kickoff time:         Wed, 06 Feb 2036 17:06:39 EET
Password last set:    Sun, 02 Feb 2025 10:47:03 EET
Password can change:  Sun, 02 Feb 2025 10:47:03 EET
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
---------------
Unix username:        user-2
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-3746210456-531547969-3149378222-1001
Primary Group SID:    S-1-5-21-3746210456-531547969-3149378222-513
Full Name:
Home Directory:       \\ALMA-1\user-2
HomeDir Drive:
Logon Script:
Profile Path:         \\ALMA-1\user-2\profile
Domain:               ALMA-1
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          Wed, 06 Feb 2036 17:06:39 EET
Kickoff time:         Wed, 06 Feb 2036 17:06:39 EET
Password last set:    Sun, 02 Feb 2025 10:47:16 EET
Password can change:  Sun, 02 Feb 2025 10:47:16 EET
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

11. Configure Samba `/etc/samba/smb.conf`. Add the following contents to share out our `department` directory to the group `department`.

```conf
[department]
    comment = Department work folder
    path = /homework/samba/department
    writable = yes
    public = yes
    valid users = @department
    write list = @department
    force group = department
    create mask = 0770
    directory mask = 2770
    hosts allow = 192.168.99.102
```

12. Test new configuration

```sh
$ sudo testparm
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Weak crypto is allowed by GnuTLS (e.g. NTLM as a compatibility fallback)

Server role: ROLE_STANDALONE

Press enter to see a dump of your service definitions
```

13. Reload two services

```sh
$ sudo systemctl reload smb nmb
```

14. Check shares locally

```sh
$ sudo smbclient -NL //localhost
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        department      Disk      Department work folder
        IPC$            IPC       IPC Service (Samba 4.20.2)
SMB1 disabled -- no workgroup available
```

15. Login to `alma-2` and install **Samba** client.

```sh
$ sudo dnf install samba-client cifs-utils
```

16. List **Samba** shares from client

```sh
$ smbclient -L alma-1 -U user-1%New_123123

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        department      Disk      Department work folder
        IPC$            IPC       IPC Service (Samba 4.20.2)
        user-1          Disk      Home Directories
SMB1 disabled -- no workgroup available
```

17. Create mount point on client and mount the group share

```sh
$ sudo mkdir -p /mnt/department

# using noperm option to disables local permission checks on the client side
$ sudo mount -o username=user-1,password=New_123123,noperm //alma-1/department /mnt/department
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.

$ sudo systemctl daemon-reload
```

18. Check mount

```sh
$ df -hT
Filesystem                      Type      Size  Used Avail Use% Mounted on
devtmpfs                        devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs                           tmpfs     888M     0  888M   0% /dev/shm
tmpfs                           tmpfs     355M  5.0M  350M   2% /run
/dev/mapper/almalinux_vbox-root xfs        17G  2.8G   15G  17% /
/dev/sda1                       xfs       960M  290M  671M  31% /boot
vagrant                         vboxsf    954G  178G  776G  19% /vagrant
tmpfs                           tmpfs     178M  4.0K  178M   1% /run/user/1000
//alma-1/department             cifs       17G  2.8G   15G  17% /mnt/department
```

19. Mount the share on boot

```sh
# unmount
$ sudo umount /mnt/department

# create credentials file to store username and password
$ sudo vi /etc/smbcredentials

$ sudo cat /etc/credentials
username=user-1
password=New_123123

# change permissions on file
$ sudo chmod 600 /etc/smbcredentials
```

20. Add new row in `/etc/fstab`

```sh
# Samba share
//alma-1/department  /mnt/department  cifs  credentials=/etc/smbcredentials,noperm,_netdev  0  0
```

21. Test

```sh
# load fstab with new record
$ sudo mount -av
/                        : ignored
/boot                    : already mounted
none                     : ignored
/vagrant                 : already mounted
Host "alma-1" resolved to the following IP addresses: 192.168.99.101
mount.cifs kernel mount options: ip=192.168.99.101,unc=\\alma-1\department,noperm,user=user-1,pass=********
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
/mnt/department          : successfully mounted

# perform test
$ cd /mnt/department/
$ touch test-from-alma-2.txt

$ ls -al
total 0
drwxr-xr-x. 2 root root  0 Feb  2 11:12 .
drwxr-xr-x. 3 root root 24 Feb  2 11:02 ..
-rwxr-xr-x. 1 root root  0 Feb  2 11:12 test-from-alma-2.txt
```

### Create an NFS share with different access (read-write and read-only) for two stations

1. Install required packages on server

```sh
$ sudo dnf install nfs-utils
```

2. Start and enable the service to start automatically on boot.

```sh
$ sudo systemctl enable --now nfs-server
Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /usr/lib/systemd/system/nfs-server.service.
```

3. Stop services not needed for NFS v4

```sh
# check running services
$ ss -4tl
State           Recv-Q          Send-Q                     Local Address:Port                             Peer Address:Port          Process
LISTEN          0               4096                             0.0.0.0:39753                                 0.0.0.0:*
LISTEN          0               64                               0.0.0.0:41761                                 0.0.0.0:*
LISTEN          0               4096                             0.0.0.0:mountd                                0.0.0.0:*
LISTEN          0               50                               0.0.0.0:microsoft-ds                          0.0.0.0:*
LISTEN          0               50                               0.0.0.0:netbios-ssn                           0.0.0.0:*
LISTEN          0               4096                             0.0.0.0:sunrpc                                0.0.0.0:*
LISTEN          0               128                              0.0.0.0:ssh                                   0.0.0.0:*
LISTEN          0               64                               0.0.0.0:nfs                                   0.0.0.0:*

# Mask them
$ sudo systemctl mask --now rpcbind.service rpcbind.socket rpc-statd.service
Created symlink /etc/systemd/system/rpcbind.service → /dev/null.
Created symlink /etc/systemd/system/rpcbind.socket → /dev/null.
Created symlink /etc/systemd/system/rpc-statd.service → /dev/null.
```

4. Create folder and allow everyone to be able to do anything

```sh
$ sudo mkdir -p /homework/nfs/share

# allow everyone to be able to do anything
$ sudo chmod -R 777 /homework/nfs/share
```

5. Setup **SELinux** Booleans

```sh
# nsf booleans are ON
$ getsebool -a | grep nfs
cobbler_use_nfs --> off
colord_use_nfs --> off
conman_use_nfs --> off
ftpd_use_nfs --> off
git_cgi_use_nfs --> off
git_system_use_nfs --> off
httpd_use_nfs --> off
ksmtuned_use_nfs --> off
logrotate_use_nfs --> off
mpd_use_nfs --> off
nagios_use_nfs --> off
nfs_export_all_ro --> on
nfs_export_all_rw --> on
nfsd_anon_write --> off
openshift_use_nfs --> off
polipo_use_nfs --> off
samba_share_nfs --> off
sanlock_use_nfs --> off
sge_use_nfs --> off
tmpreaper_use_nfs --> off
use_nfs_home_dirs --> off
virt_use_nfs --> off
xen_use_nfs --> off
```

6. Add record in `/etc/exports`

```sh
$ cat /etc/exports
/homework/nfs/share 192.168.99.102(rw) 192.168.99.103(ro)
```

7. Apply changes

```sh
$ sudo exportfs -rav
exporting 192.168.99.102:/homework/nfs/share
exporting 192.168.99.103:/homework/nfs/share

# more info about shares
$ sudo exportfs -v
/homework/nfs/share
                192.168.99.102(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
/homework/nfs/share
                192.168.99.103(sync,wdelay,hide,no_subtree_check,sec=sys,ro,secure,root_squash,no_all_squash)
```

8. Add **NFS** service to firewall allow list

```sh
$ sudo firewall-cmd --add-service nfs --permanent
success
$ sudo firewall-cmd --reload
success
```

9. Login to `alma-2` (IP address: 192.168.99.102) and install nfs client.

```sh
$ sudo dnf install nfs-utils
```

10. Create a mount point

```sh
$ sudo mkdir -pv /mnt/nfs/share
mkdir: created directory '/mnt/nfs'
mkdir: created directory '/mnt/nfs/share'
```

11. Mount **NFS** share

```sh
$ sudo mount -t nfs4 alma-1:/homework/nfs/share /mnt/nfs/share

$ df -hT
Filesystem                      Type      Size  Used Avail Use% Mounted on
devtmpfs                        devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs                           tmpfs     888M     0  888M   0% /dev/shm
tmpfs                           tmpfs     355M  5.0M  350M   2% /run
/dev/mapper/almalinux_vbox-root xfs        17G  2.8G   15G  17% /
/dev/sda1                       xfs       960M  290M  671M  31% /boot
vagrant                         vboxsf    954G  178G  776G  19% /vagrant
tmpfs                           tmpfs     178M  4.0K  178M   1% /run/user/1000
//alma-1/department             cifs       17G  2.8G   15G  17% /mnt/department
alma-1:/homework/nfs/share      nfs4       17G  2.8G   15G  17% /mnt/nfs/share
```

12. Check the access from `alma-2`

```sh
$ cd /mnt/nfs/share/
$ echo "test from suse-2" | tee /mnt/nfs/share/new-file-suse-2.txt
test from suse-2

$ ls -al
total 4
drwxrwxrwx. 2 root    root    33 Feb  2 11:30 .
drwxr-xr-x. 3 root    root    19 Feb  2 11:24 ..
-rw-r--r--. 1 vagrant vagrant 17 Feb  2 11:30 new-file-suse-2.txt
```

13. Login on `alma-3` (IP address: 192.168.99.103) and install nfs client.

```sh
$ sudo dnf install nfs-utils
```

14. Create a mount point

```sh
$ sudo mkdir -pv /mnt/nfs/share
mkdir: created directory '/mnt/nfs'
mkdir: created directory '/mnt/nfs/share'
```

15. Mount **NFS** share

```sh
$ sudo mount -t nfs4 alma-1:/homework/nfs/share /mnt/nfs/share

$ df -hT
Filesystem                      Type      Size  Used Avail Use% Mounted on
devtmpfs                        devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs                           tmpfs     888M     0  888M   0% /dev/shm
tmpfs                           tmpfs     355M  5.0M  350M   2% /run
/dev/mapper/almalinux_vbox-root xfs        17G  2.8G   15G  17% /
/dev/sda1                       xfs       960M  290M  671M  31% /boot
vagrant                         vboxsf    954G  178G  776G  19% /vagrant
tmpfs                           tmpfs     178M  4.0K  178M   1% /run/user/1000
alma-1:/homework/nfs/share      nfs4       17G  2.8G   15G  17% /mnt/nfs/share
```

16. Check the access from `alma-3`

```sh
$ cd /mnt/nfs/share/
$ echo "test from suse-3" | tee /mnt/nfs/share/new-file-suse-3.txt
tee: /mnt/nfs/share/new-file-suse-3.txt: Read-only file system
test from suse-3

$ cat new-file-suse-2.txt
test from suse-2
```

### Create an iSCSI disk-based target

1. On Target (alma-1) install required package

```sh
$ sudo dnf install targetcli
```

2. Create a folder to store the iSCSI disk files

```sh
$ sudo mkdir -pv /var/lib/iscsi_disks/
mkdir: created directory '/var/lib/iscsi_disks/'
```

3. Start the admin tool

```sh
$ sudo targetcli
```

4. Create iSCSI disk

```sh
/> cd backstores/fileio
/backstores/fileio> create disk1 /var/lib/iscsi_disks/disk1.img 2G
Created fileio disk1 with size 2147483648
/backstores/fileio> ls
o- fileio ..................................................................................................... [Storage Objects: 1]
  o- disk1 ........................................................ [/var/lib/iscsi_disks/disk1.img (2.0GiB) write-back deactivated]
    o- alua ....................................................................................................... [ALUA Groups: 1]
      o- default_tg_pt_gp ........................................................................... [ALUA state: Active/optimized]
```

5. Create target IQN (define new Target)

```sh
/backstores/fileio> cd /iscsi
/iscsi> create iqn.2025-02.lab.homework:alma-1.target1
Created target iqn.2025-02.lab.homework:alma-1.target1.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
```

6. Create a LUN

```sh
/iscsi> cd iqn.2025-02.lab.homework:alma-1.target1/tpg1/luns
/iscsi/iqn.20...et1/tpg1/luns> create /backstores/fileio/disk1
Created LUN 0.
```

7. Register a initiator

```sh
/iscsi/iqn.20...et1/tpg1/luns> cd ../acls
/iscsi/iqn.20...et1/tpg1/acls> create iqn.2025-02.lab.homework:alma-2.init1
Created Node ACL for iqn.2025-02.lab.homework:alma-2.init1
Created mapped LUN 0.
```

8. Set username and password for initiator

```sh
/iscsi/iqn.20...et1/tpg1/acls> cd iqn.2025-02.lab.homework:alma-2.init1
/iscsi/iqn.20...:alma-2.init1> set auth userid=user-1
Parameter userid is now 'user-1'.
/iscsi/iqn.20...:alma-2.init1> set auth password=New_123123
Parameter password is now 'New_123123'.
```

9. Set authentication flag on for the target portal group (tpg1)

```sh
/iscsi/iqn.20...:alma-2.init1> cd /iscsi/iqn.2025-02.lab.homework:alma-1.target1/tpg1/
/iscsi/iqn.20....target1/tpg1> set attribute authentication=1
Parameter authentication is now '1'.
```

10. Final setup

```sh
/> ls /
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 1]
  | | o- disk1 ...................................................... [/var/lib/iscsi_disks/disk1.img (2.0GiB) write-back activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2025-02.lab.homework:alma-1.target1 ........................................................................... [TPGs: 1]
  |   o- tpg1 .......................................................................................... [no-gen-acls, auth per-acl]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.2025-02.lab.homework:alma-2.init1 .................................................... [1-way auth, Mapped LUNs: 1]
  |     |   o- mapped_lun0 ................................................................................ [lun0 fileio/disk1 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ............................................... [fileio/disk1 (/var/lib/iscsi_disks/disk1.img) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]
```

11. Setup firewall

```sh
$ sudo firewall-cmd --add-service iscsi-target --permanent
success
$ sudo firewall-cmd --reload
success
```

12. Enable and start the Terget service

```sh
$ sudo systemctl enable --now target
Created symlink /etc/systemd/system/multi-user.target.wants/target.service → /usr/lib/systemd/system/target.service.
```

13. Login on `alma-2` and install iscsi initiator.

```sh
$ sudo dnf install iscsi-initiator-utils
```

14. Set the initiator name `/etc/iscsi/initiatorname.iscsi`

```sh
$ cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.2025-02.lab.homework:alma-2.init1
```

15. Set the behavior of initiator in `/etc/iscsi/iscsid.conf`

```sh
# uncomment CHAP settings
node.session.auth.authmethod = CHAP

# uncoment and setup username and password
node.session.auth.username = user-1
node.session.auth.password = New_123123
```

16. Restart `iSCSI` service

```sh
sudo systemctl restart iscsi
```

17. Initiate a target discovery

```sh
$ sudo iscsiadm -m discovery -t sendtargets -p alma-1
192.168.99.101:3260,1 iqn.2025-02.lab.homework:alma-1.target1
```

18. Try to login on target (this also attach the disk)

```sh
$ sudo iscsiadm -m node --login
Logging in to [iface: default, target: iqn.2025-02.lab.homework:alma-1.target1, portal: 192.168.99.101,3260]
Login to [iface: default, target: iqn.2025-02.lab.homework:alma-1.target1, portal: 192.168.99.101,3260] successful.

# new disk is sdc
$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                       8:0    0   20G  0 disk
├─sda1                    8:1    0    1G  0 part /boot
└─sda2                    8:2    0   19G  0 part
  ├─almalinux_vbox-root 253:0    0   17G  0 lvm  /
  └─almalinux_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                       8:16   0    5G  0 disk
sdc                       8:32   0    2G  0 disk
sr0                      11:0    1 1024M  0 rom
```

19. Crearte patition, format and mount

```sh
# create mount point
$ sudo mkdir -p /mnt/iscsi/

# create partition
$ sudo parted -s /dev/sdc -- mklabel msdos mkpart primary 16384s -0m

# create file system
$ sudo mkfs.ext4 /dev/sdc1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 522240 4k blocks and 130560 inodes
Filesystem UUID: e56b7564-3643-48e8-89cd-4bf7e48fb757
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

# mount
$ sudo mount /dev/sdc1 /mnt/iscsi/

$ df -hT
Filesystem                      Type      Size  Used Avail Use% Mounted on
devtmpfs                        devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs                           tmpfs     888M     0  888M   0% /dev/shm
tmpfs                           tmpfs     355M  5.1M  350M   2% /run
/dev/mapper/almalinux_vbox-root xfs        17G  2.8G   15G  17% /
/dev/sda1                       xfs       960M  290M  671M  31% /boot
vagrant                         vboxsf    954G  178G  776G  19% /vagrant
//alma-1/department             cifs       17G  2.9G   15G  17% /mnt/department
alma-1:/homework/nfs/share      nfs4       17G  2.9G   15G  17% /mnt/nfs/share
tmpfs                           tmpfs     178M  4.0K  178M   1% /run/user/1000
/dev/sdc1                       ext4      2.0G   24K  1.9G   1% /mnt/iscsi
```

20. Mount on boot

```sh
# take UUID of iscsi disk
$ sudo blkid /dev/sdc1
/dev/sdc1: UUID="e56b7564-3643-48e8-89cd-4bf7e48fb757" TYPE="ext4" PARTUUID="834bb923-01"

# add new record in /etc/fstab
# iSCSI
UUID="e56b7564-3643-48e8-89cd-4bf7e48fb757" /mnt/iscsi ext4 _netdev 0 0
```

### Create a GlusterFS dispersed volume

1. Isntall required packages on `alma-1`, `alma-2` and `alma-3`. These three vms will act as GlusterFS bricks.

```sh
# add repository
$ sudo dnf install centos-release-gluster10

# install required packages
$ sudo dnf install glusterfs glusterfs-libs glusterfs-server
```

2. Enable and start the service

```sh
$ sudo systemctl enable --now glusterd
```

3. Setup firewall

```sh
$ sudo firewall-cmd --add-service=glusterfs --permanent
success
$ sudo firewall-cmd --reload
success
```

4. Create partition on `/dev/sdb` and format (for all bricks)

```sh
$ sudo parted -s /dev/sdb -- mklabel msdos mkpart primary 16384s -0m

$ sudo mkfs.ext4 /dev/sdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 1308672 4k blocks and 327680 inodes
Filesystem UUID: 4fbcdc8d-de1d-4771-afb2-ee461a68c1ab
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

5. Mount new partition (for all bricks)

```sh
# create mount point
$ sudo mkdir -p /mnt/glusterfs

# mount fs
$ sudo mount /dev/sdb1 /mnt/glusterfs

$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                       8:0    0   20G  0 disk
├─sda1                    8:1    0    1G  0 part /boot
└─sda2                    8:2    0   19G  0 part
  ├─almalinux_vbox-root 253:0    0   17G  0 lvm  /
  └─almalinux_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                       8:16   0    5G  0 disk
└─sdb1                    8:17   0    5G  0 part /mnt/glusterfs
sr0                      11:0    1 1024M  0 rom
```

6. Test communication and add bricks from `alma-1`.

```sh
$ sudo gluster peer probe alma-2
peer probe: success

$ sudo gluster peer probe alma-3
peer probe: success

$ sudo gluster peer status
Number of Peers: 2

Hostname: alma-2
Uuid: 2f7ceb7a-5dbd-4e66-ad90-acb96054a8ff
State: Peer in Cluster (Connected)

Hostname: alma-3
Uuid: a1a76b37-b7f8-4538-ad6f-42a694c52129
State: Peer in Cluster (Connected)
```

7. Create a volume (execute from `alma-1`)

```sh
$ sudo gluster volume create homework-vol disperse 3 redundancy 1 alma-1:/mnt/glusterfs alma-2:/mnt/glusterfs alma-3:/mnt/glusterfs force
volume create: homework-vol: success: please start the volume to access data
```

8. Get information about the volume

```sh
$ sudo gluster volume info homework-vol

Volume Name: homework-vol
Type: Disperse
Volume ID: 8880d819-e5d3-4bc5-9c26-4b38924b9357
Status: Created
Snapshot Count: 0
Number of Bricks: 1 x (2 + 1) = 3
Transport-type: tcp
Bricks:
Brick1: alma-1:/mnt/glusterfs
Brick2: alma-2:/mnt/glusterfs
Brick3: alma-3:/mnt/glusterfs
Options Reconfigured:
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on
```

9. Start the volume

```sh
$ sudo gluster volume start homework-vol
volume start: homework-vol: success
```

10. Login on `alma-4` and install neccessary packages.

```sh
# add repo
$ sudo dnf install centos-release-gluster10

# install packages
$ sudo dnf install glusterfs glusterfs-fuse
```

11. Create a mount point

```sh
$ sudo mkdir -p /mnt/glusterfs
```

12. Mount the volume

```sh
$ sudo mount -t glusterfs alma-2:/homework-vol /mnt/glusterfs

$ df -hT
Filesystem                      Type            Size  Used Avail Use% Mounted on
devtmpfs                        devtmpfs        4.0M     0  4.0M   0% /dev
tmpfs                           tmpfs           888M     0  888M   0% /dev/shm
tmpfs                           tmpfs           355M  5.0M  350M   2% /run
/dev/mapper/almalinux_vbox-root xfs              17G  2.8G   15G  17% /
/dev/sda1                       xfs             960M  290M  671M  31% /boot
vagrant                         vboxsf          954G  178G  776G  19% /vagrant
tmpfs                           tmpfs           178M  4.0K  178M   1% /run/user/1000
alma-2:/homework-vol            fuse.glusterfs  9.7G  102M  9.2G   2% /mnt/glusterfs
```

13. Write some files on mounted volume

```sh
$ echo "test" | sudo tee /mnt/glusterfs/test-file-0{1..9}
test

$ ls -al  /mnt/glusterfs/
total 49
drwxr-xr-x. 4 root root  4096 Feb  2 12:21 .
drwxr-xr-x. 3 root root    23 Feb  2 12:19 ..
drwx------. 2 root root 16384 Feb  2 12:11 lost+found
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-01
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-02
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-03
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-04
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-05
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-06
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-07
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-08
-rw-r--r--. 1 root root     5 Feb  2 12:21 test-file-09
```
14. Check the hashes on all bricks to ensure Dispersed mode

We had different md5sum values for the same file (test-file-01) across all bricks. This means the file is being dispersed (erasure coded).

```sh
# alma-1
$ md5sum /mnt/glusterfs/test-file-01
24b1d3fc0aaf94b269dee4a6a0f1ff63  /mnt/glusterfs/test-file-01

# alma-2
$ md5sum /mnt/glusterfs/test-file-01
f66587b187a2e86849e0c5427292aace  /mnt/glusterfs/test-file-01

# alma-3
$  md5sum /mnt/glusterfs/test-file-01
9761724f232eedffa8a7857d2eac5c75  /mnt/glusterfs/test-file-01
```