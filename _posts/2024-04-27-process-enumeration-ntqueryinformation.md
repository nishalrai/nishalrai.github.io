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


## [NtQuerySystemInformation](https://learn.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntquerysysteminformation)
NtQuerySystemInformation is a function derived from the Native APIs in the Windows system that retrieves specified system information. This function can retrieve information that the Windows APIs discussed earlier cannot. It is usually used by applications like Task Manager, Process Hacker, and other applications that need kernel mode access to the system. It is not only used to enumerate processes but can also gather basic system information, policy details, performance metrics, code integrity, exceptions, kernel and shadow information, and much more. However, for the sake of this blog, our focus will be on enumerating process-related information using this function.

**SYNTAX**
```c++
__kernel_entry NTSTATUS NtQuerySystemInformation(
  [in]            SYSTEM_INFORMATION_CLASS SystemInformationClass,
  [in, out]       PVOID                    SystemInformation,
  [in]            ULONG                    SystemInformationLength,
  [out, optional] PULONG                   ReturnLength
);
```

Explanation: TODO

```c++

#include <iostream>
#include <Windows.h>
#include <phnt_windows.h>
#include <phnt.h>
#include <stdio.h>


// Instruct the linker to link with 'ntdll' library, otherwise NtQuerySystemInformation not be compiled.
#pragma comment(lib, "ntdll")

int main()
{
    // Setup the NTSTATUS code 
    NTSTATUS nisStat;

    // Declaring the pointer sysProc of type SYSTEM_PROCESS_INFORMATION 
    // and initializes it to nullptr as until now we do not know the size of the system information.
    SYSTEM_PROCESS_INFORMATION* sysProc = nullptr;

    // Defining the ULONG variable for SystemInformationLength. Size of the buffer used to store system process information.
    unsigned long sysInfoLen = 1024 * 1024;

    // Creates and allocate the memory defined above with read/write permission.
    void* nisBuf = VirtualAlloc(nullptr, sysInfoLen, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

    // Cast the allocated memory to SYSTEM_PROCESS_INFORMATION pointer and assigns it to 'sysProc'
    sysProc = reinterpret_cast<SYSTEM_PROCESS_INFORMATION*>(nisBuf);

    // Call NtQuerySystemInformation function with the type of information needed
    // SystemProcessInformation - Enumerate process
    // sysProc - buffer to receive the information
    // sysInfoLen - size of the buffer.
    // nullptr - Optional, nullptr means it do not need to return the length of the data
    nisStat = NtQuerySystemInformation(SystemProcessInformation, sysProc, sysInfoLen, nullptr);

    // Just some fancy printing format.
    wprintf(L"%-10ls %-14ls %-30ls\n", L"ProcessID", L"Thread Count", L"Process Name");
    wprintf(L"%-10ls %-14ls %-30ls\n", L"----------", L"----------",L"------------------------------");

    // Iterate over each SYSTEM_PROCESS_INFORMATION entry in the buffer and prints its information.
    do {
        wprintf(L"%-10lu %-14lu %-30ls\n",
            
            reinterpret_cast<ULONG_PTR>(sysProc->UniqueProcessId),
            sysProc->NumberOfThreads,
            sysProc->ImageName.Buffer ? sysProc->ImageName.Buffer : L"(null)"
        );

        // When NextEntryOffset is 0, then there is no process left to enumerate.
        if (sysProc->NextEntryOffset == 0) {
            break;
        }
        
        // Moves the sysProc pointer from the current SYSTEM_PROCESS_INFORMATION entry to the next one in the buffer.
        // The reinterpret_cast<BYTE*>(sysProc) allows the code to add NextEntryOffset as a byte offset.
        // The final reinterpret_cast<SYSTEM_PROCESS_INFORMATION*>(...) converts the byte pointer back to a SYSTEM_PROCESS_INFORMATION* type, 
        // so sysProc can be used as before, but now pointing to the next process entry.
        sysProc = reinterpret_cast<SYSTEM_PROCESS_INFORMATION*>(reinterpret_cast<BYTE*>(sysProc) + sysProc->NextEntryOffset);
    } while (true);

    // Clean the memory once the program is done.
    VirtualFree(nisBuf, 0, MEM_RELEASE);
    return true;
}

```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-4.gif">