---
title: Offensive C++ - Process Injection - DLL Injection
author: nirajkharel
date: 2024-07-05 15:10:00 +0800
categories: [Red Teaming, Offensive Programming]
tags: [Red Teaming, Offensive Programming]
render_with_liquid: false
---


## Process Injection - DLL Injection
DLL Injection is a kind of process injection techniques but unlike loading shellcode into a running process, DLL Injection involves injecting and loading a malicious DLL into the running process.

**What is DLL?**    
Think of it as libraries needed to run a program. These libraries can be used by different programs at the same time. But in order to use such libraries/DLL/modules, a program needs to have them imported while compiling their code. When the DLL is loaded, it will execute the main function within the DLL, which is **DllMain**. Here in the image below, we can see the list of modules/DLLs that a program can have, for example, for `Calc.exe`.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-injection-dll-inject-2.png">

Or, you can go through the [blog](https://nirajkharel.com.np/posts/process-module-enumeration/) I wrote before to enumerate the loaded DLLs/modules in the running process.

As we discussed earlier, `DllMain` serves as the entry point function for each loaded DLL. Let's explore what `DllMain` is and how we can create our own malicious DLL:
- On Microsoft Visual Code Studio, create a new project and select **Dynamic-Link Library** as the project type.
- Open the project, and you will be presented with default code for the DLL.


```c++
// dllmain.cpp : Defines the entry point for the DLL application.
#include "pch.h"

BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

In the code above, we can see that it contains a `DllMain` function which has three different parameters:

- **HMODULE hModule**: Handle to the DLL module.
- **DWORD ul_reason_for_call**: Reason for calling the function. It contains different cases like `DLL_PROCESS_ATTACH`, `DLL_THREAD_ATTACH`, `DLL_THREAD_DETACH`, and `DLL_PROCESS_DETACH`. It performs actions based on the reason for calling. For example, the case `DLL_PROCESS_ATTACH` is executed when the DLL is loaded into the address space of the process.
- **LPVOID lpReserved**: Reserved for future use and is typically `NULL`.

If we want to execute malicious code or run any program within the process through our malicious DLL, we need to place that code inside the `DLL_PROCESS_ATTACH` case. For example, to simply open `Notepad.exe` through this DLL, we can use `CreateProcess` inside the `DllMain` function.

```c++
#include "pch.h"
#include <windows.h>
#pragma comment(lib, "user32.lib")

BOOL APIENTRY DllMain(HMODULE hModule, DWORD  nReason, LPVOID lpReserved) {
    switch (nReason) {
    case DLL_PROCESS_ATTACH:
        STARTUPINFO si;
        PROCESS_INFORMATION pi;
        ZeroMemory(&si, sizeof(si));
        si.cb = sizeof(si);
        ZeroMemory(&pi, sizeof(pi));

        // Create the calculator process
        if (CreateProcess(
            L"C:\\Windows\\System32\\Notepad.exe", // Application name
            NULL,    // Command line arguments
            NULL,    // Process handle not inheritable
            NULL,    // Thread handle not inheritable
            FALSE,   // Set handle inheritance to FALSE
            0,       // No creation flags
            NULL,    // Use parent's environment block
            NULL,    // Use parent's starting directory 
            &si,     // Pointer to STARTUPINFO structure
            &pi)     // Pointer to PROCESS_INFORMATION structure
            ) {
            // Close process and thread handles. 
            CloseHandle(pi.hProcess);
            CloseHandle(pi.hThread);
        }
        break;
    case DLL_PROCESS_DETACH:
        break;
    case DLL_THREAD_ATTACH:
        break;
    case DLL_THREAD_DETACH:
        break;
    }
    return TRUE;
}

```

I would also recommend you watch the video **[Everything You Ever Wanted to Know about DLLs](https://www.youtube.com/watch?v=JPQWQfDhICA&t=1749s)** on YouTube. It explains pretty much everything you need to know.

Compile the code and execute the DLL using **rundll32.exe**. You should see that the DLL file has been loaded successfully and that `Notepad.exe` has been opened, which means the `CreateProcess` function inside `DllMain` has been executed.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-injection-dll-inject-4.png">

Now that we have our malicious DLL ready, let’s go through the process of injecting it into a running process.

With our malicious DLL compiled, we first create a handle to the target process using **OpenProcess**. We then use that handle to allocate virtual address space in the memory of the running process with **VirtualAllocEx** and write the path of the DLL file into this memory using **WriteProcessMemory**.

Once that is done, we need to load the DLL into memory. We have written the DLL path into the memory using **WriteProcessMemory**, and now we use **CreateRemoteThread** to execute the **LoadLibrary** API. This will load the DLL specified by the path on the buffer address into the target process. **CreateRemoteThread** provides a handle to the newly created thread, which executes the DLL loading function.

By doing this, we successfully inject the DLL into the running process.


<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-injection-dll-inject.png">