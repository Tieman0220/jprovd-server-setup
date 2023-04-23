## Initial Setup

This documentation is assuming you have a domain already purchased and a DNS A record pointed to the server. It also assumes you are running this on an OVH Bare Metal server but with a good amount of judgement can be used otherwise as well.

### Housekeeping/Security Standardization
SSH into server using credentials provided by OVH
1. Download and install updates  
	 `sudo apt update`  
	 `sudo apt upgrade`
2. Add new user  
	 `sudo adduser [USERNAME]`
3. Add new used to sudo group  
	 `sudo usermod -aG sudo [USERNAME]`
4. Reboot server  
	 `sudo reboot now`
5. Login after reboot as new user and disable default user   
	 `sudo usermod --expiredate 1 ubuntu`
#### [Firewall Setup](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-22-04)
6. Allow port 22 for SSH access  
	`sudo ufw allow 22`
7. Allow port 80 for Certbot/NGINX  
    `sudo ufw allow 80`
8. Allow port 443 for Certbot/NGINX  
    `sudo ufw allow 443`
9. Allow port 3333 for jprovd  
	`sudo ufw allow 3333`
10. Enable firewall  
	 `sudo ufw enable`
11. Verify settings  
	 `sudo ufw status`
#### [Fail2Ban Setup](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-20-04)
12. Install fail2ban  
	 `sudo apt install fail2ban`
13. Run fail2ban as a system service  
	 `sudo systemctl enable fail2ban.service`
#### Monitoring
14. [Netdata](https://learn.netdata.cloud/docs/getting-started/)
#### [Enable Unattended Upgrades](https://www.digitalocean.com/community/tutorials/how-to-keep-ubuntu-22-04-servers-updated)
15. Install Unattended Upgrades package  
    `sudo apt install unattended-upgrades`
16. Configure Unattended Upgrades  
 `sudo nano /etc/apt/apt.conf.d/50unattended-upgrades`
17. Restart Unattended Upgrades (if changes made)  
    `sudo systemctl restart unattended-upgrades.service`
18. Make sure Unattended Upgrades is running  
    `sudo systemctl status unattended-upgrades.service`
#### [Disable phased updates](https://askubuntu.com/questions/1421222/update-upgrade-not-working-because-of-phased-updates)
19. Create new apt configuration  
    `sudo nano /etc/apt/apt.conf.d/30phased-upgrades`  
    `APT::Get::Always-Include-Phased-Updates "true";`
#### [Allocate Swap Space](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-22-04)
20. Check free disk space  
    `df -h`
21. Create swap file of appropriate size  
    `sudo fallocate -l 260G /swapfile`
22. Make swapfile only accessible to root  
    `sudo chmod 600 /swapfile`
23. Mark file as swap space  
    `sudo mkswap /swapfile`
24. Enable swap file  
    `sudo swapon /swapfile`
25. Check to make sure new swap space is available  
    `free -h`
26. Backup existing fstab file  
    `sudo cp /etc/fstab /etc/fstab.bak`
27. Add new swap to the end of fstab to enable automatic mounting during boot   
    `echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab`
28. Change swappiness  
    `sudo sysctl vm.swappiness=10`
29. Change cache pressure  
    `sudo sysctl vm.vfs_cache_pressure=50`
30. Persist changes across boot  
    `sudo nano /etc/sysctl.conf`  
        ```
         vm.swappiness=10
         vm.vfs_cache_pressure=50
        ```
### ZFS Setup
31. Install ZFS prerequisites  
    `sudo apt install zfsutils-linux`
32. List disks  
    `sudo fdisk -l`
33. List disks again to verify  
    `lsblk`
34. Create striped zpool  
    `sudo zpool create [ZFS MOUNT POINT] /dev/(whatever)`
35. Check pool status  
    `sudo zpool status`
### [NGINX Setup](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04)
36. Install NGINX      
    `sudo apt install nginx`
37. Create configuration for jprovd  
    `sudo nano /etc/nginx/sites-available/[SUBDOMAIN]`

         server {
        
                server_name [DOMAIN];
        
                location / {
                        proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_pass http://localhost:3333;
                }
        

         }
        
         server {
                listen 443;
                listen [::]:433;
        
                server_name [DOMAIN];
        
                location / {
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                        proxy_pass http://localhost:3333;
                }
         }

         
38. Symlink to enable config   
    `sudo ln -s /etc/nginx/sites-available/[SUBDOMAIN] /etc/nginx/sites-enabled/`
39. Check NGINX config for relevant jprovd settings  
    `sudo nano /etc/nginx/conf.d/settings.conf`
     	
	    client_max_body_size 0;  
        proxy_http_version 1.1;  
        proxy_request_buffering off;  
        proxy_read_timeout 1800;  
        proxy_connect_timeout 1800;  
        proxy_send_timeout 1800;  
	
	 
40. Check NGINX config for relevant jprovd settings pt 2  
    `sudo nano /etc/nginx/nginx.conf`
     
         user www-data;
         worker_processes auto;
         pid /run/nginx.pid;
         include /etc/nginx/modules-enabled/*.conf;
         
         events {
             worker_connections 768;
             # multi_accept on;
         }
         
         http {
         
             ##
             # Basic Settings
             ##
         
             sendfile on;
             tcp_nopush on;
             types_hash_max_size 2048;
             # server_tokens off;
         
             # server_names_hash_bucket_size 64;
             # server_name_in_redirect off;
         
             include /etc/nginx/mime.types;
             default_type application/octet-stream;
         
             ##
             # SSL Settings
             ##
         
             ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
             ssl_prefer_server_ciphers on;
         
             ##
             # Logging Settings
             ##
         
             access_log /var/log/nginx/access.log;
             error_log /var/log/nginx/error.log;
         
             ##
             # Gzip Settings
             ##
         
             gzip on;
         
             # gzip_vary on;
             # gzip_proxied any;
             # gzip_comp_level 6;
             # gzip_buffers 16 8k;
             # gzip_http_version 1.1;
             # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
         
             ##
             # Virtual Host Configs
             ##
         
             include /etc/nginx/conf.d/*.conf;
             include /etc/nginx/sites-enabled/*;
         }
         
41. Verify NGINX configuration
    `sudo nginx -t`
42. Restart NGINX to apply configuration changes  
    `sudo systemctl restart nginx`
 #### [Certbot Setup](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04)
43. Install Certbot Prerequisites  
    `sudo snap install core; sudo snap refresh core`  
    `sudo snap install --classic certbot`
44. Link the certbot command from the snap install directory to your path  
    `sudo ln -s /snap/bin/certbot /usr/bin/certbot`
45. Run certbot to obtain certificate  
    `sudo certbot --nginx -d [DOMAIN]`
46. Verify NGINX configuration  
    `sudo nginx -t`
47. Restart NGINX to apply changes  
    `sudo systemctl restart nginx`
 ### [jprovd Installation and Setup](https://docs.jackaldao.com/docs/nodes/providers/setting_up)
48. Install prerequisites  
    `sudo apt install build-essential lz4 jq`
49. Install Go  
    `GOVER=$(curl https://go.dev/VERSION?m=text)`  
    `wget https://golang.org/dl/${GOVER}.linux-amd64.tar.gz`  
    `sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf ${GOVER}.linux-amd64.tar.gz`
50. Add golang path info to profile  
    `nano ~/.profile`
     
         # add environmental variables for Go
         if [ -f "/usr/local/go/bin/go" ] ; then
             export GOROOT=/usr/local/go
             export GOPATH=${HOME}/go
             export GOBIN=$GOPATH/bin
             export PATH=${PATH}:${GOROOT}/bin:${GOBIN}
             export GO111MODULE=on
         fi
         
51. Reload profile  
    `source ~/.profile`
52. Install using git  
    `git clone https://github.com/JackalLabs/canine-provider.git`  
    `cd canine-provider`  
    `git pull`  
    `git checkout [VERSION]`
53. Install jprovd  
    `make install`
54. Generate private key and wallet for provider  
    `jprovd client gen-key --home=/[ZFS MOUNT POINT]`
55. Add funds to your new provider wallet to cover gas. 50 or so jkl should do.
56. Edit jprovd configuration  
    `cd /[ZFS MOUNT POINT]/config`  
    `nano client.toml`
     
         # This is a TOML config file.
         # For more information, see https://github.com/toml-lang/toml
         
         ###############################################################################
         ###                           Client Configuration                            ###
         ###############################################################################
         
         # The network chain ID
         chain-id = "jackal-1"
         # The keyring's backend, where the keys are stored (os|file|kwallet|pass|test|memory)
         keyring-backend = ""
         # CLI output format (text|json)
         output = "text"
         # <host>:<port> to Tendermint RPC interface for this chain
         node = "[YOUR RPC NODE]"
         # Transaction broadcasting mode (sync|async|block)
         broadcast-mode = ""

57. Initialize provider  
    `jprovd init "https://[DOMAIN]" "[STORAGE SPACE IN BYTES]" "" --home=/[ZFS MOUNT POINT]`
58. Create [unit file](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files) to run jprovd as a service  
    `sudo nano /etc/systemd/system/jprovd.service`
     
         [Unit]
         Description=Jackal Provider
         After=network-online.target
         [Service]
         User=[USERNAME]
         ExecStart=/home/[USERNAME]/go/bin/jprovd start --home=/[ZFS MOUNT POINT]
         Restart=always
         RestartSec=3
         LimitNOFILE=4096
         [Install]
         WantedBy=multi-user.target
         
59. Reload unit files  
    `sudo systemctl daemon-reload`
60. Enable jprovd as a service  
    `sudo systemctl enable jprovd.service`
61. Start jprovd  
    `sudo systemctl start jprovd.service`
62. Verify jprovd status  
    `sudo systemctl status jprovd.service`
63. Backup your provider private key somehow - write it down or put it in your password manager  
    `cat /[ZFS MOUNT POINT]/config/priv_storkey.json`
