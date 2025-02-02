# Homework M3: Distributed and Shared Storage

Main goal is to build further on what was demonstrated during the practice
Prerequisites may vary between different tasks. You should adjust your infrastructure according to the task you chose to implement

## Tasks

Choose and implement one or more of the following:

- Create a Samba group share (one folder, two users, one group, accessible only by the group)
- Create an NFS share with different access (read-write and read-only) for two stations
- Create an iSCSI disk-based target
- Create a GlusterFS dispersed volume

* Please note that even if you choose to implement more than one task, they are quite independent and different, so you may need to create a separate infrastructure (environment) for each

## Proof

Prepare a document that shows what you accomplished and how you did it. It can include (not limited to):

- The commands you used to achieve the above tasks
- A few pictures showing intermediary steps or results

## Solutions

- alma/homework-alma.md
- debian/homework-debian.md
- suse/homework-suse.md

## OpenSUSE solution

### Create a Samba group share (one folder, two users, one group, accessible only by the group)

1. Install **Samba** packages

```sh
$ sudo zypper install samba samba-client
```

2. Enable and start the service

```sh
$ sudo systemctl enable smb nmb
Created symlink /etc/systemd/system/multi-user.target.wants/smb.service → /usr/lib/systemd/system/smb.service.
Created symlink /etc/systemd/system/multi-user.target.wants/nmb.service → /usr/lib/systemd/system/nmb.service.

$ sudo systemctl start smb nmb
```

3. Add firewall exception for **Samba** and reload it.

```sh
$ sudo firewall-cmd --add-service samba --permanent
success

$ sudo firewall-cmd --reload
success
```

4. Create folder

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
total 0
drwxr-xr-x 1 root root       20 Feb  1 09:18 .
drwxr-xr-x 1 root root       10 Feb  1 09:17 ..
drwxrwx--- 1 root department  0 Feb  1 09:18 department
```

7. Create two users on **Samba** server and add users in `department` group

```sh
$ sudo useradd user-1
$ sudo useradd user-2

$ sudo usermod -aG department user-1
$ sudo usermod -aG department user-2
```

8. Add the users to **Samba** server

```sh
# add user-1
$ sudo smbpasswd -a user-1
New SMB password: # type password
Retype new SMB password: # re-type password
Added user user-1.

# add user-2
$ sudo smbpasswd -a user-2
New SMB password: # type password
Retype new SMB password: # re-type password
Added user user-2.

# check
$ sudo pdbedit -Lv
---------------
Unix username:        user-1
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-2203497296-1643005602-2407561709-1000
Primary Group SID:    S-1-5-21-2203497296-1643005602-2407561709-513
Full Name:
Home Directory:       \\SUSE-1\user-1\.9xprofile
HomeDir Drive:        P:
Logon Script:
Profile Path:         \\SUSE-1\profiles\.msprofile
Domain:               SUSE-1
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          Wed, 06 Feb 2036 17:06:39 EET
Kickoff time:         Wed, 06 Feb 2036 17:06:39 EET
Password last set:    Sat, 01 Feb 2025 09:39:27 EET
Password can change:  Sat, 01 Feb 2025 09:39:27 EET
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
---------------
Unix username:        user-2
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-2203497296-1643005602-2407561709-1001
Primary Group SID:    S-1-5-21-2203497296-1643005602-2407561709-513
Full Name:
Home Directory:       \\SUSE-1\user-2\.9xprofile
HomeDir Drive:        P:
Logon Script:
Profile Path:         \\SUSE-1\profiles\.msprofile
Domain:               SUSE-1
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          Wed, 06 Feb 2036 17:06:39 EET
Kickoff time:         Wed, 06 Feb 2036 17:06:39 EET
Password last set:    Sat, 01 Feb 2025 09:40:04 EET
Password can change:  Sat, 01 Feb 2025 09:40:04 EET
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

9. Configure **Samba** `sudo nano /etc/samba/smb.conf`. Add the following contents to share out our `department` directory to the group `department`.

```conf
...
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

10. Test configuration

```sh
$ sudo testparm
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Weak crypto is allowed by GnuTLS (e.g. NTLM as a compatibility fallback)

Server role: ROLE_STANDALONE

Press enter to see a dump of your service definitions
```

11. Restart two deamons

```sh
$ sudo systemctl reload smb nmb
```

12. Check for available shares locally

```sh
$ sudo smbclient -NL //localhost

        Sharename       Type      Comment
        ---------       ----      -------
        profiles        Disk      Network Profiles Service
        users           Disk      All users
        groups          Disk      All groups
        print$          Disk      Printer Drivers
        department      Disk      Department work folder
        IPC$            IPC       IPC Service (Samba 4.19.9-git.399.71536ca297e150600.3.9.6SUSE-oS15.0-x86_64)
        nobody          Disk      Home Directories
SMB1 disabled -- no workgroup available
```

13. Install **Samba** client on `suse-2` vm.

```sh
$ sudo zypper install samba-client
```

14. List shares from client

```sh
$ smbclient -L //192.168.99.101 -U user-1
Password for [WORKGROUP\user-1]:

        Sharename       Type      Comment
        ---------       ----      -------
        profiles        Disk      Network Profiles Service
        users           Disk      All users
        groups          Disk      All groups
        print$          Disk      Printer Drivers
        department      Disk      Department work folder
        IPC$            IPC       IPC Service (Samba 4.19.9-git.399.71536ca297e150600.3.9.6SUSE-oS15.0-x86_64)
        user-1          Disk      Home Directories
SMB1 disabled -- no workgroup available
```

15. Create mount point and mount the group share

```sh
$ sudo mkdir -p /mnt/department

# using noperm option to disables local permission checks on the client side
$ sudo mount -o username=user-1,password=New_123123,noperm //192.168.99.101/department /mnt/department
```

16. Test

```sh
$ echo "test with new params" | tee /mnt/department/test-new.txt
test with new params
```

17. Mount the share on boot

```sh
# unmount
$ sudo umount /mnt/department

# create credentials file to store username and password
$ sudo touch /etc/smbcredentials

$ sudo cat /etc/credentials
username=user-1
password=New_123123

# change permissions on file
$ sudo chmod 600 /etc/smbcredentials

```

18. add new row in `/etc/fstab`

```plain
# Samba share
//192.168.99.101/department  /mnt/department  cifs  credentials=/etc/smbcredentials,noperm,_netdev  0  0
```

19. Test

```sh
# umount the share
$ sudo umount /mnt/department

# load fstab with new record
$ sudo mount -a

# perform test
$ cd /mnt/department
$ touch test-new-file.txt

$ ls -al
total 0
drwxr-xr-x 2 root root  0 Feb  1 12:01 .
drwxr-xr-x 1 root root 20 Feb  1 11:00 ..
-rwxr-xr-x 1 root root  0 Feb  1 11:58 test-new-file.txt
```

### Create an NFS share with different access (read-write and read-only) for two stations

1. Install **NFS** server

```sh
$ sudo zypper install nfs-kernel-server
```

2. Enable and start service

```sh
$ sudo systemctl enable --now nfsserver
Created symlink /etc/systemd/system/multi-user.target.wants/nfsserver.service → /usr/lib/systemd/system/nfsserver.service.
```

3. Create folder and allow everyone to be able to do anything

```sh
$ sudo mkdir -p /homework/nfs/share
```

4. Add record in `/etc/exports`

```plain
# See the exports(5) manpage for a description of the syntax of this file.
# This file contains a list of all directories that are to be exported to
# other computers via NFS (Network File System).
# This file used by rpc.nfsd and rpc.mountd. See their manpages for details
# on how make changes in this file effective.

/homework/nfs/share 192.168.99.102(rw) 192.168.99.103(ro)
```

5. Apply changes

```sh
$ sudo exportfs -rav
exporting 192.168.99.102:/homework/nfs/share
exporting 192.168.99.103:/homework/nfs/share
```

6. More information about shares

```sh
$ sudo exportfs -v
/homework/nfs/share
                192.168.99.102(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
/homework/nfs/share
                192.168.99.103(sync,wdelay,hide,no_subtree_check,sec=sys,ro,secure,root_squash,no_all_squash)
```

7. Check what services are allowed in firewall

```sh
$ sudo firewall-cmd --list-services
dhcpv6-client ssh
```

8. Add **NFS** service to firewall allow list

```sh
$ sudo firewall-cmd --add-service nfs --permanent
success

# reload the firewall service
$ sudo firewall-cmd --reload
success
```

9. Login on `suse-2` and install `nfs-client` if necessary.

```sh
$ sudo zypper install nfs-client
```

10. Create mount point

```sh
$ sudo mkdir -pv /mnt/nfs/share
mkdir: created directory '/mnt/nfs'
mkdir: created directory '/mnt/nfs/share'
```

11. Mount **NFS** share

```sh
$ sudo mount -t nfs4 suse-1:/homework/nfs/share /mnt/nfs/share

$ df
Filesystem                 1K-blocks      Used Available Use% Mounted on
/dev/sda2                   10476524   3421216   6140880  36% /
devtmpfs                        4096         0      4096   0% /dev
tmpfs                        1010176         0   1010176   0% /dev/shm
tmpfs                         404072      6340    397732   2% /run
/dev/sda2                   10476524   3421216   6140880  36% /boot/grub2/i386-pc
/dev/sda2                   10476524   3421216   6140880  36% /opt
/dev/sda2                   10476524   3421216   6140880  36% /boot/grub2/x86_64-efi
/dev/sda2                   10476524   3421216   6140880  36% /home
/dev/sda2                   10476524   3421216   6140880  36% /root
/dev/sda2                   10476524   3421216   6140880  36% /srv
/dev/sda2                   10476524   3421216   6140880  36% /tmp
/dev/sda2                   10476524   3421216   6140880  36% /usr/local
/dev/sda2                   10476524   3421216   6140880  36% /var
vagrant                    999297020 184249424 815047596  19% /vagrant
tmpfs                         202032         4    202028   1% /run/user/1000
suse-1:/homework/nfs/share  10476544   3392256   6169856  36% /mnt/nfs/share
```

12. Check the access

```sh
$ cd /mnt/nfs/share/
$ echo "test from suse-2" | tee /mnt/nfs/share/new-file-suse-2.txt
test from suse-2

$ ls -al
total 0
drwxrwxrwx 1 root    root  38 Feb  1 14:27 .
drwxr-xr-x 1 root    root  10 Feb  1 14:24 ..
-rw-r--r-- 1 vagrant users  0 Feb  1 14:27 new-file-suse-2.txt
```

13. Login on `suse-3` and install `nfs-client` if necessary.

```sh
$ sudo zypper install nfs-client
```

14. Create mount point on `suse-3`

```sh
$ sudo mkdir -pv /mnt/nfs/share
mkdir: created directory '/mnt/nfs'
mkdir: created directory '/mnt/nfs/share'
```

15. Mount **NFS** share

```sh
$ sudo mount -t nfs4 suse-1:/homework/nfs/share /mnt/nfs/share

$ df
Filesystem                 1K-blocks      Used Available Use% Mounted on
/dev/sda2                   10476524   3395596   6166516  36% /
devtmpfs                        4096         0      4096   0% /dev
tmpfs                        1010176         0   1010176   0% /dev/shm
tmpfs                         404072      6340    397732   2% /run
/dev/sda2                   10476524   3395596   6166516  36% /boot/grub2/i386-pc
/dev/sda2                   10476524   3395596   6166516  36% /boot/grub2/x86_64-efi
/dev/sda2                   10476524   3395596   6166516  36% /opt
/dev/sda2                   10476524   3395596   6166516  36% /home
/dev/sda2                   10476524   3395596   6166516  36% /root
/dev/sda2                   10476524   3395596   6166516  36% /srv
/dev/sda2                   10476524   3395596   6166516  36% /tmp
/dev/sda2                   10476524   3395596   6166516  36% /usr/local
/dev/sda2                   10476524   3395596   6166516  36% /var
vagrant                    999297020 184257124 815039896  19% /vagrant
tmpfs                         202032         4    202028   1% /run/user/1000
suse-1:/homework/nfs/share  10476544   3392256   6169856  36% /mnt/nfs/share
```

16. Test access form `suse-3`

```sh
$ cd /mnt/nfs/share/
$ cat new-file-suse-2.txt
test from suse-2

$ echo "test from suse-3" | tee new-file-suse-3.txt
tee: new-file-suse-3.txt: Read-only file system
test from suse-3
```

### Create an iSCSI disk-based target

1. On Target (suse-1) install required package

```sh
$ sudo zypper install targetcli-fb
```

2. Create a folder to store the iSCSI disk files

```sh
$ sudo mkdir -pv /var/lib/iscsi_disks/
mkdir: created directory '/var/lib/iscsi_disks/'
```

3. Start the admin tool

```sh
sudo targetcli
```

4. Create iSCSI disk

```sh
/> ls /
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 0]
  o- loopback ......................................................................................................... [Targets: 0]
  o- vhost ............................................................................................................ [Targets: 0]
  o- xen-pvscsi ....................................................................................................... [Targets: 0]
/> cd backstores/fileio
/backstores/fileio> create disk1 /var/lib/iscsi_disks/disk1.img 1G
Created fileio disk1 with size 1073741824
/backstores/fileio> ls
o- fileio ..................................................................................................... [Storage Objects: 1]
  o- disk1 ........................................................ [/var/lib/iscsi_disks/disk1.img (1.0GiB) write-back deactivated]
    o- alua ....................................................................................................... [ALUA Groups: 1]
      o- default_tg_pt_gp ........................................................................... [ALUA state: Active/optimized]
```

5. Create target IQN

```sh
/backstores/fileio> cd /iscsi
/iscsi> create iqn.2025-02.lab.homework:suse-1.target1
Created target iqn.2025-02.lab.homework:suse-1.target1.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
```

6. Create LUN

```sh
/iscsi> cd iqn.2025-02.lab.homework:suse-1.target1/tpg1/luns
/iscsi/iqn.20...et1/tpg1/luns> create /backstores/fileio/disk1
Created LUN 0.
```

7. Register initiator

```sh
/iscsi/iqn.20...et1/tpg1/luns> cd ../acls
/iscsi/iqn.20...et1/tpg1/acls> create iqn.2025-02.lab.homework:suse-2.init1
Created Node ACL for iqn.2025-02.lab.homework:suse-2.init1
Created mapped LUN 0.
```

8. Set username and password for initiator

```sh
/iscsi/iqn.20...et1/tpg1/acls> ls
o- acls .................................................................................................................. [ACLs: 1]
  o- iqn.2025-02.lab.homework:suse-2.init1 ........................................................................ [Mapped LUNs: 1]
    o- mapped_lun0 ........................................................................................ [lun0 fileio/disk1 (rw)]
/iscsi/iqn.20...et1/tpg1/acls> cd iqn.2025-02.lab.homework:suse-2.init1
/iscsi/iqn.20...:suse-2.init1> set auth userid=user-1
Parameter userid is now 'user-1'.
/iscsi/iqn.20...:suse-2.init1> set auth password=New_123123
Parameter password is now 'New_123123'.
```

9. Setup authentication flag of the target portal group (tpg1)

```sh
cd /iscsi/iqn.2025-02.lab.homework:suse-1.target1/tpg1/
/iscsi/iqn.20....target1/tpg1> set attribute authentication=1
Parameter authentication is now '1'.
/iscsi/iqn.20....target1/tpg1> cd /
/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 1]
  | | o- disk1 ...................................................... [/var/lib/iscsi_disks/disk1.img (1.0GiB) write-back activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2025-02.lab.homework:suse-1.target1 ........................................................................... [TPGs: 1]
  |   o- tpg1 .......................................................................................... [no-gen-acls, auth per-acl]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.2025-02.lab.homework:suse-2.init1 .................................................... [1-way auth, Mapped LUNs: 1]
  |     |   o- mapped_lun0 ................................................................................ [lun0 fileio/disk1 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ............................................... [fileio/disk1 (/var/lib/iscsi_disks/disk1.img) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]
  o- vhost ............................................................................................................ [Targets: 0]
  o- xen-pvscsi ....................................................................................................... [Targets: 0]
```

10. setup firewall

```sh
$ sudo firewall-cmd --add-service iscsi-target --permanent
success
$ sudo firewall-cmd --reload
success
```

11. Start the **iSCSI** service

```sh
$ sudo systemctl enable --now targetcli.service
Created symlink /etc/systemd/system/remote-fs.target.wants/targetcli.service → /usr/lib/systemd/system/targetcli.service.
```

12. Login on `suse-2` and install iscsi client

```sh
sudo zypper install open-iscsi
```

13. Set the initiator name `/etc/iscsi/initiatorname.iscsi`

```plain
##
## /etc/iscsi/initiatorname.iscsi
##
## Default iSCSI Initiatorname.
##
## DO NOT EDIT OR REMOVE THIS FILE!
## If you remove this file, the iSCSI daemon will not start.
## If you change the InitiatorName, existing access control lists
## may reject this initiator. The InitiatorName must be unique
## for each iSCSI initiator. Do NOT duplicate iSCSI InitiatorNames.

InitiatorName=iqn.2025-02.lab.homework:suse-2.init1
```

14. Set the behavior of initiator ib `/etc/iscsi/iscsid.conf`

```sh
# change the start mode to automatic
node.startup = automatic

# uncomment CHAP settings
node.session.auth.authmethod = CHAP

# uncoment and setup username and password
node.session.auth.username = user-1
node.session.auth.password = New_123123
```

15. Initiate a target discovery

```sh
$ sudo iscsiadm -m discovery -t sendtargets -p suse-1
192.168.99.101:3260,1 iqn.2025-02.lab.homework:suse-1.target1
```

16. Try to login on target (this also mean to attach the disk)

```sh
$ sudo iscsiadm -m node --login
Logging in to [iface: default, target: iqn.2025-02.lab.homework:suse-1.target1, portal: 192.168.99.101,3260]
Login to [iface: default, target: iqn.2025-02.lab.homework:suse-1.target1, portal: 192.168.99.101,3260] successful.

$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   10G  0 disk
├─sda1   8:1    0    8M  0 part
└─sda2   8:2    0   10G  0 part /var
                                /usr/local
                                /tmp
                                /srv
                                /root
                                /home
                                /boot/grub2/x86_64-efi
                                /opt
                                /boot/grub2/i386-pc
                                /
sdb      8:16   0    1G  0 disk
sr0     11:0    1 1024M  0 rom
```

17. Crearte patition, format and mount

```sh
$ sudo parted -s /dev/sdb -- mklabel msdos mkpart primary 16384s -0m

$ sudo mkfs.ext4 /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 260096 4k blocks and 65024 inodes
Filesystem UUID: 2f1dadf2-a6ac-4e65-a828-c509e3daaecb
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

$ sudo mount /dev/sdb1 /mnt/iscsi/

$ df
Filesystem     1K-blocks      Used Available Use% Mounted on
/dev/sda2       10476524   3435436   6126660  36% /
devtmpfs            4096         8      4088   1% /dev
tmpfs            1010176         0   1010176   0% /dev/shm
tmpfs             404072      6356    397716   2% /run
/dev/sda2       10476524   3435436   6126660  36% /boot/grub2/i386-pc
/dev/sda2       10476524   3435436   6126660  36% /opt
/dev/sda2       10476524   3435436   6126660  36% /boot/grub2/x86_64-efi
/dev/sda2       10476524   3435436   6126660  36% /home
/dev/sda2       10476524   3435436   6126660  36% /root
/dev/sda2       10476524   3435436   6126660  36% /srv
/dev/sda2       10476524   3435436   6126660  36% /tmp
/dev/sda2       10476524   3435436   6126660  36% /usr/local
/dev/sda2       10476524   3435436   6126660  36% /var
vagrant        999297020 184760992 814536028  19% /vagrant
tmpfs             202032         4    202028   1% /run/user/1000
/dev/sdb1        1005120        24    936696   1% /mnt/iscsi
```

18. Mount on boot

```sh
# take UUID of partition
$ sudo blkid /dev/sdb1
/dev/sdb1: UUID="2f1dadf2-a6ac-4e65-a828-c509e3daaecb" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="0dcb9cd6-01"

# add record in /etc/fstab
# iSCSI
UUID="2f1dadf2-a6ac-4e65-a828-c509e3daaecb" /mnt/iscsi ext4 _netdev 0 0
```

### Create a GlusterFS dispersed volume

1. Add repo for **GlusterFS** on `suse-1`, `suse-2` and `suse-3`

```sh
$ sudo zypper ar https://download.opensuse.org/repositories/home:/glusterfs:/SLES15SP5-10/15.5/home:glusterfs:SLES15SP5-10.repo
```

2. install the package

```sh
$ sudo zypper install glusterfs
```

3. Enable and start the service

```sh
$ sudo systemctl enable --now glusterd
Created symlink /etc/systemd/system/multi-user.target.wants/glusterd.service → /usr/lib/systemd/system/glusterd.service.
```

4. Create firewall service for GlusterFS - `sudo vi /etc/firewalld/services/glusterfs.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
<short>glusterfs-static</short>
<description>Default ports for gluster-distributed storage</description>
<port protocol="tcp" port="24007"/>    <!--For glusterd -->
<port protocol="tcp" port="24008"/>    <!--For glusterd RDMA port management -->
<port protocol="tcp" port="55555"/>    <!--For glustereventsd -->
<port protocol="tcp" port="38465"/>    <!--Gluster NFS service -->
<port protocol="tcp" port="38466"/>    <!--Gluster NFS service -->
<port protocol="tcp" port="38467"/>    <!--Gluster NFS service -->
<port protocol="tcp" port="38468"/>    <!--Gluster NFS service -->
<port protocol="tcp" port="38469"/>    <!--Gluster NFS service -->
<port protocol="tcp" port="49152-60999"/>  <!--Ports needed for bricks -->
</service>
```

5. Reload firewall service

```sh
$ sudo systemctl reload firewalld
```

6. Reload firewall settings

```sh
$ sudo firewall-cmd --add-service=glusterfs --permanent
success
$ sudo firewall-cmd --reload
success
```

7. Create partition on `/dev/sdb` and format

```sh
$ sudo parted -s /dev/sdb -- mklabel msdos mkpart primary 16384s -0m

$ sudo mkfs.ext4 /dev/sdb1
```

8. Mount partition

```sh
$ sudo mkdir -p /mnt/glusterfs

$ sudo mount /dev/sdb1 /mnt/glusterfs

$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   10G  0 disk
├─sda1   8:1    0    8M  0 part
└─sda2   8:2    0   10G  0 part /usr/local
                                /var
                                /boot/grub2/x86_64-efi
                                /tmp
                                /srv
                                /root
                                /opt
                                /home
                                /boot/grub2/i386-pc
                                /
sdb      8:16   0    5G  0 disk
└─sdb1   8:17   0    5G  0 part /mnt/glusterfs
sr0     11:0    1 1024M  0 rom
```

9. Test communication and add brics from `suse-1`.

```sh
$ sudo gluster peer probe suse-2
peer probe: success

$ sudo gluster peer probe suse-3
peer probe: success

$ sudo gluster peer status
Number of Peers: 2

Hostname: suse-2
Uuid: cf1a18be-bc27-44b7-8eea-1f8c8e975d3e
State: Peer in Cluster (Connected)

Hostname: suse-3
Uuid: f419e833-2449-4238-b264-f92f82c3539d
State: Peer in Cluster (Connected)
```

10. Create a volume

```sh
$ sudo gluster volume create homework-vol disperse 3 redundancy 1 suse-1:/mnt/glusterfs suse-2:/mnt/glusterfs suse-3:/mnt/glusterfs force
volume create: homework-vol: success: please start the volume to access data
```

11. Get information about the volume

```sh
$ sudo gluster volume info homework-vol

Volume Name: homework-vol
Type: Disperse
Volume ID: ab22ce1e-ea44-4f25-9f88-ec2f383f344f
Status: Created
Snapshot Count: 0
Number of Bricks: 1 x (2 + 1) = 3
Transport-type: tcp
Bricks:
Brick1: suse-1:/mnt/glusterfs
Brick2: suse-2:/mnt/glusterfs
Brick3: suse-3:/mnt/glusterfs
Options Reconfigured:
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on
```

12. Start the volume

```sh
$ sudo gluster volume start homework-vol
volume start: homework-vol: success
```

13. Login on `suse-4` and install neccessary package.

```sh
$ sudo zypper ar https://download.opensuse.org/repositories/home:/glusterfs:/SLES15SP5-10/15.5/home:glusterfs:SLES15SP5-10.repo

$ sudo zypper install glusterfs
```

14. Create mount point

```sh
$ sudo mkdir -p /mnt/glusterfs
```

14. mount the volume

```sh
$ sudo mount -t glusterfs suse-1:/homework-vol /mnt/glusterfs

$ df -hT
Filesystem           Type            Size  Used Avail Use% Mounted on
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /
devtmpfs             devtmpfs        4.0M     0  4.0M   0% /dev
tmpfs                tmpfs           987M     0  987M   0% /dev/shm
tmpfs                tmpfs           395M  6.2M  389M   2% /run
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /boot/grub2/i386-pc
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /boot/grub2/x86_64-efi
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /opt
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /tmp
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /srv
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /root
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /usr/local
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /home
/dev/sda2            btrfs            10G  3.3G  5.9G  36% /var
vagrant              vboxsf          954G  181G  773G  19% /vagrant
tmpfs                tmpfs           198M  4.0K  198M   1% /run/user/1000
suse-1:/homework-vol fuse.glusterfs  9.7G  102M  9.2G   2% /mnt/glusterfs
```

15. Write some files on mounted volume

```sh
$ echo "test" | sudo tee /mnt/glusterfs/test-file-0{1..9}
test

$ ls -al  /mnt/glusterfs/
total 9
drwxr-xr-x 1 root root 236 Feb  1 16:29 .
drwxr-xr-x 1 root root  18 Feb  1 16:21 ..
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-01
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-02
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-03
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-04
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-05
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-06
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-07
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-08
-rw-r--r-- 1 root root   5 Feb  1 16:29 test-file-09
```

16. Check the hashes on all bricks to ensure Dispersed mode

```sh
# suse-1
$ md5sum /mnt/glusterfs/test-file-01
24b1d3fc0aaf94b269dee4a6a0f1ff63  /mnt/glusterfs/test-file-01

# suse-2
$ md5sum /mnt/glusterfs/test-file-01
f66587b187a2e86849e0c5427292aace  /mnt/glusterfs/test-file-01

# suse-3
$ md5sum /mnt/glusterfs/test-file-01
9761724f232eedffa8a7857d2eac5c75  /mnt/glusterfs/test-file-01
```

We had different md5sum values for the same file (test-file-01) across all bricks. This means the file is being dispersed (erasure coded).
