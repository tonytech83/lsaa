## Create a RAID10-based pool in ZFS out of six devices each 5 GB in size. Do it in two different configurations – 3x2 and 2x3
### Preparations
1. Add the ZFS on Linux repository
```sh
$ sudo dnf install https://zfsonlinux.org/epel/zfs-release-2-3.el9.noarch.rpm
$ sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-openzfs-el-9
```
2. Install the **kABI-tracking** kmods the default repository in the /etc/yum.repos.d/zfs.repo file must be switch from **zfs** to **zfs-kmod**
```sh
$ sudo dnf config-manager --enable zfs-kmod
```
3. Install the ZFS packages
```sh
$ sudo dnf install zfs
```
4. Autoload the module
```sh
$ echo "zfs" | sudo tee -a /etc/modules-load.d/zfs.conf
```
#### Creating 3x2 Configuration (three mirrored pairs)
Each mirrored pair is created using two devices, and then the three pairs are striped.
1. Create a mount point
```sh
$ sudo mkdir -p /homework/zfs 
```
2. Crate a RAID10 3x2
```sh
$ sudo zpool create -m /homework/zfs raid10_3x2 mirror /dev/sdb /dev/sdc mirror /dev/sdd /dev/sde mirror /dev/sdf /dev/sdg

$ sudo zpool status raid10_3x2
  pool: raid10_3x2
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        raid10_3x2  ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0
          mirror-1  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0
          mirror-2  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors
```
3. Clean up
```sh
$ sudo umount /homework/zfs
$ sudo zpool destroy raid10_3x2
$ sudo wipefs --all /dev/sd[b-g]
```
#### Creating 2x3 Configuration (two groups of three striped devices, mirrored together)
Three devices are striped together in two groups, and these groups are mirrored.
1. Crate a RAID10 2x3
```sh
$ sudo zpool create -m /homework/zfs raid10_2x3 mirror /dev/sdb /dev/sdc /dev/sdd mirror /dev/sde /dev/sdf /dev/sdg

$ sudo zpool status raid10_2x3
  pool: raid10_2x3
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        raid10_2x3  ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0
            sdd     ONLINE       0     0     0
          mirror-1  ONLINE       0     0     0
            sde     ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors
```
2. Clean up
```sh
$ sudo umount /homework/zfs
$ sudo zpool destroy raid10_3x2
$ sudo wipefs --all /dev/sd[b-g]
```
## Create a RAID6-based pool in ZFS out of five devices each 5 GB in size
1. Create a mount point
```sh
$ sudo mkdir -p /homework/zfs-raid6
```
2. Create the RAID-Z2 Pool
```sh
$ sudo zpool create -m /homework/zfs-raid6 raid6_pool raidz2 /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf

$ sudo zpool status raid6_pool
  pool: raid6_pool
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        raid6_pool  ONLINE       0     0     0
          raidz2-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0
            sdf     ONLINE       0     0     0

errors: No known data errors
```
3. Perform test 
```sh
$ sudo touch /homework/zfs-raid6/testfile
$ ls -l /homework/zfs-raid6/
total 1
-rw-r--r--. 1 root root 0 Jan 18 15:15 testfile
```
4. Clean up
```sh
$ sudo umount /homework/zfs-raid6
$ sudo zpool destroy raid6_pool
$ sudo wipefs --all /dev/sd[b-f]
```
## Research on how to use a key file to automount an encrypted volume on boot and demonstrate it for one drive
```sh
$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                       8:0    0   20G  0 disk
├─sda1                    8:1    0    1G  0 part /boot
└─sda2                    8:2    0   19G  0 part
  ├─almalinux_vbox-root 253:0    0   17G  0 lvm  /
  └─almalinux_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                       8:16   0    5G  0 disk
sdc                       8:32   0    5G  0 disk
sdd                       8:48   0    5G  0 disk
sde                       8:64   0    5G  0 disk
sdf                       8:80   0    5G  0 disk
sdg                       8:96   0    5G  0 disk
sr0                      11:0    1 1024M  0 rom
```
#### Prepare encrypted volume
1. Check is crypt module is available
```sh
$ grep -i DM_CRYPT /boot/config-$(uname -r)
CONFIG_DM_CRYPT=m
```
2. Load the module
```sh
$ sudo modprobe dm_crypt

$ sudo lsmod | grep dm_crypt
dm_crypt               69632  0
dm_mod                249856  10 dm_crypt,dm_log,dm_mirror
```
3. Install crypt tools
```sh
$ sudo dnf install cryptsetup
```
4. Create partition on sdb drive
```sh
$ sudo parted -s /dev/sdb -- mklabel msdos mkpart primary 2048s 5G
```
5. Initialize LUKS encryption
```sh
$ sudo cryptsetup luksFormat /dev/sdb1
WARNING: Device /dev/sdb1 already contains a 'zfs_member' superblock signature.

WARNING!
========
This will overwrite data on /dev/sdb1 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdb1:
Verify passphrase: # enter twice same passcode
```
6. Open the encrypted volume
```sh
$ sudo cryptsetup open /dev/sdb1 encrypted_volume
Enter passphrase for /dev/sdb1: # enter the passcode
```
7. Format the volume
```sh
$ sudo mkfs.xfs /dev/mapper/encrypted_volume
meta-data=/dev/mapper/encrypted_volume isize=512    agcount=4, agsize=326592 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=1306368, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
8. Mount the volume
```sh
$ sudo mkdir -p /homework/encr-data

$ sudo mount /dev/mapper/encrypted_volume /homework/encr-data
```
#### Generate a key file and configure autofs
1. Create a random key file
```sh
$ sudo mkdir -p /etc/luks-keys

$ sudo dd if=/dev/urandom of=/etc/luks-keys/crypto_keyfile.bin bs=1024 count=4
4+0 records in
4+0 records out
4096 bytes (4.1 kB, 4.0 KiB) copied, 0.00267606 s, 1.5 MB/s
```
2. Set proper permissions
```sh
$ sudo chmod 400 /etc/luks-keys/crypto_keyfile.bin
$ sudo chown root:root /etc/luks-keys/crypto_keyfile.bin
```
3. Add the key file to LUKS volume
```sh
$ sudo cryptsetup luksAddKey /dev/sdb1 /etc/luks-keys/crypto_keyfile.bin
Enter any existing passphrase: # enter the passcode
```
4. Install autofs
```sh
$ sudo dnf update
$ sudo dnf install autofs
```
5. Enable and start autofs service
```sh
$ sudo systemctl enable autofs
$ sudo systemctl start autofs
```
6. Add line to map file `sudo vi /etc/auto.master.d/encrypted.autofs`. The timeout option - how long device stay inactive before unmount.
```plain
/homework /etc/auto.encrypted --timeout=60
```
7. Create the mapping file `sudo vi /etc/auto.encrypted`.
```plain
encr-data -fstype=xfs,rw :/dev/mapper/encrypted_volume
```
8. Configure crypttab `sudo vi /etc/crypttab`.
```plain
encrypted_volume /dev/sdb1 /etc/luks-keys/crypto_keyfile.bin luks
```
9. Set correct context for the key file
```sh
$ sudo semanage fcontext -a -t unlabeled_t "/etc/luks-keys/crypto_keyfile.bin"
$ sudo restorecon -v /etc/luks-keys/crypto_keyfile.bin
```
8 Restart autofs service.
```sh
$ sudo systemctl restart autofs
```
#### Test with reboot of VM
1. Try accessing the directory
```sh
Last login: Sun Jan 19 09:58:49 2025 from 10.0.2.2
[vagrant@alma ~]$ ls /homework/encr-data
[vagrant@alma ~]$ ls -al /homework/encr-data
total 0
drwxr-xr-x. 2 root root 6 Jan 19 09:07 .
drwxr-xr-x. 3 root root 0 Jan 19 10:54 ..
```
2. Check if it's mounted
```sh
$ mount | grep encrypted_volume
/dev/mapper/encrypted_volume on /homework/encr-data type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```