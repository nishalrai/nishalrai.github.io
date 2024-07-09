---
title: Offensive C++ - Process Injection (ShellCode) - QueueUserAPC
author: nirajkharel
date: 2024-06-29 15:10:00 +0800
categories: [Red Teaming, Offensive Programming]
tags: [Red Teaming, Offensive Programming]
render_with_liquid: false
---


## Process Injection (ShellCode) - QueueUserAPC

APC (Asynchronous Procedure Call) on Windows involves threads having `APC queues` for functions that execute only under specific thread conditions. When an application queues an APC before a thread starts, the thread initiates by invoking the APC function. By starting a process in a suspended state, queuing our shellcode as an APC, and then resuming the main thread, our shellcode executes. This method is stealthier than using `CreateRemoteThread` because processes commonly use the `QueueUserAPC` function and we can do that without triggering user popups, as the main thread remains inactive.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-injection-queueUserAPC.png">

As shown in the image above, we first need to create a process, example Notepad.exe, in a `suspended` state. A suspended state allows a process to be opened but interrupts its normal execution. Once the process is opened, we can obtain a handle to it and its threads. We use `VirtualAllocEx` to allocate memory for the code and `WriteProcess` to write our shellcode into that virtual address space. Instead of using `CreateRemoteThreadEx`, we implement `QueueUserAPC` to queue our malicious code as an APC on the main thread of the process. Then, we resume the thread using the `ResumeThread` function. When `ResumeThread` is called, the Windows system starts with the APC queue created, executing our malicious code without executing the main code on the thread.


## References
- [https://www.youtube.com/watch?v=aMkMkkClXVc&t=468s](https://www.youtube.com/watch?v=aMkMkkClXVc&t=468s)
- [http://rinseandrepeatanalysis.blogspot.com/2019/04/early-bird-injection-apc-abuse.html?m=1](http://rinseandrepeatanalysis.blogspot.com/2019/04/early-bird-injection-apc-abuse.html?m=1)
- [https://www.ired.team/offensive-security/code-injection-process-injection/apc-queue-code-injection](https://www.ired.team/offensive-security/code-injection-process-injection/apc-queue-code-injection)
- [https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc)