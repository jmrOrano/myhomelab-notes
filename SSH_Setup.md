

# OpenSSH Server Setup (Mint / Ubuntu-based)
---
*03/29/2026*
*mint cinnamon 22.03 Zena, Kernel 6.8.0, Ubuntu base 24.04 noble*  
Purpose: *To keep track of how i configured it and set it up*

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
   sudo systemctl enable ssh   #auto start on boot*
   sudo systemctl start ssh    #START
   sudo systemctl status ssh   #check if running
```

Output should be : *`active(running)`*

#### **Configuration File**

```Bash
sudo nano /etc/ssh/sshd_config
```
   **Settings** :
```text
Port 22 #default port
PermitRootLogin no           # or prohibit-password, yes (not recommended)
PasswordAuthentication yes   #if want to use password login (disable or comment this if using key pairs instead)
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
sudo ufw allow ssh #(not recommended for public exposure)
#or with rate limiting within its LAN
sudo ufw limit from 192.168.1.0/24 to any port 22 proto tcp
# or Restrict + Rate limit 
sudo ufw limit ssh
``` 
   
   Its better to recreate the rule than edit it.
   To delete : 
```Bash
sudo ufw status numbered
#then
sudo ufw delete [number]
```
---

# **SETTING IT UP WITH KEYPAIRS**

## 1.Generate an SSH key pair (do this to the client side)
```Bash
ssh-keygen -t ed25519 -C "yourComment"
```
   **Breakdown:**  
  - `-t ed25519` - modern and secure key type  
  - `-C` - for the comment, for easy identification in scalability.   
   
   **A message will appear :**   
	   `Enter file in which to save the key (home/linuxuser/.ssh/id_ed25519):`  
	   Press enter to accept the default location.  
	   
   **Then it will ask:**  
	`Enter passphrase (empty for passphrase)`	  
	Enter a passphrase for extra security or leave empty for convenience  
	
   Result: Two files are actually created in `~/.ssh/`:  
-	`id_ed25519` -> **private key** (keep it secret)  
-	`id_ed25519.pub` -> **Public key** (to share with server)  

## 2.Copy the public key to the ssh server
```bash
ssh-copy-id linuxuser@theIp
```
	 
  Manual method (alternative)
> Run this on the SERVER
```Bash
  cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
  chmod 600 ~/.ssh/authorized_keys
```  

## 3.Test key-based login
```Bash
ssh linuxuser@ipaddress
```
	
  If setup is correct, it should not ask for pass, only the passphrase if set one.
  if it still ask for password, then check the permission
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
## 4. RECOMMENDED: Disable password login  
  Refer to config file [OpenSSH_Config](#configuration-file)

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
   port = 22
   filter = sshd
   logpath = /var/log/auth.log
   maxretry = 5
   bantime = 1h  # set to -1 for perma ban
   findtime = 10m
   
   [DEFAULT]
   ignoreip = 127.0.0.1/8 ::1
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


## ❓ Potential Questions

Not everyone can simply copy their keys to the server. How can a fresh user authenticate to the SSH server?

If `PasswordAuthentication` is temporarily enabled in `/etc/ssh/sshd_config`, does that partially defeat the purpose of preventing brute-force attacks?

---

## ℹ️ Stopping the SSH Service

Halt the SSH daemon (`sshd`):

```bash
sudo systemctl stop ssh
```

Disable SSH from starting at boot:

```bash
sudo systemctl disable ssh
```

Stop and disable `ssh.socket` for additional security:

```bash
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket
```

---

## 🧹 Removing SSH Including Configurations (Fresh Start)

Delete the SSH rule from UFW:

```bash
sudo ufw status numbered
```

Then remove the rule:

```bash
sudo ufw delete <number>
```

Remove the package:

```bash
sudo apt purge openssh-server
```

Clean residual package files:

```bash
sudo apt autoclean
sudo apt clean
```

Check for leftover configuration files:

```bash
ls /etc/ssh/
```

If files such as `sshd_config` still exist, remove them manually:

```bash
sudo rm -rf /etc/ssh
```

---

## ⚠️ Regenerating Host Keys

Generate a fresh host key pair:

```bash
sudo rm /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
```

> **Warning**
>
> Regenerating host keys changes the server's identity. Clients that have previously connected will receive a **"host key changed"** warning and may refuse to connect until their `known_hosts` entries are updated.

# COMMON SSH USAGE (NETWORK ACCESS & TUNNELING)

## Accessing Local Network of Server

*Overview: it’s possible, but not automatic. You need tunneling or a jump host setup, and the network must allow it.*

Depends on configuration of the server and network:

- **Basic SSH session**  
  You only access the server itself. You can run commands, transfer files, and manage that machine only.

- **Network visibility**  
  If routing/firewall allows it, the server can act as a jump host to reach other devices on its LAN.

- **SSH tunneling / port forwarding**  
  SSH supports local forwarding, remote forwarding, and dynamic SOCKS proxying to access internal services.

- **Restrictions**  
  Access depends on firewall rules, routing, and security policies. Many networks block lateral movement.

---

> **Example**

Suppose there is a service running:

```bash
curl http://192.168.1.50:8080
```

This works inside the SSH session if the server can reach that IP.

To access it from your own machine:

```bash
ssh -L 8080:192.168.1.50:8080 user@your-ssh-server
```

Then:

```bash
curl http://localhost:8080
```

---

> **Note: Testing local server**

If you get an error like:

```text
curl: (7) Failed to connect to 127.0.0.1 port 5000
```

Test with:

```bash
python3 -m http.server 5000
```

Then verify:

```bash
curl http://127.0.0.1:5000
```
