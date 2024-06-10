---
title: Offensive C++ Basics
author: nirajkharel
date: 2024-05-15 14:10:00 +0800
categories: [Red Teaming, Malware Development]
tags: [Red Teaming, Malware Development]
render_with_liquid: false
---


Supposed to be Intro
================

The code snippet below and its explanations cover various useful Windows functions and APIs that can be implemented using the C++ language for system programming purposes. Here, we'll discuss the basic terms associated with these functions. If you're new to C++, don't worry, we'll walk through it together, as I'm also new to it (no, please learn some syntax first).
  

Since we are dealing with the Windows API, we need to use **Windows.h** as a header file which contains declarations for all of the functions in the Windows API. Also **iostream** is used for data input and output.  
Basic Syntax Format for this blog:
```c++
#include<Windows.h> // Import Windows API Functions
#include<iostream> // Input and Output
using namespace std; // Allows a program to use names for objections and variables from the standard library like  cout and endl

int main() // Entry point of the program.
{
    // YOUR CODE HERE
    system("PAUSE"); // Wait for the user to exit the program.
    return 0;
}
```
# Directories and Files
## Create a Directory
The **CreateDirectory** function creates a new directory on the specified path. The return type of the function is **BOOL** and it contains two parameters **lpPathName** and **lpSecurityAttributes**.

- **lpPathName:** Contains value for the path of the directory which needs to be created. Its data type is [**LPCTSTR**](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/f8d4fe46-6be8-44c9-8823-615a21d17a61) which means Long Pointer to Constant TCHAR String. It is commonly used in Windows API functions to pass string parameters.
- **lpSecurityAttributes:** Used to define the security attributes for the folder being created like defining who can access it and what operations they can perform. More info about it on [MSDN](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/legacy/aa379560(v=vs.85)). 

**Syntax for CreateDirectory**
```c++
BOOL CreateDirectory(
  [in]           LPCTSTR               lpPathName,
  [in, optional] LPSECURITY_ATTRIBUTES lpSecurityAttributes
);
```
Here on the above syntax, **[in]** means that it is an input parameter and  **[in, optional]** means that it is an optional input parameter which means we can set it as **NULL**. When the **lpSecurityAttributes** is set to Null, the directory gets a default security descriptor.

```c++
#include<Windows.h> 
#include<iostream>
using namespace std;
int main()
{

	// Define a Global variable, b in bCreateDir is the indication that the variable is BOOL.
	BOOL bCreateDir;

	// Create a Function and pass the directory path
	bCreateDir = CreateDirectory(
		L"C:\\Users\\theni\\Desktop\\Dir1", // L indicates for Long
		NULL);

	// If the function fails, the return value will be zero i.e. FALSE. 
	// To get the information about the error, we can use GetLastError function.
	if (bCreateDir == FALSE)
	{
		cout << "CreateDirectory Function Failed. Error No:  " << GetLastError() << endl;
	}

	cout << "CreateDirectory Function Succeed" << endl;
	getchar();
	system("PAUSE");
	return 0;
}
```
**Run the above Code**
- Click on 'Build -> Build Solution' to compile the code.
- Click on 'Debug -> Start Debugging' to run the program.

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/cplusplus1.gif">


## Delete a Directory
Windows API contains a function [**RemoveDirectory**](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-removedirectorya) to delete the directory from the specified path. It consists of a single parameter **lpPathName** which specifies the path of the directory to be removed. It works like the rmdir command on Linux, and the directory should be empty when invoking this API.

**Syntax**
```c++
BOOL RemoveDirectoryA(
  [in] LPCSTR lpPathName
);
```

```c++
#include<Windows.h>
#include<iostream>
using namespace std;
int main()
{

// Define a Global variable, b in bRemoveDir is the indication that the variable is BOOL.
BOOL bRemoveDir;

// Call RemoveDirectory funtion and specify the direcoty to be removed
bRemoveDir = RemoveDirectory(L"C:\\Users\\theni\\Desktop\\NewFolder1");

// If the function fails, the return value will be zero i.e. FALSE. 
// To get the information about the error, we can use GetLastError function.
if (bRemoveDir == FALSE)
{
cout << "RemoveDirectory Function Failed. Error No: " << GetLastError() << endl;
}
cout << "RemoveDirectory Function Succeed" << endl;
system("PAUSE");
return 0;
}
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/cplusplus2.gif">

## Create File
Windows contains [**CreateFile**](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea) function which creates or opens a file. The return type of the function is a **handle**, which is a unique identifier used by Windows programs to manage and interact with resources, such as files, devices, windows, or memory blocks. When a program requests access to a resource, the operating system assigns a handle to it. For this function, we are using a file handle.

The **CreateFile** function contains seven different arguments with one optional.
- lpFileName: Similar to the functions above, this specifies the name or path of the file or device to be created or opened.
- dwDesiredAccess: The mode in which you want to open the file, mostly **GENERIC_READ** and **GENERIC_WRITE**. It includes two additional modes: **GENERIC_ALL**, which allows all possible access rights, and **GENERIC_EXECUTE**, which allows the file to be executed.
- dwShareMode: The mode in which you want to share the file, mostly **FILE_SHARE_READ**. It includes three additional modes: 0, which prevents other processes from opening the file; **FILE_SHARE_DELETE**, which enables the operation to delete the file; and **FILE_SHARE_WRITE**, which enables write access to the file. Note that the modes specified in **dwShareMode** should align with the modes in **dwDesiredAccess**, meaning you cannot grant a permission that conflicts with the access mode specified in an existing request with an open handle. In such cases, it will return an **ERROR_SHARING_VIOLATION** error.
- lpSecurityAttributes: Optional parameter, can be left as NULL to assign it the default value.
- dwCreationDisposition: This parameter specifies an action to take on a file if it already exists. Actions include **CREATE_ALWAYS**, which overwrites an existing file; **CREATE_NEW**, which creates a new file only if it does not exist; **OPEN_ALWAYS**, which opens a file if it exists and creates a new file if it does not; **OPEN_EXISTING**, which only opens a file if it exists; and **TRUNCATE_EXISTING**, which truncates the file to zero if it exists.
- dwFlagsAndAttributes: Defines the attributes for the file, whether the file should be archived, encrypted, hidden, offline, read-only, used only by the system, or available temporarily. We can use **FILE_ATTRIBUTE_NORMAL** to not set any attributes on it.
- hTemplateFile: Optional parameter that contains a handle to a template file. Can be set to NULL.

The return value for the function is an open handle to the specified file when the function succeeds and **INVALID_HANDLE_VALUE** when the function fails.

**Syntax**
```c++
HANDLE CreateFileA(
  [in]           LPCSTR                lpFileName,
  [in]           DWORD                 dwDesiredAccess,
  [in]           DWORD                 dwShareMode,
  [in, optional] LPSECURITY_ATTRIBUTES lpSecurityAttributes,
  [in]           DWORD                 dwCreationDisposition,
  [in]           DWORD                 dwFlagsAndAttributes,
  [in, optional] HANDLE                hTemplateFile
);
```

```c++
#include<Windows.h>
#include<iostream>
using namespace std;
int main()
{
	// Define a handle, h on hFile is known as handle.
	HANDLE hFile;

	// Create a function
	hFile = CreateFile(
		L"C:\\Users\\theni\\Desktop\\Dir1\\CreateFile.txt", // Creates a file
		GENERIC_READ | GENERIC_WRITE, // Opens the file in read and write mode.
		FILE_SHARE_READ, // Enable other process to open the file in read mode.
		NULL,
		CREATE_NEW, // Creates a new file only when it does not exists.
		FILE_ATTRIBUTE_NORMAL, // Does not sets any attributes
		NULL);

	// Returns a error message if INVALID_HANDLE_VALUE
	if (hFile == INVALID_HANDLE_VALUE)
	{
		cout << "CreateFile Failed & Error No =" << GetLastError() << endl;
	}
	else {
		// Otherwise the operation is success
		cout << "CreateFile Success" << endl;
	}
	
	// close the handle
	CloseHandle(hFile);

	system("PAUSE");
	return 0;
}
```
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/cplusplus3.gif">

## Copying File
The [**CopyFile**](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-copyfile) function generally copies an existing file and its contents into a new file. It consists of three parameters and the return type is **BOOL**. The first parameter is **lpExistingFileName**, which holds the name of an existing file. The second one is **lpNewFileName**, which holds the name of the new file. The last one is **bFailIfExists**, which is a **BOOL** type that determines the action based on its value. If the value is **TRUE** and **lpNewFileName** specifies a file name that already exists, the function fails. If the value is FALSE and **lpNewFileName** specifies a file name that already exists, the new file overwrites the existing file.

Regarding the return type, if the function fails, it returns zero. It returns a nonzero value for successful execution.

**Syntax**
```c++
BOOL CopyFile(
  [in] LPCTSTR lpExistingFileName,
  [in] LPCTSTR lpNewFileName,
  [in] BOOL    bFailIfExists
);
```

```c++
#include<Windows.h>
#include<iostream>
using namespace std;

int main()
{
	// Define a global variable. 
	BOOL bFile;

	// Execute the function and assign the return type to a variable
	bFile = CopyFile(
		L"C:\\Users\\theni\\Desktop\\File1.txt",
		L"C:\\Users\\theni\\Desktop\\File2.txt",
		TRUE);

// if the return value is zero/false, the function fails.
if (bFile == FALSE)
{
	cout << "CopyFile Failed & Error No - " << GetLastError() << endl;
}

// otherwise it succeeds
else {
	cout << "CopyFile Success" << endl;
}
system("PAUSE");
return 0;
}
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/cplusplus4.gif">


## Reading the File

TODO

```c++
#include <Windows.h>
#include <iostream>

using namespace std;

int main() {
    HANDLE hFile;
    BOOL bRFile;
    char* chBuffer = nullptr;
    DWORD dwFileSize;
    DWORD dwNoByteRead;

    // Open file for reading
    hFile = CreateFile(
        L"C:\\Users\\theni\\Desktop\\File1.txt",
        GENERIC_READ,
        FILE_SHARE_READ,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hFile == INVALID_HANDLE_VALUE) {
        cout << "Unable to Open the File, Error: " << GetLastError() << endl;
        return 1;
    }

    // Get the file size
    dwFileSize = GetFileSize(hFile, NULL);
    

    // Allocate memory for the buffer
    chBuffer = new char[dwFileSize + 1]; // Add 1 for null terminator

    // Read file content into the buffer
    bRFile = ReadFile(
        hFile,
        chBuffer,
        dwFileSize,
        &dwNoByteRead,
        NULL
    );

    if (bRFile == FALSE) {
        cout << "ReadFile failed with error: " << GetLastError() << endl;
    }
    else {
        // Null-terminate the buffer
        chBuffer[dwNoByteRead] = '\0';
        cout << "ReadFile succeeded, and the data is: " << chBuffer << endl;
    }

    // Cleanup
    delete[] chBuffer;
    CloseHandle(hFile);

    system("PAUSE");
    return 0;
}
```

<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/cplusplus5.gif">