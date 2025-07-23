---
title: Harekaze CTF 2018 - Div N
date: '2018-02-11'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- writeup
- harekaze18
tags:
- crypto
- z3
authors:
- chq-matteo
---

We are given a c function that does the integer division of the argument by a constant N.
```c
long long div(long long x) {
    return x / N;
}
```

The constant is set at compile time
```
$ gcc -DN=$N -c -O2 foo.c
```

Given the disassembly of the function we need to recover N
```
$ objdump -d foo.o

foo.o:     file format elf64-x86-64

Disassembly of section .text:
0000000000000000
:
   0:	48 89 f8             	mov    %rdi,%rax
   3:	48 ba 01 0d 1a 82 9a 	movabs $0x49ea309a821a0d01,%rdx
   a:	30 ea 49
   d:	48 c1 ff 3f          	sar    $0x3f,%rdi
  11:	48 f7 ea             	imul   %rdx
  14:	48 c1 fa 30          	sar    $0x30,%rdx
  18:	48 89 d0             	mov    %rdx,%rax
  1b:	48 29 f8             	sub    %rdi,%rax
  1e:	c3                   	retq
$ echo “HarekazeCTF{$N}” > /dev/null
```

I tried at first to run the code in an emulator, but the movabs with a quad word immediate wasn't supported, so I rewrote the code in python.
In the System V x86_64 calling convenction rdi holds the first argument of a function.
The disassembly is in the AT&T syntax, not the usual Intel syntax (e.g. mov %rdi, %rax means rax = rdi, that is move rdi to rax)

```python
def div(rdi):
    rax = rdi
    rdx = 0x49ea309a821a0d01
    rdi = rdi >> 0x3f
    rdxrax = rdx * rax
    rdx, rax = rdxrax >> 64, rdxrax % (2 ** 64)
    rdx = rdx >> 0x30
    rax = rdx
    rax = rax - rdi
    return rax
```

Since this function computes x / N, we have that the smallest x such that x / N = 1 is x = N.   Since x / N is a monotone function we can use binary search to efficiently find this x.

```python
print(div(2 ** 64 - 1))
h = 2 ** 64 - 1
l = 0
while h - l > 1:
    m = (h + l) / 2
    if div(m) >= 1:
        h = m
    else:
        l = m
# we have that div(l) < 1 and l is non decreasing after each iteration
# so at the end of the binary search l is the greatest x such that div(x) = 0
# that in turn means that l + 1 is the smallest x such that div(x) = 1
print(l + 1)
print(div(l))

print(div(l + 1))
```
