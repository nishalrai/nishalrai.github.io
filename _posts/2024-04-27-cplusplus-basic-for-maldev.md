---
title: C++ Basics For Malware Development
author: nirajkharel
date: 2024-04-27 14:10:00 +0800
categories: [Red Teaming, Malware Development]
tags: [Red Teaming, Malware Development]
render_with_liquid: false
---


Supposed to be Intro
================

The code snippet below and its explanations cover various useful Windows functions and APIs that can be implemented using the C++ language for system programming purposes (aka not a Malware Development 😉). Here, we'll discuss the basic terms associated with these functions. If you're new to C++, don't worry, we'll walk through it together, as I'm also new to it (no, please learn some syntax first).
  

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
Windows API contains a function **RemoveDirectory** to delete the directory from the specified path. It consists of a single parameter **lpPathName** which specifies the path of the directory to be removed. It works like the rmdir command on Linux, and the directory should be empty when invoking this API.

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