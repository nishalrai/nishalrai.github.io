---
title: Automate the backup of F5 BIG-IP – UCS, SCF and SSL
description: >-
  Get started with Chirpy basics in this comprehensive overview.
  You will learn how to install, configure, and use your first Chirpy-based website, as well as deploy it to a web server.
author: nisharai
date: 2023-05-19 20:55:00 +0800
categories: [F5 Networks, Backup]
tags: [getting started]
---

One of the most tedious jobs for the F5 Administrator is the periodic backup of the F5 BIG-IP system. F5 BIG-IQ does solve the problem but, this guide is for someone who wants the workaround for the standalone subscription of F5 BIG-IP.

The idea behind the automation of F5 BIG-IP backup has been referenced from the work of:

[Reference of Code](https://community.f5.com/t5/codeshare/automated-backup-f5-configuration-to-remote-server/ta-p/286400)


UCS
– Stands for User Configuration Set.
– Saves all the configuration files in a tar file.
– Consists the following elements (Configuration files, BIG-IP License File, User Account Information, SSL certificates with private keys and DNS zone files.

To create UCS using CLI:  

On Advanced (bash shell):
`tmsh save /sys ucs [filename] The default file location is /var/local/ucs directory`

To restore the UCS using CLI:
`tmsh load /sys ucs [filename]`


SCF
- Stands for Single Configuration File
- Archives the configuration backup in the form of tmsh commands executed on the BIG-IP system.Can be used to replicate the configuration from the hardware system to the virtual system and vice-versa.
- Since SCF is platform independent so it can be used across multiple F5 BIG-IP system.  

<br>

**To create a SCF file**
On Advanced (bash shell):
`tmsh save /sys config file [filename] no-passphrase`

The default file location is `/var/local/scf` directory    


The overall idea behind the automation in a summary:

- Generate three different files as the backup on the F5 BIG-IP system – UCS, SCF and archive file of SSL.
- Transfer the file using the SCF protocol from F5 BIG-IP to the destinated server.
- Delete the recently created three different files on the F5 BIG-IP system after successfully transferring the files.
- The whole configuration has been described with the various configuration section:

**Connectivity**
To verify the connectivity between the F5 BIG-IP system and the destinated remote server.

- ping <destinated-ip>
- telnet <destinated-ip> <port>
- Finally, establish ssh connection:

`ssh user@remote-server`

<br>

In our case-scenario:

F5 BIG-IP system (Management-IP): 192.168.59.128
The source for the remote server would be:
ip route get <destinated-ip>

Remote-Server: 12.0.1.129
<br>

# Configure SSH-based authentication
This is performed to avoid the password prompt during the transfer of the files to the remote server using SCP protocol.

Generate an SSH key pair on the F5 BIG-IP system
`ssh-keygen -t rsa -b 4096`

The public key will be saved in /root/.ssh/id_rsa.pub

Copy the public key to the destination server using ssh-copy-id command.
`ssh-copy-id -i /root/.ssh/id_rsa.pub user@destination_server`

*This command will prompt the user for the password of the destination server. Enter the password to complete the copy.*

Test the connection


# Script
The script will perform the following tasks:

Takes UCS backup
Takes SCF backup
Archives SSL backup of the F5 BIG-IP system

```bash
OUT_DIR='/var/tmp'

# Generate timestamp in YYYYMMDD_HHMMSS format
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

# Saves the files ucs, scf and archive tar on the defined file format with timestamp
FILE_UCS="${HOSTNAME}_${TIMESTAMP}.ucs"
FILE_SCF="${HOSTNAME}_${TIMESTAMP}.scf"
FILE_CERT="${HOSTNAME}_${TIMESTAMP}.cert.tar"

# Changes to the defined directory
cd "${OUT_DIR}" || exit

# Perform the ucs, scf backup and archive the ssl directory
tmsh save /sys ucs "${OUT_DIR}/${FILE_UCS}"
tmsh save /sys config file "${OUT_DIR}/${FILE_SCF}" no-passphrase
tar -cf "${OUT_DIR}/${FILE_CERT}" /config/ssl

# Define the destination server and directory to save the files
DESTINATION_SERVER="user@destinated-ip-address"
DESTINATION_DIR="/directory"

# Transfer files using scp
scp "${OUT_DIR}/${FILE_UCS}" "${DESTINATION_SERVER}:${DESTINATION_DIR}"
scp "${OUT_DIR}/${FILE_SCF}" "${DESTINATION_SERVER}:${DESTINATION_DIR}"
scp "${OUT_DIR}/${FILE_CERT}" "${DESTINATION_SERVER}:${DESTINATION_DIR}"

# Removes the files after successfully transferring the file
rm -f "${OUT_DIR}/${FILE_UCS}"
rm -f "${OUT_DIR}/${FILE_SCF}"
rm -f "${OUT_DIR}/${FILE_CERT}"

RTN_CODE=$?
exit $RTN_CODE
```

Configuration
Log into f5 big-ip cli
On the /var/tmp directory
Create a script:
touch <filename>.sh
i.e. touch backup.sh

Change the file permission:
`chmod 755 ./backup.sh`

### Test
Once the script has been developed and the required permissions on the script has been configured it’s time to test.

On the directory where the script resides.
`./.sh`

i.e. ./backup.sh


Once the script has been completed, check the files on the destinated server.

### Schedule
After the successful completion of the scripts and the files has been transferred the file on the destinated remote server.
You need to schedule the script to execute on the required periodic manner using Crontab on the F5 BIG-IP.

Crontab is a time-based job scheduler in Unix-like operating systems. It allows users to schedule and automate the execution of commands or scripts at specified intervals, such as daily, weekly, monthly, or on specific days and times.

Using crontab and append the line using the command:
crontab -e

```crontab
0 23 * * 5 <command_to_execute>
This will configure the crontab to execute the script on every Friday at 11 P.M.

0 23 * * 5 /var/tmp/backup.sh
```

Here’s an explanation of the fields in the crontab entry:

- The first field, 0, represents the minute at which the command should run (in this case, at the 0th minute).
- The second field, 23, represents the hour at which the command should run (in this case, at 11 PM in 24-hour format).
- The third field, *, represents the day of the month. Using * means the command can run on any day of the month.
- The fourth field, *, represents the month. Using * means the command can run in any month.
- The fifth field, 5, represents the day of the week. Here, 5 corresponds to Friday (Sunday is 0, Monday is 1, and so on).
- The last field, command_to_execute, is the actual command or script you want to run at the specified time.
- 
The last field, command_to_execute, is the actual command or script you want to run at the specified time.
