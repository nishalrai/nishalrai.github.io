---
title: Red Teaming - Pivoting Perspectives
author: nirajkharel
date: 2024-01-22 20:55:00 +0800
categories: [Red Teaming, Pivoting]
tags: [Red Teaming, Pivoting]
---


Alright, imagine this hilarious little memo I wrote down for my future self: "Yo, Future Me! I made this easy-peasy guide for ya, just in case you hack into that machine someday. No hacking stuff here, no sneaky exploits or network shenanigans. You know me, always forgetting the fancy tricks!" 🕵️‍♂️

This refers to the _**assume breach**_ scenario, where it is expected that you already have initial access to the network.

## Port Forwarding
Let's suppose you have gained access to a machine that has certain open ports on the localhost, making them inaccessible to other devices on the network. However, the machine may have interesting ports related to HTTP, MySQL, Mssql, PostgreSQL, and others. If you can interact with the machine through a command line or shell, it is still possible for an attacker machine to access those services using techniques like port forwarding and reverse port forwarding.

But before that, let's summarize the **local port forwarding**, **remote port forwarding**, **dynamic port forwarding** and **reverse dynamic port forwarding**.

### Local Port Forwarding
- Gain access to a port that is exclusively reachable by the victim's machine or network (e.g., firewalled or internal localhost network). Local port forwarding enables the Pentester to make a remote service accessible on their own machine.
- _I want to access remote resources that I can't access._
- Let's consider a situation where the attacker can reach the machine at IP address 10.10.0.3. However, this machine has an additional interface, IP address 10.20.0.6. In the same network, there is another machine with IP address 10.20.0.5, which has an open port 8080. Despite having access to the first machine, the attacker cannot directly interact with port 8080 on the second machine.
- To overcome this restriction, we can establish an SSH tunnel between IP addresses 10.10.0.2 and 10.10.0.3. By doing this, we can create a pathway to transmit data. The attacker sets up a local port on their own machine, and any data traffic directed to port 9090 on the attacker's machine will be redirected through the tunnel to port 8080 on the machine with IP address 10.20.0.5.
- Now, if the attacker accesses localhost:9090, the SSH tunnel will cleverly embed the content within the TCP packet, allowing it to be transmitted through the tunnel to the target machine at IP address 10.20.0.5 on port 8080.
<p align="center">
  <img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/pivot-1.png">
</p>

#### Local Port Forwarding with SSH
```bash
ssh -L localport:remote-ip:remote-port user@remote-ip

# In case of above theoretical condition
ssh -L 9090:10.20.0.5:8080 user@10.10.0.3

ssh -L localport:localhost:remoteport user@remote-ip

# Forwarding multipe ports
ssh -L localport:remote-ip:remote-port localport2:remote-ip:remote-port2 user@remote-ip
```

#### Local Port Forwarding with Metasploit
```bash
# After you get the meterpreter session
portfwd add -L <attacker-ip> -l <local port> -p <remote port on targets machine> -r <target ip>

# List the forwarded ports
portfwd list

# Delete the forwarded ports
portfwd delete -l <local port> -p <remote port on targets machine> -r <target ip>

# Delete all
portfwd flush

# In case of above theoretical condition
portfwd add -L 10.10.0.2 -l 9090 -p 8080 -r 10.20.0.5
```
- Where,
  - "-l" is the local port that an attacker will communicate on its machine
  - "-p" is the remote port that an attacker will communicate through its local port.
  - "-r" is the ip address of a machine whose remote port is required to be accessed.

#### Local Port Forwarding with Chisel
```bash
# In remote machine
chisel server -p <listen-port>

# In  attacker machine
chisel client <listen-ip>:<listen-port> <local-port>:<target-ip>:<target-port>

# In case of above theoretical condition

# In remote machine
chisel server -p 9999

# In attacker machine
chisel client 10.10.0.3:9999 9090:10.20.0.5:8080

# After that we can connect to the port 8080 from local machine through port 9090 i.e. localhost:9090

# Forwarding multiple ports
chisel client 10.10.0.3:9999 9090:10.20.0.5:8080 9091:10.20.0.5:8081
```

#### Local Port Forwarding with Plink
```bash
plink.exe -ssh -L localport:remote-ip:remoteport user@remote-ip

# In case of above theoretical condition
plink.exe -ssh -L 9090:10.20.0.5:8080 user@10.10.0.3

# Create a firewall rule allowing incoming connections on the TCP port
netsh advfirewall firewall add rule name="ALLOW TCP PORT 9090" dir=in action=allow protocol=TCP localport=9090
```

### Remote/Reverse Port Forwarding
- With remote port forwarding, the pentester/attacker machine is the one listening locally for traffic from the compromised server. This will be useful if you found the internal services running on a victim machine or on it's local network and firewal does not allow inbound traffic for these services. It can also be used for relaying traffic from internal running srvices to be accessible externally for a specific machine.
- _I want people to access local resoures that they don't have access to._
- The scenario is same as above where we have comproised or have an SSH credentials of machine 10.10.0.3. However, the firewall has been reconfigured to prevent incoming traffic on port 8080 of the IP address 10.20.0.5. As a result, direct communication with the device using local port forwarding is no longer feasible. In this scenario, remote port forwarding comes into play. Here, it's the attacker's machine that actively listens for traffic from the compromised server. To achieve this, the compromised machine initiates an SSH session with the attacker's machine, which then sets up a listener on port 9090. Any incoming traffic from the attacker's machine on it's port 9090 is then redirected to the compromised machine's mapped services. This allows us to access these services, despite their filtration at the firewall.
<p align="center">
  <img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/pivot-2.png">
</p>

#### Remote Port Forwarding with SSH
```bash
ssh -R attackerport:remote-ip:remote-port user@attacker-ip

# In case of above theoretical condition
ssh -R 9090:10.20.0.5:8080 user@10.10.0.2

ssh -R attackerport:localhost:remoteport user@attacker-ip

# Forwarding multipe ports
ssh -R attackerport:remote-ip:remote-port attackerport2:remote-ip:remote-port2 user@attacker-ip
```

#### Remote Port Forwarding with Metasploit
```bash
# After you get the meterpreter session
portfwd add -R -L attacker-ip -l attacker-port -p <remote port on targets machine> -r <target ip>

# List the forwarded ports
portfwd list

# Delete all
portfwd flush

# In case of above theoretical condition
portfwd add -R -L 10.10.0.2 -l 9090 -p 8080 -r 10.20.0.5
```
- Where,
  - "-R" this instructs meterpreter that the rule is a reverse one.
  - "-L" tells meterpreter the IP address on which the multi handler is running; this is out metasploit system.
  - "-l" is the local port that an attacker will communicate on its machine
  - "-p" is the remote port that an attacker will communicate through its local port.
  - "-r" is the ip address of a machine whose remote port is required to be accessed.

#### Remote Port Forwarding with Chisel
```bash
# In attacker machine
chisel server -p <listen-port> --reverse

# In  remote machine
chisel client <listen-ip>:<listen-port> R:<attacker-port>:<target-ip>:<target-port>

# In case of above theoretical condition

# In attacker machine
chisel server -p 9999

# In remote machine
chisel client 10.10.0.2:9999 R:9090:10.20.0.5:8080

# After that we can connect to the port 8080 from local machine through port 9090 i.e. localhost:9090

# Forwarding multiple ports
chisel client 10.10.0.2:9999 9090:10.20.0.5:8080 9091:10.20.0.5:8081
```
 
#### Remote Port Forwarding with Plink
```bash
# In an attacker machine, open the port
nc -lvnp 9090

# In a compromised machine
plink.exe -ssh -R attacker-port:victim-ip:victim-port user@attacker-ip

# In case of above theoretical condition
plink.exe -ssh -R 9090:10.20.0.5:8080 user@10.10.0.2
```

### Dynamic Port Forwarding 
- The concept of dynamic port forwarding is introduced to address situations where access or connections to multiple services are needed. It establishes a SOCKS proxy server, enabling communication across a varities of ports. Once a client connects to this specific port, the connection is directed to the remote (SSH server) machine. Subsequently, the remote machine redirects it to a dynamic port on the target destination machine. All connections utilizing the SOCKS proxy are routed to the SSH server, which then manages the transmission of all traffic to their respective ultimate destinations.
<p align="center">
  <p align="center">
  <img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/pivot-3.png">
</p>
</p>

#### Dynamic Port Forwarding with SSH
```bash
ssh -D 127.0.0.1:9999 user@remote-ip

# Here we need to proxy the requests via SOCKS proxy from the browser on host 127.0.0.1 and port 9999.
```
#### Dynamic Port Forwarding with chisel
```bash
# In compromised machine
chisel server -p 7777 --socks5

# In an attacker machine
chisel client 10.10.0.3:7777 9999:socks

# Modify the /etc/proxychains.conf in local machine
socks5 127.0.0.1 9999
```
#### Dynamic Port Forwarding with Plink
```bash
plink.exe -D 9090 user@remote-ip
```

## References
- https://pentest.blog/explore-hidden-networks-with-double-pivoting/
- https://erev0s.com/blog/ssh-local-remote-and-dynamic-port-forwarding-explain-it-i-am-five/
- https://blog.certcube.com/pivoting-port-forwarding/
- https://medium.com/@ja3c/hackthebox-skill-assignment-pivoting-tunneling-and-port-forwarding-e6ceb091f241
- https://www.hdysec.com/port-forward-tunnels/
- https://neutronsec.com/ptpf/meterpreter_tunneling_and_port_forwarding/
- https://goteleport.com/blog/ssh-tunneling-explained/
- https://viperone.gitbook.io/pentest-everything/everything/pivoting-and-portforwarding
- https://medium.com/axon-technologies/how-to-implement-pivoting-and-relaying-techniques-using-meterpreter-b6f5ec666795
- https://www.golinuxcloud.com/setup-ssh-port-forwarding/
- https://medium.com/yavar/reverse-ssh-tunnelling-why-and-how-to-make-it-d40cd98d143b
- https://medium.com/r3d-buck3t/remote-local-port-tunneling-2b6a2fc1cab4
- https://medium.com/sysco-labs/ssh-port-forwarding-a634f2dd0a4f
- https://z-r0crypt.github.io/blog/2020/02/12/port-forward-tunnelling-quick-reference-guide/
- https://book.hacktricks.xyz/generic-methodologies-and-resources/tunneling-and-port-forwarding
- https://infosecwriteups.com/gain-access-to-an-internal-machine-using-port-forwarding-penetration-testing-518c0b6a4a0e
- https://guide.offsecnewbie.com/port-forwarding-ssh-tunneling
- https://www.thehacker.recipes/infra/pivoting/port-forwarding
- https://exploit-notes.hdks.org/exploit/network/port-forwarding/port-forwarding-with-chisel/
- https://techexpert.tips/windows/windows-port-forwarding-using-plink/
