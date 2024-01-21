---
title: Red Teaming - Havoc C2 Team Server and Profiles
author: nirajkharel
date: 2024-01-21 20:55:00 +0800
categories: [Red Teaming, C2]
tags: [Red Teaming, C2]
---

# The Team Server


# The C2 Profiles
You can probably refer to the [Havoc's documentation](https://havocframework.com/docs/profiles) to understand the basic syntax of the profile in detail.
I will try to explain the default havoc profile which is located on `Havoc/profiles` directory.
```bash
cd Havoc/profiles
cat havoc.yaotl

# Part 1 
Teamserver {
    Host = "0.0.0.0"
    Port = 40056

    Build {
        Compiler64 = "data/x86_64-w64-mingw32-cross/bin/x86_64-w64-mingw32-gcc"
        Compiler86 = "data/i686-w64-mingw32-cross/bin/i686-w64-mingw32-gcc"
        Nasm = "/usr/bin/nasm"
    }
}

# Part 2
Operators {
    user "5pider" {
        Password = "password1234"
    }

    user "Neo" {
        Password = "password1234"
    }
}

# Part 3
# this is optional. if you dont use it you can remove it.
Service {
    Endpoint = "service-endpoint"
    Password = "service-password"
}

# Part 4
Demon {
    Sleep = 2
    Jitter = 15

    TrustXForwardedFor = false

    Injection {
        Spawn64 = "C:\\Windows\\System32\\notepad.exe"
        Spawn32 = "C:\\Windows\\SysWOW64\\notepad.exe"
    }
}
```

Here I have divided the code blocks into four parts and we will go through it one by one.

**Part 1**
```
Teamserver {
    Host = "0.0.0.0"
    Port = 40056

    Build {
        Compiler64 = "data/x86_64-w64-mingw32-cross/bin/x86_64-w64-mingw32-gcc"
        Compiler86 = "data/i686-w64-mingw32-cross/bin/i686-w64-mingw32-gcc"
        Nasm = "/usr/bin/nasm"
    }
}
```

It contains the necessary configuration to deploy the team server on a specified host and port. Within this setup, the client can establish a connection with the team server using the designated host and port. For real-time red team engagements, it is vital to modify the host and port values, given that the default C2 port can act as an indicator for the blue team.

