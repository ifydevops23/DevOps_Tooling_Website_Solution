# 3-TIER WEB APP USING A SINGLE DATABASE AND A SINGLE NFS SERVER AS STORAGE SERVER

On the diagram below you can see a common pattern where several stateless Web Servers share a common database and also access the same files using Network File Sytem (NFS) as a shared file storage. Even though the NFS server might be located on a completely separate hardware – for Web Servers it look like a local file system from where they can serve the same files.


It is important to know what storage solution is suitable for what use cases, for this – you need to answer the following questions: what data will be stored, in what format, how this data will be accessed, by whom, from where, how frequently, etc. Based on this you will be able to choose the right storage system for your solution.


Network File Sharing (NFS) is a protocol that allows you to share directories and files with other Linux clients over a network. Shared directories are typically created on a file server, running the NFS server component.

## STEP 1 - PREPARE NFS SERVER
- Spin up a new EC2 instance with RHEL Linux 8 Operating System.<br>
- Configure LVM on the Server. <br>
* List attached volumes`ls /dev/`<br>
* Install lvm2`sudo yum install lvm2` <br>*
* Use gdisk utility to create single partitions of 10G in each disk. <br>
```
`sudo gdisk /dev/xvdf`
`sudo gdisk /dev/xvdg`
`sudo gdisk /dev/xvdh`
```
* Create physical volumes<br>
`sudo pvcreate /dev/xvdf1`
`sudo pvcreate /dev/xvdg1`
`sudo pvcreate /dev/xvdh1`

* Create Volume group VG <br>
`sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1` 

* Create Logical Volume <br>
`sudo lvcreate -n lv-apps -L 10G webdata-vg`
`sudo lvcreate -n lv-logs -L 10G webdata-vg`
`sudo lvcreate -n lv-opt -L 9G webdata-vg`

* Verify setup<br>
`sudo lsblk`<br>
* Instead of formatting the disks as ext4, you will have to format them as xfs <br>
```
`sudo mkfs -t xfs /dev/webdata-vg/lv-apps`
`sudo mkfs -t xfs /dev/webdata-vg/lv-logs`
`sudo mkfs -t xfs /dev/webdata-vg/lv-opt`
```
- Ensure there are 3 Logical Volumes. lv-opt lv-apps, and lv-logs.<br>
`sudo lsblk`<br>
- Create the root directories to be shared by our web servers (client) <br>
```
sudo mkdir -p /mnt/apps 
sudo mkdir -p /mnt/logs
sudo mkdir -p /mnt/opt 
```
We set up permission that will allow our Web servers access the folder to read, write and execute files on NFS:<br>
#No-one is Owner<br>
```
sudo chown -R nobody: /mnt/apps 
sudo chown -R nobody: /mnt/logs 
sudo chown -R nobody: /mnt/opt  
```

#Everyone can modify files <br>
```
sudo chmod -R 777 /mnt/apps 
sudo chmod -R 777 /mnt/logs 
sudo chmod -R 777 /mnt/opt  
```

**_Mount the shared directories to the logical volumes for persistent data_** <br>
- Create mount points on /mnt directory for the logical volumes as follow: <br>


* Mount lv-apps on /mnt/apps – To be used by webservers <br>
`sudo mount /dev/webdata-vg/lv-apps /mnt/apps` <br>


*Mount lv-logs on /mnt/logs – To be used by webserver logs <br>
`sudo mount /dev/webdata-vg/lv-logs /mnt/logs` <br>


* Mount lv-opt on /mnt/opt – To be used by Jenkins server in Project 8 <br>
`sudo mount /dev/webdata-vg/lv-opt /mnt/opt` <br>

- Install NFS server, configure it to start on reboot and make sure it is u and running. <br>
```
sudo yum -y update
sudo yum install nfs-utils -y 
sudo systemctl start nfs-server.service 
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```
- Restart NFS server
```
sudo systemctl restart nfs-server.service
````
# To grant access to NFS clients, we’ll need to define an export file.<br>
The file is typically located at /etc/exports. <br>
`sudo vi /etc/exports` <br>
- Edit the /etc/exports file in a text editor, and add one of the following  directives.
```
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash) 
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash) 
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash) 
```
All the directives below use the options rw, which enables both read and write, sync, which writes changes to disk before allowing users to access the modified file, and no_subtree_check, which means NFS doesn’t check if each subdirectory is accessible to the user.<br>
Make the shared directory available to clients using the exportfs command.<br>
`sudo exportfs -arv`<br>
After running this command, the NFS Kernel should be restarted. <br>
```
sudo systemctl restart nfs-server.service
````
Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)<br>
`rpcinfo -p | grep nfs` <br>
**_Important note: In order for NFS server to be accessible from your client, you must also open following ports: TCP 111, UDP 111, UDP 2049_** <br>

STEP 2 - PREPARE THE DATABASE <br>
- Install MySQL server. <br>
`sudo apt install mysql-server` <br>
`sudo mysql`<br>
- Create a database and name it tooling <br>
`CREATE DATABASE tooling;`<br>
- Create a database user and name it webaccess <br>
`CREATE USER 'webaccess'@'<SUBNET-CIDR>'IDENTIFIED BY 'password';` <br>
- Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr.<br>
`GRANT ALL ON tooling.* TO 'webaccess'@'<SUBNET-CIDR>';` <br>
- Flush Priveleges<br>
`FLUSH PRIVELEGES;` <br>

## STEP 3 - PREPARE THE WEBSERVERS <br>
We need to make sure that our Web Servers can serve the same content from shared storage solutions, in our case – NFS Server and MySQL database.You already know that one DB can be accessed for reads and writes by multiple clients.<br>
For storing shared files that our Web Servers will use – we will utilize NFS and mount previously created Logical Volume "lv-apps" to the folder where Apache stores files to be served to the users (/var/www).<br>
This approach will make our Web Servers stateless, which means we will be able to add new ones or remove them whenever we need, and the integrity of the data (in the database and on NFS) will be preserved.<br>
During the next steps we will do following: <br>
- Configure NFS client (this step must be done on all three servers) <br>
- Deploy a Tooling application to our Web Servers into a shared NFS folder <br>
- Configure the Web Servers to work with a single MySQL database <br>
- Launch a new EC2 instance with RHEL 8 Operating System <br>
- Install NFS client <br>
- `sudo yum install nfs-utils nfs4-acl-tools -y` <br>
- Mount /var/www/ and target the NFS server’s export (directory) for apps <br>
`sudo mkdir /var/wwwsudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www` <br>
- Verify that NFS was mounted successfully by running `df -h`<br>
- Make sure that the changes will persist on Web Server after reboot: <br>
`sudo vi /etc/fstab` 
- Add following line <br>

```
<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
```

- Install Remi’s repository, Apache and PHP <br>
```
sudo yum install httpd -y 
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm 
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-9.rpm 
sudo dnf module reset php
sudo dnf module enable php:remi-7.4 
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd 
sudo systemctl start php-fpm 
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1 
```
**Repeat  Steps 1-5 for the other 2 Servers** <br>

- Verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps.<br>
If you see the same files – it means NFS is mounted correctly. You can try to create a new file touch test.txt from one server and check if the same file is accessible from other Web Servers.<br>

- Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. <br>
- Mount /var/log/httpd and target the NFS server’s export (directory) for logs <br>
`sudo mkdir /var/log/httpd`<br>
`sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/logs /var/log/httpd`<br>
- Verify that NFS was mounted successfully by running `df -h` <br>
- Make sure that the changes will persist on Web Server after reboot:<br>
`sudo vi /etc/fstab`<br>
- Add following line
```<NFS-Server-Private-IP-Address>:/mnt/logs /var/log/httpd nfs defaults 0 0
```
- Fork the tooling source code from Darey.io Github Account to your Github account. (Learn how to fork a repo here)<br>
- Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html <br>
- Download  DevOpsToolingWebsite and copy the html folder to var/www/html <br>

```
mkdir Tooling 
cd   Tooling
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz  sudo rm -rf latest.tar.gz   
sudo cp -R html/.  /var/www/html/ 
```
- Disable SELinux <br>
`sudo setenforce 0` <br>
- To make this change permanent – open following config file <br>
`sudo vi /etc/sysconfig/selinux` and set `SELINUX=disabled` <br>
- Restart httpd`sudo systemctl restart httpd` <br>
- Update the website’s configuration to connect to the database (in /var/www/html/functions.php file) <br>
- Apply tooling-db.sql script to your database using this command <br>
```
mysql -h <databse-private-ip> -u <db-username> -p <db-pasword> < tooling-db.sql
```
- Create in MySQL a new admin user with username: myuser and password: password:<br>
```INSERT INTO ‘users’ (‘id’, ‘username’, ‘password’, ’email’, ‘user_type’, ‘status’) VALUES-> (1, ‘myuser’, ‘5f4dcc3b5aa765d61d8327deb882cf99’, ‘user@mail.com’, ‘admin’, ‘1’);```
- Open the website in your browser http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php and make sure you can login into the website with "myuser" user. <br>

Congratulations!!!
