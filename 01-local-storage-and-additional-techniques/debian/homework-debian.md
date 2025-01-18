## Preparations

1. Add **contrib** repository
```sh
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository contrib
```
2. Install the ZFS necessary packages
```sh
$ sudo apt-get update
$ sudo apt-get install zfsutils-linux
```
3. Create folder for mount point
```sh
$ sudo mkdir -p /homework/zfs
```