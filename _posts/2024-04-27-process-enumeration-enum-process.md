---
title: Offensive C++ - Process Enumeration (EnumProcess Function)
author: nirajkharel
date: 2024-06-10 14:10:00 +0800
categories: [Red Teaming, Malware Development]
tags: [Red Teaming, Malware Development]
render_with_liquid: false
---


## [EnumProcess Function](https://learn.microsoft.com/en-us/windows/win32/api/psapi/nf-psapi-enumprocesses)
Windows also contains the additional API **EnumProcesses** to gather the process IDs for every running process in the system. It is one of the simplest and easiest functions to gather process information, but by default, the information is limited to the process identifier (i.e., PID).

This function contains three arguments: **\*lpidProcess**, **cb** and **lpcbNeeded**. 
- **\*lpidProcess**: It is a pointer which points to an array that contains the list of running processes.
- **cb**: It stores the size of array defined to contain the list of running processes.
- **lpcbNeeded**: Contains the size about the number of bytes retured by the the function.

**SYNTAX**
```c++
BOOL EnumProcesses(
  [out] DWORD   *lpidProcess,
  [in]  DWORD   cb,
  [out] LPDWORD lpcbNeeded
);
```

We will break down codes into different chunk to understand it on better way.
The first approach is to define the necessary headers in the code. The header **Psapi.h** is used to initialize the needed functions.
```c++
#include <Windows.h>
#include <iostream>
#include <string>
#include <Psapi.h>

using namespace std;
```

Define the flags needed on **EnumProcesses** as described above and create the function. The return type for the function if it fails is **FALSE**. The function requires a pointer to an array where the PID information is stored, a variable to allocate the size of the array, and an additional variable that contains the size of bytes that will be returned by the function. First, we need to define those variables and pass them into the function **EnumProcesses**.

```c++
int main() {

    // Declare the variables
    DWORD pids[1024]; // Defining an array
    DWORD bytesNeeded; // A pointer which contains the size of bytes that will be returned by EnumProcesses 
    DWORD numOfProcesses; // A variable to calculate the number of process from bytesNeeded. A pointer to this variable will be passed on the function.

    // Create the function EnumProcesses and get a list of processIds 
    BOOL processList = EnumProcesses(pids, sizeof(pids), &bytesNeeded);

    // If function fais, exit the program.
    if (processList == FALSE) {
        return 0;
    }
```
Now that we have created the function and handled the error, we need to calculate the number of items returned in the array. Since the variable **bytesNeeded** contains the size of bytes returned by the function, we can divide it by the size of **DWORD** to calculate the number of items in it. **sizeof(DWORD)** returns the size of a single process ID in bytes, which is typically 4 bytes. By dividing the total number of bytes **bytesNeeded** by the size of each item **sizeof(DWORD)**, you get the number of items in the array. Once we have that, we can iterate over the values in the **numOfProcesses** variable and print the retrieved information. Note that it will only print the running process IDs on the system.

```c++
     // Calculate the number of process  identified on the list
    numOfProcesses = bytesNeeded / sizeof(DWORD);

    // Loop through the number of process and print every entry from it.
    for (DWORD i = 0; i < numOfProcesses; i++) {
        printf("PIDs: %u\n", pids[i]);
    }
    return 0;
}
```

In order to retrieve additional information along with PIDs, we can make use of the **[QueryFullProcessImageNameA](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-queryfullprocessimagenamea?redirectedfrom=MSDN)** function to enumerate the image name, i.e., the executable path of the process. However, since it needs the **hProcess** handle to the process, it can only be accessed after we open a handle using the **[OpenProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess)** function. We can pass the enumerated process ID into the **OpenProcess** method and we can pass the **PROCESS_QUERY_LIMITED_INFORMATION** flags on `dwDesiredAccess` to get certain information from the process handle, for example, **QueryFullProcessImageName**, which contains the names of the executables.

**SYNTAX**
```c++
HANDLE OpenProcess(
  [in] DWORD dwDesiredAccess,
  [in] BOOL  bInheritHandle,
  [in] DWORD dwProcessId
);
```

Now, within the loop above, we can create a handle and pass the required information to enumerate the image name of the process.

```c++
// Loop through the number of process and print every entrry from it.
for (DWORD i = 0; i < numOfProcesses; i++) {
    printf("PIDs: %u\n", pids[i]);

    // Create a handle to open the process
    HANDLE hProcess = OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, FALSE, pids[i]);

    // Exit the program if handle fails.
    if (hProcess == NULL) {
        return 0;
    }
}
```

Once we open the handle to the process, we call the function **QueryFullProcessImageName** to retrieve the image path. Here, it requires the handle to be passed as **hProcess**. **dwFlags** defines the function to use a particular path format; we will use 0 for the Win32 path format. **lpExeName** contains the path to the executable, and **pdwSize** represents the size of **lpExeName**.

**SYNTAX**
```c++
BOOL QueryFullProcessImageNameA(
  [in]      HANDLE hProcess,
  [in]      DWORD  dwFlags,
  [out]     LPSTR  lpExeName,
  [in, out] PDWORD lpdwSize
);
```
<br>

Finally, the above For Loop would be:
```c++
// Loop through the number of process and print every entrry from it.
for (DWORD i = 0; i < numOfProcesses; i++) {
    printf("\nPIDs: %u", pids[i]);

    // Open the specific process using process id
    HANDLE hProcess = OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, FALSE, pids[i]);
    if (hProcess != NULL) {
        
       // Define variables for QueryFullProcessImageNameA
        DWORD dwFlags = 0; // Use win32 path format
        CHAR imagePath[MAX_PATH]; // LPSTR can be defined as CHAR, MAX_PATH is 260 bytes
        DWORD imageSize = MAX_PATH;

        // Open the QueryFullProcessImageNameAimagePath function and pass the handler. The function provides image file path as output via imagePath parameter.
        BOOL imageName = QueryFullProcessImageNameA(hProcess, dwFlags, imagePath, &imageSize);
        
        /// Print the executable
        printf(" Image: %s", imagePath);
        
        // Close the handle
        CloseHandle(hProcess);
    }
}
```
<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-3.gif">

Run the code using elevated privileges to enumerate the maximum Image Path.