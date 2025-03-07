# Practice M8: Exam Preparation

## Sample Exam

### Infrastructure

You will have to accomplish a set of tasks in the following infrastructure
```plain
                                                                              Network: 192.168.10.0/24
------+------------------------+---------------+---------------+---------------+---------------+------
      |                        |               |               |               |               |               
      |.10                     |.20            |.30            |.40            |.50            |.60      
+-----+-----+   +-----+  +-----+-----+   +-----+-----+   +-----+-----+   +-----+-----+   +-----+-----+  
|           |   |     |  |           |   |           |   |           |   |           |   |           |   
|           |---| 1GB |  |           |   |           |   |           |   |           |   |           |  
|           |   |     |  |           |   |           |   |           |   |           |   |           |  
|    STR    |   +-----+  |    CNT    |   |    LBA    |   |    WBA    |   |    WBB    |   |    MON    |  
|   (VM1)   |            |   (VM2)   |   |   (VM3)   |   |   (VM4)   |   |   (VM5)   |   |   (VM6)   |  
|           |   +-----+  |           |   |           |   |           |   |           |   |           | 
|           |   |     |  |           |   |           |   |           |   |           |   |           | 
|           |---| 1GB |  |           |   |           |   |           |   |           |   |           | 
+-----------+   |     |  +-----------+   +-----------+   +-----------+   +-----------+   +-----------+   
                +-----+
```
## Tasks checklist

### Containers [12 pts]
Demonstrate knowledge and readiness to work with application containers:
- (T101 / 3 pts) Install **Docker** on the **CNT** machine
  - Install repository
    ```sh
    sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
    ```    
  - Install the latest version
    ```sh
    sudo dnf install docker-ce
    ```   
- (T102 / 2 pts) Configure **Docker** to start on boot and allow the **exam** user to use it without the need of **sudo**
  - Ensure **Docker** starts on boot
    ```sh
    sudo systemctl enable --now docker
    ```
  - Add user **exam** to **docker** group
    ```sh
    sudo usermod -aG docker exam
    ```
- (T103 / 2 pts) Start (execute) a container named **tiger** based on the **hub.zahariev.pro/hello-world** image
  ```sh
  docker run -it --name tiger hub.zahariev.pro/hello-world
  ```
  
Start a container based on the **hub.zahariev.pro/httpd** that meets the following requirements:

- (T104 / 1 pts) It is run in **detached** (background) mode

- (T105 / 1 pts) It is named **httpd-web**

- (T106 / 1 pts) Port **80** of the container is forwarded to port **8080** on the **CNT** machine

- (T107 / 2 pts) The default web page (or the whole folder) in the container is mounted from a local folder (**~/html**) containing **index.html** with the sentence **Linux Guru**

  - Execute docker command
    ```sh
    docker run -d --name httpd-web -p 8080:80 -v ~/html:/usr/local/apache2/htdocs/ hub.zahariev.pro/httpd
    ```
  - Test with curl
    ```sh
    curl http://localhost:8080
    ```
  
*More about those container images could be found here: http://hub.zahariev.pro/index.html*

### Storage [16 pts]
Demonstrate knowledge and readiness to work with storage related technologies (mostly working on **STR**)

Create a simple **software RAID** with **MD**:

- (T201 / 2 pts) Utilize the two extra drives (each 1 GB) to create a **RAID1** device **/dev/md0**
  - Create partition on `/dev/sdb`
    ```sh
    sudo parted -s /dev/sdb -- mklabel msdos mkpart primary 2048s -0m set 1 raid on
    ```
  - Create partition on `/dev/sdc`
    ```sh
    sudo parted -s /dev/sdc -- mklabel msdos mkpart primary 2048s -0m set 1 raid on
    ```
  - Create RAID 1
    ```sh
    sudo mdadm --create /dev/md0 --level 1 --raid-devices 2 /dev/sd{b,c}1
    ```
- (T202 / 1 pts) Make the configuration **persistent** (alter the appropriate file, for example **/etc/mdadm.conf**)
  - Dump the configuration 
    ```sh
    sudo mdadm --detail --brief /dev/md0 | sudo tee -a /etc/mdadm.conf
    ```
- (T203 / 1 pts) Create an **ETX4** file system on the **RAID device**
  ```sh
  sudo mkfs.ext4 /dev/md0
  ```
- (T204 / 2 pts) Mount it at **/storage/raidmd** and add a record in the **/etc/fstab**
  - Create mounting point `/storage/raidmd/`
    ```sh
    sudo mkdir -p /storage/raidmd/
    ```
  - Do test mount
    ```sh
    sudo mount /dev/md0 /storage/raidmd
    ```
  - Take the UUID of file system
    ```sh
    sudo blkid /dev/md0
    ```
  - Open for editing `/etc/fstab` and add new line
    ```plain
    # RAID mount
    UUID="71b02b0c-a22e-4ee6-919d-5f847b051775"     /storage/raidmd         ext4   defaults        0 0
    ```
  - Unmount the attached file system
    ```sh
    sudo umount /storage/raidmd
    ```
  - Reload all drives from `/etc/fstab`
    ```sh
    sudo mount -av
    ```
Create an **iSCSI** storage configuration:

- (T205 / 1 pts) Install the necessary **iSCSI**-related packages on the **STR** machine
  ```sh
  sudo dnf install targetcli
  ```
- (T206 / 1 pts) Create an **iSCSI target** pointing to a **1 GB** disk image file named **D1GB.img**
  - Create folder where will store virtual disks
    ```sh
    sudo mkdir /var/lib/iscsi-images
    ```
  - Execute targetcli tool
    ```sh
    sudo targetcli
    ```
  - Create disk image file
    ```sh
    backstores/fileio create D1 /var/lib/iscsi-images/D1GB.img 1G
    ```
  - Create target IQN
    ```sh
    iscsi/ create iqn.2025-03.lab.exam:str.target1
    ```
  - Create LUN
    ```sh
    iscsi/iqn.2025-03.lab.exam:str.target1/tpg1/luns create /backstores/fileio/D1
    ```
- (T207 / 1 pts) Register the **LBA** machine as an **initiator** for the disk
  - Register initiator
    ```
    iscsi/iqn.2025-03.lab.exam:str.target1/tpg1/acls create iqn.2025-03.lab.exam:lba.init1
    ```
  - Start and enable service
    ```sh
    sudo systemctl enable --now targe
    ```
  - Setup firewall
    ```sh
    sudo firewall-cmd --add-service iscsi-target --permanent
    sudo firewall-cmd --reload
    ```
- (T208 / 2 pts) Create an **XFS** file system on the **iSCSI** disk on the **LBA** machine and add an entry in the **/etc/fstab** to mount it to **/storage/iscsi**
  - Install iSCSI client
    ```sh
    sudo dnf install iscsi-initiator-utils
    ```
  - Set the initiator name `/etc/iscsi/initiatorname.iscsi`
    ```sh
    InitiatorName=iqn.2025-03.lab.exam:lba.init1
    ```
  - Start and enable the service
    ```sh
    sudo systemctl enable --now iscsi
    ```
  - Request targets from target server
    ```sh
    sudo iscsiadm -m discovery -t sendtargets -p str
    ```
  - Login (not needed in this test scenario)
    ```sh
    sudo iscsiadm -m node --login
    ```
  - Create partition
    ```sh
    sudo parted -s /dev/sdb -- mklabel msdos mkpart primary 16384s -0m
    ```
  - Create file system
    ```sh
    sudo mkfs.xfs /dev/sdb1
    ```
  - Create mounting point
    ```sh
    sudo mkdir -p /storage/iscsi
    ```
  - Test mount
    ```sh
    sudo mount /dev/sdb1 /storage/iscsi
    ```
  - Take UUID of file system
    ```sh
    sudo blkid /dev/sdb1
    ```
  - Unmount file system
    ```sh
    sudo umount /storage/iscsi/
    ```
  - Add line to `/etc/fstab`
    ```plain
    # iSCSI
    UUID="4ac37c73-ee88-41c9-be57-2baeccd0c421" /storage/iscsi/     xfs     _netdev 0 0
    ```
  - Reload all drives from `/etc/fstab`
    ```sh
    sudo mount -av
    ```
Create a simple **NFS** export:

- (T209 / 1 pts) Install **NFS** on the **STR** machine

- (T210 / 2 pts) Export the **/storage/nfs** folder with **rw** permissions for **WBA** and **WBB**

- (T211 / 2 pts) Mount the export on both the **WBA** and **WBB** machines at **/storage/export** and add a record to the **/etc/fstab**

### Load Balancing [8 pts]

Demonstrate knowledge and readiness to work with load balancing solutions:

- (T301 / 2 pts) Install **HAProxy** on the **LBA** machine

- (T302 / 2 pts) Create a front-end configuration block to listen on port **8080**

- (T303 / 2 pts) Create a back-end configuration block that contains **WBA** and **WBB** with **roundrobin** algorithm

- (T304 / 2 pts) Make sure that when accessed the load balancer will redirect to the added **virtual host** (port **8080**) on both web servers

### Configuration Management [7 pts]

Demonstrate knowledge and readiness to work with configuration management solutions (on the **LBA** machine against both **WBA** and **WBB** machines):

- (T401 / 1 pts) Install **Ansible** on the **LBA** machine

- (T402 / 1 pts) Create **system-level inventory** that includes both **WBA** and **WBB** machines grouped as **web**

- Then create a playbook **~/exam.yml** (or **~/exam.yaml**) that when executed will do the following

    - (T403 / 1 pts) Refers to the **web** group of hosts

    - (T404 / 1 pts) Installs **NGINX** on them

    - (T405 / 1 pts) Configures the service to be **started** and configured to **start on boot**

    - (T406 / 1 pts) Changes the **default web page** to a page, containing the text **LSAA Exam**

- (T407 / 1 pts) Executes successfully

### Web Servers [5 pts]

Demonstrate knowledge and readiness to work with web servers (on both **WBA** and **WBB** machines):

- (T501 / 4 pts) Create a **virtual host** and set it to **listen** on port **8080**

- (T502 / 1 pts) Create an index page for the virtual host, that shows your **SoftUni ID** (this is your SoftUni @username) and the name of the host. For example, the result for host **WBA** may look like this **@student on WBA**

*Note that the web servers should be installed as part of the activities included in the **Configuration Management** section and the virtual hosts should be created manually as a solution to the tasks in this section*

### Monitoring [12 pts]

Demonstrate knowledge and readiness to work with monitoring solutions:

- (T601 / 3 pts) Install **Nagios** on the **MON** machine and make sure it is **started** and set to **start on boot**

- (T602 / 1 pts) Create a **nagiosadmin** user with password set to **LSAA-Exam**

- (T603 / 2 pts) Create a file **/etc/nagios/objects/exam-xxx.cfg** for every machine except the **MON**, where **xxx** is the name of the machine (adjust the path to match your situation). For example, **exam-cnt.cfg**, **exam-str.cfg**, etc. Include those files in the main configuration file

- (T604 / 4 pts) Adjust configurations made in **T603** to include the nodes (all except MON) for monitoring by adding their object definitions

- (T605 / 2 pts) Setup monitoring of the **HTTP** service running on **WBA** and **WBB** in the appropriate configuration files created earlier (**exam-wba.cfg** and **exam-wbb.cfg**)