---
title: Xiomara 2018 - Slammer
date: '2018-02-26'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- writeup
- xiomara18
tags:
- reverse
- sidechannel
- pin
authors:
- chq-matteo
---

We are given a binary that takes a password as input, sleeps for a couple of seconds and the tells us wether the password is right or wrong.

Since I didn't feel like reversing this jumpy mess, I decided to solve this challenge using the number of memory accesses as an oracle.

I can't remember/find where I saw this technique first, if you have any resources you can leave them in the comments.

I figured out that this binary was just some kind of "fancy" strcmp, so it probably terminated the execution as soon as it found a non matching character.

First of all overwrite the nanosleep syscall with nops to speed up the bruteforce.

Then use the pinatrace [pintool](https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool) to count memory reads and writes.

Try each letter, digit and probable symbols keeping the one that leads to most memory operations.

Run this script from source/tools/SimpleExamples/obj-intel64

```python
import string
import subprocess as sp
flag = ''
alphabet = string.ascii_letters + string.digits + '{} _='

for i in range(100):
	cur = 0
	ans = ''
	for j in alphabet:
		# print(j)
		cnt = int(sp.check_output(['sh', '-c', "echo '{}' | '/home/ubuntu/Desktop/pin-3.5-97503-gac534ca30-gcc-linux/pin' -t pinatrace.so -o /proc/self/fd/1 -- ~/Desktop/slammer | wc -l".format(flag + j)]))
		# print(j, cnt)
		if cnt > cur:
			cur = cnt
			ans = j
	flag += ans
	print(flag)
```

This is the count for the first byte
```
('a', 6)
('b', 6)
('c', 6)
('d', 6)
('e', 6)
('f', 6)
('g', 6)
('h', 6)
('i', 6)
('j', 6)
('k', 6)
('l', 6)
('m', 6)
('n', 6)
('o', 6)
('p', 6)
('q', 6)
('r', 6)
('s', 6)
('t', 6)
('u', 6)
('v', 6)
('w', 6)
('x', 6516)
('y', 6)
('z', 6)
('A', 6)
('B', 6)
('C', 6)
('D', 6)
('E', 6)
('F', 6)
('G', 6)
('H', 6)
('I', 6)
('J', 6)
('K', 6)
('L', 6)
('M', 6)
('N', 6)
('O', 6)
('P', 6)
('Q', 6)
('R', 6)
('S', 6)
('T', 6)
('U', 6)
('V', 6)
('W', 6)
('X', 6)
('Y', 6)
('Z', 6)
('0', 6)
('1', 6)
('2', 6)
('3', 6)
('4', 6)
('5', 6)
('6', 6)
('7', 6)
('8', 6)
('9', 6)
('{', 6)
('}', 6)
(' ', 6)
('_', 6)
('=', 6)
```
