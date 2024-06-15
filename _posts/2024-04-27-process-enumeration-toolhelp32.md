---
title: Offensive C++ - Process Enumeration (ToolHelp32 Function)
author: nirajkharel
date: 2024-06-05 14:10:00 +0800
categories: [Red Teaming, Offensive Programming]
tags: [Red Teaming, Offensive Programming]
render_with_liquid: false
---

This blog assumes that the reader has a general knowledge of C++ and system internals. For an initial overview, you can refer to [this blog](https://nirajkharel.com.np/posts/offensive-cplusplus-basics/). However, it is still under development.

# Process Enumeration
A process refers to an instance of an executable program (.exe file) running on a computer. It involves initializing the program, creating the user interface, and loading necessary drivers and DLLs. Each execution of a program creates a separate process. For instance, opening two browser windows results in two distinct processes, even though they are running the same program. Process enumerating is a technique to enumerate running instances on Windows systems. This can be achieved by using the **ToolHelp32** API, which contains the following functions:

## [CreateToolhelp32Snapshot](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-createtoolhelp32snapshot)
This function contains two arguments: **dwFlags** and **th32ProcessID**. There are different values for the dwFlags argument depending on the type of information you would like to save in the snapshot. Since in this blog we are only focused on processes, to include all processes in the system in the snapshot, we will use **TH32CS_SNAPPROCESS**. The next parameter, **th32ProcessID**, is the process identifier of the process that needs to be included in the snapshot. We can use the value 0 to indicate the current process.

**SYNTAX**
```c++
HANDLE CreateToolhelp32Snapshot(
  [in] DWORD dwFlags,
  [in] DWORD th32ProcessID
);
```

## [Process32First](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-process32first)
This can be used to extract information about the first process recorded in the snapshot. It has two arguments: **hSnapshot**, which is the handle returned by **CreateToolhelp32Snapshot**, and **lppe**, which is the pointer to a **PROCESSENTRY32** structure. This structure contains information about a single process within the snapshot in the system's address space, including the process identifier, parent process identifier, executable file, threads, and much more.

**SYNTAX**
```c++
BOOL Process32First(
  [in]      HANDLE           hSnapshot,
  [in, out] LPPROCESSENTRY32 lppe
);
```

## [Process32Next](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-process32next)
This can be used to extract information about the next process recorded on the snapshot. The arguments are same as above.

**SYNTAX**
```c++
BOOL Process32Next(
  [in]  HANDLE           hSnapshot,
  [out] LPPROCESSENTRY32 lppe
);
```

We will break down codes into different chunk to understand it on better way.
The first approach is to define the necessary headers in the code. The header **tlhelp32.h** is used to initialize the needed functions.
```c++
#include <Windows.h>
#include <iostream>
#include <string>
#include <tlhelp32.h>

using namespace std;
```

Define the function **CreateToolhelp32Snapshot** and pass the flags **TH32CS_SNAPPROCESS** and **0** to take the snapshot of the processes running and indicate the current process. The return type for the function if failed is **INVALID_HANDLE_VALUE**.
```c++
int main() {

    // TH32CS_SNAPPROCESS for the dwFlags and 0 for th32ProcessID.
    HANDLE hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

    // Handle Exceptions
    if (hSnapShot == INVALID_HANDLE_VALUE) {
        cout << "Process Snapshot creation failed" << endl;
    }
```
Now that we have process information stored in the snapshot, we can retrieve information about the first process encountered in a system snapshot using **Process32First**, and the next process using **Process32Next**. We also need to initialize the structure **PROCESSENTRY32** which contains information about a process, such as its ID, parent process ID, number of threads and the executable file name. **dwSize** is set to the size of the **PROCESSENTRY32** structure. This is necessary before using the structure in functions like **Process32First** and **Process32Next**.

I also received a suggestion that the process entry structure variable size should be set to **0** each time before using **Process32Next**, because the last ExeFilePath is an array and some bytes from the previous call to **Process32Next** may remain. This is done to clear the memory before moving to the next process, ensuring that the new entry does not overwrite the previous bytes.

```c++
    // Define processentry and the size
    PROCESSENTRY32 processEntry;
    processEntry.dwSize = sizeof(PROCESSENTRY32);

    // Checks if the first process in the snapshot can be retrieved.
    if (Process32First(hSnapShot, &processEntry) != FALSE)

    {
        // Iterates throgh the remaining processes in the snapshot. This loop continues until Process32Next fails, meaning there are no more processes in the snapshot.
        while (Process32Next(hSnapShot, &processEntry) != FALSE)
        {
            wcout << L"PID: " << processEntry.th32ProcessID // Prints the process ID
                << L"\t PPID: " << processEntry.th32ParentProcessID // Prints the parent process ID
                << L"\t Threads: " << processEntry.cntThreads // Prints the Thread running on the process
                << L"\t Name: " << processEntry.szExeFile << endl; // Prints an executable running the process
        
        // Setting the buffer of processEntry to 0
        memset(&processEntry, 0, sizeof(processEntry));
        
        // Initializing the size of processEntry again.
        processEntry.dwSize = sizeof(PROCESSENTRY32);
        }
    }
    
    // Close the handle, as always :)
    CloseHandle(hSnapShot);
    return 0;
}
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-1.gif">


**References**
- [https://www.youtube.com/watch?v=IZULG6I4z5U&t=1156s](https://www.youtube.com/watch?v=IZULG6I4z5U&t=1156s)
- [https://www.youtube.com/watch?v=nYrtkDU7uUI](https://www.youtube.com/watch?v=nYrtkDU7uUI)
- [https://www.youtube.com/watch?v=aNEqC-U5tHM](https://www.youtube.com/watch?v=aNEqC-U5tHM)
- [https://www.codeproject.com/Articles/2851/Enumerating-processes-A-practical-approach](https://www.codeproject.com/Articles/2851/Enumerating-processes-A-practical-approach)
- [https://www.irjet.net/archives/V7/i6/IRJET-V7I601.pdf](https://www.irjet.net/archives/V7/i6/IRJET-V7I601.pdf)