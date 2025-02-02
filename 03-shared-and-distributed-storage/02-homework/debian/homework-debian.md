# Tasks

Choose and implement one or more of the following:

### Create a Samba group share (one folder, two users, one group, accessible only by the group)

1. Install required packages on `debian-1` with role server.

```sh
# refresh package info
$ sudo apt update

# install samba packages
$ sudo apt install samba smbclient
```

2. Check if services are running

```sh
$ systemctl status nmbd smbd
● nmbd.service - Samba NMB Daemon
     Loaded: loaded (/lib/systemd/system/nmbd.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-02-02 13:07:10 EET; 1min 6s ago
       Docs: man:nmbd(8)
             man:samba(7)
             man:smb.conf(5)
    Process: 2972 ExecCondition=/usr/share/samba/is-configured nmb (code=exited, status=0/SUCCESS)
   Main PID: 2977 (nmbd)
     Status: "nmbd: ready to serve connections..."
      Tasks: 1 (limit: 2306)
     Memory: 4.5M
        CPU: 117ms
     CGroup: /system.slice/nmbd.service
             └─2977 /usr/sbin/nmbd --foreground --no-process-group

● smbd.service - Samba SMB Daemon
     Loaded: loaded (/lib/systemd/system/smbd.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-02-02 13:07:10 EET; 1min 6s ago
       Docs: man:smbd(8)
             man:samba(7)
             man:smb.conf(5)
    Process: 2978 ExecCondition=/usr/share/samba/is-configured smb (code=exited, status=0/SUCCESS)
    Process: 2980 ExecStartPre=/usr/share/samba/update-apparmor-samba-profile (code=exited, status=0/SUCCESS)
   Main PID: 2989 (smbd)
     Status: "smbd: ready to serve connections..."
      Tasks: 3 (limit: 2306)
     Memory: 6.7M
        CPU: 124ms
     CGroup: /system.slice/smbd.service
             ├─2989 /usr/sbin/smbd --foreground --no-process-group
             ├─2991 /usr/sbin/smbd --foreground --no-process-group
             └─2992 /usr/sbin/smbd --foreground --no-process-group
```

3. Add firewall exception for **Samba** if the firewall is running

```sh
$ sudo ufw allow samba
$ sudo ufw reload
```

4. Create share folder

```sh
$ sudo mkdir -p /homework/samba/department
```

5. Create group `department`

```sh
$ sudo groupadd department
```

6. Change the group ownership and permissions of the directory

```sh
$ sudo chgrp department /homework/samba/department/

$ sudo chmod -R 770 /homework/samba/department/

$ ls -al /homework/samba/
total 12
drwxr-xr-x 3 root root       4096 Feb  2 13:10 .
drwxr-xr-x 3 root root       4096 Feb  2 13:10 ..
drwxrwx--- 2 root department 4096 Feb  2 13:10 department
```

7. Create two users on Samba server and add users in `department` group

```sh
$ sudo useradd user-1
$ sudo useradd user-2

$ sudo usermod -aG department user-1
$ sudo usermod -aG department user-2
```

8. Add the users to Samba server

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
User SID:             S-1-5-21-189865169-1004522602-4176233501-1000
Primary Group SID:    S-1-5-21-189865169-1004522602-4176233501-513
Full Name:
Home Directory:       \\DEBIAN-1\user-1
HomeDir Drive:
Logon Script:
Profile Path:         \\DEBIAN-1\user-1\profile
Domain:               DEBIAN-1
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          Wed, 06 Feb 2036 17:06:39 EET
Kickoff time:         Wed, 06 Feb 2036 17:06:39 EET
Password last set:    Sun, 02 Feb 2025 13:13:22 EET
Password can change:  Sun, 02 Feb 2025 13:13:22 EET
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
---------------
Unix username:        user-2
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-189865169-1004522602-4176233501-1001
Primary Group SID:    S-1-5-21-189865169-1004522602-4176233501-513
Full Name:
Home Directory:       \\DEBIAN-1\user-2
HomeDir Drive:
Logon Script:
Profile Path:         \\DEBIAN-1\user-2\profile
Domain:               DEBIAN-1
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          Wed, 06 Feb 2036 17:06:39 EET
Kickoff time:         Wed, 06 Feb 2036 17:06:39 EET
Password last set:    Sun, 02 Feb 2025 13:13:33 EET
Password can change:  Sun, 02 Feb 2025 13:13:33 EET
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

9. Configure Samba `/etc/samba/smb.conf`. Add the following contents to share out our `department` directory to the group `department`.

```sh
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

10. Test new configuration

```sh
$ sudo testparm
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Weak crypto is allowed by GnuTLS (e.g. NTLM as a compatibility fallback)

Server role: ROLE_STANDALONE

Press enter to see a dump of your service definitions
```

11. Reload the **Samba** services

```sh
$ sudo systemctl restart smbd nmbd
```

12. Check shares locally

```sh
$ sudo smbclient -NL //localhost

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        department      Disk      Department work folder
        IPC$            IPC       IPC Service (Samba 4.17.12-Debian)
        nobody          Disk      Home Directories
SMB1 disabled -- no workgroup available
```

13. Login to `debian-2` and install **Samba** client.

```sh
# refresh repos
$ sudo apt update

# install required packages
$ sudo apt install smbclient cifs-utils
```

14. List **Samba** shares from client

```sh
$ smbclient -L debian-1 -U user-1%New_123123

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        department      Disk      Department work folder
        IPC$            IPC       IPC Service (Samba 4.17.12-Debian)
        user-1          Disk      Home Directories
SMB1 disabled -- no workgroup available
```

15. Create mount point on client and mount the group share

```sh
# create mount point
$ sudo mkdir -p /mnt/department

# mount share
$ sudo mount -o username=user-1,password=New_123123,noperm //debian-1/department /mnt/department
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.

# reload configuration
$ sudo systemctl daemon-reload
```

16. Check mount

```sh
$ df -hT
Filesystem            Type      Size  Used Avail Use% Mounted on
udev                  devtmpfs  962M     0  962M   0% /dev
tmpfs                 tmpfs     197M  556K  197M   1% /run
/dev/sda1             ext4       19G  2.2G   16G  13% /
tmpfs                 tmpfs     984M     0  984M   0% /dev/shm
tmpfs                 tmpfs     5.0M     0  5.0M   0% /run/lock
vagrant               vboxsf    954G  176G  778G  19% /vagrant
tmpfs                 tmpfs     197M     0  197M   0% /run/user/1000
//debian-1/department cifs       19G  3.3G   16G  18% /mnt/department
```

17. Mount the share on boot

```sh
# unmount
$ sudo umount /mnt/department

# create credentials file to store username and password
$ sudo nano /etc/smbcredentials

$ sudo cat /etc/credentials
username=user-1
password=New_123123

# change permissions on file
$ sudo chmod 600 /etc/smbcredentials
```

18. Add new row in `/etc/fstab`

```plain
# Samba
//debian-1/department  /mnt/department  cifs  credentials=/etc/smbcredentials,noperm,_netdev  0  0
```

19. Test

```sh
$ sudo mount -av
/                        : ignored
none                     : ignored
/media/cdrom0            : ignored
/vagrant                 : already mounted
mount.cifs kernel mount options: ip=192.168.99.101,unc=\\debian-1\department,noperm,user=user-1,pass=********
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
/mnt/department          : successfully mounted

# perform test
$ cd /mnt/department/
$ touch test-from-alma-2.txt

$ ls -al
total 4
drwxr-xr-x 2 root root    0 Feb  2 13:36 .
drwxr-xr-x 3 root root 4096 Feb  2 13:30 ..
-rwxr-xr-x 1 root root    0 Feb  2 13:36 test-from-alma-2.txt
```

### Create an NFS share with different access (read-write and read-only) for two stations

1. Install required packages on server

```sh
# refresh repos
$ sudo apt update

# install required packages
$ sudo apt install nfs-kernel-server
```

2. Check versions and ports

```sh
# supported protocol versions
$ sudo cat /proc/fs/nfsd/versions
+3 +4 +4.1 +4.2

# ports
$ rpcinfo -p localhost | grep nfs
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
```

3. Create folder and allow everyone to be able to do anything

```sh
$ sudo mkdir -p /homework/nfs/share

# allow everyone to be able to do anything
$ sudo chmod -R 777 /homework/nfs/share
```

4. Add record in `/etc/exports`

```sh
$ cat /etc/exports
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

/homework/nfs/share 192.168.99.102(rw) 192.168.99.103(ro)
```

5. Apply changes

```sh
$ sudo exportfs -rav
exportfs: /etc/exports [2]: Neither 'subtree_check' or 'no_subtree_check' specified for export "192.168.99.102:/homework/nfs/share".
  Assuming default behaviour ('no_subtree_check').
  NOTE: this default has changed since nfs-utils version 1.0.x

exportfs: /etc/exports [2]: Neither 'subtree_check' or 'no_subtree_check' specified for export "192.168.99.103:/homework/nfs/share".
  Assuming default behaviour ('no_subtree_check').
  NOTE: this default has changed since nfs-utils version 1.0.x

exporting 192.168.99.102:/homework/nfs/share
exporting 192.168.99.103:/homework/nfs/share

# more info about shares
$ sudo exportfs -v
/homework/nfs/share
                192.168.99.102(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
/homework/nfs/share
                192.168.99.103(sync,wdelay,hide,no_subtree_check,sec=sys,ro,secure,root_squash,no_all_squash)
```

6. Setup firewall if active

```sh
# allow communication from 192.168.99.0/24 network
sudo ufw allow from 192.168.99.0/24 to any port nfs
```

7. Login to debian-2 (IP address: 192.168.99.102) and install nfs client.

```sh
# refresh repos
$ sudo apt update

# install required packages
$ sudo apt install nfs-common
```

8. Create a mount point

```sh
$ sudo mkdir -pv /mnt/nfs/share
mkdir: created directory '/mnt/nfs'
mkdir: created directory '/mnt/nfs/share'
```

9. Mount **NFS** share

```sh
$ sudo mount -t nfs4 debian-1:/homework/nfs/share /mnt/nfs/share

$ df -hT
Filesystem                   Type      Size  Used Avail Use% Mounted on
udev                         devtmpfs  962M     0  962M   0% /dev
tmpfs                        tmpfs     197M  572K  197M   1% /run
/dev/sda1                    ext4       19G  2.2G   16G  13% /
tmpfs                        tmpfs     984M     0  984M   0% /dev/shm
tmpfs                        tmpfs     5.0M     0  5.0M   0% /run/lock
vagrant                      vboxsf    954G  176G  778G  19% /vagrant
tmpfs                        tmpfs     197M     0  197M   0% /run/user/1000
//debian-1/department        cifs       19G  3.3G   16G  18% /mnt/department
debian-1:/homework/nfs/share nfs4       19G  2.3G   16G  13% /mnt/nfs/share
```

10. Check permissions from `debian-2`

```sh
$ cd /mnt/nfs/share/
$ echo "test from suse-2" | tee /mnt/nfs/share/new-file-suse-2.txt
test from suse-2

$ ls -al
total 12
drwxrwxrwx 2 root    root    4096 Feb  2 13:49 .
drwxr-xr-x 3 root    root    4096 Feb  2 13:47 ..
-rw-r--r-- 1 vagrant vagrant   17 Feb  2 13:49 new-file-suse-2.txt
```

11. Login on `debian-3` (IP address: 192.168.99.103) and install nfs client.

```sh
# refresh repos
$ sudo apt update

# install required packages
$ sudo apt install nfs-common
```

12. Create a mount point

```sh
$ sudo mkdir -pv /mnt/nfs/share
mkdir: created directory '/mnt/nfs'
mkdir: created directory '/mnt/nfs/share'
```

13. Mount NFS share

```sh
$ sudo mount -t nfs4 debian-1:/homework/nfs/share /mnt/nfs/share

$ df -hT
Filesystem                   Type      Size  Used Avail Use% Mounted on
udev                         devtmpfs  962M     0  962M   0% /dev
tmpfs                        tmpfs     197M  556K  197M   1% /run
/dev/sda1                    ext4       19G  2.2G   16G  13% /
tmpfs                        tmpfs     984M     0  984M   0% /dev/shm
tmpfs                        tmpfs     5.0M     0  5.0M   0% /run/lock
vagrant                      vboxsf    954G  176G  778G  19% /vagrant
tmpfs                        tmpfs     197M     0  197M   0% /run/user/1000
debian-1:/homework/nfs/share nfs4       19G  2.3G   16G  13% /mnt/nfs/share
```

14. Check permissions from `debian-3`

```sh
$ echo "test from suse-3" | tee /mnt/nfs/share/new-file-suse-3.txt
tee: /mnt/nfs/share/new-file-suse-3.txt: Read-only file system
test from suse-3

$ cat /mnt/nfs/share/new-file-suse-2.txt
test from suse-2
```

### Create an iSCSI disk-based target

1. On Target (debian-1) install required package

```sh
# refresh repos
$ sudo apt update

# install required packages
$ sudo apt install targetcli-fb
```

2. Create a folder to store the iSCSI disk files

```sh
$ sudo mkdir -pv /var/lib/iscsi_disks/
mkdir: created directory '/var/lib/iscsi_disks/'
```

3. Start the _iSCSI_ admin tool

```sh
sudo targetcli
```

4. Create iSCSI disk

```sh
/> cd backstores/fileio
/backstores/fileio> create disk1 /var/lib/iscsi_disks/disk1.img 3G
Created fileio disk1 with size 3221225472
/backstores/fileio> ls
o- fileio ..................................................................................................... [Storage Objects: 1]
  o- disk1 ........................................................ [/var/lib/iscsi_disks/disk1.img (3.0GiB) write-back deactivated]
    o- alua ....................................................................................................... [ALUA Groups: 1]
      o- default_tg_pt_gp ........................................................................... [ALUA state: Active/optimized]
```

5. Create target IQN (define new Target)

```sh
/backstores/fileio> cd /iscsi
/iscsi> create iqn.2025-02.lab.homework:debian-1.target1
Created target iqn.2025-02.lab.homework:debian-1.target1.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
```

6. Create a LUN

```sh
/iscsi> cd iqn.2025-02.lab.homework:debian-1.target1/tpg1/luns
/iscsi/iqn.20...et1/tpg1/luns> create /backstores/fileio/disk1
Created LUN 0.
```

7. Register a initiator

```sh
/iscsi/iqn.20...et1/tpg1/luns> cd ../acls
/iscsi/iqn.20...et1/tpg1/acls> create iqn.2025-02.lab.homework:debian-2.init1
Created Node ACL for iqn.2025-02.lab.homework:debian-2.init1
Created mapped LUN 0.
```

8. Set username and password for initiator

```sh
/iscsi/iqn.20...et1/tpg1/acls> cd iqn.2025-02.lab.homework:debian-2.init1/
/iscsi/iqn.20...ebian-2.init1> set auth userid=user-1
Parameter userid is now 'user-1'.
/iscsi/iqn.20...ebian-2.init1> set auth password=New_123123
Parameter password is now 'New_123123'.
```

9. Set authentication flag `on` for the target portal group (tpg1)

```sh
/iscsi/iqn.20...ebian-2.init1> cd /iscsi/iqn.2025-02.lab.homework:debian-1.target1/tpg1/
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
  | | o- disk1 ...................................................... [/var/lib/iscsi_disks/disk1.img (3.0GiB) write-back activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2025-02.lab.homework:debian-1.target1 ......................................................................... [TPGs: 1]
  |   o- tpg1 .......................................................................................... [no-gen-acls, auth per-acl]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.2025-02.lab.homework:debian-2.init1 .................................................. [1-way auth, Mapped LUNs: 1]
  |     |   o- mapped_lun0 ................................................................................ [lun0 fileio/disk1 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ............................................... [fileio/disk1 (/var/lib/iscsi_disks/disk1.img) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]
  o- vhost ............................................................................................................ [Targets: 0]
  o- xen-pvscsi ....................................................................................................... [Targets: 0]
```

11. Setup firewall if active

```sh
$ sudo ufw allow 3260/tcp
$ sudo ufw reload
```

12. Enable and start service

```sh
$ sudo systemctl enable --now rtslib-fb-targetctl.service
Synchronizing state of rtslib-fb-targetctl.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable rtslib-fb-targetctl
```

13. Login on `debian-2` and install iscsi initiator.

```sh
# refresh repos
$ sudo apt update

# install required packages
$ sudo apt install open-iscsi
```

14. Set the initiator name `/etc/iscsi/initiatorname.iscsi`

```sh
$ sudo cat /etc/iscsi/initiatorname.iscsi
## DO NOT EDIT OR REMOVE THIS FILE!
## If you remove this file, the iSCSI daemon will not start.
## If you change the InitiatorName, existing access control lists
## may reject this initiator.  The InitiatorName must be unique
## for each iSCSI initiator.  Do NOT duplicate iSCSI InitiatorNames.
InitiatorName=iqn.2025-02.lab.homework:debian-2.init1
```

15. Set the behavior of initiator in /etc/iscsi/iscsid.conf

```sh
# uncomment node.startup=automatic and add commnet on node.startup=manual
node.startup = automatic

# uncomment CHAP settings
node.session.auth.authmethod = CHAP

# uncoment and setup username and password
node.session.auth.username = user-1
node.session.auth.password = New_123123
```

16. Restart iSCSI service

```sh
$ sudo systemctl restart iscsid.service
```

17. Initiate a target discovery

```sh
$ sudo iscsiadm -m discovery -t sendtargets -p debian-1
192.168.99.101:3260,1 iqn.2025-02.lab.homework:debian-1.target1
```

18. Try to login on target (this also attach the disk)

```sh
$ sudo iscsiadm -m node --login
Logging in to [iface: default, target: iqn.2025-02.lab.homework:debian-1.target1, portal: 192.168.99.101,3260]
Login to [iface: default, target: iqn.2025-02.lab.homework:debian-1.target1, portal: 192.168.99.101,3260] successful.

# new disk is sdc
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   20G  0 disk
├─sda1   8:1    0   19G  0 part /
├─sda2   8:2    0    1K  0 part
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0    5G  0 disk
sdc      8:32   0    3G  0 disk
sr0     11:0    1 1024M  0 rom
```

19. Crearte patition, format and mount

```sh
# create mount point
$ sudo mkdir -p /mnt/iscsi/

# create partition
$ sudo parted -s /dev/sdc -- mklabel msdos mkpart primary 16384s -0m

# create file system
$ sudo mkfs.ext4 /dev/sdc1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 784384 4k blocks and 196224 inodes
Filesystem UUID: 17e1a913-72ca-42a3-816b-09d860cc8aa1
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

# mount
$ sudo mount /dev/sdc1 /mnt/iscsi/

$ df -hT
Filesystem                   Type      Size  Used Avail Use% Mounted on
udev                         devtmpfs  962M     0  962M   0% /dev
tmpfs                        tmpfs     197M  584K  197M   1% /run
/dev/sda1                    ext4       19G  2.2G   16G  13% /
tmpfs                        tmpfs     984M     0  984M   0% /dev/shm
tmpfs                        tmpfs     5.0M     0  5.0M   0% /run/lock
vagrant                      vboxsf    954G  177G  777G  19% /vagrant
tmpfs                        tmpfs     197M     0  197M   0% /run/user/1000
//debian-1/department        cifs       19G  6.3G   13G  34% /mnt/department
debian-1:/homework/nfs/share nfs4       19G  5.3G   13G  30% /mnt/nfs/share
/dev/sdc1                    ext4      2.9G   24K  2.8G   1% /mnt/iscsi
```

20. Mount on boot

```sh
# take UUID of iscsi disk
$ sudo blkid /dev/sdc1
/dev/sdc1: UUID="17e1a913-72ca-42a3-816b-09d860cc8aa1" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="b42ae463-01"

# add new record in /etc/fstab
# iSCISI share
UUID="17e1a913-72ca-42a3-816b-09d860cc8aa1" /mnt/iscsi ext4 _netdev 0 0

$ sudo mount -av
/                        : ignored
none                     : ignored
/media/cdrom0            : ignored
/vagrant                 : already mounted
/mnt/department          : already mounted
/mnt/iscsi               : already mounted
```

### Create a GlusterFS dispersed volume

1. Isntall required packages on alma-1, alma-2 and alma-3. These three vms will act as GlusterFS bricks.

```sh
# refresh repos
$ sudo apt update

# install required packages
$ sudo apt install glusterfs-server glusterfs-common
```

2. Enable and start the service

```sh
$ sudo systemctl enable --now glusterd
Created symlink /etc/systemd/system/multi-user.target.wants/glusterd.service → /lib/systemd/system/glusterd.service.
```

3. Setup firewall if active (service glusterfs)

4. Create partition on `/dev/sdb` and format (for all bricks)

```sh
$ sudo parted -s /dev/sdb -- mklabel msdos mkpart primary 16384s -0

$ sudo mkfs.ext4 /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 1308672 4k blocks and 327680 inodes
Filesystem UUID: a1bb83f9-03cc-4617-b5e1-4afcba623723
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
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   20G  0 disk
├─sda1   8:1    0   19G  0 part /
├─sda2   8:2    0    1K  0 part
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0    5G  0 disk
└─sdb1   8:17   0    5G  0 part /mnt/glusterfs
sr0     11:0    1 1024M  0 rom
```

6. Test communication and add bricks from alma-1.

```sh
$ sudo gluster peer probe debian-2
peer probe: success

$ sudo gluster peer probe debian-3
peer probe: success

$ sudo gluster peer status
Number of Peers: 2

Hostname: debian-2
Uuid: 5faf433f-18a9-4674-8263-47282b71028b
State: Peer in Cluster (Connected)

Hostname: debian-3
Uuid: 66c63676-f40c-4b42-a09c-33d011ba5b57
State: Peer in Cluster (Connected)
```

7. Create a volume (execute from debian-1)

```sh
$ sudo gluster volume create homework-vol disperse 3 redundancy 1 debian-1:/mnt/glusterfs debian-2:/mnt/glusterfs debian-3:/mnt/glusterfs force
volume create: homework-vol: success: please start the volume to access data
```

8. Get information about the volume

```sh
$ sudo gluster volume info homework-vol

Volume Name: homework-vol
Type: Disperse
Volume ID: 1bb6157f-ebfb-4f0c-ab32-220a38daa916
Status: Created
Snapshot Count: 0
Number of Bricks: 1 x (2 + 1) = 3
Transport-type: tcp
Bricks:
Brick1: debian-1:/mnt/glusterfs
Brick2: debian-2:/mnt/glusterfs
Brick3: debian-3:/mnt/glusterfs
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

10. Login on `debian-4` and install neccessary packages.

```sh
# refresh repos
$ sudo apt update

# install required packages
$ sudo apt install glusterfs-client
```

11. Create a mount point

```sh
$ sudo mkdir -p /mnt/glusterfs
```

12. Mount the volume

```sh
$ sudo mount -t glusterfs debian-3:/homework-vol /mnt/glusterfs

$ df -hT
Filesystem             Type            Size  Used Avail Use% Mounted on
udev                   devtmpfs        962M     0  962M   0% /dev
tmpfs                  tmpfs           197M  540K  197M   1% /run
/dev/sda1              ext4             19G  2.2G   16G  13% /
tmpfs                  tmpfs           984M     0  984M   0% /dev/shm
tmpfs                  tmpfs           5.0M     0  5.0M   0% /run/lock
vagrant                vboxsf          954G  178G  776G  19% /vagrant
tmpfs                  tmpfs           197M     0  197M   0% /run/user/1000
debian-3:/homework-vol fuse.glusterfs  9.7G  102M  9.2G   2% /mnt/glusterfs
```

13. Write some files on mounted volume

```sh
$ echo "test" | sudo tee /mnt/glusterfs/test-file-0{1..9}
test

$ ls -al  /mnt/glusterfs/
total 53
drwxr-xr-x 4 root root  4096 Feb  2 16:05 .
drwxr-xr-x 3 root root  4096 Feb  2 16:04 ..
drwx------ 2 root root 16384 Feb  2 15:58 lost+found
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-01
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-02
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-03
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-04
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-05
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-06
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-07
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-08
-rw-r--r-- 1 root root     5 Feb  2 16:05 test-file-09
```

14. Check the hashes on all bricks to ensure Dispersed mode

We had different md5sum values for the same file (test-file-01) across all bricks. This means the file is being dispersed (erasure coded).

```sh
# debian-1
$ md5sum /mnt/glusterfs/test-file-01
24b1d3fc0aaf94b269dee4a6a0f1ff63  /mnt/glusterfs/test-file-01

# debian-2
$ md5sum /mnt/glusterfs/test-file-01
f66587b187a2e86849e0c5427292aace  /mnt/glusterfs/test-file-01

# debian-3
$ md5sum /mnt/glusterfs/test-file-01
9761724f232eedffa8a7857d2eac5c75  /mnt/glusterfs/test-file-01
```
