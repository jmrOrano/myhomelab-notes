

# Setting up ssh server with openssh.
---
*03/29/2026*
*mint cinnamon 22.03 Zena, Kernel 6.8.0, Ubuntu base 24.04 noble*
Purpose: *To keep track of how i configured it and set it up*

>[!Contents]
>- [[#**Check if SSH server is installed**]]
>- [[#**Install SSH Server (Openssh)**]]
>- [[#**Enable service and check status**]]
>- [[#**Configuration File**]]
>- [[#**Test locally**]]
>- [[#**Apply firewall rules**]]
>- [[#**SETTING IT UP WITH KEYPAIRS**]]
>- [[#SECURITY]]
>
>**OTHER INFORMATION**

---
####  **Check if SSH server is installed**
---
```Bash
dpkg -l | grep openssh
```
You should see :
`openssh-client` - for client, connecting to diff machine.
`openssh-server` - for servers, to accept ssh connections.

You can also check if its running:
```Bash
	systemctl status ssh
```

If none a message should appear:
*`Unit ssh.service could not be found`* or *`inactive`*

---
#### **Install SSH Server (Openssh)**
---
```Bash
sudo apt update
sudo apt install openssh-server
```
   
#### **Enable service and check status**
```Bash
   sudo systemctl enable ssh *#auto start on boot*
   sudo systemctl start ssh *#START*
   sudo systemctl status ssh *#check if running*
```

Output should be : *`active(running)`*

#### **Configuration File**

```Bash
sudo nano /etc/ssh/sshd_config
```
   **Settings** :
```text
Port 22 #default port
PermitRootLogin no #but the options can be prohibit-pass or yes
PasswordAuthentication yes #if want to use password login (disable or comment this if using key pairs instead)
LoginGraceTime 30
```
	   
save and restart : 
```Bash
sudo systemctl restart ssh
```
	
####  **Test locally**
From the same machine(server). Run:
```Bash
ssh your_username@localhost
```

A message will dispay :
*`The authenticity of host 'localhost (127.0.0.1) can't be established.` `fingerprint is [SHA256 here] This is not known by` `any other names.'*
*`Do you want to continue?`* Type yes


#### **Apply firewall rules**

   In terminal:
```Bash
sudo ufw allow ssh
#or
sudo ufw limit from 192.168.1.0/24 to any port 22 proto tcp   
``` 
   
   Its better to recreate the rule than edit it.
   To delete : 
```Bash
sudo ufw status numbered
#then
sudo ufw delete [number]
```
    
   In GUI based.
	   `Name: ssh`
	   `Insert: 0`
	   `Policy: Allow`
	   `Direction: In`
	   `Interface: All interface`
	   `Log: Do not log`
	   `Protocol: Both`
	   `Ip from : [192.168.1.0/24] Ip to [blank]`
	   `Port from : [22] Port to [22]`

---

# **SETTING IT UP WITH KEYPAIRS**

**1.Generate an SSH key pair (do this to the client side)**
```Bash
ssh-keygen -t ed25519 -C "yourComment"
```
   **Breakdown:**
   `-t ed25519` - modern and secure key type
   `-C` - for the comment, for easy identification in scalability. 
   
   **A message will appear :** 
	   `Enter file in which to save the key (home/linuxuser/.ssh/id_ed25519):`
	   Press enter to accept the default location.
	   
   **Then it will ask:**
	`Enter passphrase (empty for passphrase)`	
	Enter a passphrase for extra security or leave empty for convenience
	
   Result: Two files are actually created in `~/.ssh/`:
	`id_ed25519` -> **private key** (keep it secret)
	`id_ed25519.pub` -> **Public key** (to share with server)

2.**Copy the public key to the ssh server**
```bash
ssh-copy-id linuxuser@theIp
```
	 
  Manual method (alternative)
```Bash
  cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
  chmod 600 ~/.ssh/authorized_keys
```  

3.**Test key-based login**
```Bash
ssh linuxuser@ipaddress
```
	
  If setup is correct, it should not ask for pass, only the passphrase if set one.
  if it still ask for password, then check the permission
	`chmod 700 ~/.ssh`
	`chmod 600 ~/.ssh/authorized_keys`

4.**{OPTIONAL}Disable password login**
	Refer to number of 4 of [[#Setting up ssh server with openssh.]]

---

# SECURITY

1. **Install fail2ban** 
```Bash
sudo apt update && sudo apt install fail2ban
```

2. **Configure fail2ban, make a 'local' copy of the `jail.conf` file in `etc/fail2ban/`**
   ```
   cd /etc/fail2ban
   sudo cp jail.conf jail.local
   ```
   Now edit the file:
```Bash
sudo nano jail.local
```
	
3. **CONFIGURATION**
```text

   [sshd]
   enabled = true
   port = 2222
   filter = sshd
   logpath = /var/log/auth.log
   maxretry = 5
   bantime = 1h  # set to -1 for perma ban
   findtime = 10m
   
   [DEFAULT]
   ignoreip = 127.0.0.1/8 ::1
   
   [DEFAULT]
   destemail = myemail@email.com
   action = %(action_mw)s
```

**Restart the service**
```
   systemctl restart fail2ban
   sudo fail2ban-client status sshd
```

4. Checking status
```Bash
sudo fail2ban-client status sshd
sudo cat /var/log/fail2ban.log.1
```


---


>[!Question]- **POTENTIAL QUESTIONS**
>Not everyone can just copy their keys to the server. So how to login to this ssh server as a fresh user?
>If other method is allowed first to authenticate like `PasswordAuthentication` which can be enable in `etc/ssh/ssh_conf` then that defeats the purpose of avoiding brute forcing.

>[!INFO]-  **STOPING SSH SERVICE**>
>
>Halt the SSH deamon(sshd)
> ```
> sudo systemctl stop ssh
> ```
>
>Disable from starting boot
>```
>sudo system disable ssh
>```
>Stop and disable ssh.socket for security
>```
>sudo systemctl stop ssh.socket
>sudo systemctl disable ssh.socket
>```


>[!Info]- **REMOVING SSH INCLUDING CONFIGS ( FOR FRESH START)**
>Delete port in ufw
>```
> sudo ufw status numbered 
>```
>Then
>```
>sudo ufw delete <number>
>```
> 
>Remove package 
>```
>sudo apt purge openssh-server
>```
>   
>Clean residual files
>```
>sudo apt autoclean
>sudo apt clean
>```
>    
> Check for leftover config files `ls /etc/ssh/`
> If still see files like `sshd_config` , delete it manually 
> ```
> sudo rm -rf /etc/ssh
> ```
> 
> 

> [!WARNING]-
> You can regenerate for fresh keypair with
> ```
> sudo rm /etc/ssh/ssh_host_*
> sudo dpkg-reconfigure openssh-server
> ```
> 
>Removing and regenerating host keys will change the server's identity. Client that previously connected will see a "host key changed" warning and may refure to connect until they update their `known_host`


# COMMON USAGE AND COMMANDS

### ACCESSING LOCAL NETWORK OF SERVER
*Overview: **it’s possible**, but it’s not automatic. You’d need to configure tunneling or use the server as a gateway, and the network must allow it.*

depends on how the server and its network are configured:

- **Basic SSH session**: By default, you only have access to the server itself. You can run commands, transfer files, and interact with that machine, but not automatically with other devices on its local LAN.
    
-  **Network visibility**: If the server has routing or firewall rules that allow it, you can use the server as a “jump host” to reach other machines on the same local network. For example, you might SSH from the server into another device on that LAN.
    
- **SSH tunneling / port forwarding**: SSH supports features like local port forwarding, remote port forwarding, and dynamic SOCKS proxying. These can let you route your traffic through the SSH server, effectively giving you access to services on its local network (if permitted).
	
-  **Restrictions**: Access depends on permissions, firewall rules, and security policies. Some networks deliberately block this kind of lateral access for safety reasons.

>[!Example] 
>*Suppose there's a web service running *
>```Bash
>curl http://192.168.1.50:8080
>#This command will work if you run it inside your SSH session, because the server can see that local IP and port.
>```
>
>If you want to access that local service **from your own computer’s browser or curl**, you’ll need to set up SSH port forwarding. For instance:
>```Bash
>ssh -L 8080:192.168.1.50:8080 user@your-ssh-server
>#- `-L` means local port forwarding.
>#- The first `8080` is the port on your local machine.
>#- `192.168.1.50:8080` is the target inside the SSH server’s LAN. - `user@your-ssh-server` is your SSH login.
>```
>After running that, you can open your browser at `http://localhost:8080` or run:
>```Bash
>curl http://localhost:8080
>```

>[!Note] Getting an error
>*If you get an error like the ff below:*
>```Error Message 
>curl: (7) Failed to connect to 127.0.0.1 port 5000 after 0 ms: Couldn't connect to server
>```
>
>Perform a test 
>```Bash
>python3 -m http.server 5000
>#That will start a simple HTTP server bound to `127.0.0.1:5000`
>```
>
>Verify locally via curl or accessing the browser directly
>```Bash
>curl http://127.0.0.1:5000
>```
