---
title: N1CTF 2018 - MathGame
date: '2018-03-12'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- writeup
- n1ctf18
tags:
- ppc
- geometry
authors:
- dp1
---

<script type="text/javascript" async
  src="https://cdn.rawgit.com/mathjax/MathJax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  TeX: { equationNumbers: { autoNumber: "AMS" } },
  tex2jax: {
    inlineMath: [['$','$'], ['\\(','\\)']],
    processEscapes: true
  }
});
</script>

> Game rules:
>
>   A 7\*7\*7 cube consists of 343 cubes, These are 218 cubic cells in the surface, and 125 cubic cells in the internal.
>
> We put numbers in 218 cubes, of which seven cubes number have different attributes than others (for example, odd and even, prime and non-prime).
>
> Connect these seven cubes to form 21 straight lines. Only two of these 21 straight lines are perpendicular and go through the same internal cube.
>
> Please give the coordinates of this internal cube. If you answer 5 Question, you will get the flag.

Basically, we're given the six faces of the cube as 7\*7 grids from which we have to reconstruct the cube, find the seven "different" numbers, connect them, find two perpendicular lines which go through the same internal cube and return its coordinates.

This would seem like an easy ppc challenge: read the numbers, build the cube, find a few lines and give back the solution. Right? Hell, no. While _theoretically_ it's easy, we had many roadblocks in front of us to step over before finally reaching our flag.

At first there was a bug in the challenge itself that meant that the faces given weren't consistent with one another. Fine, go to sleep and wake up the next morning while the organizers fix it. After building the cube we implemented some checks (even/odd and prime/not prime were the first ones) and went on to finding the perpendicular lines. While it may seem complicated, it's actually pretty easy with some analytical geometry. Basically, a line in 3D space can be described with the following equation:

$$\frac{x-x_a}{l} = \frac{y-y_a}{m} = \frac{z-z_a}{n}$$

$$l = x_b-x_a, m = y_b-y_a, n = z_b - z_a$$

where $(x_a,y_a)$ and $(x_b,y_b)$ are the start and end points of the segment.

Two lines are perpendicular if and only if $l\cdot l'+m\cdot m' + n\cdot n' = 0$. If you're like me, you're probably staring at those equations and wondering why they work - let's just say they do.

The other geometry task we have is finding which cubes a given segment goes through. I did it the simple way, by evaluating the segment at various points and rounding the coordinates to the nearest integer. Let's see the code, shall we?

```python
# Segment between a and b
class Line:
	def __init__(self, a, b):
		self.l, self.m, self.n = b[0] - a[0], b[1] - a[1], b[2] - a[2]
		self.a, self.b = a, b
	def perpendicular(self, b):
		return self.l * b.l + self.m * b.m + self.n * b.n == 0
	def points(self):
		# Just go into parametric form and evaluate. It's ugly but it kinda works
		out = set()
		for t in range(0, 10001):
			x = int(round(self.a[0] + t * self.l / 10000.0))
			y = int(round(self.a[1] + t * self.m / 10000.0))
			z = int(round(self.a[2] + t * self.n / 10000.0))
			out.add((x,y,z))
		return out
	def __repr__(self):
		return str(self.a) + ' -> ' + str(self.b)
```

It was at this point that we encountered another roadblock: the code didn't work all the time - in fact, it often gave multiple solutions or none at all. But frankly I was tired and went forward nonetheless.

The main part was now done, all that remained was finding the characteristics we needed to select the seven numbers. Even/odd and prime/nonprime were already in place and managed to solve the first four levels of the challenge, while the fifth took us a bit more time. We tried a lot of possibilities and in the end the one that worked was based on the size of the prime factors of the numbers. More specifically, the seven numers had "small" factors - less than 10000, while the others only had bigger factors.

The code was now complete but it would still often fail, so I just left it in a bash loop running in the background. After a few minutes, it stopped and finally gave us the flag: `N1CTF{This_1s_a_1j_Math_Game4!}`

As you may have noticed, I've written this article in plural: while in the end it was my code to get the flag, this challenge only got solved because a few of us worked on it. Code, as usual:

---

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from pwn import *
from math import *
from collections import defaultdict
import random, string, hashlib, sys
import sympy, numpy

def captcha():
	line = r.readline()
	while len(line.strip()) == 0:
		line = r.readline().strip()
	base, target = line.split('"')[1::2]
	print 'Solving captcha for', base, target

	while True:
		z = ''.join(random.choice(string.ascii_letters) for _ in range(10))
		h = hashlib.sha256(base + z).hexdigest()
		if h.startswith(target):
			r.sendline(z)
			break

def readFace(name):
	print 'Reading face', name
	line = r.readline().strip()
	while line != name:
		line = r.readline().strip()

	out = []
	for i in range(7):
		r.readline()
		out.append(map(int, r.readline().strip()[1:-1].split('|')))
	return out

# 90° clockwise
def rotate(face):
	out = []
	for i in range(7):
		row = []
		for j in range(6, -1, -1):
			row.append(face[j][i])
		out += [row]
	return out

def flip(face):
	return face[::-1]

def getFace(a, b, faces, avoid):
	for i in range(6):
		if i == avoid:
			continue
		face = faces[i]
		for _ in range(2):
			for _ in range(4):
				if face[0][0] == a and face[0][6] == b:
					return i, face
				face = rotate(face)
			face = flip(face)
	print "Couldn't get face for", a, b
	quit()

def build(faces):
	nums = set()
	for f in faces:
		for l in f:
			for e in l:
				nums.add(e)
	print 'Nums:', len(nums)

	coords = {}
	# Back face
	for i in range(7):
		for j in range(7):
			coords[faces[0][i][j]] = (j, 6 - i, 0)

	# Left face
	a, b = faces[0][6][0], faces[0][0][0]
	idx, f = getFace(a, b, faces, 0)
	for i in range(7):
		for j in range(7):
			if f[i][j] in coords:
				assert coords[f[i][j]] == (0, j, i)
			else: coords[f[i][j]] = (0, j, i)

	# Right face
	a, b = faces[0][0][6], faces[0][6][6]
	idx, f = getFace(a, b, faces, 0)
	for i in range(7):
		for j in range(7):
			if f[i][j] in coords:
				assert coords[f[i][j]] == (6, 6 - j, i)
			else: coords[f[i][j]] = (6, 6 - j, i)

	# Bottom face
	a, b = faces[0][6][0], faces[0][6][6]
	idx, f = getFace(a, b, faces, 0)
	for i in range(7):
		for j in range(7):
			if f[i][j] in coords:
				assert coords[f[i][j]] == (j, 0, i)
			else: coords[f[i][j]] = (j, 0, i)

	# Top face
	a, b = faces[0][0][0], faces[0][0][6]
	idx, f = getFace(a, b, faces, 0)
	for i in range(7):
		for j in range(7):
			if f[i][j] in coords:
				assert coords[f[i][j]] == (j, 6, i)
			else: coords[f[i][j]] = (j, 6, i)

	# Front face
	a, b = f[6][0], f[6][6]
	idx, f = getFace(a, b, faces, idx)
	for i in range(7):
		for j in range(7):
			if f[i][j] in coords:
				assert coords[f[i][j]] == (j, 6 - i, 6)
			else: coords[f[i][j]] = (j, 6 - i, 6)

	return (coords, nums)

# Line between a and b
class Line:
	def __init__(self, a, b):
		self.l, self.m, self.n = b[0] - a[0], b[1] - a[1], b[2] - a[2]
		self.a, self.b = a, b
	def perpendicular(self, b):
		return self.l * b.l + self.m * b.m + self.n * b.n == 0
	def points(self):
		# Just go into parametric form and evaluate. It's ugly but it kinda works
		out = set()
		for t in range(0, 10001):
			f = round#floor if self.l > 0 else ceil
			x = int(f(self.a[0] + t * self.l / 10000.0))
			y = int(f(self.a[1] + t * self.m / 10000.0))
			z = int(f(self.a[2] + t * self.n / 10000.0))
			out.add((x,y,z))
		return out
	def __repr__(self):
		return str(self.a) + ' -> ' + str(self.b)


# Only even, prime and primefactors were actually needed
def even(x): return x % 2 == 0
def prime(x): return sympy.isprime(x)
def pal(x): return str(x) == str(x)[::-1]
def power(x, b):
	p = int(log(x, b) + 0.5)
	return b ** p == x
def perfect(x, p):
	r = x ** (1.0 / p)
	return int(r + 0.5) ** p == x
def primefactors(x): return sympy.primefactors(x)[0] < 10000

checks = [even, prime, pal, primefactors]

for i in range(2, 30):
	checks.append(lambda c: power(i, c))
	checks.append(lambda c: perfect(i, c))

def check(nums, f):
	m = defaultdict(list)
	for e in nums:
		m[f(e)].append(e)
	if len(m) != 2:
		#print 'Fails', m
		return ([], [])
	out = [x for x in m]
	return (m[out[0]], m[out[1]])

def solve(faces):
	coords, nums = build(faces)

	for c in checks:
		n = check(nums, c)
		if len(n[0]) == 7 or len(n[1]) == 7:
			print 'Check succedeed:', c
			vals = n[0] if len(n[0]) == 7 else n[1]
			print vals

			lines = []
			for i in range(len(vals)):
				for j in range(i):
					lines.append(Line(coords[vals[i]], coords[vals[j]]))

			sol = []
			for i in range(len(lines)):
				for j in range(i):
					a, b = lines[i], lines[j]
					if a.perpendicular(b):
						# print 'Perpendicular:', a, b
						pa = a.points()
						pb = b.points()
						inter = [x for x in pa.intersection(pb) if x[0] * x[1] * x[2] > 0 and x[0] < 6 and x[1] < 6 and x[2] < 6]
						# print inter
						sol += inter
			print 'Solutions:', sol
			if len(sol) == 0:
				print 'No solution found'
				# print faces
				quit()

			return sol[0]

	print 'No solver found'
	print faces
	quit()

if __name__ == "__main__":
	r = remote('47.75.60.212', 11011)

	captcha()
	if len(sys.argv) > 1:
		r.interactive()
		quit()

	for _ in range(5):
		print "=== Solving level", _, '==='

		try:
			faces = [readFace(ch) for ch in 'ABCDEF']
		except:
			print "Nope, failed again :/"
			quit()

		sol = solve(faces)

		r.recvuntil('x:')
		r.sendline(str(sol[0]))
		r.recvuntil('y:')
		r.sendline(str(sol[1]))
		r.recvuntil('z:')
		r.sendline(str(sol[2]))

	r.interactive()

```

```bash
[+] Opening connection to 47.75.60.212 on port 11011: Done
Solving captcha for qslkSm 35833
=== Solving level 0 ===
Reading face A
Reading face B
Reading face C
Reading face D
Reading face E
Reading face F
Nums: 218
Check succedeed: <function even at 0xcafebabe>
[4769684666, 6209217742, 6023565020, 7908922760, 6252119964, 4968995262, 6573984764]
Solutions: [(2, 2, 2)]
=== Solving level 1 ===
Reading face A
Reading face B
Reading face C
Reading face D
Reading face E
Reading face F
Nums: 218
Check succedeed: <function even at 0xdeadbeef>
[6904338941, 5817587951, 7750062409, 8317821287, 4624044927, 5387137963, 8289714159]
Solutions: [(3, 2, 5), (3, 2, 4)]
=== Solving level 2 ===
Reading face A
Reading face B
Reading face C
Reading face D
Reading face E
Reading face F
Nums: 218
Check succedeed: <function prime at 0xbaadcafe>
[4204061747, 3292669631, 3645316397, 3266714431, 2589252553, 2910816229, 3490277359]
Solutions: [(5, 3, 4), (5, 4, 4)]
=== Solving level 3 ===
Reading face A
Reading face B
Reading face C
Reading face D
Reading face E
Reading face F
Nums: 218
Check succedeed: <function prime at 0xbaadf00d>
[6813711975, 7612598477, 8432017163, 7160284219, 6027848751, 8535782963, 6826067957]
Solutions: [(5, 4, 5)]
=== Solving level 4 ===
Reading face A
Reading face B
Reading face C
Reading face D
Reading face E
Reading face F
Nums: 218
Check succedeed: <function primefactors at 0xb000dead>
[194668571, 288244259, 825663691, 406614797, 593067869, 353542379, 845340763]
Solutions: [(3, 3, 3)]
[*] Switching to interactive mode
 N1CTF{This_1s_a_1j_Math_Game4!}
[*] Got EOF while reading in interactive
```
