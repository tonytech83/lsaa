# Research and implement a 389-ds single-node solution
## Create a directory with three users and two groups. One of the users should belong to two groups, while the other two just to one of them

### Server setup (192.168.99.101)
1. Install 389 Directory Server
```sh
$ sudo dnf install -y 389-ds-base
```
2.  Enable and start the service
```sh
$ sudo systemctl enable --now dirsrv.target
```
3. Configuring 389 Directory Server instance interactively.
```sh
$ sudo dscreate interactive
Install Directory Server (interactive mode)
===========================================

Enter system's hostname [ds.homework.lab]: ds.homework.lab

Enter the instance name [ds]: ds

Enter port number [389]: 389

Create self-signed certificate database [yes]: yes

Enter secure port number [636]: 636

Enter Directory Manager DN [cn=Directory Manager]: cn=Admin

Enter the Directory Manager password: 
Confirm the Directory Manager Password: 

Choose whether mdb or bdb is used. [bdb]: mdb

Enter the lmdb database size [12165339545.6]: 1073741824

Enter the database suffix (or enter "none" to skip) [dc=ds,dc=homework,dc=lab]: 

Create sample entries in the suffix [no]: no

Create just the top suffix entry [no]: no

Do you want to start the instance after the installation? [yes]: yes

Are you ready to install? [no]: yes
Starting installation ...
Validate installation settings ...
Create file system structures ...
Create self-signed certificate database ...
Perform SELinux labeling ...
Create database backend: dc=ds,dc=homework,dc=lab ...
Perform post-installation tasks ...
Completed installation for instance: slapd-ds
```
4. Check server status
```sh
$ sudo systemctl status dirsrv@ds           
● dirsrv@ds.service - 389 Directory Server ds.
     Loaded: loaded (/usr/lib/systemd/system/dirsrv@.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/dirsrv@.service.d
             └─custom.conf
     Active: active (running) since Tue 2025-02-11 08:33:29 EET; 58s ago
    Process: 6830 ExecStartPre=/usr/libexec/dirsrv/ds_systemd_ask_password_acl /etc/dirsrv/slapd-ds/dse.ldif (code=exited, status=0/SUCCESS)
    Process: 6835 ExecStartPre=/usr/libexec/dirsrv/ds_selinux_restorecon.sh /etc/dirsrv/slapd-ds/dse.ldif (code=exited, status=0/SUCCESS)
   Main PID: 6840 (ns-slapd)
     Status: "slapd started: Ready to process requests"
      Tasks: 24 (limit: 11085)
     Memory: 65.8M
        CPU: 435ms
     CGroup: /system.slice/system-dirsrv.slice/dirsrv@ds.service
             └─6840 /usr/sbin/ns-slapd -D /etc/dirsrv/slapd-ds -i /run/dirsrv/slapd-ds.pid

Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.834039938 +0200] - ERR - attrcrypt_cipher_init - No symmetric key found for>
Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.836105273 +0200] - INFO - attrcrypt_cipher_init - Key for cipher AES succes>
Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.837153066 +0200] - ERR - attrcrypt_cipher_init - No symmetric key found for>
Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.839517873 +0200] - INFO - attrcrypt_cipher_init - Key for cipher 3DES succe>
Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.840584415 +0200] - INFO - slapd_daemon - New referral entries are detected >
Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.875479458 +0200] - INFO - connection_table_new - Number of connection sub-t>
Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.901811951 +0200] - INFO - slapd_daemon - slapd started.  Listening on All I>
Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.903435068 +0200] - INFO - slapd_daemon - Listening on All Interfaces port 6>
Feb 11 08:33:29 ds.homework.lab ns-slapd[6840]: [11/Feb/2025:08:33:29.904844985 +0200] - INFO - slapd_daemon - Listening on /run/slapd-ds.socket >
Feb 11 08:33:29 ds.homework.lab systemd[1]: Started 389 Directory Server ds..
```
5. Allow LDAP traffic via firewall
```sh
# open standard ldap port 389
$ sudo firewall-cmd --permanent --add-service=ldap
success

# open secure ldap port 636
$ sudo firewall-cmd --permanent --add-service=ldaps
success

# apply changes
$ sudo firewall-cmd --reload
success
```
6. Perform LDAP search
```sh
$ ldapsearch -x -H ldap://localhost -D "cn=Admin" -W -b "dc=example,dc=com"
Enter LDAP Password: # enter the password
# extended LDIF
#
# LDAPv3
# base <dc=example,dc=com> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 32 No such object

# numResponses: 1
```
7. Prepare hole DS structure in one file `add_ds_structure.ldif`
```ldif
# Add root
dn: dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: domain
dc: ds
description: LDAP Root for ds.homework.lab

# Add Groups OU
dn: ou=Groups,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: organizationalUnit
ou: Groups

# Add People OU
dn: ou=People,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: organizationalUnit
ou: People

# Add two groups
dn: cn=Accountant,ou=Groups,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: groupOfNames
cn: Accountant

dn: cn=Commercial,ou=Groups,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: groupOfNames
cn: Commercial

# Add users
dn: uid=ivan.ivanov,ou=People,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Ivan
sn: Ivanov
displayName: Ivan Ivanov
uid: ivan.ivanov
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/ivan.ivanov
loginShell: /bin/bash
userPassword: pass_123456

dn: uid=mariya.ilieva,ou=People,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Mariya
sn: Ilieva
displayName: Mariya Ilieva
uid: mariya.ilieva
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/mariya.ilieva
loginShell: /bin/bash
userPassword: pass_123456

dn: uid=petar.radev,ou=People,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Petar
sn: Radev
displayName: Petar Radev
uid: petar.radev
uidNumber: 10003
gidNumber: 10003
homeDirectory: /home/petar.radev
loginShell: /bin/bash
userPassword: pass_123123

# Add users to groups
dn: cn=Accountant,ou=Groups,dc=ds,dc=homework,dc=lab
changetype: modify
add: member
member: uid=ivan.ivanov,ou=People,dc=ds,dc=homework,dc=lab
member: uid=mariya.ilieva,ou=People,dc=ds,dc=homework,dc=lab

dn: cn=Commercial,ou=Groups,dc=ds,dc=homework,dc=lab
changetype: modify
add: member
member: uid=ivan.ivanov,ou=People,dc=ds,dc=homework,dc=lab
member: uid=petar.radev,ou=People,dc=ds,dc=homework,dc=lab
```
8. Import file to Directory server.
```sh
$ ldapadd -x -D "cn=Admin" -W -H ldap://localhost -f add_ds_structure.ldif 
Enter LDAP Password: # enter the password
adding new entry "dc=ds,dc=homework,dc=lab"

adding new entry "ou=Groups,dc=ds,dc=homework,dc=lab"

adding new entry "ou=People,dc=ds,dc=homework,dc=lab"

adding new entry "cn=Accountant,ou=Groups,dc=ds,dc=homework,dc=lab"

adding new entry "cn=Commercial,ou=Groups,dc=ds,dc=homework,dc=lab"

adding new entry "uid=ivan.ivanov,ou=People,dc=ds,dc=homework,dc=lab"

adding new entry "uid=mariya.ilieva,ou=People,dc=ds,dc=homework,dc=lab"

adding new entry "uid=petar.radev,ou=People,dc=ds,dc=homework,dc=lab"

modifying entry "cn=Accountant,ou=Groups,dc=ds,dc=homework,dc=lab"

modifying entry "cn=Commercial,ou=Groups,dc=ds,dc=homework,dc=lab"
```
9. Verify structure after import
```sh
$ ldapsearch -x -LLL -H ldap://localhost -D "cn=Admin" -W -b "dc=ds,dc=homework,dc=lab"
Enter LDAP Password: # enter the password
dn: dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: domain
dc: ds
description: LDAP Root for ds.homework.lab

dn: ou=Groups,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: organizationalUnit
ou: Groups

dn: ou=People,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: organizationalUnit
ou: People

dn: cn=Accountant,ou=Groups,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: groupOfNames
cn: Accountant
member: uid=ivan.ivanov,ou=People,dc=ds,dc=homework,dc=lab
member: uid=mariya.ilieva,ou=People,dc=ds,dc=homework,dc=lab

dn: cn=Commercial,ou=Groups,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: groupOfNames
cn: Commercial
member: uid=ivan.ivanov,ou=People,dc=ds,dc=homework,dc=lab
member: uid=petar.radev,ou=People,dc=ds,dc=homework,dc=lab

dn: uid=ivan.ivanov,ou=People,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Ivan
sn: Ivanov
displayName: Ivan Ivanov
uid: ivan.ivanov
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/ivan.ivanov
loginShell: /bin/bash
userPassword:: e1BCS0RGMi1TSEE1MTJ9MTAwMDAkRVBvRzE0TndCU3BIa3F6QjFzSVR3WUVwQi9
 qcmw2L3QkWDdieEZ4UjNvMzdueWs2aGE4MFZML21QMGgzYjVpWVZFM1NzSTZHQUJ6NURmRmRMcUJT
 NnN4SVJPVGVIOFZlOTA0SzNQUXRPbTBmeFAyRktFMEU3dlE9PQ==

dn: uid=mariya.ilieva,ou=People,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Mariya
sn: Ilieva
displayName: Mariya Ilieva
uid: mariya.ilieva
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/mariya.ilieva
loginShell: /bin/bash
userPassword:: e1BCS0RGMi1TSEE1MTJ9MTAwMDAkOXMrbTcxaWZkK1RBVzVoRDN4WW5RdldWb2J
 LNTVmQSskV1VuaFVWMmx0czBheFpiUTBuWlpwNEFvcE1YQk1rRzAwZnhSZkpvWXMyT3hNRzFpTEhL
 djhWa3JDYmx1cjhqVDV5MXY4RVdKcysvbnlBUFNWdWE2T1E9PQ==

dn: uid=petar.radev,ou=People,dc=ds,dc=homework,dc=lab
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Petar
sn: Radev
displayName: Petar Radev
uid: petar.radev
uidNumber: 10003
gidNumber: 10003
homeDirectory: /home/petar.radev
loginShell: /bin/bash
userPassword:: e1BCS0RGMi1TSEE1MTJ9MTAwMDAkRnh3SDJ2ZDNzTlREa1lNMytPK0VtR1Vwazd
 IRXRWSUgkYUZob1oyaDF1WnhwVWhwMnE3Q2Q3d0VuSTlaZS9DVkRsK0plMFZPZVlubG1rSHNOWEZI
 RDNacXN4NVFvWmtFRzFaOTB2S2FCT1hXK3JvTENBdnordUE9PQ==
```
### Clien setup (192.168.99.102)
1. Install required packages
```sh
$ sudo dnf install -y sssd sssd-ldap oddjob-mkhomedir authselect-compat openldap-clients
```
Packages:
- **sssd** → Core SSSD package.
- **sssd-ldap** → LDAP support for SSSD.
- **oddjob-mkhomedir** → Automatically creates home directories for LDAP users.
- **authselect-compat** → Helps configure authentication.
- **openldap-clients** → Provides LDAP utilities like ldapsearch.
2. Enable and start service
```sh
$ sudo systemctl enable --now sssd oddjobd
Created symlink /etc/systemd/system/multi-user.target.wants/oddjobd.service → /usr/lib/systemd/system/oddjobd.service.
```
3. Test connectivity to LDAP server
```sh
$ ldapsearch -x -H ldap://192.168.99.101 -b "dc=ds,dc=homework,dc=lab"
# extended LDIF
#
# LDAPv3
# base <dc=ds,dc=homework,dc=lab> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 0 Success

# numResponses: 1
```
4. Configure SSSD. Creare or edit `/etc/sssd/sssd.conf`
```conf
[sssd]
services = nss, pam
config_file_version = 2
domains = ldap

[domain/ldap]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap
cache_credentials = True

# LDAP Server and Search Base
ldap_uri = ldap://192.168.99.101
ldap_search_base = dc=ds,dc=homework,dc=lab
ldap_default_bind_dn = cn=Admin
ldap_default_authtok_type = password
ldap_default_authtok = New_123123
ldap_tls_reqcert = never # Force disable tls (for testing only)

# User and Group Mappings
ldap_user_search_base = ou=People,dc=ds,dc=homework,dc=lab
ldap_group_search_base = ou=Groups,dc=ds,dc=homework,dc=lab

# Map LDAP attributes to local Unix attributes
ldap_user_object_class = inetOrgPerson
ldap_user_name = uid
ldap_user_uid_number = uidNumber
ldap_user_gid_number = gidNumber
ldap_user_home_directory = homeDirectory
ldap_user_shell = loginShell

ldap_group_object_class = groupOfNames
ldap_group_name = cn

# Enable enumeration (optional but useful for debugging)
enumerate = True

# Enable home directory creation on first login
override_homedir = /home/%u
```
5. Set permissions on file
```sh
$ sudo chmod 600 /etc/sssd/sssd.conf
```
6. Restart **sssd** service
```sh
$ sudo systemctl restart sssd
```
7. Verify LDAP user lookup
```sh
$ getent passwd ivan.ivanov
ivan.ivanov:*:10001:10001:Ivan:/home/ivan.ivanov:/bin/bash
```
8. Try login with LDAP user
```sh
$ su - ivan.ivanov
Password: # enter LDAP user password
Creating home directory for ivan.ivanov.
Last failed login: Tue Feb 11 09:27:11 EET 2025 on pts/0
There were 6 failed login attempts since the last successful login.

[ivan.ivanov@client ~]$

[ivan.ivanov@client ~]$ id
uid=10001(ivan.ivanov) gid=10001 groups=10001 context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```