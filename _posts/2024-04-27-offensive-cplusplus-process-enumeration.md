---
title: Offensive C++ - Process Enumeration
author: nirajkharel
date: 2024-04-27 14:10:00 +0800
categories: [Red Teaming, Malware Development]
tags: [Red Teaming, Malware Development]
render_with_liquid: false
---

# Process Enumeration
Process Enumerating is a technique to enumerate running process on the Window Systems. This can be achive by using **ToolHelp32** API which contains the below functions:

## CreateToolhelp32Snapshot**
- This function contains two arguments which are **dwFlags** and **th32ProcessID**. There are different values for **dwFlags** argument depending upon type of information you would like to save it on the snapshot. Since on this blog, we are only focused on process, inorder to include all processes in the system in the snapshot, we will use **TH32CS_SNAPPROCESS**. Next parameter **th32ProcessID** is the process identifier of the process which needs to be included in the snapshot. We can use the value as **0** to indicate the current process for that.

**SYNTAX**
```c++
HANDLE CreateToolhelp32Snapshot(
  [in] DWORD dwFlags,
  [in] DWORD th32ProcessID
);
```

## Process32First
This can be used to extract the information about the first process recorded on the snapshot. It has two arguments, one is **hSnapshot** which is the handle returned by **CreateToolhelp32Snapshot** and **lppe** which is the pointer to **PROCESSENTRY32** structure. This structure contains information about a single process within the snapshot in the system's address space. Information such as process identifier, parent process's identifier, executable file, threads and much more.

**SYNTAX**
```c++
BOOL Process32First(
  [in]      HANDLE           hSnapshot,
  [in, out] LPPROCESSENTRY32 lppe
);
```

## Process32Next
This can be used to extract information about the next process recorded on the snapshot. The arguments are same as above.
**SYNTAX**
```c++
BOOL Process32Next(
  [in]  HANDLE           hSnapshot,
  [out] LPPROCESSENTRY32 lppe
);
```


```c++
#include <Windows.h>
#include <iostream>
#include <string>
#include <tlhelp32.h>

using namespace std;

int main() {

    // TH32CS_SNAPPROCESS for the dwFlags and 0 for th32ProcessID.
    HANDLE hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

    // Handle Exceptions
    if (hSnapShot == INVALID_HANDLE_VALUE) {
        cout << "Process Snapshot creation failed" << endl;
    }

    // Now we have process information stored on the snapshot, we can retrieve information about the first process encountered in a system snapshot using Process32First
    // and Next process with Process32Next

    // Define processentry and the size
    PROCESSENTRY32 processEntry;
    processEntry.dwSize = sizeof(PROCESSENTRY32);

    // Checks if the first process in the snapshot can be retrieved.
    if (Process32First(hSnapShot, &processEntry) != FALSE)

    {
        // Iterates throgh the remaining processes in the snapshot
        while (Process32Next(hSnapShot, &processEntry) != FALSE)
        {
            wcout << L"PID: " << processEntry.th32ProcessID // Prints the process ID
                << L"\t PPID: " << processEntry.th32ParentProcessID // Prints the parent process ID
                << L"\t Threads: " << processEntry.cntThreads // Prints the Thread running on the process
                << L"\t Name: " << processEntry.szExeFile << endl; // Prints an executable running the process
        }
    }
    
    // Close the handle, as always :)
    CloseHandle(hSnapShot);
    return 0;
}
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-1.gif">