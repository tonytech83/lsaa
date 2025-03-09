# Practice M4: Network Services (Red Hat)

This practice assumes that you are working in an on-premises environment

All tasks can be achieved under different configurations (host OS and/or virtualization solution) with the appropriate adjustments

This practice is oriented towards **Red Hat**-based distributions and more precisely, **AlmaLinux 9.5**. You can use any other distribution from the same family as **AlmaLinux 9.x**, **Fedora Server 40+**, **Oracle Linux 9.x**, **Rocky Linux 9.x**, etc. but may have to make some adjustments here and there

The infrastructure will vary during the practice but in its most complete stage will include up to three machines:
```
NETWORK: 192.168.81.0/24                                        Public / External
------------+---------------------------+---------------------------+------------
            |                           |                           |
            |.IP1                       |.IP2                       |.IP3
+-----------+-----------+   +-----------+-----------+   +-----------+-----------+
|    [ almalinux-m1 ]   |   |    [ almalinux-m2 ]   |   |    [ almalinux-m3 ]   |
|                       |   |                       |   |                       |
|                       |   |                       |   |                       |
|                       |   |                       |   |                       |
|           M1          |   |           M2          |   |           M3          |
|                       |   |                       |   |                       |
|                       |   |                       |   |                       |
|                       |   |                       |   |                       |
+-----------------------+   +-----------------------+   +-----------------------+
```
*We assume that all machines are set up with FQDN set to **distribution-mX.lsaa.lab** where **distribution** is either almalinux, debian, or opensuse, and **X** is 1, 2, or 3 and can reach each other either via the FQDN or just by the hostname*

*Hint: Create a snapshot/checkpoint of the machines while they are clean and powered off. This will save you time later as we will need a fresh set a few times during the practice*

## Part 1: Web Servers

For this part we will need an infrastructure with all three machines

Machines can be with or without a graphical environment

Network settings shown on the picture reflect the ones, used during the demonstration. You should adjust them according to your setup

### Apache

Let us log on to the **M1** machine with a regular user account

#### Installation and Configuration

The installation of **Apache** is straight forward. Execute
```sh
sudo dnf install httpd
```
*Basically, we can also install the **tree** command to explore the **/etc/httpd** hierarchy*
```sh
sudo dnf install tree
tree /etc/httpd
```
Once it is installed, we can turn off the default (welcome) web page

We can remove the **/etc/httpd/conf.d/welcome.conf** file but it will be recreated during an upgrade

Instead, we will comment out its content
```sh
sudo sed -i 's/^/#/g' /etc/httpd/conf.d/welcome.conf
```
Now, we can modify a little bit the main configuration file **/etc/httpd/conf/httpd.conf**
```sh
sudo vi /etc/httpd/conf/httpd.conf
```
(line 91) Set the **ServerAdmin** for example to **root@lsaa.lab**

(line 100) Set the **ServerName** for example to **almalinux-m1.lsaa.lab:80**

(line 149) Change **Options** to **FollowSymLinks** (*remove the **Indexes** directive if there*)

(line 169) And add **index.php** to **DirectoryIndex**

Save and close the file

More information on the core **Apache** features can be found here: https://httpd.apache.org/docs/2.4/mod/core.html

Check the configuration with
```sh
apachectl configtest
```
Enable and start the **httpd** service
```sh
sudo systemctl enable --now httpd
```
Check the status of the **httpd** service
```sh
systemctl status httpd
```
Don’t forget to open the appropriate port in the firewall if running
```sh
sudo firewall-cmd --add-service http --permanent
sudo firewall-cmd --reload
```
Set a custom **index.html** file
```sh
echo '<h1>Hello from AlmaLinux M1</h1>' | sudo tee /var/www/html/index.html
```
And finally, test it with
```sh
curl http://localhost
```
### Virtual Hosts

Quite often we would want to have multiple sites on one server. One way to achieve this is to use virtual hosts

#### Preparation

Create a file **/etc/httpd/conf.d/main.conf**
```sh
sudo vi /etc/httpd/conf.d/main.conf
```
With the following content
```conf
<VirtualHost *:80>
    DocumentRoot /var/www/html
    ServerName almalinux-m1.lsaa.lab
</VirtualHost>
```
Save and close the file

#### Virtual Hosts (by port)

Create a file **/etc/httpd/conf.d/vhost-port.conf**
```sh
sudo vi /etc/httpd/conf.d/vhost-port.conf
```
With the following content
```conf
Listen 8080

<VirtualHost *:8080>
    DocumentRoot /var/www/vhost-port
    ServerName almalinux-m1.lsaa.lab
</VirtualHost>
```
Save and close the file

The **Listen 8080** instruction can be placed in the main configuration file instead in this one

#### Virtual Hosts (by name)

Create a file **/etc/httpd/conf.d/vhost-name.conf**
```sh
sudo vi /etc/httpd/conf.d/vhost-name.conf
```
With the following content
```conf
<VirtualHost *:80>
    DocumentRoot /var/www/vhost-name
    ServerName www.demo.lab
    ServerAdmin admin@demo.lab
    ErrorLog logs/vhost-name-error.log
    CustomLog logs/vhost-name-access.log combined
</VirtualHost>
```
Save and close the file

#### Finalization

Create the corresponding **DocumentRoot** folders
```sh
sudo mkdir /var/www/vhost-{name,port}
```
Crete two new **index.html** files
```sh
echo '<h1>Hello from vhost by port</h1>' | sudo tee /var/www/vhost-port/index.html
```
And
```sh
echo '<h1>Hello from vhost by name</h1>' | sudo tee /var/www/vhost-name/index.html
```
Test the new configuration with
```sh
apachectl configtest
```
Restart the **httpd** service
```sh
sudo systemctl restart httpd
```
*You can use **reload** instead of **restart** as well*

Open port **8080/tcp** in the firewall
```sh
sudo firewall-cmd --add-port 8080/tcp --permanent
sudo firewall-cmd --reload
```
Add new record in the **/etc/hosts** file
```sh
echo '<m1-ip> www.demo.lab www' | sudo tee -a /etc/hosts
```
*Don’t forget to change <m1-ip> with the IP address of your M1 machine

Finally, test both virtual hosts and the main site with*
```sh
curl http://localhost
curl http://localhost:8080
curl http://www.demo.lab
```
More information and samples about **Apache** virtual hosts can be found here: https://httpd.apache.org/docs/2.4/vhosts/examples.html

### TLS/SSL

We can use certificates issued from a trusted certificate authority or create self-signed certificate

As this is a demo, let us create a self-signed certificate

#### Preparation

Install the necessary packages with
```sh
sudo dnf install mod_ssl openssl
```
Generate the private key
```sh
openssl genrsa -out ca.key 2048
```
Create a certificate signing request (CSR)
```sh
openssl req -new -key ca.key -out ca.csr
```
Answer the questions by modifying or accepting the default values

Generate the self-signed certificate
```sh
openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
```
We can see the result with
```sh
openssl x509 -text -in ca.crt
```
Copy the files to the appropriate folders
```sh
sudo cp ca.crt /etc/pki/tls/certs/ca.crt
sudo cp ca.key /etc/pki/tls/private/ca.key
sudo cp ca.csr /etc/pki/tls/private/ca.csr
```
#### Apache settings

Open the **/etc/httpd/conf.d/ssl.conf** file and navigate to the section of interest with
```sh
sudo vi +/SSLCertificateFile /etc/httpd/conf.d/ssl.conf
```
Change the **SSLCertificateFile** and **SSLCertificateKeyFile** to match our path (change the name to **ca.key** or **ca.crt**)

Save and close the file

Restart the **httpd** service
```sh
sudo systemctl restart httpd
```
Ask for the status
```sh
systemctl status httpd
```
You can see that **Apache** is listening on port **443** as well

Test the default site with 
```sh
curl https://localhost
```
Because the certificate is self-signed you will see an error. Try again with
```sh
curl -k https://localhost
```

Now, you should see our default site

#### Final touches

We must open the appropriate port in the firewall
```sh
sudo firewall-cmd --add-service https --permanent
sudo firewall-cmd --reload
```
Should we want to be automatically redirected to **https** when visiting **http**, we can modify the virtual host configuration for our default site
```sh
sudo vi /etc/httpd/conf.d/main.conf
```
Add the following three lines just before the closing **</VirtualHost>**
```
    RewriteEngine on
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
```
Save and close the file

Restart the **httpd** service
```sh
sudo systemctl restart httpd
```
Open a browser tab on the host and navigate to `http://<m1-ip>/`

A warning should appear, accept it. Now, you should see our default site

Or use
```sh
curl -kL http://localhost
```
More information about the **mod_rewrite** module of **Apache** can be found here: https://httpd.apache.org/docs/2.4/mod/mod_rewrite.html

### PHP

Let us install the necessary packages. This time we will utilize the new **module** functionality
```sh
sudo dnf module install php:8.2
```
Once the installation is complete, we can check with
```sh
php -v
```
Then we can restart the **httpd** service
```sh
sudo systemctl restart httpd
```
Remove the existing **index.html** file
```sh
sudo rm /var/www/html/index.html
```
Create a new **index.php** file with
```sh
echo '<?php phpinfo(); ?>' | sudo tee /var/www/html/index.php
```
Do test with either **curl** locally or with browser tab on the host

### Reverse Proxy

#### Preparation

Log on to the **M2** machine and install **httpd**
```sh
sudo dnf install httpd
```
Then create a custom **index.html** page
```sh
echo '<h1>Hello from AlmaLinux M2</h1>' | sudo tee /var/www/html/index.html
```
Don’t forget to enable and start the service
```sh
sudo systemctl enable --now httpd
```
And last, but not least, don’t forget to adjust the firewall
```sh
sudo firewall-cmd --add-service http --permanent
sudo firewall-cmd --reload
```
#### Configuration

Return to the **M1** machine
```sh
mod_proxy is installed together with the httpd package
```
We can check the **Apache** configuration for the module in question with
```sh
grep mod_proxy /etc/httpd/conf.modules.d/*
```
Now, let us create a configuration for the reverse proxy module
```sh
sudo vi /etc/httpd/conf.d/reverse-proxy.conf
```
Enter the following
```conf
<IfModule mod_proxy.c>
    ProxyRequests Off
    <Proxy *>
        Require all granted
    </Proxy>
    ProxyPass / http://almalinux-m2.lsaa.lab/
    ProxyPassReverse / http://almalinux-m2.lsaa.lab/
</IfModule>
```
Save and close the file

Test the configuration
```sh
apachectl configtest
```
Restart the **httpd** service
```sh
sudo systemctl restart httpd
```
Test by opening a browser tab on the host and navigating to `https://<m1-ip>/`

You will see an error

Return on **M1** and adjust the following **SELinux Boolean**
```sh
sudo setsebool -P httpd_can_network_connect on
```
Refresh the browser tab, now it must work as expected

You should see the index page of **M2**

More information about the **mod_proxy** module of **Apache** can be found here: https://httpd.apache.org/docs/2.4/mod/mod_proxy.html

### Load Balancing

#### Preparation

Log on to the **M3** machine and install **httpd**
```sh
sudo dnf install httpd
```
Then create a custom **index.html** page
```sh
echo '<h1>Hello from AlmaLinux M3</h1>' | sudo tee /var/www/html/index.html
```
Don’t forget to enable and start the service
```sh
sudo systemctl enable --now httpd
```
And last, but not least, don’t forget to adjust the firewall
```sh
sudo firewall-cmd --add-service http --permanent
sudo firewall-cmd --reload
```
#### Configuration

We are one step away from turning our setup into a fully functional load balancing solution

Return on **M1**

Open the proxy module configuration file
```sh
sudo vi /etc/httpd/conf.d/reverse-proxy.conf
```
And modify the file by deleting the **ProxyPass** and **ProxyPassReverse** lines

Then enter the following
```conf
ProxyPass / balancer://demo/
ProxyPassReverse / balancer://demo/
<Proxy balancer://demo>
    BalancerMember http://almalinux-m2.lsaa.lab
    BalancerMember http://almalinux-m3.lsaa.lab
    ProxySet lbmethod=bytraffic
</Proxy>
```
Save and close the file

Check the configuration
```sh
apachectl configtest
```
Restart the **httpd** service
```sh
sudo systemctl restart httpd
```
Test by opening a browser tab on the host and navigating to `https://<m1-ip>/`

Refresh a few times. Everything should work as expected

More information about the **mod_proxy** module of **Apache** can be found here: https://httpd.apache.org/docs/2.4/mod/mod_proxy.html

And about the load balancing extension, here: https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html

### NGINX

We can continue with the same infrastructure, just reset the machines. Alternatively, spin up a new set

Let us log on to the **M1** machine with a regular user account

#### Installation and Configuration

The installation of **NGINX** is straight forward, execute
```sh
sudo dnf install nginx
```
Once it is installed, we can change the default (welcome) web page

Execute the following to change it
```sh
echo '<h1>Hello from AlmaLinux M1</h1>' | sudo tee /usr/share/nginx/html/index.html
```
Now, we can modify a little bit the main configuration file `/etc/nginx/nginx.conf`
```sh
sudo vi /etc/nginx/nginx.conf
```
(line 41) Set the **server_name** for example to **amalinux-m1.lsaa.lab**

Save and close the file

Test the **NGINX** configuration with
```sh
sudo nginx -t
```
Enable and start the **nginx** service
```sh
sudo systemctl enable --now nginx
```
Check the status of the **nginx** service
```sh
systemctl status nginx
```
Don’t forget to open the appropriate port in the firewall if running
```sh
sudo firewall-cmd --add-service http --permanent
sudo firewall-cmd --reload
```
And finally, test with
```sh
curl http://localhost
```
### Virtual Hosts

Quite often we would want to have multiple sites on one server. One way to achieve this is to use virtual hosts

#### Virtual Hosts (by port)

Create a file `/etc/nginx/conf.d/vhost-port.conf`
```sh
sudo vi /etc/nginx/conf.d/vhost-port.conf
```
With the following content
```conf
server {
    listen 8080;

    location / {
        root /usr/share/nginx/vhost-port;
        index index.html;
    }

}
```
Save and close the file

#### Virtual Hosts (by name)

Create a file `/etc/nginx/conf.d/vhost-name.conf`
```sh
sudo vi /etc/nginx/conf.d/vhost-name.conf
```
With the following content
```conf
server {
    listen 80;
    server_name www.demo.lab;

    location / {
        root /usr/share/nginx/vhost-name;
        index index.html;
    }
}
```
Save and close the file

#### Finalization

Create the corresponding **root** folders
```sh
sudo mkdir /usr/share/nginx/vhost-{name,port}
```
Crete two new **index.html** files
```sh
echo '<h1>Hello from AlmaLinux M1 (vhost by port)</h1>' | sudo tee /usr/share/nginx/vhost-port/index.html
```
And
```sh
echo '<h1>Hello from AlmaLinux M1 (vhost by name)</h1>' | sudo tee /usr/share/nginx/vhost-name/index.html
```
Test the new configuration with
```sh
sudo nginx -t
```
Restart the **nginx** service
```sh
sudo systemctl restart nginx
```
Open port **8080/tcp** in the firewall
```sh
sudo firewall-cmd --add-port 8080/tcp --permanent
sudo firewall-cmd --reload
```
Add new record in the **/etc/hosts** file
```sh
echo '<m1-ip> www.demo.lab www' | sudo tee -a /etc/hosts
```
*Don’t forget to change `<m1-ip>` with the IP address of your M1 machine*

Finally, test both virtual hosts and the main site with
```sh
curl http://localhost
curl http://localhost:8080
curl http://www.demo.lab
```
### TLS/SSL

We can use certificates issued from a trusted certificate authority or create self-signed certificate

As this is a demo, let us create a self-signed certificate

#### Preparation

Repeat the same steps as with Apache

Install the necessary packages with
```sh
sudo dnf install openssl
```
Generate the private key
```sh
openssl genrsa -out ca.key 2048
```
Create a certificate signing request (CSR)
```sh
openssl req -new -key ca.key -out ca.csr
```
Generate the self-signed certificate
```sh
openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
```
We can we the result with
```sh
openssl x509 -text -in ca.crt
```
Copy the files to the appropriate folders
```sh
sudo cp ca.crt /etc/pki/tls/certs/ca.crt
sudo cp ca.key /etc/pki/tls/private/ca.key
sudo cp ca.csr /etc/pki/tls/private/ca.csr
```
#### NGINX settings

Open the **/etc/nginx/nginx.conf** file
```sh
sudo vi +58 /etc/nginx/nginx.conf
```
Uncomment lines between **58** and **81**

You can do it automatically (while in vi) with the following
```plain
:.,$s/^#//g
```
Then adjust the **server_name**, **ssl_certificate**, and **ssl_certificate_key** to match your settings

Save and close the file

Test the new configuration with
```sh
sudo nginx -t
```
Restart the **nginx** service
```sh
sudo systemctl restart nginx
```
Test the default site with
```sh
curl -k https://localhost
```
Now, you should see our default site

#### Final touches

We must open the appropriate port in the firewall
```sh
sudo firewall-cmd --add-service https --permanent
sudo firewall-cmd --reload
```
Should we want to be automatically redirected to **https** when visiting **http**, we can modify the virtual host configuration for our default site
```sh
sudo vi /etc/nginx/nginx.conf
```
Add the following line between **listen** (line 40) and **server_name** (line 41)
```plain
return 301 https://$host$request_uri;
```
Save and close the file

Test the new configuration with
```sh
sudo nginx -t
```
Restart the **nginx** service
```sh
sudo systemctl restart nginx
```
Open a browser tab on the host and navigate to `http://<m1-ip>/`

A warning should appear, accept it. Now, you should see our default site

Or use
```sh
curl -kL http://localhost
```
### PHP

Let us install the necessary packages
```sh
sudo dnf module install php:8.2
```
Once the installation is complete, we can check with
```sh
php -v
```
Then we can restart the **nginx** service
```sh
sudo systemctl restart nginx
```
Create a new **index.php** file with
```sh
echo '<?php phpinfo(); ?>' | sudo tee /usr/share/nginx/html/index.php
```
Do test with either **curl** locally or with browser tab on the host

### Reverse Proxy

#### Preparation

Log on to the **M2** machine and install **nginx**
```sh
sudo dnf install nginx
```
Change the **/etc/nginx/nginx.conf** file
```sh
sudo vi /etc/nginx/nginx.conf
```
By adding the following after the **root** directive (line 42)
```conf
set_real_ip_from <machines-network>/24; <- for example 192.168.99.0/24
real_ip_header X-Forwarded-For;
```
And adjusting the **server_name** line

Save and close the file

Test the new configuration with
```sh
sudo nginx -t
```
Start the **nginx** service
```sh
sudo systemctl enable --now nginx
```
Then create a custom **index.html** page
```sh
echo '<h1>Hello from AlmaLinux M2</h1>' | sudo tee /usr/share/nginx/html/index.html
```
Don’t forget to adjust the firewall
```sh
sudo firewall-cmd --add-service http --permanent
sudo firewall-cmd --reload
```
#### Configuration

Return to the **M1** machine

Open the main configuration file for editing
```sh
sudo vi /etc/nginx/nginx.conf
```
Enter the following lines just above the **include** (line 45) directive or the **location** block
```conf
    roxy_redirect off;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
```
Add the following to the **location** block (if one exists)
```conf
proxy_pass http://almalinux-m2.lsaa.lab/;
```
Or create a new one (above the **include** directive) like this
```conf
location / { 
    proxy_pass http://almalinux-m2.lsaa.lab/;
}
```
Repeat the above steps for the **TLS** section as well

Save and close the file

Test the configuration
```sh
sudo nginx -t
```
Restart the **nginx** service
```sh
sudo systemctl restart nginx
```
Adjust the following **SELinux Boolean**
```sh
sudo setsebool -P httpd_can_network_connect on
```
Open a browser tab on the host and navigate to `https://<m1-ip>/`

You should see the index page of **M2**

### Load Balancing

#### Preparation

Log on to the **M3** machine and install **nginx**
```sh
sudo dnf install nginx
```
Change the **/etc/nginx/nginx.conf** file
```sh
sudo vi /etc/nginx/nginx.conf
```
By adding the following after the **root** directive (line 42)
```conf
set_real_ip_from <machines-network>/24; <- for example 192.168.99.0/24
real_ip_header X-Forwarded-For;
```
And adjusting the **server_name** line

Save and close the file

Test the new configuration with
```sh
sudo nginx -t
```
Start the **nginx** service
```sh
sudo systemctl enable --now nginx
```
Then create a custom **index.html** page
```sh
echo '<h1>Hello from AlmaLinux M3</h1>' | sudo tee /usr/share/nginx/html/index.html
```
Don’t forget to adjust the firewall
```sh
sudo firewall-cmd --add-service http --permanent
sudo firewall-cmd --reload
```
#### Configuration

We are one step away from turning our setup into a fully functional load balancing solution

Return on the **M1** machine

Open the main configuration file for editing
```sh
sudo vi /etc/nginx/nginx.conf
```
Add the following lines in the top of the **http** section
```sh
upstream backend {
    server almalinux-m2.lsaa.lab;
    server almalinux-m3.lsaa.lab;
}
```
Then change the location of both sections for plain http and TLS to
```sh
proxy_pass http://backend;
```
Save and close the file

Check the configuration
```sh
sudo nginx -t
```
Restart the **nginx** service
```sh
sudo systemctl restart nginx
```
Test by opening a browser tab on the host and navigating to `https://<m1-ip>/`

Refresh a few times. Everything should work as expected

## Part 2: Domain Name System

For this part we will need an infrastructure with just two machines

Machines can be with or without a graphical environment

Network settings shown on the picture reflect the ones, used during the demonstration. You should adjust them according to your setup

### BIND

#### Caching Server

Log on to the **M1** machine

Start with installing the required packages
```sh
sudo dnf install bind bind-utils
```
Open the main configuration file
```sg
sudo vi /etc/named.conf
```
Add the following block above the options instruction
```conf
acl trusted-clients {
    localhost;
    <machines-network>/24; <- for example 192.168.99.0/24
}
```
This will allow us to grant all stations in our internal network the right to query the DNS server

Next, change the interfaces the **bind** service listens on

For example, change the IPv4 **localhost** (**127.0.0.1**) to **any** (to listen on all interfaces) or to **none** (to stop listen)

Same applies for IPv6 but instead of **127.0.0.1** the address is **::1**

Let us set both **listen-on** and **listen-on-v6** to **any**

Now, change the **allow-query** to **{ trusted-clients; }**

Save and close the file

Check that everything with the configuration is okay
```sh
sudo named-checkconf
```
It appears that have an error due to a missing semicolon

Open again the file and correct the error

Check the configuration again. Now all should be fine

It is a good idea to check the ownership of the file
```sh
ls -l /etc/named.conf
```
It must be owned by the **root** user and the **named** group

In the same manner, we must check the **SELinux** label for the file
```sh
ls -Z /etc/named.conf
```
The type part must be **named_conf_t**

Enable and start the service
```sh
sudo systemctl enable --now named
```
And check its status
```sh
systemctl status named
```
Additionally, the status can be checked with
```sh
sudo rndc status
```
Don’t forget to allow the DNS service in the firewall
```sh
sudo firewall-cmd --add-service dns --permanent
sudo firewall-cmd --reload
```
Now, log on to the **M2** machine

Install the required packages
```sh
sudo dnf install bind-utils
```
Set **M1** as a DNS server

Change local settings for the DNS server
```sh
sudo nmcli conn modify ens192 ipv4.dns <m1-ip>
sudo nmcli conn down ens192; sudo nmcli conn up ens192
```
*Please note that you may need to adjust the network interface name as well. For example, **enp0s3** or **eth0**

You may need to check and adjust the */etc/resolv.conf* file as well*

Let us look up information for a domain, for example **opensuse.org**
```sh
dig opensuse.org
```
We can see plenty of information

Notice the time it took to answer the query

Repeat the command once more

Now, the answer is returned faster, because of the cache

Let us try a reverse lookup query with one of the returned IP addresses
```sh
dig -x 195.135.223.50
```
#### Forwarding Server

Now, we can modify the settings of our server and turn it into a forwarding server

Return on **M1** machine

Open the main configuration file
```sh
sudo vi /etc/named.conf
```
Insert the following block after the **allow-query** instruction
```conf
forwarders {
    8.8.8.8;
    8.8.4.4;
};

forward only;
```
Save and close the file

Then, check it for errors
```sh
sudo named-checkconf
```
Execute the following to make **bind** reload its configuration
```sh
sudo rndc reload
```
And this one to flush the cache
```sh
sudo rndc flush
```
Switch to the **M2** machine and repeat the lookup queries

Even the first attempt is resolved much faster now

#### Internal DNS server

Return on the **M1** machine

Open the main configuration file for editing
```sh
sudo vi /etc/named.conf
```
Position the cursor after the last **zone** block and type the following for the forward lookup zone
```conf
zone "lsaa.lab" IN {
    type master;
    file "lsaa.lab.zone";
    allow-update { none; };
};
```
And then add the following for the reverse lookup zone (you must change 81.168.192 to match yours)
```conf
zone "99.168.192.in-addr.arpa" IN {
    type master;
    file "99.168.192.zone";
    allow-update { none; };
};

Save and close the file

Check the configuration with
```sh
sudo named-checkconf
```
Create a new forward lookup zone file
```sh
sudo vi /var/named/lsaa.lab.zone
```
And enter the following
```zone
$ORIGIN lsaa.lab.
$TTL 86400
@ IN SOA almalinux-m1.lsaa.lab. root.lsaa.lab. (
    2025020501 ; serial
    3600 ; refresh in 1 hour
    1800 ; retry in 30 minutes
    604800 ; expires after 7 days
    86400 ; minimum TTL of 1 day
)
    IN NS almalinux-m1.lsaa.lab.
    IN A <m1-ip>
    IN MX 10 almalinux-m1.lsaa.lab.
almalinux-m1 IN A <m1-ip>
almalinux-m2 IN A <m2-ip>
client IN CNAME almalinux-m2.lsaa.lab.
```
*Make sure that you filled in the `<m1-ip>` and `<m2-ip>` placeholders with the correct IP addresses*

Save and close the file

Ensure that the file permissions and ownership are as expected
```sh
sudo chmod 640 /var/named/lsaa.lab.zone
sudo chown root:named /var/named/lsaa.lab.zone
```
Check the zone with
```sh
sudo named-checkzone lsaa.lab /var/named/lsaa.lab.zone
```
Create a new reverse lookup zone file
```sh
sudo vi /var/named/99.168.192.zone
```
And enter the following
```zone
$TTL 86400
99.168.192.in-addr.arpa. IN SOA almalinux-m1.lsaa.lab. root.lsaa.lab. (
    2025020501 ; serial
    3600 ; refresh in 1 hour
    1800 ; retry in 30 minutes
    604800 ; expires after 7 days
    86400 ; minimum TTL of 1 day
)
    IN NS almalinux-m1.lsaa.lab.
<last-part-ip-m1> IN PTR almalinux-m1.lsaa.lab.
<last-part-ip-m2> IN PTR almalinux-m2.lsaa.lab.
```
*Make sure that you filled in the `<last-part-ip-m1>` and `<last-part-ip-m2>` placeholders with the correct IP addresses*

Save and close the file

Check the zone
```sh
sudo named-checkzone 99.168.192.in-addr.arpa /var/named/99.168.192.zone
```
Adjust the permissions and ownership
```sh
sudo chmod 640 /var/named/99.168.192.zone
sudo chown root:named /var/named/99.168.192.zone
```
Restart the **named** service
```sh
sudo systemctl restart named
```
And check its status
```sh
systemctl status named
```
Change local settings for the DNS server
```sh
sudo nmcli conn modify ens192 ipv4.dns <m1-ip>
sudo nmcli conn down ens192; sudo nmcli conn up ens192
```
*Please note that you may need to adjust the network interface name as well. For example, **enp0s3** or **eth0**

You may need to check and adjust the **/etc/resolv.conf** file as well*

Do a forward query for the **almalinux-m2.lsaa.lab** machine
```sh
dig almalinux-m2.lsaa.lab
```
Then do a reverse lookup
```sh
dig -x <m2-ip>
```
You can switch to **M2** machine and test the same

## Part 3: Directory Services

For this part we will need an infrastructure with just two machines

Machines can be with or without a graphical environment

Do not forget to reset the machines used during the previous part or spin a new set

Network settings shown in the picture reflect the ones used during the demonstration. You should adjust them according to your setup

### OpenLDAP

#### Installation

Log in to the first machine and install the packages
```sh
sudo dnf install -y openldap openldap-clients openldap-servers
```
Check the directory structure
```sh
sudo tree /etc/openldap/
```
Check the status of the service
```sh
systemctl status slapd
```
Adjust the default settings
```sh
sudo vi /etc/openldap/ldap.conf
```
Make sure that the following two are present and have the appropriate values
```conf
BASE dc=lsaa,dc=lab
URI ldap://almalinux-m1.lsaa.lab
```
Once done with the changes, start the service
```sh
sudo systemctl enable --now slapd
```
And check the status of the service
```sh
systemctl status slapd
```
Finally, if **FirewallD** is installed and active, adjust it to allow external connections
```sh
sudo firewall-cmd --permanent --add-service={ldap,ldaps}
sudo firewall-cmd --reload
```
Check that we can get response
```sh
sudo ldapsearch -H ldapi:/// -Y EXTERNAL -b cn=config
```
Note the initial domain - it should be something like

*olcSuffix: dc=my-domain,dc=com*
*olcRootDN: cn=Manager,dc=my-domain,dc=com*

Do another check but this time ask for just the DN's
```sh
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config dn
```
Note the number of active/loaded schemas – just one

#### Basic Configuration

Generate password for the root user (of the OpenLDAP server)
```sh
slappasswd
```
If you enter ***Parolka-12345***, the output should be something like ***{SSHA}7/v9wy6YQruW6bk1jhn2M3Nq3HLCoKpc***

Remember the value and use it in the paragraphs that follow

Prepare an LDIF file to change/set the root password to new one
```sh
vi rootpw.ldif
```
Enter the following
```ldfi
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}7/v9wy6YQruW6bk1jhn2M3Nq3HLCoKpc
```
Apply the LDIF file
```sh
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f rootpw.ldif
```
The output should be similar to this

*SASL/EXTERNAL authentication started*

*SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth*

*SASL SSF: 0*

*modifying entry "olcDatabase={0}config,cn=config"*

Now extend the active schemas
```sh
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```
And ask again for the configuration
```sh
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config dn
```
The newly added ones should appear in the list

Pay attention to the **mdb** database - it should be listed as **{2}mdb,cn=config**

Now, prepare another LDIF file to adjust the manager/administrator of the directory

Open a new file
```sh
vi manager.ldif
```
And enter the following
```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=lsaa,dc=lab

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=lsaa,dc=lab

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}7/v9wy6YQruW6bk1jhn2M3Nq3HLCoKpc
```
Apply the newly created LDIF file
```sh
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f manager.ldif
```
The output should be similar to

*SASL/EXTERNAL authentication started*

*SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth*

*SASL SSF: 0*

*modifying entry "olcDatabase={2}mdb,cn=config"*

*modifying entry "olcDatabase={2}mdb,cn=config"*

*modifying entry "olcDatabase={2}mdb,cn=config"*

Now, if we ask for detailed configuration information
```sh
sudo ldapsearch -H ldapi:/// -Y EXTERNAL -b cn=config
```
We should notice that the name of our domain has changed to

*olcSuffix: dc=lsaa,dc=lab*

*olcRootDN: cn=Manager,dc=lsaa,dc=lab*

Now, let's prepare a file to lay down the base organization of our directory

Open a new file
```sh
vi baseorg.ldif
```
And enter the following
```ldif
dn: dc=lsaa,dc=lab
objectClass: top
objectClass: dcObject
objectclass: organization
o: Demo LDAP Server (LSAA)
dc: lsaa
dn: cn=Manager,dc=lsaa,dc=lab
objectClass: organizationalRole
cn: Manager
description: LDAP Manager

dn: ou=Users,dc=lsaa,dc=lab
objectClass: organizationalUnit
ou: Users

dn: ou=Groups,dc=lsaa,dc=lab
objectClass: organizationalUnit
ou: Groups
```
Now apply the file
```sh
ldapadd -x -D cn=Manager,dc=lsaa,dc=lab -W -f baseorg.ldif
```
You will be asked for the root password created earlier, enter it

The output should be similar to

*adding new entry "dc=lsaa,dc=lab"*

*adding new entry "cn=Manager,dc=lsaa,dc=lab"*

*adding new entry "ou=Users,dc=lsaa,dc=lab"*

*adding new entry "ou=Groups,dc=lsaa,dc=lab"*

Now, let's check what we have so far
```sh
ldapsearch -x -b dc=lsaa,dc=lab
ldapsearch -x -LLL -H ldap:/// -b dc=lsaa,dc=lab dn
```
It is time to add a few other objects

Let's first create a group

Open an empty file
```sh
vi group.ldif
```
And enter the following
```ldif
dn: cn=penguins,ou=Groups,dc=lsaa,dc=lab
objectClass: posixGroup
cn: penguins
gidNumber: 10000
```
Apply the LDIF file
```sh
ldapadd -x -D cn=manager,dc=lsaa,dc=lab -W -f group.ldif
```
You will be asked for the root password created earlier, enter it

The output should be similar to

*adding new entry "cn=penguins,ou=Groups,dc=lsaa,dc=lab"*

Let's continue with the users

Create an empty file for the users
```sh
vi users.ldif
```
Enter the following
```ldif
dn: uid=john,ou=Users,dc=lsaa,dc=lab
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: john
sn: Smith
givenName: John
cn: John Smith
displayName: John Smith
uidNumber: 10000
gidNumber: 10000
userPassword: {CRYPT}x
gecos: John Smith
loginShell: /bin/bash
homeDirectory: /home/john

dn: uid=jane,ou=Users,dc=lsaa,dc=lab
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: jane
sn: Hudson
givenName: Jane
cn: Jane Hudson
displayName: Jane Hudson
uidNumber: 10001
gidNumber: 10000
userPassword: {CRYPT}x
gecos: Jane Hudson
loginShell: /bin/bash
homeDirectory: /home/jane
```
Please note that it is important that **uid** and **gid** values in your directory do not collide with local values

You can use high number ranges, such as starting at **5000 or even higher**

Apply the LDIF file
```sh
ldapadd -x -D cn=manager,dc=lsaa,dc=lab -W -f users.ldif
```
You will be asked for the root password created earlier, enter it

The output should be similar to

*adding new entry "uid=john,ou=Users,dc=lsaa,dc=lab"*

*adding new entry "uid=jane,ou=Users,dc=lsaa,dc=lab"*

Now we can ask for all the content of the directory
```sh
ldapsearch -x -b dc=lsaa,dc=lab
ldapsearch -x -LLL -H ldap:/// -b dc=lsaa,dc=lab dn
```
Or we can ask for particular object. For example, for the user Jane
```sh
ldapsearch -x -LLL -b dc=lsaa,dc=lab '(uid=jane)' cn gidNumber homeDirectory
```
The output should be similar to

*dn: uid=jane,ou=Users,dc=lsaa,dc=lab*

*cn: Jane Hudson*

*gidNumber: 10000*

*homeDirectory: /home/jane*

Of course, a simple filter like the one above could be written without the quotes and parentheses

We can change both the filter and the list of properties

Let's change both and ask for all the properties of the groups in which Jane is a member
```sh
ldapsearch -x -LLL -b dc=lsaa,dc=lab '(&(objectClass=posixGroup)(memberUid=jane))'
```
Ha, nothing. Why? If we check how we created the group, we will see that there is no reference to the users

We can correct this by creating another LDIF file with suitable content

Open an empty file
```sh
vi group-fix.ldif
```
And enter the following
```ldif
dn: cn=penguins,ou=Groups,dc=lsaa,dc=lab
changetype: modify
add: memberUid
memberUid: john
memberUid: jane
```
Apply the LDIF file
```sh
ldapmodify -x -D cn=manager,dc=lsaa,dc=lab -W -f group-fix.ldif
```
You will be asked for the root password created earlier, enter it

The output should be similar to

*modifying entry "cn=penguins,ou=Groups,dc=lsaa,dc=lab"*

Now, if we repeat the last search command, we should see something different
```sh
ldapsearch -x -LLL -b dc=lsaa,dc=lab '(&(objectClass=posixGroup)(memberUid=jane))'
```
The output should be something like

*dn: cn=penguins,ou=Groups,dc=lsaa,dc=lab*

*objectClass: posixGroup*

*cn: penguins*

*gidNumber: 10000*

*memberUid: john*

*memberUid: jane*

We are done with this. There is just one more thing - our users do not have passwords. We could have specified those during the creation, or we can set them later. Let's see how this could be done

Set a password for **John**
```sh
ldappasswd -x -D cn=manager,dc=lsaa,dc=lab -W -S uid=john,ou=Users,dc=lsaa,dc=lab
```
The output should look like

*New password:*

*Re-enter new password:*

*Enter LDAP Password:*

So, we are entering twice the password for the target user, and then the password for the Manager

Do it for **Jane** as well
```sh
ldappasswd -x -D cn=manager,dc=lsaa,dc=lab -W -S uid=jane,ou=Users,dc=lsaa,dc=lab
```
Before we switch to our client station, let's check a few more things around authentication

Execute the following
```sh
ldapwhoami -x
```
The result should be

*anonymous*

Now try with **John**
```sh
ldapwhoami -x -D uid=john,ou=Users,dc=lsaa,dc=lab -W
```
After entering his password, we should see something like

*dn:uid=john,ou=Users,dc=lsaa,dc=lab*

All is working as expected. We are done here, let's continue with the SSL/TLS and client station

#### SSL/TLS

Check the current state and availability of TLS related options
```sh
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config | grep olcTLS
```
The output is not very easy/suitable for interpretation, but we can see that there are some options available like **olcTLSCACertificate**, **olcTLSCertificateFile**, etc.

Let's ask differently and see just how those are currently configured (if at all configured - no, they are not)
```sh
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config | grep ^olcTLS
```
The output should be an empty list, however there will be some text like

*SASL/EXTERNAL authentication started*

*SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth*

*SASL SSF: 0*

Now, let's prepare the necessary files

Create a CA certificate and key
```sh
openssl req -new -x509 -nodes -out ca.crt -keyout ca.key -days 3650
```
Generate the server's private key
```sh
openssl genpkey -algorithm RSA -out server.key
```
Create a CSR for the server
```sh
openssl req -new -key server.key -out server.csr
```
Finally, create the server’s certificate
```sh
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 3650
```
By now, if we check
```sh
ls ca.* server.*
```
We should have all the needed files

*ca.crt ca.key ca.srl server.crt server.csr server.key*

Let's copy those to the appropriate place
```sh
sudo cp -v *.{key,crt} /etc/pki/tls/certs/
```
Change their ownership
```sh
sudo chown -v ldap:ldap /etc/pki/tls/certs/ca.*
sudo chown -v ldap:ldap /etc/pki/tls/certs/server.*
```
Now, we must create an LDIF file to change the respective settings

Open an empty LDIF file
```sh
vi tls.ldif
```
Enter the following
```ldif
dn: cn=config
changetype: modify
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/pki/tls/certs/server.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/pki/tls/certs/server.key
-
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/pki/tls/certs/ca.crt
-
add: olcTLSVerifyClient
olcTLSVerifyClient: never
```
Apply the file
```sh
sudo ldapadd -H ldapi:// -Y EXTERNAL -f tls.ldif
```
The output should be like

*SASL/EXTERNAL authentication started*

*SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth*

*SASL SSF: 0*

*modifying entry "cn=config"*

Now, if we check the respective settings
```sh
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config | grep ^olcTLS
```
The output should be the following

*SASL/EXTERNAL authentication started*

*SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth*

*SASL SSF: 0*

*olcTLSCertificateFile: /etc/pki/tls/certs/server.crt*

*olcTLSCertificateKeyFile: /etc/pki/tls/certs/server.key*

*olcTLSCACertificateFile: /etc/pki/tls/certs/ca.crt*

*olcTLSVerifyClient: never*

Restart the service
```sh
sudo systemctl restart slapd.service
```
Open the **/etc/openldap/ldap.conf** file
```sh
sudo vi /etc/openldap/ldap.conf
```
And add the following
```conf
TLS_CACERT /etc/pki/tls/certs/ca.crt
TLS_REQCERT never
```
Now, let's test that we can use secure communication with our **OpenLDAP** server

First use this one
```sh
ldapwhoami -H ldap://almalinux-m1.lsaa.lab -x -ZZ
```
The result should be

*anonymous*

Now, let's do a search
```sh
ldapsearch -H ldap://almalinux-m1.lsaa.lab -x -Z
```
We should see the objects in our directory

And we should be able to achieve the same result with this command
```sh
ldapsearch -H ldaps://almalinux-m1.lsaa.lab -x
```
If so, then we are done here and now, we can move on to the client station

#### Client Station

Log in to **M2**

Install the required/needed packages
```sh
sudo dnf install -y openldap-clients sssd sssd-ldap oddjob-mkhomedir
```
Then, we must change the identification provider to **SSSD**
```sh
sudo authselect select sssd with-mkhomedir --force
```
Next, change the **OpenLDAP** client settings. Open the file for editing
```sh
sudo vi /etc/openldap/ldap.conf
```
And make sure it contains the following lines
```conf
BASE dc=lsaa,dc=lab
URI ldap://almalinux-m1.lsaa.lab
TLS_CACERT /etc/openldap/certs/ca.crt
TLS_REQCERT never
```
Copy the CA certificate from the other machine
```sh
scp <user-name>@almalinux-m1:/etc/pki/tls/certs/ca.crt .
```

And then copy (or move it) to the right place (the one we mentioned in the configuration file)
```sh
sudo cp -v ca.crt /etc/openldap/certs/
```
Open the /etc/sssd/sssd.conf file for editing
```sh
sudo vi /etc/sssd/sssd.conf
```
And make sure it has content like this
```conf
[domain/default]
id_provider = ldap
autofs_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldap://almalinux-m1.lsaa.lab/
ldap_search_base = dc=lsaa,dc=lab
ldap_id_use_start_tls = True
ldap_tls_cacertdir = /etc/openldap/certs
cache_credentials = True
ldap_tls_reqcert = allow

[sssd]
services = nss, pam, autofs
domains = default

[nss]
homedir_substring = /home
```
Change its permissions
```sh
sudo chmod 0600 /etc/sssd/sssd.conf
```
Now enable and start the two important services
```sh
sudo systemctl enable --now oddjobd.service
sudo systemctl enable --now sssd.service
```
And check they started normally
```sh
systemctl status oddjobd.service
systemctl status sssd.service
```
Now, we should be able to interact with our **OpenLDAP** server in a (relatively) secure manner

Check our two users
```sh
id john

id jane
```

We should see something similar to

*uid=10000(john) gid=10000(penguins) groups=10000(penguins)*

*uid=10001(jane) gid=10000(penguins) groups=10000(penguins)*

We can use also the getent command
```sh
getent passwd john

getent passwd jane
```

And the result should be

*john:*:10000:10000:John Smith:/home/john:/bin/bash*

*jane:*:10001:10000:Jane Hudson:/home/jane:/bin/bash*

A few more tests

Let's initiate a search for objects with
```sh
ldapsearch -H ldap://almalinux-m1.lsaa.lab -x -Z
```
We should see them (as expected)

Or we can use the **LDAPS** protocol instead with
```sh
ldapsearch -H ldaps://almalinux-m1.lsaa.lab -x
```
Again, we should see the same

Last, but not least, let's try to log in as **John**
```sh
su - john
```
We should see something like

Creating home directory for john.

We managed to successfully log in as **John**

We are done

And we have a basic, but fully working **OpenLDAP** setup