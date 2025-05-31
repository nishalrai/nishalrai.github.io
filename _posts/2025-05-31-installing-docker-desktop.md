---
title: Installing Docker Desktop on a Drive Other Than C drive
description: >-
  How to install the docker desktop in another drive instead of the default C drive.
author: nishalrai
date: 2025-05-31 20:55:00 +0800
categories: [Cloud Native, Docker]
tags: [getting started]
---


By default, Docker Desktop installs itself on the C: drive, but if you're tight on space or prefer keeping your dev tools on a different partition, you can install Docker Desktop to another drive (like D:) — even when using WSL2 under the hood. This will be a step-by-step guide to help you get it done cleanly and without surprises during the installation of the docker desktop.


# Prerequisites
- Windows OS with WSL2 enabled
  <br>(Make sure you've set WSL2 as your default with wsl --set-default-version 2)
- Docker Desktop installer .exe file
  <br>(Download it from [Docker's official site](https://www.docker.com/products/docker-desktop/))
- Administrative privileges

<br>

## Step 1: Prepare Custom Install Directory
We first need to create the two directory where Docker Desktop and its WSL data should live, and the other for Docker Images to be stored. Here, we will use `D:\WSL\docker` for the Docker Desktop data, and `D:\WSL\docker-image` for the docker images.


### Open the Powershell in the administrator mode
`New-Item -ItemType Directory -Path D:\WSL\docker`<br>
`New-Item -ItemType Directory -Path D:\WSL\docker-image`


**To check if the directory was created or not**<br>
`Test-Path D:\WSL\docker`<br>
`Test-Path D:\WSL\docker-image`

And if returns true, then move to the second step, otherwise, re-run the command again.
> The Powershell should be running in administrator mode for later use case.

<br>

# Step 2: Install Docker Destop to the Custom Directory
Run this powershell command as **Administrator**

```
Start-Process "Docker Desktop Installer.exe" -Wait -ArgumentList "install", "-accept-license", "--installation-dir=D:\WSL\docker", "--wsl-default-data-root=D:\WSL\docker", "--windows-containers-default-data-root=D:\WSL\docker-image"
```

**What These flags Do:**
 - `--installation-dir`: Where Docker Desktop's binaries and related files go.
 - `--wsl-default-data-root`: WSL2's storage location for Docker data.
 - `-windows-containers-default-data-root`: Where Docker stores data for Windows containers.

<br>

# Got an Error? Here's How to Fix It
After the successful installtion of the docker desktop, you are prompted with the following error when the Docker engine failed to start.


> deploying WSL2 distributions
ensuring main distro is deployed: deploying "docker-desktop": preparing directory "D:\\WSL\\docker\\main" for WSL distro "docker-desktop": creating distro destination dir "D:\\WSL\\docker\\main": mkdir D:\WSL\docker\main: Access is denied.
checking if isocache exists: CreateFile \\wsl$\docker-desktop-data\isocache\: The network name cannot be found.

This usually means Docker couldn’t create or write to the required subfolder. It’s a permission issue.

<br>

## Set Directory Permissions Manually
Create the missing main directory if it doesn't exist:<br>
`New-Item -ItemType Directory -Path D:\WSL\docker\main -Force`

Grant full access to administrators:<br>
`icacls D:\WSL\docker\main /grant Administrators:F /t`

- `icacls`: Windows command to change file and folder permissions.
- `/grant Administrators:F`: Gives full control (F) to the Administrators group.
- `/t`: Applies the permission change to all files and subfolders recursively.

> (Optional but recommended) Grant access to the current user:<br>
`icacls D:\WSL\docker\main /grant "$($env:USERNAME):F" /t`<br>
This ensures you, the logged-in user, can access and manage files inside the Docker folder.

<br>

## Take Ownership if You Still Have Access Issues
If permissions are still acting up, try taking ownership of the folder:
`takeown /f D:\WSL\docker /r /d y`<br>

- `/f`: Target folder
- `/r`: Recurse into subdirectories
- `/d y`: Assume "Yes" for all confirmation prompts


## Verify Permissions
You can check permissions on a file or folder like this:<br>
`icacls D:\WSL\docker\app.json`

A successful output looks like:<br>
`Successfully processed 1 files; Failed processing 0 files`

This means the permissions were updated successfully.

<br>

To start the Docker Desktop form Powershell:<br>
`& "D:\WSL\docker\Docker Desktop.exe"`

<br>

![Docker Desktop](/assets/img/images/cloud-native/dockerDesktop01.png)

<br>

**Powershell CLI**<br>
![Docker Desktop](/assets/img/images/cloud-native/dockerDesktop02.png)


# Final Notes
- After setting permissions, re-run the installer if it previously failed.
- Installing Docker Desktop on another drive is perfectly safe and helps keep your C: drive light.
- Make sure you always run PowerShell as Administrator for these steps.

<br>

# Reference
- https://stackoverflow.com/questions/75727062/how-to-install-docker-desktop-on-a-different-drive-location-on-windows
- https://docs.docker.com/desktop/setup/install/windows-install/#install-from-the-command-line