---
title: 3DS CTF 2017 - W32.killah
date: '2018-01-02'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- ctf_3ds2017
- writeup
- 3ds2017
tags:
- malware
- analisys
authors:
- chq-matteo
---

We are given a malware sample. We take a snapshot on our VM and run it.
It asks for administrative privileges that we proptly grant.
After the execution our machine shuts down and cannot reboot.

Let's open it with [x64dbg](https://x64dbg.com/) and step through the code.
We notice that:
- the string @0x403223 is modified many times.
- @0x4010e0 and @0x401100 the program tries to exit so we just nop out a couple instructions
- @0x0040115f it creates a file with name 'flag_is_here' then writes 13 bytes from 0x403223 and 16 from 0x403231
There are two decryption routines
```
│              ; CALL XREF from 0x00401011 (entry0)                                                  
│              ; CALL XREF from 0x00401048 (entry0)                                                  
│              ; CALL XREF from 0x0040112d (entry0)                                                  
│              ; CALL XREF from 0x00401143 (entry0)                                                  
│           0x004011a9      51             push ecx                                                  
│           0x004011aa      52             push edx                                                  
│              ; JMP XREF from 0x004011b0 (entry0)                                                   
│       ┌─> 0x004011ab      0002           add byte [edx], al                                        
│       ⁝   0x004011ad      3002           xor byte [edx], al                                        
│       ⁝   0x004011af      42             inc edx                                                   
│       └─< 0x004011b0      e2f9           loop 0x4011ab               ;[1]                          
│           0x004011b2      5a             pop edx                                                   
│           0x004011b3      59             pop ecx                                                   
└           0x004011b4      c3             ret 
```

And

```
┌ (fcn) fcn.004011b5 10                                                                              
│   fcn.004011b5 ();                                                                                 
│              ; UNKNOWN XREF from 0x0040107e (entry0)                                               
│              ; CALL XREF from 0x0040107e (entry0)                                                  
│           0x004011b5      51             push ecx                                                  
│           0x004011b6      52             push edx                                                  
│              ; JMP XREF from 0x004011ba (fcn.004011b5)                                             
│       ┌─> 0x004011b7      3002           xor byte [edx], al                                        
│       ⁝   0x004011b9      42             inc edx                                                   
│       └─< 0x004011ba      e2fb           loop 0x4011b7               ;[3]                          
│           0x004011bc      5a             pop edx                                                   
│           0x004011bd      59             pop ecx                                                   
└           0x004011be      c3             ret   
```

So we try jumping around at each function call prelude. 
The first half of the flag is easy to get (you can actually read it in the file), for the latter half I missed a call at first so I just got rubbish and it cost me a couple of resets.
