---
title: Insomnihack Teaser 2018 - Rule86
date: '2018-01-24'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- writeup
- insomnihackteaser2018
tags:
- crypto
- z3
authors:
- chq-matteo
---

## Why

There are already couple writeups available, so I just want to show a different approach to this challenge. I will not discuss the first part of this challenge since it has been explained before.

The writeup is kind of long, but it doesn't take much time or effort to use these ideas during a competition.

After we have obtained the decrypted python script and gif, we have to obtain the seed of the PRNG.

The algorithm is quite simple.
```python
RULE = [86 >> i & 1 for i in range(8)]
N_BYTES = 32
N = 8 * N_BYTES

def next(x):
  x = (x & 1) << N+1 | x << 1 | x >> N-1
  y = 0
  for i in range(N):
    y |= RULE[(x >> i) & 7] << i
  return y

# Bootstrap the PNRG
keystream = int.from_bytes(args.key.encode(),'little')
for i in range(N//2):
  print(i)
  keystream = next(keystream)
```

At the time of writing, every writeup focuses on *reversing* the `next` function.
However there is a trick that can be used to find preimages for similar simple algorithms which does not require to a lot of understanding of the algorithm to reverse.

The idea is that we can use a SAT or SMT solver to find the preimage of a function. We will use [Z3](https://github.com/Z3Prover/z3).

So we rewrite each function to take an expression and output an expression.

## Rewriting the functions
For function `next` we take an array of booleans (z3.Bool) which represent the bits of the seed with the least significant bit in `x[0]` and the most significant bit in `x[-1]`
```python
def next(x):
		# we translate the bit operations
    x = [x[-1]] + x + [x[0]]
		# we convert RULE to a function that takes 3 bits
    return [RULE(x[i + 2], x[i + 1], x[i]) for i in range(256)]
```

For RULE we take the [truth table](https://en.wikipedia.org/wiki/Truth_table) associated with the array and write its associated boolean function in [disjunctive normal form](https://en.wikipedia.org/wiki/Disjunctive_normal_form).
```python
def RULE(x, y, z):
    return Or(
        And(Not(x), Not(y), z), # 001 = 1 -> 1
        And(Not(x), y, Not(z)), # 010 = 2 -> 1
        And(x, Not(y), Not(z)), # 100 = 4 -> 1
        And(x, y, Not(z))       # 110 = 6 -> 1
    )
```

Now we just need to obtain the constraits.
We create a Solver, an array of 256 Bools to model our ~~64~~ 32 byte integer and we save the expression for the output of next.
```python
s = Solver()
seed = [Bool('seed[{}]'.format(i)) for i in range(256)]
next_value = next(seed)
```

We set as a target the first known output value of the PRNG.
Since we want to know the preimage of this value, we add constraints for bit to bit equality to `next_value`.
If the constraints are satisfiable we get a model (some concrete values for each of the variables we defined) for these constraints and then convert the bits to an integer
```python
target = 37450399269036614778703305999225837723915454186067915626747458322635448226786
for j in range(256):
        bit = ((1 << j) & target) >= 1
        s.add(next_value[j] == bit)
if s.check() == sat:
        m = s.model()
        ans = 0
        for j in range(256):
            ans |= (1 if m[seed[j]] else 0) << j
```

We just repeat this process 128 times to get the original seed and print the flag

## Putting it all together
### Final notes
- we actually can reverse multiple applications of the `next` function just by calling multiple time `next`
- we actually may *want* to do this since next is actually **not injective** (another plus of SAT and SMT solvers is that we can and proved this `next(0x73245e28a24b91090a0967c212e2496612c42512e29c0a224945c428b0b39271) == next(0x892814514904a727172102ce41492116429c8a41442f14c922829c50b08490a)`)
- however notice that if you compute too many iterations at the same time, you kill performance
- if we had used more of the SMT features that z3 offers, we could have almost copy pasted the original code

```python
from z3 import *
from binascii import unhexlify
s = Solver()
seed = [Bool('seed[{}]'.format(i)) for i in range(256)]

def RULE(x, y, z):
    return Or(
        And(Not(x), Not(y), z), # 001 = 1 -> 1
        And(Not(x), y, Not(z)), # 010 = 2 -> 1
        And(x, Not(y), Not(z)), # 100 = 4 -> 1
        And(x, y, Not(z))       # 110 = 6 -> 1
    )

def next(x):
    x = [x[-1]] + x + [x[0]]
    return [RULE(x[i + 2], x[i + 1], x[i]) for i in range(256)]

target = 37450399269036614778703305999225837723915454186067915626747458322635448226786
next_value = seed

steps = 8

for i in range(steps):
    next_value = next(next_value)

for i in range(128 // steps):
    # take a snapshots of the solver constraints
    s.push()
    for j in range(256):
        bit = ((1 << j) & target) >= 1
        s.add(next_value[j] == bit)
    if s.check() == sat:
        m = s.model()
        ans = 0
        for j in range(256):
            ans |= (1 if m[seed[j]] else 0) << j
        # renew the target
        target = ans

        if 'INS' in unhexlify('%064x' % ans)[::-1]:
            print(repr(unhexlify('%064x' % ans)[::-1]))
    # restore the snapshot
    s.pop()
```

### next is not injective (e.g. not invertible)

```python
s = Solver()
seed = [Bool('seed[{}]'.format(i)) for i in range(256)]
seed1 = [Bool('seed1[{}]'.format(i)) for i in range(256)]

nv = next(seed)
nv1 = next(seed1)
# we want same output
for j in range(256):
    s.add(nv[j] == nv1[j])

# we want different input
dis = False
for i in range(256):
    dis = Or(dis, seed[i] != seed1[i])

s.add(dis)
print(s.check())

m = s.model()
ans = 0
for j in range(256):
    ans |= (1 if m[seed[j]] else 0) << j
print(ans)
ans = 0
for j in range(256):
    ans |= (1 if m[seed1[j]] else 0) << j
print(ans)
s.pop()
```
