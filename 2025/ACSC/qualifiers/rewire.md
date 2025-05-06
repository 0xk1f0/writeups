# rewire

## Description

```
This firmware is responsible for managing the CCTV system. Maybe we can modify it?
```

## Categories

- `rev`

## Provided Files

- `rewire.zip`

## Solve Process

This was a pretty simple "analyze and patch" challenge. I solved it using (cutter)[https://cutter.re/], but [Ghidra](https://ghidra-sre.org/) or similar will work as well.

Cutter shows us three functions we're interested in: `main()`, `sym.disable_cctv()`, `sym.disable_cctv()`.

```c
// --- snip ---
undefined8 main(void)
{
    puts("NexTek Vision Pro");
    puts("CCTV Solution");
    enable_cctv();
    return 0;
}
// --- snip ---
void enable_cctv(void)
{
    puts("\nRunning CCTV feed...");
    do {
        putchar(0x2e);
        fflush(_stdout);
        sleep(1);
    } while( true );
}
// --- snip ---
void disable_cctv(void)
{
    int64_t in_FS_OFFSET;
    int64_t var_58h;
    uint64_t var_50h;
    int64_t var_48h;
    int64_t var_40h;
    int64_t var_38h;
    int64_t var_30h;
    int64_t var_28h;
    int64_t var_20h;
    int64_t var_18h;
    int64_t canary;

    canary = *(int64_t *)(in_FS_OFFSET + 0x28);
    if (_key != 0x4e657854) {
        puts("GLOBAL key is invalid");
    }
    puts("\nDisabling CCTV feed...");
    var_48h = 0x7b57486626061930;
    var_40h = 0x3c25273a23511c2f;
    var_38h = 0x110427217e1c2767;
    var_30h = 0x7e010a313e154926;
    var_28h = 0x7a0d0f0b3c0a2737;
    var_20h = 0x2b034f62115a5920;
    var_18h._0_4_ = 0x2b544a35;
    var_18h._4_1_ = 0x29;
    for (var_58h = 0; (uint64_t)var_58h < 0x35; var_58h = var_58h + 1) {
        *(uint8_t *)((int64_t)&var_48h + var_58h) =
             *(uint8_t *)((int64_t)&var_48h + var_58h) ^ (uint8_t)(_key >> (int8_t)(((uint32_t)var_58h & 3) << 3));
    }
    printf(data.00002032, &var_48h);
    if (canary != *(int64_t *)(in_FS_OFFSET + 0x28)) {
        __stack_chk_fail();
    }
    return;
}
// --- snip ---
```

Looking at `disable_cctv()` I think this challenge could also be solvable by reversing the code that shifts around the hardcoded values, but the simpler approach, since this is a local challenge, is to modify the binary directly.

In `main()`, we directly move to `enable_cctv()` that puts us in a `while(true)` loop. So this has to be discarded first, we can do that by replacing the address at the correct instruction so it moves us to `disable_cctv()` instead.

We can achieve this by switching to the Disassembly view in cutter and patching the instruction at `0x0000137d` to point to address `0x000011e9`, which is where `disable_cctv()` resides.

```text
; initial instruction
0x0000137d      call 0x00001316
; becomes this
0x0000137d      call 0x000011e9
```

Cool, but when executing this, we can see that there is more to this.

```bash
$ ./rewire
NexTek Vision Pro
CCTV Solution
GLOBAL key is invalid

Disabling CCTV feed...
```

So there is some key we need to change. We can spot this in the disassembled code from `disable_cctv()`.

```c
// --- snip ---
if (_key != 0x4e657854) {
    puts("GLOBAL key is invalid");
}
// --- snip ---
for (var_58h = 0; (uint64_t)var_58h < 0x35; var_58h = var_58h + 1) {
    *(uint8_t *)((int64_t)&var_48h + var_58h) =
         *(uint8_t *)((int64_t)&var_48h + var_58h) ^ (uint8_t)(_key >> (int8_t)(((uint32_t)var_58h & 3) << 3));
}
// --- snip ---
```

Important to note is that `_key` is being used twice here. First it's compared to the value `0x4e657854` (which is the key, yay) and later used in the `for` loop that is used to compute the final output. So we need to change it twice.

We can do this again from the Disassembly view in Cutter.

```text
; first occurence
0x00001204      mov eax, dword [key]
; becomes
0x00001204      mov eax, 0x4e657854
; and the second occurence
0x000012ae      mov edx, dword [key]
; becomes this
0x000012ae      mov edx, 0x4e657854
```

Alright so that should be it, when we now execute the binary, we should get correct output.

```bash
$ ./rewire
NexTek Vision Pro
CCTV Solution

Disabling CCTV feed...
dach2025{d4mn_@r3_y0u_a_r1pperd0c_or_wh4t!?_67fea21e}
```

And that's another solve.
