## STEP 0 - REQUIREMENTS
1. Two RHEL8 Web Servers
2. One MySQL DB Server (based on Ubuntu 20.04)
3. One RHEL8 NFS server
## STEP 1 - PREREQUISITE CONFIGURATION
- Apache (httpd) process running in both webservers.
- /var/www mounted on /mnt/apps of the NFS server
- TCP and UDP ports open.(80,3306,111,2049)
- Tooling website can be open via the Public IPs of the various webservers.
