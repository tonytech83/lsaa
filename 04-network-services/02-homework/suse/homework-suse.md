# Research and implement a 389-ds single-node solution
## Create a directory with three users and two groups. One of the users should belong to two groups, while the other two just to one of them

###Server setup (192.168.99.101)
1. Install 389 Directory Server
```sh
$ sudo zypper refresh
$ sudo zypper install 389-ds
```
2. Initialize and configure the 389 Directory Server
```sh
$ sudo dscreate interactive
Install Directory Server (interactive mode)
===========================================
selinux is disabled, will not relabel ports or files.

Selinux support will be disabled, continue? [yes]: yes

Enter system's hostname [ds.homework.lab]: ds.homework.lab

Enter the instance name [ds]: ds

Enter port number [389]: 389

Create self-signed certificate database [yes]: yes

Enter secure port number [636]: 636

Enter Directory Manager DN [cn=Directory Manager]: cn=Admin

Enter the Directory Manager password: 
Confirm the Directory Manager Password: 

Enter the database suffix (or enter "none" to skip) [dc=ds,dc=homework,dc=lab]: 

Create sample entries in the suffix [no]: no

Create just the top suffix entry [no]: no

Do you want to start the instance after the installation? [yes]: yes

Are you ready to install? [no]: yes
Starting installation ...
Validate installation settings ...
Create file system structures ...
Create self-signed certificate database ...
selinux is disabled, will not relabel ports or files.
selinux is disabled, will not relabel ports or files.
Create database backend: dc=ds,dc=homework,dc=lab ...
Perform post-installation tasks ...
Completed installation for instance: slapd-ds
```
3. Enable and start 389 Directory Server service
```sh
$ sudo systemctl enable --now dirsrv@ds

$ sudo systemctl status dirsrv@ds
● dirsrv@ds.service - 389 Directory Server ds.
     Loaded: loaded (/usr/lib/systemd/system/dirsrv@.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/dirsrv@.service.d
             └─krbkdcbefore.conf
     Active: active (running) since Tue 2025-02-11 09:49:56 EET; 1min 4s ago
   Main PID: 28378 (ns-slapd)
     Status: "slapd started: Ready to process requests"
      Tasks: 29 (limit: 2333)
        CPU: 614ms
     CGroup: /system.slice/system-dirsrv.slice/dirsrv@ds.service
             └─28378 /usr/sbin/ns-slapd -D /etc/dirsrv/slapd-ds -i /run/dirsrv/slapd-ds.pid

Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.659065377 +0200] - ERR - attrcrypt_cipher_init - No symmetric key found for cipher AES >
Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.662136545 +0200] - INFO - attrcrypt_cipher_init - Key for cipher AES successfully gener>
Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.662913176 +0200] - ERR - attrcrypt_cipher_init - No symmetric key found for cipher 3DES>
Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.665476170 +0200] - INFO - attrcrypt_cipher_init - Key for cipher 3DES successfully gene>
Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.666986475 +0200] - INFO - slapd_daemon - New referral entries are detected under dc=ds,>
Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.710080435 +0200] - INFO - connection_table_new - conntablesize:64001
Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.729125219 +0200] - INFO - slapd_daemon - slapd started.  Listening on All Interfaces po>
Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.730717691 +0200] - INFO - slapd_daemon - Listening on All Interfaces port 636 for LDAPS>
Feb 11 09:49:56 ds ns-slapd[28378]: [11/Feb/2025:09:49:56.732324724 +0200] - INFO - slapd_daemon - Listening on /run/slapd-ds.socket for LDAPI re>
Feb 11 09:49:56 ds systemd[1]: Started 389 Directory Server ds..
```
4. Allow LDAP traffic through firewall
```sh
$ sudo firewall-cmd --add-service=ldap --permanent

success
$ sudo firewall-cmd --add-service=ldaps --permanent

success
$ sudo firewall-cmd --reload

success
```
5. Perform LDAP search
```sh
$ ldapsearch -x -H ldap://localhost -D "cn=Admin" -W -b "dc=example,dc=com"
Enter LDAP Password: 
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
6. Prepare DS structure in one file `add_ds_structure.ldif`
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
7. Import file to Directory server.
```sh
$ ldapadd -x -D "cn=Admin" -W -H ldap://localhost -f add_ds_structure.ldif
Enter LDAP Password: # enter ADmin password
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
8. Verify structure after import
```sh
$ ldapsearch -x -LLL -H ldap://localhost -D "cn=Admin" -W -b "dc=ds,dc=homework,dc=lab"
Enter LDAP Password: 
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
userPassword:: e1BCS0RGMi1TSEE1MTJ9MTAwMDAkT0x1YlRrNU9BdG5FR2FxL21IOWFwSEZEY3Z
 vcE0zcjckUm9lQndpZSt0ZllPMk1hZHRBbFJ1bjZnTGY3TThVcXc2anRickUzY1NOVzNCZENUaEVh
 YlE0L1JVL1lsTUtnZjRPeDBFOEtXOElvTG9mQlRjYkRQRXc9PQ==

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
userPassword:: e1BCS0RGMi1TSEE1MTJ9MTAwMDAkRElLZi9DK2VxYkRTY056azhGTGVxUDBSS0t
 3cHRaUWckdE45WFdHSVZKZENNUGNEWUpyL055QzRUa0ROT0dsMHcyTHJKaDZoeVhoY2ZHdmxrcjFq
 UkdrQzhtdzZLVWw3d3BpNzZXTDJDNVJJZ3h0dUJVNjZvbWc9PQ==

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
userPassword:: e1BCS0RGMi1TSEE1MTJ9MTAwMDAkUmg1dFN6Q1FVdG9yMXVyZE8ycXJFRHNsUnR
 Uc05vNnckcXZlczRsbGlpNmhkWFBjajFzbzFvam80K3VnU3RmdDJrM1d0VEtvdndOK05WSEgyY3VG
 UU1CWFZkTnk3bW41SVdyeDh5d2dHTE9UQm9kdzFXb1pKY3c9PQ==
```
### Client setup (192.168.99.102)
1. Install required packages 
```sh
$ sudo zypper install sssd sssd-ldap sssd-tools oddjob oddjob-mkhomedir pam_krb5 krb5-client openldap2-client nss-pam-ldapd
```
2. Configure SSSD service. Create or edit `/etc/sssd/sssd.conf`
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
ldap_tls_reqcert = never

# User and Group Mappings
ldap_user_search_base = ou=People,dc=ds,dc=homework,dc=lab
ldap_group_search_base = ou=Groups,dc=ds,dc=homework,dc=lab

# Mapping of LDAP attributes
ldap_user_object_class = posixAccount
ldap_user_name = uid
ldap_user_uid_number = uidNumber
ldap_user_gid_number = gidNumber
ldap_user_home_directory = homeDirectory
ldap_user_shell = loginShell

ldap_group_object_class = posixGroup
ldap_group_name = cn

# Enable enumeration (optional but useful for debugging)
enumerate = True

# Enable home directory creation on first login
override_homedir = /home/%u
```
3. Set proper permissions
```sh
$ sudo chmod 600 /etc/sssd/sssd.conf
```
4. Edit `/etc/nsswitch.conf` ensure following lines exist in file
```conf
...
passwd:         compat sss
group:          compat sss
shadow:         compat sss
...
```
5. Enable and restart the SSSD service to laod configuration.
```sh
$ sudo systemctl enable --now sssd
```

6. Verify LDAP user lookup
```sh
$ getent passwd ivan.ivanov
ivan.ivanov:*:10001:10001:Ivan:/home/ivan.ivanov:/bin/bash
```
7. Try login with LDAP user

_I got problem here, maybe I will try to solve the issue in the future._