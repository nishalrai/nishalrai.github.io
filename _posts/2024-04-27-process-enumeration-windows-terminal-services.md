---
title: Offensive C++ - Process Enumeration (Windows Terminal Services API)
author: nirajkharel
date: 2024-06-09 14:10:00 +0800
categories: [Red Teaming, Malware Development]
tags: [Red Teaming, Malware Development]
render_with_liquid: false
---


## [Windows Terminal Services - WTS API](https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/nf-wtsapi32-wtsenumerateprocessesa)
Windows also contains **WTSEnumerateProcessesExW** function to gather inforamtion about currently active processes on the remote session via RDP or Virualization. It is not necessary for this to work only on a remote machine as it also has the capability to enumerate the running processes local machine. The target can depend on the value you pass to its handle.

This function contains five arguments: **hServer**, **\*pLevel**, **SessionId**, **\*ppProcessInfo**, **\*pCount**. 
- **hServer**: Opens a handle to open Remote Desktop session to the server. If needed to be done on the local machine, we can specify **WTS_CURRENT_SERVER_HANDLE** value on it.
- **\*pLevel**: It is a pointer which defines what kind of information is needed in return. Inorder to get maximum information, we can specify **WTS_PROCESS_INFO_EX** which is equivalent to **1** otherwise 0.
- **SessionId**: Specify which sessions to use inorder to enumerate the process. Process can be dependednt to different users logged in into the machine. To get the maximum result, we can specify **WTS_ANY_SESSION** to get list of all processes running by all users.
- **\*ppProcessInfo**: It is a pointer which contains the references to all the information gatherred via **WTS_PROCESS_INFO_EX** function.
- **\*pCount**: It is a pointer which contains the number of processes captured. This is used to iterate through the list of process.

**SYNTAX**
```c++
BOOL WTSEnumerateProcessesExW(
  [in]      HANDLE hServer,
  [in, out] DWORD  *pLevel,
  [in]      DWORD  SessionId,
  [out]     LPWSTR *ppProcessInfo,
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

Define the flags needed on **WTSEnumerateProcessesExW** as desribed above and create the function. The function will return **False** if failed and returns non zero if suceeeds. Here on the below code we suggested the program to enumerate the process on the local machine, asked for all the detialed information about the process, defined the pointer to an array of [**WTS_PROCESS_INFO_EX**](https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/ns-wtsapi32-wts_process_info_exa) structure which contains the information about running processes like SessionId, ProcessId, ProcessName, UserSid, NumberofThreads, HandleCount and much more along with the pointer which points to the variable that contains the number of structures returned by **\*ppProcessInfo**. 

As we know, **&processInfo** is used to obtain the address of the variable which is a pointer to a structure of type **WTS_PROCESS_INFO_EX**. But the function **WTSEnumerateProcessesExW** expects the fourth parameter to be of type **LPWSTR\***, which is a pointer to a wide string (i.e., a wchar_t*). However, **processInfo** is declared as a pointer to a **WTS_PROCESS_INFO_EX structure**. To make **&processInfo** compatible with the function's expected parameter type, we cast **&processInfo** to **LPWSTR\*** using **(LPWSTR\*)&processInfo**. This tells the compiler to treat the address of **processInfo** as a **LPWSTR\***, thus allowing the function to write to the pointer **processInfo** in a way that aligns with its expectations. You can learn more about [pointers here.](https://www.youtube.com/watch?v=h-HBipu_1P0&list=PL2_aWCzGMAwLZp6LMUKI3cc7pgGsasm2_&index=3)

```c++
int main() {

    HANDLE hMachine = WTS_CURRENT_SERVER_HANDLE; // Use local machine for the enumeration
    DWORD infoLevel = 1; // All detailed information is requested.
    WTS_PROCESS_INFO_EX* processInfo; // Pointer to an array of WTS_PROCESS_INFO_EX
    DWORD noOfProcess; // Store the number of processes enumerated.

    BOOL WTSProcess = WTSEnumerateProcessesExW(hMachine, &infoLevel, WTS_ANY_SESSION, (LPWSTR*)&processInfo, &noOfProcess);

    // If the function fails, exit the program.
    if (WTSProcess == FALSE) {
        return 0;
    }
```

Now that we have created the function and handled the error, we can iterate over the values in the **noOfProcess** variable and print the retrieved information. Since **processInfo** is an array, the code snippet **auto& pInfo = processInfo[i]** iterates through the `i`th index and assigns it to the variable **pInfo**. 

The **auto&** is used to tell the compiler to figure out the data type of **pInfo** based on the type of **processInfo[i]**, which is **WTS_PROCESS_INFO_EX**. You can also write **WTS_PROCESS_INFO_EX& pInfo = processInfo[i];** instead to explicitly specify the data type. The **pInfo** variable holds information such as Process ID, Session ID, number of Threads, and Process Name.

As per the documentation, while using **WTS_PROCESS_INFO_EX**, we need to free this structure by calling the method **WTSFreeMemory**.

```c++
    // Just structuring the print format
    wprintf(L"%-14ls %-10ls %-20ls\n", L"ProcessID", L"Session", L"Process Name");
    wprintf(L"%-14ls %-10ls %-20ls\n", L"----------", L"----------", L"----------");

    //
    for (DWORD i = 0; i < noOfProcess; i++) {
        
       auto& pInfo = processInfo[i];
        
        wprintf(L"%-14lu %-14lu %-20ls %-30ls\n",
            pInfo.ProcessId, pInfo.SessionId,
            pInfo.pProcessName);
    }

    WTSFreeMemory(processInfo);
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

Call the function **SidToStringSid** inside print statement above and pass the value as **processInfo.pUserSid**. The function **c_str()** is  used when you need to pass the contents of a **std::string** or **std::wstring**, otherwise the SID value will be empty.
Replace the above printf statement with:
```c++
  // Just structuring the print format
  wprintf(L"%-14ls %-10ls %-20ls %-30ls\n", L"ProcessID", L"Session", L"Process Name", L"SID");
  wprintf(L"%-14ls %-10ls %-20ls %-30ls\n", L"----------", L"----------", L"----------", L"------------------------------");

  //
  for (DWORD i = 0; i < noOfProcess; i++) {
      
    &auto pInfo = processInfo[i];
      
      wprintf(L"%-14lu %-14lu %-20ls %-30ls\n",
          pInfo.ProcessId, pInfo.SessionId,
          pInfo.pProcessName, SidToStringSid(pInfo.pUserSid).c_str());
  }
```
You can further convert that SID into username using [LookUpAccountSidA](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-lookupaccountsida) function.

Before building the project, you will also require to include the dependency **Wtsapi32.lib** on it.
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/proc-enum-wts-1.png">

<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-2.gif">

Run the code using elevated privileges to enumerate the maximum SIDs.

**References**
- [https://stackoverflow.com/questions/75605656/how-to-use-the-api-wtsenumerateprocessesexw](https://stackoverflow.com/questions/75605656/how-to-use-the-api-wtsenumerateprocessesexw)
- [https://www.youtube.com/watch?v=h-HBipu_1P0&list=PL2_aWCzGMAwLZp6LMUKI3cc7pgGsasm2_&index=2](https://www.youtube.com/watch?v=h-HBipu_1P0&list=PL2_aWCzGMAwLZp6LMUKI3cc7pgGsasm2_&index=2)
- [https://tbhaxor.com/windows-process-listing-using-wtsapi32/](https://tbhaxor.com/windows-process-listing-using-wtsapi32/)
- [https://www.youtube.com/watch?v=IZULG6I4z5U&t=324s](https://www.youtube.com/watch?v=IZULG6I4z5U&t=324s)