## STEP 0 - REQUIREMENTS
1. Two RHEL9 Web Servers
2. One MySQL DB Server (based on Ubuntu 20.04)
3. One RHEL9 NFS server
   
## STEP 1 - PREREQUISITE CONFIGURATION
- Apache (httpd) process running in both webservers.
- /var/www mounted on /mnt/apps of the NFS server
- TCP and UDP ports open.(80,3306,111,2049)
- Tooling website can be open via the Public IPs of the various webservers.

### WEB SERVER PREP
Install Apache and PHP
```
sudo yum install httpd -y 
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm  
sudo dnf module list php 
sudo dnf module reset php
sudo dnf module enable php:remi-7.4 
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd --nobest
sudo systemctl start php-fpm 
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
```

Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html<br>
Download  DevOpsToolingWebsite and copy the html folder to var/www/html <br>
```
mkdir Tooling 
cd   Tooling
sudo wget https://github.com/ifydevops23/ToolingWebsite/archive/master.zip
sudo unzip master.zip 
sudo rm -rf master.zip   
sudo cp -R html/.  /var/www/html/
```
Disable SELinux <br>
`sudo setenforce 0`<br>

To make this change permanent – open following config file<br>
`sudo vi /etc/sysconfig/selinux` <br>
Set `SELINUX=disabled` <br>

Resttart apache<b>
`sudo systemctl restart httpd`<br>

Restart httpd <br>
`sudo systemctl restart httpd`<br>

Open the website in your browser http://Public-IP-Address/index.php and make sure you can login into the website with "admin" user.
