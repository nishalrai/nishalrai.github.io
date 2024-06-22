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

<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-5.gif">

### EnumProcessModules


<br>
<img alt="" class="bf jp jq dj" loading="lazy" role="presentation" src="https://raw.githubusercontent.com/nirajkharel/nirajkharel.github.io/master/assets/img/images/process-enum-5.gif">