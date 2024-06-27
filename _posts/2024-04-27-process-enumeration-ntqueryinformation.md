---
title: Offensive C++ - Process Enumeration (The Native NtQuerySytemInformation)
author: nirajkharel
date: 2024-06-25 14:10:00 +0800
categories: [Red Teaming, Offensive Programming]
tags: [Red Teaming, Offensive Programming]
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

Let's break down the flags needed for `NtQuerySystemInformation`:
- `SYSTEM_INFORMATION_CLASS SystemInformationClass`: It specifies the type of information needed from the system, such as `SystemBasicInformation`, `SystemPolicyInformation`, or `SystemProcessInformation`. Here, we will use `SystemProcessInformation`, which returns an array of process information, one for each running process.
- `PVOID SystemInformation`: This is a void pointer. It points to the buffer that receives the requested information from the `SystemInformationClass` flag.
- `ULONG SystemInformationLength`: An unsigned long integer datatype that holds the size of the buffer pointed to by `SystemInformation`.
- `PULONG ReturnLength`: This is an optional parameter that contains a pointer to the location where the acquired information length is written. It can be declared as `nullptr`.


**Important:** Before compiling the Native API functions into our code, we need to integrate the Native API definitions, which are not built-in within C++ or Visual Studio. We can get them through the GitHub repository maintained by the developers of **[Process Hacker 2](https://github.com/winsiderss/systeminformer)**. You need to download and place all the header files within your project directory.

Or, if you want to use Native APIs for multiple projects, you can do it via **[Vcpkg](https://github.com/microsoft/vcpkg/releases)**, which is an open-source C/C++ dependency manager.

```powershell
Vcpkg.exe integrate install
Vcpkg.exe install phnt:x64-windows 

// Or depending upon your architecture
Vcpkg.exe install phnt:arm64-windows
```

Since we have our requirements installed, let's break down the code that we can use to enumerate process information using Native APIs.

The first approach is to define the necessary headers in the code. The headers **phnt_windows.h** and **phnt.h** are used to initialize the needed NT API functions.

```c++
#include <iostream>
#include <Windows.h>
#include <phnt_windows.h>
#include <phnt.h>
#include <stdio.h>
```

Instruct the linker to link with 'ntdll' library, otherwise NtQuerySystemInformation will not be compiled.
```c++
#pragma comment(lib, "ntdll")
```

Within the main function, first set up the **NTSTATUS** code, a data type defined in the Windows SDK to indicate the success or failure of function calls. Declare a pointer to **SYSTEM_PROCESS_INFORMATION** named **sysProc** and initialize it to `nullptr` since the size of the system information is not yet known. Define a `ULONG` variable for **SystemInformationLength**, representing the size of the buffer used to store system process information. Allocate memory for this buffer using **VirtualAlloc** with read/write permissions. Finally, cast the allocated memory to a **SYSTEM_PROCESS_INFORMATION** pointer and assign it to the **sysProc** pointer.

```c++
int main()
{
    // Setup the NTSTATUS code 
    NTSTATUS nisStat;

    // Declaring the pointer sysProc of type SYSTEM_PROCESS_INFORMATION 
    // and initializes it to nullptr as until now we do not know the size of the system information.
    SYSTEM_PROCESS_INFORMATION* sysProc = nullptr;

    // Defining the ULONG variable for SystemInformationLength. Size of the buffer used to store system process information. This size is an example and can need adjustment based on the buffer size.
    unsigned long sysInfoLen = 1024 * 1024;

    // Creates and allocate the memory defined above with read/write permission.
    void* nisBuf = VirtualAlloc(nullptr, sysInfoLen, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

    // Cast the allocated memory to SYSTEM_PROCESS_INFORMATION pointer and assigns it to 'sysProc'
    sysProc = reinterpret_cast<SYSTEM_PROCESS_INFORMATION*>(nisBuf);
```

After defining all the required parameters and allocating the virtual memory, call the **NtQuerySystemInformation** function with the above-defined flags.

```c++
    // Call NtQuerySystemInformation function with the type of information needed
    // SystemProcessInformation - Enumerate process
    // sysProc - buffer to receive the information
    // sysInfoLen - size of the buffer.
    // nullptr - Optional, nullptr means it do not need to return the length of the data
    nisStat = NtQuerySystemInformation(SystemProcessInformation, sysProc, sysInfoLen, nullptr);

    // Just some fancy printing format.
    wprintf(L"%-10ls %-10ls %-14ls %-10ls %-30ls\n", L"ProcessID", L"ParentPID", L"Thread Count", L"Session", L"Process Name");
wprintf(L"%-10ls %-10ls %-14ls %-10ls %-30ls\n", L"----------", L"----------", L"----------", L"----------", L"------------------------------");

```

Iterate over each **SYSTEM_PROCESS_INFORMATION** entry in the buffer and print its information. During iteration, when the entry in the array is exhausted, break the loop since there are no processes left to enumerate. As long as the loop continues, cast the original type of **UniqueProcessId** into **ULONG_PTR** so that the process ID can be printed on the console. The next two entries are for threads and the name of the executable.

Since we have converted **sysProc** into **ULONG_PTR**, in order to move to the next entry in **sysProc**, the `reinterpret_cast<BYTE*>(sysProc)` is used. This allows the code to add **NextEntryOffset** as a byte offset. The final `reinterpret_cast<SYSTEM_PROCESS_INFORMATION*>(...)` converts the byte pointer back to a **SYSTEM_PROCESS_INFORMATION*** type, so **sysProc** can be used as before, but now pointing to the next process entry.

```c++
    do {
         wprintf(L"%-10lu %-10lu %-14lu %-10lu %-30ls\n",
         reinterpret_cast<ULONG_PTR>(sysProc->UniqueProcessId),
         sysProc->InheritedFromUniqueProcessId,
         sysProc->NumberOfThreads,
         sysProc->SessionId,
         sysProc->ImageName.Buffer ? sysProc->ImageName.Buffer : L"(null)"
 );

        // When NextEntryOffset is 0, then there is no process left to enumerate.
        if (sysProc->NextEntryOffset == 0) {
            break;
        }
        
        // Moves the sysProc pointer from the current SYSTEM_PROCESS_INFORMATION entry to the next one in the buffer.
        sysProc = reinterpret_cast<SYSTEM_PROCESS_INFORMATION*>(reinterpret_cast<BYTE*>(sysProc) + sysProc->NextEntryOffset);
    } while (true);

    // Clean the memory once the program is done.
    VirtualFree(nisBuf, 0, MEM_RELEASE);
    return true;
}
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-4.gif">