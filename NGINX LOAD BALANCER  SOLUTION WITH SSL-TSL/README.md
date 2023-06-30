# LOAD BALANCER SOLUTION WITH NGINX AND SSL/TLS
CompletedSubmit Project for Review
By now you have learned what Load Balancing is used for and have configured an LB solution using Apache, but a DevOps engineer must be a versatile professional and know different alternative solutions for the same problem.<br>
That is why, in this project we will configure an Nginx Load Balancer solution.<br>
It is also extremely important to ensure that connections to your Web solutions are secure and information is encrypted in transit – we will also cover connection over secured HTTP (HTTPS protocol), its purpose and what is required to implement it.<br>
When data is moving between a client (browser) and a Web Server over the Internet – it passes through multiple network devices and, if the data is not encrypted, it can be relatively easy intercepted by someone who has access to the intermediate equipment. This kind of information security threat is called Man-In-The-Middle (MIMT) attack.
This threat is real – users that share sensitive information (bank details, social media access credentials, etc.) via non-secured channels, risk their data to be compromised and used by cybercriminals.<br>
SSL and its newer version, TSL – is a security technology that protects connection from MITM attacks by creating an encrypted session between browser and Web server. Here we will refer this family of cryptographic protocols as SSL/TLS – even though SSL was replaced by TLS, the term is still being widely used.
SSL/TLS uses digital certificates to identify and validate a Website. A browser reads the certificate issued by a Certificate Authority (CA) to make sure that the website is registered in the CA so it can be trusted to establish a secured connection.<br>
There are different types of SSL/TLS certificates – you can learn more about them here. You can also watch a tutorial on how SSL works here or an additional resource here<br>
In this project you will register your website with LetsEnrcypt Certificate Authority, to automate certificate issuance you will use a shell client recommended by LetsEncrypt – cetrbot.<br>

## Task<br>
This project consists of two parts:<br>
- Configure Nginx as a Load Balancer 
- Register a new domain name and configure secured connection using SSL/TLS certificates

Your target architecture will look like this:

![0_setup](https://github.com/ifydevops23/DevOps_Tooling_Website_Solution/assets/126971054/b0b87554-aa1b-4a0c-903b-cec51afda452)

Let us get started!<br>

### CONFIGURE NGINX AS A LOAD BALANCER <br>
You can either uninstall Apache from the existing Load Balancer server, or create a fresh installation of Linux for Nginx.<br>
- Create an EC2 VM based on Ubuntu Server 20.04 LTS and name it Nginx LB (do not forget to open TCP port 80 for HTTP connections, also open TCP port 443 – this port is used for secured HTTPS connections)
- Update /etc/hosts file for local DNS with Web Servers’ names (e.g. Web1 and Web2) and their local IP addresses
- Install and configure Nginx as a load balancer to point traffic to the resolvable DNS names of the webservers
- Update the instance and Install Nginx<br>
`sudo apt update`<br>
`sudo apt install nginx`<br>
- Configure Nginx LB using Web Servers’ names defined in /etc/hosts<br>
Hint: Read this blog to read about /etc/host
- Open the default nginx configuration file<br>
`sudo vi /etc/nginx/nginx.conf`<br>
#insert following configuration into http section

```
 upstream myproject {
    server Web1 weight=5;
    server Web2 weight=5;
  }


server {
    listen 80;
    server_name www.domain.com;
    location / {
      proxy_pass http://myproject;
    }
  }
```

#comment out this line <br>
#       include /etc/nginx/sites-enabled/*;<br>

- Restart Nginx and make sure the service is up and running <br>
`sudo systemctl restart nginx`<br>
`sudo systemctl status nginx`<br>

Side Self Study: Read more about HTTP load balancing methods and features supported by Nginx on this page

### REGISTER A NEW DOMAIN NAME AND CONFIGURE SECURED CONNECTION USING SSL/TLS CERTIFICATES
Let us make necessary configurations to make connections to our Tooling Web Solution secured!
- In order to get a valid SSL certificate – you need to register a new domain name, you can do it using any Domain name registrar – a company that manages reservation of domain names. The most popular ones are: Godaddy.com, Domain.com, Bluehost.com.
- Register a new domain name with any registrar of your choice in any domain zone (e.g. .com, .net, .org, .edu, .info, .xyz or any other)
- Assign an Elastic IP to your Nginx LB server and associate your domain name with this Elastic IP<br>
You might have noticed, that every time you restart or stop/start your EC2 instance – you get a new public IP address. When you want to associate your domain name – it is better to have a static IP address that does not change after reboot. Elastic IP is the solution for this problem, learn how to allocate an Elastic IP and associate it with an EC2 server on this page.
Update A record in your registrar to point to Nginx LB using Elastic IP address
Learn how associate your domain name to your Elastic IP on this page.<br>

Side Self Study: Read about different DNS record types and learn what they are used for.<br>

Check that your Web Servers can be reached from your browser using new domain name using HTTP protocol – http://<your-domain-name.com><br>
Configure Nginx to recognize your new domain name<br>
Update your nginx.conf with server_name www.<your-domain-name.com> instead of server_name www.domain.com<br>
- Install certbot and request for an SSL/TLS certificate<br>
- Make sure snapd service is active and running<br>
`sudo systemctl status snapd`
- Install certbot <br>
`sudo snap install --classic certbot`<br>
- Request your certificate (just follow the certbot instructions – you will need to choose which domain you want your certificate to be issued for, domain name will be looked up from nginx.conf file so make sure you have updated it on step 4).<br>
`sudo ln -s /snap/bin/certbot /usr/bin/certbot`<br>
`sudo certbot --nginx`<br>
- Test secured access to your Web Solution by trying to reach https://<your-domain-name.com><br>
You shall be able to access your website by using HTTPS protocol (that uses TCP port 443) and see a padlock pictogram in your browser’s search string.
Click on the padlock icon and you can see the details of the certificate issued for your website.<br>

- Set up periodical renewal of your SSL/TLS certificate
By default, LetsEncrypt certificate is valid for 90 days, so it is recommended to renew it at least every 60 days or more frequently.
You can test renewal command in dry-run mode<br>
`sudo certbot renew --dry-run`<br>
Best pracice is to have a scheduled job that to run renew command periodically. Let us configure a cronjob to run the command twice a day.
To do so, lets edit the crontab file with the following command:<br>
`crontab -e`<br>
Add following line:<br>
```
* */12 * * *   root /usr/bin/certbot renew > /dev/null 2>&1
```
You can always change the interval of this cronjob if twice a day is too often by adjusting schedule expression.<br>
Side Self Study: Refresh your cron configuration knowledge by watching this video.<br>
You can also use this handy online cron expression editor.<br>
Congratulations!<br>
You have just implemented an Nginx Load Balancing Web Solution with secured HTTPS connection with periodically updated SSL/TLS certificates.<br>


    
