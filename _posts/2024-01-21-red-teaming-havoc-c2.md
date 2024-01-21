---
title: Red Teaming - Havoc C2 Introduction
author: nirajkharel
date: 2024-01-20 20:55:00 +0800
categories: [Red Teaming, C2]
tags: [Red Teaming, C2]
---

# Introduction
C2 frameworks, also known as command and control, enables red teamers to control and communicate with compromised systems. Havoc is a modern and malleable post-exploitation command and control framework, created by [@C5pider.](https://twitter.com/C5pider)

# Installation

## Clone the Havoc Repo
Havoc source code is avalable on the [github](https://github.com/HavocFramework/Havoc.git). **Note:** I will be installing Havoc Framework on Kali Linux for the demonstration purpose.

```bash
git clone https://github.com/HavocFramework/Havoc.git
```

## Install dependencies
**Install Python 3.10:**   
It is necessary to install python 3.10 inorder to install the Havoc. You must setup the bookwork repo for Python 3.10.
```bash
echo 'deb http://ftp.de.debian.org/debian bookworm main' >> /etc/apt/sources.list
sudo apt update
sudo apt install python3-dev python3.10-dev libpython3.10 libpython3.10-dev python3.10
```

Also you need to install below additional libraries. 
```bash
sudo apt install -y git build-essential apt-utils cmake libfontconfig1 libglu1-mesa-dev libgtest-dev libspdlog-dev libboost-all-dev libncurses5-dev libgdbm-dev libssl-dev libreadline-dev libffi-dev libsqlite3-dev libbz2-dev mesa-common-dev qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools libqt5websockets5 libqt5websockets5-dev qtdeclarative5-dev golang-go qtbase5-dev libqt5websockets5-dev python3-dev libboost-all-dev mingw-w64 nasm
```

**Install Mingw32**   
Navigate inside the `teamserver` directory and run the script.
```bash
cd Havoc/teamserver
./Install.sh
```
**Install Go dependencies.**  
Navigate inside the `teamserver` directory
```bash
cd Havoc/teamserver
go mod download golang.org/x/sys
go mod download github.com/ugorji/go 
cd .. 
```

## Build
Make sure you are on the base cloned directory 
```bash
# Install server
make ts-build

# Install client
make client-build
```

## Run
**Running the teamserver.**

All files created during interaction with the Teamserver are stored within the `/Havoc/data/*` folder.
```bash
./havoc server --default
              _______           _______  _______ 
    │\     /│(  ___  )│\     /│(  ___  )(  ____ \
    │ )   ( ││ (   ) ││ )   ( ││ (   ) ││ (    \/
    │ (___) ││ (___) ││ │   │ ││ │   │ ││ │      
    │  ___  ││  ___  │( (   ) )│ │   │ ││ │      
    │ (   ) ││ (   ) │ \ \_/ / │ │   │ ││ │      
    │ )   ( ││ )   ( │  \   /  │ (___) ││ (____/\
    │/     \││/     \│   \_/   (_______)(_______/

         pwn and elevate until it's done

[INFO] Havoc Framework [Version: 0.7] [CodeName: Bites The Dust]
[INFO] Use default profile
[INFO] Build: 
 - Compiler x64 : /usr/bin/x86_64-w64-mingw32-gcc
 - Compiler x86 : /usr/bin/i686-w64-mingw32-gcc
 - Nasm         : /usr/bin/nasm
[INFO] Time: 21/01/2024 06:59:07
[INFO] Teamserver logs saved under: data/loot/2024.01.21._06:59:07
[INFO] Starting Teamserver on wss://0.0.0.0:40056
[INFO] Opens existing database: data/teamserver.db
[INFO] Starting 1 listeners from last session
[INFO] Started "http" listener: https://172.16.18.128:443
[INFO] Restored 1 agents from last session
```

**Running the client**
```bash
./havoc client 
```
A TeamServer prompt will be opened after running the client as below:
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/favicons/havoc-1.png">

- The Name field can be any profile name.
- In the fields, Host and Port should contain the teamserver host address/domain and port.
- The fields, User and Password should contain the username and password specified in your Yaotl profile. You can change the credentials by navigating inside the **profile** directory.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/favicons/havoc-2.png">

After successfull connection, we can have the clean and awesome Havoc C2 framework infront of us.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/favicons/havoc-3.png">