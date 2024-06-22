---
title: Offensive C++ - Process Modules Enumeration
author: nirajkharel
date: 2024-06-20 14:10:00 +0800
categories: [Red Teaming, Offensive Programming]
tags: [Red Teaming, Offensive Programming]
render_with_liquid: false
---


## Module Enumeration
We have already discussed different ways to enumerate processes, and one additional enumeration crucial for offensive programming is the enumeration of modules inside processes. Modules are generally DLL files and can be used to perform attacks such as DLL injections. Here, we will discuss the way to enumerate and load the DLL files (modules) associated with a specified process using the **[EnumProcessModules](https://learn.microsoft.com/en-us/windows/win32/api/psapi/nf-psapi-enumprocessmodules)** function.

The image below can describe the flow that we will cover in order to enumerate the modules. First, we need to enumerate all processes using either of the methods. Once we have the process ID, we pass that ID to open a handle to the process with **OpenProcess**. We will then use that handle with **EnumProcessModules**, which opens handles to the modules within that process. Both the process handle and module handles are passed to **GetModuleFileNameExA**, which enumerates the fully qualified path for the file containing the specified module.


<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-5.png">

### Enumerate Process Ids
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
    FindProcessModule(pid);
}
```
### [OpenProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess)
The next step involves opening a handle to the process using the **OpenProcess** function. This function requires three flags: **dwDesiredAccess**, **bInheritHandle**, and **dwProcessId**.


**SYNTAX**
```c++
HANDLE OpenProcess(
  [in] DWORD dwDesiredAccess,
  [in] BOOL  bInheritHandle,
  [in] DWORD dwProcessId
);
```
- **DWORD dwDesiredAccess**: Contains the access rights to the process object. Since we are using the function **GetModuleFileNameExA**, it requires **PROCESS_QUERY_INFORMATION** and **PROCESS_VM_READ** attributes.
- **BOOL bInheritHandle**: Specifies whether the created process should inherit the handle of the original process. Since we do not need that now, we can set it to **FALSE**.
- **DWORD dwProcessId**: ID of the process that we want to open.

Create another function such as `FindProcessModule` and open the handle for the process.

```c++
void FindProcessModule(DWORD pid) {
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
### [EnumProcessModules](https://learn.microsoft.com/en-us/windows/win32/api/psapi/nf-psapi-enumprocessmodules)
Once we have opened the handle for the specified process, the next step is to retrieve handles for each module within that process using the handle we created. The return type is boolean and the function return with zero if fails.

**SYNTAX**
```c++
BOOL EnumProcessModules(
  [in]  HANDLE  hProcess,
  [out] HMODULE *lphModule,
  [in]  DWORD   cb,
  [out] LPDWORD lpcbNeeded
);
```
- **HANDLE hProcess**: This parameter requires a handle to the process.
- **HMODULE \*lpModule**: An array where the module handles for the process are stored.  HMODULE provides handle to the module (EXE or DLL) within the selected process. We can give it the size of 1024 or 2048 initially since we do not have an idea of the potentital number of the  modules here.
- **DWORD cb**: Size of the `lpModule` array in bytes.
- **LPDWORD lpcbNeeded**: Number of bytes required to store the list of module handles in the `lpModule` array.

```c++
    // Output variable which points to an array of HMODULE values. HMODULE provides handle to the module (EXE or DLL) within the selected process.
    HMODULE lphModule[1024];

    // Size in bytes for lphModule
    DWORD cbNeeded;

    // Opens handle for each module (DLLs) within the selected process.
    BOOL bProcessModule = EnumProcessModules(hProcess, lphModule, sizeof(lphModule), &cbNeeded);
```

### [GetModuleFileNameExA](https://learn.microsoft.com/en-us/windows/win32/api/psapi/nf-psapi-getmodulefilenameexa)
Once we have the handle for each module in the process, we can use `GetModuleFileNameExA` to retrieve the full path of the file for each module, i.e., the DLL file. 

**SYNTAX**
```c++
DWORD GetModuleFileNameExA(
  [in]           HANDLE  hProcess,
  [in, optional] HMODULE hModule,
  [out]          LPSTR   lpFilename,
  [in]           DWORD   nSize
);
```

- **HANDLE hProcess:** It represents the handle to the process which contains the module.
- **HMODULE hModule:** It represents the handle to the modules within that process.
- **LPSTR lpFilename:** Provides the full path for the specified modules.
- **DWORD nSize:** It contains the buffer size for the path in bytes.

Before calling the `GetModuleFileNameExA` function, we need to iterate through the list of modules in the array because `lphModule` contains this list in an array format. To determine the number of entries in `lphModule`, we divide `lpcbNeeded` (the number of bytes required to store the list of module handles) by the size in bytes of each element in module handle (`sizeof(HMODULE)`). This calculation, represented as `int numberOfModules = cbNeeded / sizeof(HMODULE);`, gives us the count of modules based on the information returned by `EnumProcessModules`.


```c++
    // Identify the number of HMODULE values
    int numberOfModules = cbNeeded / sizeof(HMODULE);
    {
        printf("\n%s \t\t%s\n", "Address", "DLLs");

        // Iterate through each modlues
        for (int i = 0; i < numberOfModules; i++)
        {
            TCHAR szModName[MAX_PATH];
            DWORD szModNameSize = sizeof(szModName);

            // Get the full path to the module's file.
            // Process - Handle to the process, lphModule[i] - handle to the module, szModName - Filename (Output) that means it provides DLL, szModNameSize - size of the buffer
            if (GetModuleFileNameExA(hProcess, lphModule[i], szModName,szModNameSize))
            {
                // Print the module name and handle value.
                printf("%#llx \t\t%s\n", lphModule[i], szModName);
            }
        }
    }
}
```

<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-5.gif">