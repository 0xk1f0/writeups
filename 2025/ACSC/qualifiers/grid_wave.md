# GRID_WAVE

## Description

```
Word’s been going around about an old piece of anti-system tech. No one's laid a hand on it in decades, but I got the specs.
```

## Categories

- `rev`

## Provided Files

- `GRID_WAVE.zip`

## Solve Process

PLACEHOLDER

```py
# number registers
r18 = 0x00
r20 = 0x00

# direct value registers
r19 = 0x00
r21 = 0x00
r22 = 0x00

def calc_chars(char_1: str, char_2: str):
    # ld r18, Z
    r18 = int(hex(ord(char_1)), 16)
    # cpi r18, 0x61
    # brge
    if (r18 >= 0x61):
        # ldi 19, 0x57
        r19 = 0x57
    else:
        # ldi 19, 0x30
        r19 = 0x30
    # sub r18, r19
    r18 = r18 - r19
    # ld r20, Z+1
    r20 = int(hex(ord(char_2)), 16)
    # cpi r20, 0x61
    # brge
    if (r20 >= 0x61):
        # ldi 21, 0x57
        r21 = 0x57
    else:
        # ldi 21, 0x30
        r21 = 0x30
    # ldi r22, 0x10
    r22 = 0x10
    # mul r18, r22
    # movw r18, r0
    r18 = r18 * r22
    # sub r20, r21
    r20 = r20 - r21
    # or r18, r20
    r18 = r18 | r20
    # st X+1, r18
    return r18

def main():
    # 'a', 'b' prints 0xab
    # so does every other combination of chars in [0-9][a-z][A-Z]
    print(hex(calc_chars('2', '8')))
    # the program checks with function at 0x8a4
    # and expects a sequence

main()
```
