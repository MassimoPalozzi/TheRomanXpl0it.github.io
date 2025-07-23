---
title: Xiomara 2018 - Custom HEN
date: '2018-02-26'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- writeup
- xiomara18
tags:
- crypto
authors:
- chq-matteo
---

After a rocky start we managed to get a kind of understandable problem statement.

We are given a custom cipher with references to an "alphametic puzzle".

After a quick search I found [http://www.tkcs-collins.com/truman/alphamet/alpha_solve.shtml](http://www.tkcs-collins.com/truman/alphamet/alpha_solve.shtml)

And I got a mapping from letters to digits


```python
eflag = '082_336_88_167755403'.split('_')
enc = {'C': 0, 'F': 2, 'G': 1, 'H': 9, 'K': 3, 'N': 4, 'P': 5, 'U': 6, 'Y': 7, 'Z': 8}
dec = {str(y): x for x, y in enc.items()}
```


```python
def alphametic(x):
    return ''.join([str(dec[y]) for y in x])
```

The second part of the cipher was a bit hard to decipher because of language barriers.

What it did was basically:
1. convert each letter of the plain text to its position in the alphabet starting from 0
2. substitute each number following this rule `plain_text[i] = (plain_text[i] - (i+1) * len(plain_text)) % 26`
3. convert the numbers back

To decrypt we just add instead of subtract


```python
import string
i = lambda x: string.uppercase.index(x)

def custom(x):
    size = len(x)
    for j, c in enumerate(x):
        yield string.uppercase[(i(c) + (j + 1) * size) % 26]
```


```python
efl = ''
for ef in eflag:
    efl += (alphametic(ef))
```


```python
print(''.join(custom(efl)))
```

    THEARSOFDIDUCTION
