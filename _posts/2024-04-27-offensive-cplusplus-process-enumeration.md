---
title: Offensive C++ - Process Enumeration
author: nirajkharel
date: 2024-04-27 14:10:00 +0800
categories: [Red Teaming, Malware Development]
tags: [Red Teaming, Malware Development]
render_with_liquid: false
---

```c++
#include <Windows.h>
#include <iostream>
#include <string>
#include <tlhelp32.h>

// Functions used
    // CreateToolhelp32Snapshot
    // PROCESSENTRY32
        // Process32First
        // Process32Next

using namespace std;

int main() {
    
    cout << endl << "Running Processes" << endl;

    // Create Snapshot of currently running processes (SYNTAX)
    HANDLE CreateToolhelp32Snapshot(
        DWORD dwFlags,
        DWORD th32ProcessID
    );

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