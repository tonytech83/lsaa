## Create a RAID10-based pool in ZFS out of six devices each 5 GB in size. Do it in two different configurations – 3x2 and 2x3

### Preparations

1. Add the ZFS on Linux repository and refresh

```sh
$ sudo zypper addrepo https://download.opensuse.org/repositories/filesystems/15.6/filesystems.repo
$ sudo zypper refresh
```

2. Install the ZFS packages

```sh
$ sudo zypper install zfs
```

3. Autoload the module

```sh
$ echo "zfs" | sudo tee -a /etc/modules-load.d/zfs.conf
$ echo "allow_unsupported_modules 1" | sudo tee /etc/modprobe.d/10-unsupported-modules.conf
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

$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   10G  0 disk
├─sda1   8:1    0    8M  0 part
└─sda2   8:2    0   10G  0 part /var
                                /usr/local
                                /tmp
                                /srv
                                /root
                                /opt
                                /home
                                /boot/grub2/x86_64-efi
                                /boot/grub2/i386-pc
                                /
sdb      8:16   0    5G  0 disk
├─sdb1   8:17   0    5G  0 part
└─sdb9   8:25   0    8M  0 part
sdc      8:32   0    5G  0 disk
├─sdc1   8:33   0    5G  0 part
└─sdc9   8:41   0    8M  0 part
sdd      8:48   0    5G  0 disk
├─sdd1   8:49   0    5G  0 part
└─sdd9   8:57   0    8M  0 part
sde      8:64   0    5G  0 disk
├─sde1   8:65   0    5G  0 part
└─sde9   8:73   0    8M  0 part
sdf      8:80   0    5G  0 disk
├─sdf1   8:81   0    5G  0 part
└─sdf9   8:89   0    8M  0 part
sdg      8:96   0    5G  0 disk
sr0     11:0    1 1024M  0 rom

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
-rw-r--r-- 1 root root 0 Jan 18 15:17 testfile
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
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   10G  0 disk
├─sda1   8:1    0    8M  0 part
└─sda2   8:2    0   10G  0 part /var
                                /usr/local
                                /tmp
                                /srv
                                /root
                                /opt
                                /home
                                /boot/grub2/x86_64-efi
                                /boot/grub2/i386-pc
                                /
sdb      8:16   0    5G  0 disk
sdc      8:32   0    5G  0 disk
sdd      8:48   0    5G  0 disk
sde      8:64   0    5G  0 disk
sdf      8:80   0    5G  0 disk
sdg      8:96   0    5G  0 disk
sr0     11:0    1 1024M  0 rom
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
dm_crypt               65536  0
dm_mod                237568  1 dm_crypt
```

3. Create partition on sdb drive

```sh
$ sudo parted -s /dev/sdb -- mklabel msdos mkpart primary 2048s 5G
```

4. Initialize LUKS encryption on `/dev/sdb1`

```sh
$ sudo cryptsetup luksFormat /dev/sdb1
WARNING: Device /dev/sdb1 already contains a 'zfs_member' superblock signature.

WARNING!
========
This will overwrite data on /dev/sdb1 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdb1:
Verify passphrase: # enter passcode twice
```

5. Open the encrypted volume

```sh
$ sudo cryptsetup open /dev/sdb1 encrypted_volume
Enter passphrase for /dev/sdb1: # enter the passcode
```

6. Format the volume

```sh
$ sudo mkfs.ext3 /dev/mapper/encrypted_volume
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 1309952 4k blocks and 327680 inodes
Filesystem UUID: 85b5215e-e1dc-49ac-b7e8-83466d11c049
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

7. Mount the volume

```sh
$ sudo mkdir -p /homework/encr-data
$ sudo mount /dev/mapper/encrypted_volume /homework/encr-data
$ echo 'Hello encrypted drive' | sudo tee /homework/encr-data/hello.txt
Hello encrypted drive
```

#### Generate a key file and configure autofs

1. Create a random key file

```sh
$ sudo dd if=/dev/urandom of=/root/crypto_keyfile.bin bs=1024 count=4
4+0 records in
4+0 records out
4096 bytes (4.1 kB, 4.0 KiB) copied, 0.00425322 s, 963 kB/s
```

2. Secure the key file

```sh
$ sudo chmod 400 /root/crypto_keyfile.bin
```

3. Add the key file to LUKS volume

```sh
$ sudo cryptsetup luksAddKey /dev/sdb1 /root/crypto_keyfile.bin
Enter any existing passphrase: # enter passcode
```

4. Add line to map file `sudo nano /etc/auto.master`. The timeout option - how long device stay inactive before unmount.

```plain
/homework /etc/auto.encrypted --timeout=60
```

5. Create the mapping file `sudo nano /etc/auto.encrypted`.

```plain
encr-data -fstype=ext3 :/dev/mapper/encrypted_volume
```

6. Configure crypttab to use the key file `sudo nano /etc/crypttab`.

```plain
encrypted_volume /dev/sdb1 /root/crypto_keyfile.bin luks
```

7. Enable and start **autofs** service.

```sh
$ sudo systemctl enable autofs.service
Created symlink /etc/systemd/system/multi-user.target.wants/autofs.service → /usr/lib/systemd/system/autofs.service.

$ sudo systemctl start autofs.service
```

#### Test

1. Try accessing the directory

```sh
Have a lot of fun...
Last login: Sat Jan 18 15:06:18 2025 from 10.0.2.2
vagrant@suse:~> ls /homework/encr-data
hello.txt  lost+found
```

2. Check if it's mounted

```sh
$ mount | grep encrypted_volume
/dev/mapper/encrypted_volume on /homework/encr-data type ext3 (rw,relatime)
```
