# simpleC2Redir
Simple C2 Path Redirector with nginx to help learn the basics of C2 infrastructure.
Requirements:
  2 VMs in the cloud
  1 Domain

# 1. Create VMs in Cloud
First create two VMS in your favorite Cloud service (AWS, Azure, etc)
Redir-VM: Will be exposed to the internet as serve as the 'fake website' using nginx 
C2-VM: Will be your C2 (in this case Sliver) and NOT exposed to the internet at all

These two VMs will be in the SAME private network with the following network configs
C2-VM: NO Public IP or Ports open to the Internet
Redir-VM: 
  Inbound 
    - 22 to only your IP(s)
    - 80 to any IP
    - 443 to any IP
    - 8443 to any IP
  Outbound
    - 80 to any IP
    - 443 to any IP
    - 8443 to any IP 

Once you get your public IP for Redir-VM, make an 'A-record' DNS entry to your domain and the public IP.


# 2. Configure each VM
Redir-VM: First lets configure the redirector which will host a fake website using nginx and when a certain path is given, it will redirect to the C2 server.
```
sudo apt update && sudo apt upgrade
sudo apt-get install -y nginx certbot
sudo systemctl stop nginx
sudo certbot certonly --register-unsafely-without-email -d <www.youdomain.com>
Note: for certbot, enter [1] then [y] when prompted
```

Create your fake website in /var/www/html by changing index.html to your own code or use the one in the repo and change the wording a bit to make sense for your domain.

Create your redirector in /etc/nginx/sites-available by adding the default file from the repo to replace the one there.
CHANGE the 'secretpath' to something custom, make sure its random looking and long. so something like:
```
  location /ajisdfYUVSbgseoznauitrcbXBWDG {
                proxy_pass https://<C2-ip>:443;
        }
```
Finally start up nginx
```
sudo systemctl start nginx
```



