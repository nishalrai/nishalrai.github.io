---
title: Investigate and confirm the status of your System
author: nishalrai
date: 2022-08-16 20:55:00 +0800
categories: [Offensive, Forensics]
tags: [forensics]
---

Investigate and confirm whether the system is under attack or just being paranoid ie. e client in a cooperative environment – Want to know how and where to start from? Follow these steps to be almost sure of the issue.

Investigation can be carried out by adopting some predefined cybersecurity frameworks developed by different well-esteemed Organizations in the field of Information Technology and cybersecurity to tackle such a scenario. NIST Cyber Security Framework, COBIT, ISO/IEC Standards, COSO, NERC, TY Cyber, HITRUST CSF, and many others are some of the well defined Cyber Security Framework is available to use.

However, there are no specific rules to undergo such an investigation and can proceed through anyway to solve such a scenario in the best possible way. We can divide such investigation into two major parts:

# For the verification of Cyber Attack
- If there is a firewall, then it might have logged suspicious programs trying to set up a server, or the antivirus alerted about some trojan in the system.
- Scanning the packets of the network using tools like WireShark, CloudShark, and others to analyze any suspicious packets in the system network.
- Examing the resources of the system which might contain Phishing Emails, Malicious Attachments, Password, and other suspicious requests.
- Run a full-system-wide file check of the system using specific tools like NIS filecheck that will create cryptographically strong hashes from the files which are specified or all files based on their extension specified. Since the two files cannot have the identical hash unless they are exactly the same file. So, it will detect the changes in the file on the system easily.
- Run the command prompt and type “netstat” to see all the connections in and out of the computer. To check whether an unauthorized user is “Listening” or “connected” in the system with what port or not. Tools like Active Ports displays what programs are using what ports to connect where. It is very helpful to find the remote access trojans (RAT:s) or other software that is on the computer and has a connection to the outside world without any user’s authorization. 

## cmd
- Check what processes are running in the background and check for strange activities like “backdoor.exe” or “app.exe” or “tool.exe”, “service.exe”, “help.exe”, “system.exe”, “windows.exe” or anything new in the system in the taskmanager. However, some trojans can even ‘tap’ into existing programs using a trick called .dll injection, so it cannot be easily spotted from the task manager. So, Process Explorer can be used to show every program and dll that is running in the system to be sure of unwanted programs.

- To check the startup, to know what gets started up during the reboot of the system.

## startup manager
- Run a full deep system scan by antivirus to encounter any malicious program in the system.

Even if we follow all these steps mentioned above and don’t encounter any suspicious activity or program in the system then it’s almost possible to consider as paranoid.

But if we encounter any cyber attacks activity during the above process. Here,


# After verification of Cyber Attack
- Protection of Devices
  The compromised host of the system needs to find and eliminate the malicious activity either by running an anti-malware program or disconnecting from the system in severe cases until the problem gets resolved.

- Response to the threat
  A well-prepared Incident Response Planning (IRP) is very crucial in such a scenario and also applying the necessary safeguard mechanism will mitigate the risk and catastrophe in an Organization. Then, the legal action comes into play after the problem gets resolved where all the log activity and damages of the system will be used as evidence against such attackers.

IRP (Incident Response Planning), an element in the triad of Business Continuity and Disaster Recovery (BCDR or BC/DR) Planning. IRP is the documented and coordinated method of addressing and managing a security breach or attack in the system. The ultimate goal behind IRP is to effectively resolve the incident so that the scale of damage is limited. Also, both recovery time and costs, as well as collateral damage such as brand reputation, is kept a minimum.