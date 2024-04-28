# anti-rev

## Description

```
Good luck finding the secret word for my super secure program!
```

## Categories

- `rev`

## Provided Files

- `anti-rev`

## Solve Process

Analyzing the binary we can see that the program does some rather complex stuff to our input.

```t
#  --- snip ---
char * var_74;
    var_53 = &var_0;
    var_73 = __TMC_END__;
    fgets(/* buf */ &var_2, /* n */ 31, /* fp */ var_73);
    var_72 = strncmp(&var_2, "openECSC{", 9UL);
    if (var_72 != 0) {
        var_72 = 0;
    }
    else if (var_52 != '}') {
        var_72 = 0;
    }
    else {
        var_56 = &var_0 + 19L;
        var_55 = &var_0 + 10L;
        var_57 = &var_0 + 15L;
        var_54 = &var_0 + 1L;
        var_58 = &var_0 + 5L;
        if (((unsigned int)*var_53 << 4 & 0xff) + (unsigned char)(unsigned int)*var_53 - '}' + *var_54 * -57 + (-(unsigned int)*var_58 << 2 & 0xff) + *var_55 * '|' + (unsigned char)(unsigned int)*var_57 - ((unsigned int)*var_57 << 3 & 0xff) + *var_56 * ';' != 'V') {
            var_72 = 0;
        }
        else {
            var_59 = &var_0 + 3L;
            if (*var_53 * '2' + 'j' + ((unsigned int)*var_54 << 3 & 0xff) + (unsigned char)(unsigned int)*var_54 + ((unsigned int)*var_54 << 3 & 0xff) + (unsigned char)(unsigned int)*var_54 + *var_59 * -107 + *var_55 * -118 + *var_56 * -39 != ')') {
                var_72 = 0;
            }
            else {
                var_62 = &var_0 + 13L;
                var_60 = &var_0 + 18L;
                var_61 = &var_0 + 9L;
                if (*var_59 * -81 - '0' + ((unsigned int)*var_61 << 7 & 0xff) + *var_62 * 'g' + ((unsigned int)*var_57 << 5 & 0xff) - (unsigned char)(unsigned int)*var_57 + (((unsigned int)*var_60 << 3) + (unsigned int)*var_60 << 2 & 0xff) != -35) {
                    var_72 = 0;
                }
                else {
                    var_65 = &var_0 + 12L;
                    var_64 = &var_0 + 11L;
                    var_63 = &var_0 + 7L;
                    if ((-(unsigned int)*var_53 << 6 & 0xff) - 'J' + ((unsigned int)*var_59 << 6 & 0xff) + (unsigned char)(unsigned int)*var_59 + (-(unsigned int)*var_58 << 4 & 0xff) + *var_63 * -102 + *var_55 * -78 + *var_64 * -107 + *var_65 * -84 != 'B') {
                        var_72 = 0;
                    }
                    else if (*var_59 * 'x' - 23 + *var_60 * -73 != -88) {
                        var_72 = 0;
                    }
                    else {
                        var_68 = &var_0 + 17L;
                        var_66 = &var_0 + 8L;
                        var_67 = &var_0 + 6L;
#  --- snip ---
```

What stands out is the flag prefix `openECSC{` and the `fgets()` call, that is limited to 31 chars.

So we have two conditions that are to be met for the input, out pseudo flag would look something like `openECSC{XXXXXXXXXXXXXXXXXXXX}`.

Unless you are mentally insane like me, you would not want to try and solve each `if` case in this check manually. But since I probably had too much time (I did not actually), I tried to reverse this by writing down each if case as a equation and trying to do a `linSolve()` on it.

This abomination of a code structure looked like this

```py
import sympy as sy

a, b, c, d, e, f, g, h, i, j, k, m, n, o, p, q, r, s, t, u = sy.symbols('a b c d e f g h i j k m n o p q r s t u')

c01 = sy.Eq((a * 16) + a - 125 + b * -57 + (-f * 4) + k * 124 + q - (q * 8) + u * 59, 86)
c02 = sy.Eq(a * 50 + 106 + (b * 8) + b + (b * 8) + b + d * -107 + k * -118 + u * -39, 41)
c03 = sy.Eq(d * -81 - 48 + (j * 128) + o * 103 + (q * 32) - q + ((t * 8) + t * 4), -35)
c04 = sy.Eq((a * 64) - 74 + (d * 64) + d + (-f * 16) + h * -102 + k * -78 + m * -107 + n * -84, 66)
c05 = sy.Eq(d * 120 - 23 + t * -73, -88)
c06 = sy.Eq(b * 71 + 37 + f * -69 + g * -81 + ((h * 4) + h * 8) + i * -73 + ((j * 8) + j * 16) + m * -82 + s, 126)
c07 = sy.Eq(a * -50 - 110 + d - (d * 16) + i * -23 + m * -25 + (-q * 64) + r * -68 + u * -116, -14)
c08 = sy.Eq(k * 117 + 61 + o * 46 + q * 67 + r * 107, -33)
c09 = sy.Eq((-h * 4) + 5 + m * 113 + r * -118, 8)
c10 = sy.Eq(c * 101 - 15 + (-h * 64) + ((j * 4) + j) + ((j * 4) + j * 4), -124)
c11 = sy.Eq(c * 67 + 45 + (f * 128) - f + (g * 16) + k * 39 + (-((m * 4) + m)) + (r * 16) + r + u * 47, -44)
c12 = sy.Eq(a * -121 + 37 + o * -34, 93)

# swap var_61 to e

c13 = sy.Eq(e * -91 - 121 + (g * 4) + s * -103 + (-u * 4), 56)
c14 = sy.Eq(c * -99 + 39 + f * 97 + (g + g + g * 4) + g + ((k * 4) + k * 4) + m * 34 + o * 23 - p + (s * 4) + s + (s * 4) + s + s + t * 88, -16)
c15 = sy.Eq(d * -90 + 5 + (i * 4) + i + (i * 4) + i + n * -117 + r * 56 + t * -35, -37)
c16 = sy.Eq(((c * 4) + c * 32) - 62 + g * -57, 4)
c17 = sy.Eq(b - (b * 16) - 33 + s * 83 + u * -70, -109)
c18 = sy.Eq(c * -93 - 12 + e * 43 + ((f * 4) + f) + ((f * 4) + f * 4) + h * -38 + i * -61 + ((k * 8) + k) + ((k * 8) + k * 8), 57)
c19 = sy.Eq(p * 98 + 55 + s * 113 + t * 88, -63)
c20 = sy.Eq(a * -11 + 64 + b * -100 + h * 117 + (m * 4) + m + (m * 4) + m + m + q * -79, -42)

# solve for all

flag = sy.solve((c01, c02, c02, c03, c04, c05, c06, c07, c08, c09, c10, c11, c12, c13, c14, c15, c16, c17, c18, c19, c20), (a, b, c, d, e, f, g, h, i, j, k, m, n, o, p, q, r, s, t, u))
print(flag)
```

It did go through, but not yield any results. So I put this challenge on hold because I had a headache after trying to solve it like this.

After some well deserved sleep and a few Google searches later, I came across a library named `angr`. This was pretty new to me since I never used it before, but it seemed to be a perfect fit for the job.

With `angr` you can basically solve such complex checks in a binary, by providing it with a few constraints and conditions, as well as addreses to look for.

The final solver script looked like this.

```py
import angr
import claripy

def main():
    project = angr.Project("./anti-rev", main_opts = {"base_addr": 0x0})

    chars = [claripy.BVS(f"flag{i}", 8) for i in range(31)] # <-- define the flag length here
    flag = claripy.Concat(*chars + [claripy.BVV(b'\n')])

    initial = project.factory.entry_state(stdin=flag) # <-- pass it as stdin

    for char in chars:
        initial.solver.add(char >= ord('!')) # <-- only allow characters greater than "!" (printable)
        initial.solver.add(char <= ord('~')) # <-- only allow characters smaller than "~" (printable)

    simulation = project.factory.simgr(initial)

    simulation.explore(find=0x1def, avoid=0x1df8) # <-- tell angr where 'good' and 'bad' addresses are

    if len(simulation.found) > 0:
        for found in simulation.found:
            print(found.posix.dumps(0))

main()
```

We can find the 'good' and 'bad' addresses above when looking through the binary. I did this locally with cutter which supports copying the address of the current line.

```t
#  --- snip ---
0x00001de8      nop
0x00001de9      cmp     dword [var_44h], 0
0x00001ded      je      0x1df8
0x00001def      lea     rax, [str.Correct] ; 0x200e # <---- We want this address here
0x00001df6      jmp     0x1dff
0x00001df8      lea     rax, [str.Wrong] ; 0x2017 # <------ We dont want this one
0x00001dff      mov     rdi, rax   ; const char *s
0x00001e02      call    puts       ; sym.imp.puts ; int puts(const char *s)
0x00001e07      mov     eax, 0
0x00001e0c      mov     rcx, qword [canary]
0x00001e10      xor     rcx, qword fs:[0x28]
0x00001e19      je      0x1e20
#  --- snip ---
```

That is basically all there is to it. After running the script (a few times, since it took some retries), `angr` is able to find the flag successfully and prints it.
