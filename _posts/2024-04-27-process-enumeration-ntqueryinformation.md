---
title: Offensive C++ - Process Enumeration (The Native NtQuerySytemInformation)
author: nirajkharel
date: 2024-06-10 14:10:00 +0800
categories: [Red Teaming, Malware Development]
tags: [Red Teaming, Malware Development]
render_with_liquid: false
---


## [Native APIs](https://learn.microsoft.com/en-us/sysinternals/resources/inside-native-applications)
Most of the time, when we develop code to interact with the Windows API, we use the **Kernel32** library, which includes thousands of documented Windows APIs. However, the Windows system contains more than just the public Windows API. There is a lesser-known and undocumented set of APIs that the NT system uses internally, referred to as the Native API, which is not generally accessible or published in standard documentation.

To understand Native APIs properly, one first needs to understand the different types of modes in the Windows system: user mode and kernel mode. User application code runs in user mode, whereas OS code (such as system services and device drivers) runs in kernel mode. Kernel mode refers to a mode of execution in a processor that grants access to all system memory and all CPU instructions. Using Win32 APIs in malware is easily spotted by antivirus and EDR systems because these APIs are well-documented and can be tracked for unusual activity.

So, how can user mode access kernel mode? There is no direct way to access it. This is achieved through the **Win32 API**, which translates requests into the **NT API** and then passes them to the **Kernel**.

The Native API is exported by a library called **Ntdll.dll**. This library is the final stop that most of our Windows API functions end up in after being called from **Kernel32.dll** before reaching the threshold into the kernel space.

When you call a function like **ReadFile** or **OpenProcess**, these functions are exported by a library called **Kernel32.dll** and end up in **Ntdll.dll**. However, we can use **kernelbase.dll** to forward the calls down to the **Ntdll.dll** API, which acts as a proxy between user mode and kernel mode.

**Win32API (ReadFile) -> kernel32!ReadFile() -> kernelbase!ReadFile() [kernelbase.dll works as a proxy] -> ntdll!NtReadFile() -> Syscall -> Kernel mode tasks.**

I also suggest you to go through this blog to have a brief overview on the Native API **[Inside the Native API](http://mirrors.arcadecontrols.com/www.sysinternals.com/Information/NativeApi.html)**. Also [Crow](https://www.youtube.com/watch?v=P1PHRcmPM7c&t=2395s) has explained about this on interesting way.

## [NtQuerySystemInformation]()
