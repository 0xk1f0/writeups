# neon-cirquit

## Description

```
In the sprawling digital metropolis of NEXUS-42, a mysterious app called Neon Circuit has surfaced, claiming to provide untraceable access to the dark net. Your mission: infiltrate its native add-in, uncover the secrets hidden in its code, and expose the shadowy figure behind the chaos.
```

## Categories

- `rev`

## Provided Files

- `neon-cirquit-1.0.0.AppImage`

## Solve Process

Ah yes, AppImage and Electron, by some it may be under the most hated things the Linux desktop experience has ever been pleased with. Now let's dive into this by first extracting the AppImage using built-in functionality

```bash
$ ./neon-cirquit-1.0.0.AppImage --appimage-extract
$ cd squashfs-root/
```

The output folder contains the things you would expect from a finished and shipped Electron application, which I had the opportunity to develop a few times. Based on that I looked for a `app.asar` file, which is located under `resources/`.

This grants me the ability to restore the original JavaScript sourcecode.

```bash
$ npx @electron/asar extract app.asar src
$ ls src/
index.html
index.js
main.js
native-addon
node_modules
package.json
preload.js
renderer.js
```

The `index.html` file presents the main app logic, which is just a password comparison.

```html
<body>
    <h1 class="glitch" data-text="NEON CIRQUIT">NEON CIRQUIT</h1>
    <div style="text-align:center;">
        <input type="text" id="passwordInput" placeholder="Enter Access Code" />
        <br />
        <button id="submitBtn">VERIFY</button>
        <div id="result"></div>
    </div>
    <script src="renderer.js"></script>
</body>
```

In `renderer.js`, we can see a call being made to `window.nativeAddon.checkPassword()`, which points to inspecting the `native-addon` folder in our extracted source code.

In this folder we find the built binaries that provide this checkPassword functionality.

```bash
$ cd native-addon/build/Release/
$ file addon.node
addon.node: ELF 64-bit LSB shared object, x86-64
```

Let's throw this into [dogbolt](https://dogbolt.org/) or similar to see what we're working with.

Searching for a `checkPassword` something function, BinaryNinja actually shows some of the most workable output. When digging into the correct function, this piece of code appears.

```c
char* rax_42 = operator new(0x1e);
uint8_t* s_7 = &rax_42[0x1e];
var_78 = rax_42;
__builtin_memcpy(rax_42, 
    "\x9b\x9e\x9c\x97\xcd\xcf\xcd\xca\x84\x88\xcc\xa0\x9e\x8d\xcc\xa0\xb1\xcc\xa7\xaa\xac\xa0\xcb\xcd\xa0\xce\xcc\xcc\xcb\x82", 
    0x1e);
var_68 = s_7;
s_1 = s_7;

do
{
    *rax_42 = ~*rax_42;
    rax_42[1] = ~rax_42[1];
    rax_42 = &rax_42[2];
} while (s_7 != rax_42);

var_f8 = 0;
std::vector<uint8_t>::_M_realloc_insert<uint8_t>(&var_78, s_7);
char* r15_1 = var_78;
data_409200 = &data_409210;

if (r15_1) {
    // --- snip ---
    // code logic hidden
    // --- snip ---
    return result
}
```

So there is a hardcoded byte sequence present which is `memcpy`'d to `rax_42` and then shifted around in a `while` loop, before being compared and outputted.

I could have just recoded this in C but quick dirty solver scripts are always more fun to write in Python, so I opted to code out this logic from scratch and see what I get.

```py
rax_42 = bytearray(0x1E)

values = bytearray.fromhex("9b9e9c97cdcfcdca8488cca09e8dcca0b1cca7aaaca0cbcda0cecccccb82")
for i in range(0x1E):
    rax_42[i] = values[i]

s_7 = len(rax_42)
var_68 = s_7
s_1 = s_7

i = 0
while i < s_7:
    rax_42[i] = ~rax_42[i] & 0xFF
    if i + 1 < s_7:
        rax_42[i + 1] = ~(rax_42[i + 1]) & 0xFF
    i += 2

print(rax_42.decode())
```

To my surprise this was actually all there is to it, the above code prints the correct flag, which allows us to move on from this very cool challenge.
