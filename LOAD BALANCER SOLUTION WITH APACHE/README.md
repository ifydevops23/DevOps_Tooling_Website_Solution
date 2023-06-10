# LOAD BALANCER SOLUTION WITH APACHE <br>

![3_architecture_1](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/3c7250a2-d343-4edf-b38a-3f24d4c51e5d)

When we access a website in the Internet we use an URL and we do not really know how many servers are out there serving our requests, this complexity is hidden from a regular user, but in the case of websites that are being visited by millions of users per day (like Google or Reddit), it is impossible to serve all the users from a single Web Server (it is also applicable to databases, but for now we will not focus on distributed DBs).<br>
Each URL contains a domain name part, which is translated (resolved) to the IP address of a target server that will serve requests when open a website in the Internet. Translation (resolution) of domain names is performed by DNS servers, the most commonly used one has a public IP address 8.8.8.8 and belongs to Google. You can try to query it with nslookup command:<br>
`nslookup 8.8.8.8`<br>

![1_nslookup](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/c04a5347-c261-4849-bbb7-279e17fb76f4)

In our set-up in Project-7, we had 3 Web Servers and each of them had its own public IP address and public DNS name. <br>
A client has to access them by using different URLs, which is not a nice user experience to remember the addresses/names of even 3 servers, let alone millions of Google servers. <br>
In order to hide all this complexity and to have a single point of access with a single public IP address/name, a Load Balancer can be used. <br>

A Load Balancer (LB) distributes clients’ requests among underlying Web Servers and makes sure that the load is distributed in an optimal way.<br>

In this project, we will enhance our Tooling Website solution by adding a Load Balancer to distribute traffic between Web Servers and allow users to access our website using a single URL. <br>

## STEP 0 - REQUIREMENTS
1. Two RHEL8 Web Servers
2. One MySQL DB Server (based on Ubuntu 20.04)
3. One RHEL8 NFS server

## STEP 1 - PREREQUISITE CONFIGURATION
- Apache (httpd) process running in both webservers.
- /var/www mounted on /mnt/apps of the NFS server
- TCP and UDP ports open.(80,3306,111,2049)
- Tooling website can be open via the Public IPs of the various webservers.


## STEP 2 - CONFIGURE APACHE AS A LOAD BALANCER
1. Create an Ubuntu Server 20.04 EC2 instance and name it Project-8-apache-lb, so your EC2 list will look like this:<br>

![3_running_instances_prereq](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/2b15e83e-36cf-460d-8e6d-15b2d6985bbd)

2. Open TCP port 80 on Project-8-apache-lb by creating an Inbound Rule in the Security Group.
3. Install Apache Load Balancer on Project-8-apache-lb server and configure it to point traffic coming to LB to both Web Servers:<br>
#Install apache2 <br>
`sudo apt update` <br>
`sudo apt install apache2 -y` <br>
![3_install_apache](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/e744df2b-5fb8-48db-85a7-b10c0d149516)
`sudo apt-get install libxml2-dev` <br>
![3_install_libxml](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/c367c8bc-2139-401c-bde6-69189625f896)

#Enable the following modules: <br>
```
sudo a2enmod rewrite 
sudo a2enmod proxy 
sudo a2enmod proxy_balancer 
sudo a2enmod proxy_http 
sudo a2enmod headers 
sudo a2enmod lbmethod_bytraffic 
```
![3_enable_proxy](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/bf9fd5c8-8c73-425e-a37f-7bfd2105be16)<br>

#Restart apache2 service <br>
`sudo systemctl restart apache2`
Make sure apache2 is up and running <br>
`sudo systemctl status apache2`<br>
Configure load balancing <br>
`sudo vi /etc/apache2/sites-available/000-default.conf` <br>
#Add this configuration into this section <VirtualHost *:80>  </VirtualHost>

```
<Proxy "balancer://mycluster">
               BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=1
               BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=1
               ProxySet lbmethod=bytraffic
               # ProxySet lbmethod=byrequests
        </Proxy>


        ProxyPreserveHost On
        ProxyPass / balancer://mycluster/
        ProxyPassReverse / balancer://mycluster/
```

![3_edit_config_file22](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/6ae0639f-63da-488c-a417-219b433beca1)

#Restart apache server<br>
`sudo systemctl restart apache2`<br>

'bytraffic' balancing method will distribute the incoming load between your Web Servers according to the current traffic load. <br>
We can control in which proportion the traffic must be distributed by loadfactor parameter. <br>

4. Verify that our configuration works – try to access your LB’s public IP address or Public DNS name from your browser:<br>
`http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php` <br>
![3_login_page_from loadbalancer](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/d9ef1855-7976-444e-a597-03ae809ce258)<br>

Login as 'admin' user <br>

![3_hello_from_propitix_after_Login](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/4346d4fb-49a9-4143-adec-e46ec472163e)

Note: If in Project-7 you mounted /var/log/httpd/ from your Web Servers to the NFS server – unmount them and make sure that each Web Server has its own log directory.<br>
`sudo umount -f /var/log/httpd`

Open two ssh/Putty consoles for both Web Servers and run the following command:<br>
`sudo tail -f /var/log/httpd/access_log`<br>

![3_log_from_webserver_2](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/6f706860-1c2a-4d4d-8455-a6e20637e627)

Try to refresh your browser page <br>
`http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php` several times and ensure both servers receive HTTP GET requests from your LB – new records must appear in each server’s log file. <br>

![3_check_log_for_get_requests](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/397d95fa-68d6-46ec-afe9-a6cb30af64e0)

The number of requests to each server will be approximately the same since we set loadfactor to the same value for both servers – it means that traffic will be distributed evenly between them.If you have configured everything correctly – your users will not even notice that their requests are served by more than one server. <br>

### **Optional Step – Configure Local DNS Names Resolution**<br>
Sometimes it is tedious to remember and switch between IP addresses, especially if you have a lot of servers under your management.<br>What we can do, is to configure local domain name resolution.<br>
The easiest way is to use /etc/hosts file, although this approach is not very scalable, it is very easy to configure and shows the concept well. <br>
So let us configure the IP address to domain name mapping for our LB.#Open this file on your LB server <br>
`sudo vi /etc/hosts`<br>

![3_etc_hosts](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/db3a47bc-35a4-44e0-a6a4-accdcb91ca6c)

#Add 2 records into this file with the Local IP address and arbitrary name for both of your Web Servers
```
<WebServer1-Private-IP-Address> Web1
<WebServer2-Private-IP-Address> Web2
```
Now you can update your LB config file with those names instead of IP addresses.
```
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
```
![3_edit_config_to_hostname](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/0d6a5da9-0212-4cca-b106-adb2f03df1bc)

You can try to curl your Web Servers from LB locally<br> 
`curl http://Web1` or `curl http://Web2`<br>

![3_curl_from_web1](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/8c7ebfc3-a6f8-4daa-ad22-0d5a2c8b1c1a)

Remember, this is only an internal configuration and it is also local to your LB server, these names will neither be ‘resolvable’ from **other servers** internally nor from the Internet.<br>

![desired_architecture](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/58b51d85-88ae-4070-bc61-fbac0c24509c)

Congratulations!!!
