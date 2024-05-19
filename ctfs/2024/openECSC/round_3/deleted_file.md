# Deleted file

## Description

```
Oh no, I deleted a file. I need to get it back.
```

## Categories

- `misc`

## Provided Files

- `disk.img`

## Solve Process

After making my life way too hard using `testdisk` to analyze the image file, which would yield a textfile that says:

```plain
The password is: password
```

I discovered that I could simply have used the `strings` command, which yields the same result

```bash
strings disk.img
# --- snip ---
U6U)m
VJO!
flag.txtUT
<fux
The password is: password
```

So this tells us that the image contains some sort of deleted file that we may need a password for. Using `testdisk` I indeed found a `.zip` archive, but wasn't able to restore it properly. This is why I then tried to do it using `binwalk`, which worked beautifully

```bash
binwalk -e disk.img
# this would yield
file 5200.zip
5200.zip: Zip archive data, at least v1.0 to extract, compression method=store
```

now all that is left is to unpack it and give it the correct password, which yields another file named `flag.txt` that now actually contains the flag.

```bash
7z e -ppassword 5200.zip
# --- snip ---
Everything is Ok

Archives with Warnings: 1

Warnings: 1
Size:       42
Compressed: 1027584
# --- snip ---
cat flag.txt
openECSC{thank_you_for_recovering_my_file}
```
