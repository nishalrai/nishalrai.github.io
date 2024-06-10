---
title: Offensive C++ - Process Enumeration (Windows Terminal Services API)
author: nirajkharel
date: 2024-06-09 14:10:00 +0800
categories: [Red Teaming, Malware Development]
tags: [Red Teaming, Malware Development]
render_with_liquid: false
---


## [Windows Terminal Services - WTS API](https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/nf-wtsapi32-wtsenumerateprocessesa)
Windows also contains **WTSEnumerateProcessesExA** function to gather inforamtion about currently active processes on the remote session via RDP or Virualization. It is not necessary for this to work only on a remote machine; it also has the capability to enumerate a remote machine. The target can depend on the value you pass to its handle.

This function contains five arguments: **hServer**, **\*pLevel**, **SessionId**, **\*ppProcessInfo**, **\*pCount**. 
- **hServer**: Opens a handle to open Remote Desktop session to the server. If needed to be done on the local machine, we can specify **WTS_CURRENT_SERVER_HANDLE** value on it.
- **\*pLevel**: It is a pointer which defines what kind of information is needed in return. Inorder to get maximum information, we can specify **WTS_PROCESS_INFO_EX** which is equivalent to **1** otherwise 0.
- **SessionId**: Specify which sessions to use inorder to enumerate the process. Process can be dependednt to different users logged in into the machine. To get the maximum result, we can specify **WTS_ANY_SESSION** to get list of all processes running by all users.
- **\*ppProcessInfo**: It is a pointer which contains the references to all the information gatherred via **WTS_PROCESS_INFO_EX** function.
- **\*pCount**: It is a pointer which contains the number of processes captured. This is used to iterate through the list of process.

**SYNTAX**
```c++
BOOL WTSEnumerateProcessesExA(
  [in]      HANDLE hServer,
  [in, out] DWORD  *pLevel,
  [in]      DWORD  SessionId,
  [out]     LPSTR  *ppProcessInfo,
  [out]     DWORD  *pCount
);
```

We will break down codes into different chunk to understand it on better way.
The first approach is to define the necessary headers in the code. The header **Wtsapi32.h** is used to initialize the needed functions.
```c++
#include <Windows.h>
#include <iostream>
#include <string>
#include <WtsApi32.h>

using namespace std;
```

Define the flags needed on **WTSEnumerateProcessesEx** as desribed above and create the function. . The return type for the function if failed is **FALSE**. Everything on the code below should be clear other then **(PWSTR*)&processInfo** where **PWSTR** is **Pointer to a Wide String** which are typically 16-bit characters in Windows and the **processInfo** pointer will piont to an array of **WTS_PROCESS_INFO_EX** structures.
```c++
int main() {

    DWORD level = 1; // All detailed information is requested.
    WTS_PROCESS_INFO_EX* processInfo; // Pointer to an array of WTS_PROCESS_INFO_EX
    DWORD count; // Store the number of processes enumerated.

    BOOL WTSProcess = WTSEnumerateProcessesEx(WTS_CURRENT_SERVER_HANDLE, &level, WTS_ANY_SESSION, (PWSTR*)&processInfo, &count);

    // If the function fails, exit the program.
    if (WTSProcess == FALSE){
      return 0;
    }
```
Now that we have created the function and handled the error, we can iterate over the value in **count** variable and print the retrieved information. Since **processInfo** contains the value in array, the code snippet `auto& p = processInfo[i]` iterates the i'th index and assigns it to a variable **pInfo**. The **pInfo** variable holds the informations like Process ID, Session ID, Threads, Process Name.

As per the documentation, while using **WTS_PROCESS_INFO_EX**, we need to free this structure by calling the method **WTSFreeMemory**.

```c++
    for (DWORD i = 0; i < count; i++) {
    auto& pInfo = processInfo[i];
    printf("PID: %u     Session: %u     Threads: %u      H: %u   Name: %ws\n",
        pInfo.ProcessId, pInfo.SessionId, pInfo.NumberOfThreads, pInfo.HandleCount, pInfo.pProcessName);
        }
    WTSFreeMemory(pInfo);
    return 0;
}
```

The method **WTSEnumerateProcessesEx** also contains a capability to retrieve the information about current user running the process. This can be useful while enumerating the privileges for the enumerated processes. For this, we need to use **[ConvertSidToStringSidA](https://learn.microsoft.com/en-us/windows/win32/api/sddl/nf-sddl-convertsidtostringsida)** inorder to convert SID into string format to supply it into **printf** function.

**SYNTAX**
```c++
BOOL ConvertSidToStringSidA(
  [in]  PSID  Sid,
  [out] LPSTR *StringSid
);
```

Create a Function to Convert SID into printable format using below code. This function also requires the header **sddl.h** to be added on the header section. `#include <sddl.h>`

```c++
//std::wstring to instruct the funtion to return wide string.
std::wstring SidToStringSid(PSID sid) {

    PWSTR ssid; // Pointer to a Wide String ssid, can also be defined as wchar_t
    if (ConvertSidToStringSid(sid, &ssid)) {
        std::wstring result(ssid);
        
        // Frees the memory allocated for ssid since it allocates memory for it. It is used to avoid memory leaks.
        LocalFree(ssid);
        return result;
    }
    // In case the SID is not present, return empty string otherwise function might fail.
    return L"";
```

Call the function **SidToStringSid** on the printf function above and pass the value as **processInfo.pUserSid**. The function **c_str()** is  used when you need to pass the contents of a **std::string** or **std::wstring**, otherwise the SID value will be empty.
Replace the above printf function with:
```c++
for (DWORD i = 0; i < count; i++) {
    auto& pInfo = processInfo[i];
    printf("PID: %u     Session: %u     Threads: %u      Handles: %u   Name: %ws    SID: %ws\n",
        pInfo.ProcessId, pInfo.SessionId, pInfo.NumberOfThreads, pInfo.HandleCount, pInfo.pProcessName, SidToStringSid(pInfo.pUserSid).c_str());
}
```

Before building the project, you will also require to include the dependency **Wtsapi32.lib** on it.
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-enum-wts-1.png">

<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-2.gif">

Run the code using elevated privileges to enumerate the maximum SIDs.