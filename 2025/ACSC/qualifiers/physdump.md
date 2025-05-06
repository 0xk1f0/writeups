# physdump

## Description

```
A captured a very spooky Windows malware. It featured all kind of VM escapes, so I didn't trust any established hypervisors like VMWare or VirtualBox to execute it. However, I executed it in my own hypervisor, which it didn't knew.

With my limited hypervisor features I was only able to capture a raw physical memory and register dump. The guest OS physical memory was mapped @ 0x100000. The Windows OS is a Windwos 10 22H2 () Lets go for a walk through various Windows kernel structures, identify the malware and get the flag!
```

## Categories

- `misc`

## Provided Files

- `physdump.zip`

## Solve Process

PLACEHOLDER

```py
var_50 = b'\xf6\xd8\xfe\x07'[::-1]
var_4c = b'\x92\x4e\x15\xe2'[::-1]
var_48 = b'\x60\x32\x16\xef'[::-1]
var_44 = b'\x90\x8e\x7c\xb1'[::-1]
var_40 = b'\x71\x6b\x2d\x4b'[::-1]
var_3c = var_50 + var_4c + var_48 + var_44 + var_40 + b'\x4d\xdc\x3d\xca\xb8\xa9\x0c\xd2\x3a\xde\x02\x22\xa3\xc8\x9e\xa7\x1e\x79\xe3\x46\xda\x0d\x30\xa7\xe4\x9a\x2f\x1a\x89\xf6\xe9\xc2\xb3\x26'
var_88 = b'\x63\x9f\xbb\x9e\xd0\x25\x7c\xa7\x94\x71\x47\x53\xc2\x0f\xd1\xe9\x7b\x58\x34\x1d\x28\xe8\x4f\xa4\xdd\xcd\x53\xa1\x0a\xb3\x67\x7d\xd1\xfc\xe9\xf8\x6c\x4d\x94\x19\xa8\x39\x47\xf8\x82\xaa\x5d\x7f\xe7\x85\xd8\xa1\xc0\x5b'
i = 0
rbx = 0

var_3c = var_3c[::-1]
var_88 = var_88[::-1]

assert len(var_3c) == len(var_88)

full_string = ""

for rbx in range(len(var_3c)):
    byte_to_write = (var_3c[rbx] ^ var_88[rbx])    
    full_string += chr(byte_to_write)
    i += 1
    rbx += 1

print(full_string[::-1])
```
