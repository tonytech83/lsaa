## Preparations

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

## Creating 3x2 Configuration (three mirrored pairs)
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
## Creating 2x3 Configuration (two groups of three striped devices, mirrored together)
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