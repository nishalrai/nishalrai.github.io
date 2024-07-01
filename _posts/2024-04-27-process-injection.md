---
title: Offensive C++ - Process Injection
author: nirajkharel
date: 2024-06-27 14:10:00 +0800
categories: [Red Teaming, Offensive Programming]
tags: [Red Teaming, Offensive Programming]
render_with_liquid: false
---


## Process Injection
It is a technique to inject malicious code (can be on any form, ex shellcode, DLLs) into the legitimate process. It executes the code in the address space of the running process 

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-injection-1.png">

Image is referenced from:<font size="2"> <u> http://struppigel.blogspot.com/2017/07/process-injection-info-graphic.html<u>

- **Shellcode Injection**
- **DLL Injection**
- **Process Hollowing**
- **Thread Execution Hijacking**
- **APC (Asynchronous Procedure Call) Injection**
- **PE Injection (Portable Executable Injection)**
- **Code Cavities**
- **Inline Hooking and Patching**
- **SetWindowsHookEx Injection**
- **VMI (Virtual Machine Introspection) Injection**