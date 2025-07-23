---
title: Codeblue CTF 2017 - Paillier Oracle
date: '2017-11-11'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- ctf_codeblue2017
- writeup
- codeblue2017
tags:
- number
- theory
- crypto
- rsa
authors:
- chq-matteo
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

We have a service and we are given the source code.  
When we connect we have to submit a proof of work, after that we will receive the encrypted flag.  
We can then send cipher text to the server and get the least significant bit of the decrypted text.

The first thing you find looking for the [Pailler Cryptosystem](https://en.wikipedia.org/wiki/Paillier_cryptosystem#Homomorphic_properties) is that:
1. $(n, g)$ is the public key
2. given an ecrypted message $c = enc(m)$ you can craft another encrypted message so that is decrypts to $m + m_1$ or $mm_2$.  

So we can control the decrypted text (kind of).

I worked offline at first with a mock flag.

The first idea I came up with was to send a message that decrypts to $m2^{-1}$ and build the flag 1 bit at a time. This worked for a couple of bit, but sometimes randomly not.

That's because multiplying for $2^{-1} \mod n^2$ is not really an integer division!

So I came up with this:

1. if the binary representation of $m$ (the flag for example) ends with $0$ we can craft a message that decrypts to $m2^{-1}$ and get the bit following the last zero (like a shift to the right)
2. if $m$ does *not* end with $0$ we can send a message that decrypts to $(m - 1) * 2^{-1}$ and get the next bit

The server can tell us the last bit of $m$.
So how we get the flag?
1. Get the `encrypted_flag` (ef) from the server
2. Get the least significant bit and set it on our partial `flag`
3. Send `m` so that `dec(m)` $ = (ef - flag)/ 2^{len(flag)} = ef \gg len(flag)$ 
4. Repeat from 2

It was pretty cool because when I started the script and I got `}` as the first character and I was like 'wow it works', but then I started to get random hex characters and not my mock flag.

Well I had forgotten that I switched to the remote server :P

I had a video of the run, but it got overwritten by an attempt to solve Smoke on the Water.

________________

## Python solution

I have pwntools on a virtualenv and it didn't play well with SageMath, so back to plain old python

```python
flag = 0
l = 0
# i = g^(-1), j = 2^(-1)
while l < 8 * 100:
    # dec((ef * g^(-flag)) ^ (2^(-l))) = (flag - knownbits) >> l
    flag |= (getLSB(pow(encrypted_flag * pow(i, flag, n2), pow(j, l, n2), n2)) << l)
    l += 1
    # print pt
    if l % 8 == 0:
        print unhexlify(format(pt, 'x'))

```
____________
We have these two nice properties in the Paillier Cryptosystem

$$dec(m g^{k}) = m + k \mod n^2$$

$$dec(m^{k}) = mk \mod n^2$$

### Helper functions

```python
def getLSB(m):
    '''
    Get LSB from the server
    '''
    r.sendline(str(m))
    es = r.recvline()
    lsb = int(es[-2])
    r.recvuntil(loop)
    return lsb

```
