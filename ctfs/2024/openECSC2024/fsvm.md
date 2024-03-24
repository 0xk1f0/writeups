# fsvm

## Description

```
I want this VM to generate a good description, but all I get is "no".
```

## Provided Files

- `vm`
- `bytecode`

## Solve Process

Firstly I threw the binary into [dogbolt](https://dogbolt.org/), which is pretty handy for quick decompilation. I observed a pretty long switch statement that modified some local files under the `regs/` directory.

```t
#  --- snip ---
int32_t main(int32_t argc, char** argv, char** envp)
#  --- snip ---
    mkdir("regs", 0x1c0);
    interpret(&var_228);
    void* var_258 = &regs;
    for (void* i = &regs; i != _ITM_deregisterTMCloneTable; i = (i + 0x20))
    {
        void var_248;
        std::string::string(&var_248);
        remove(std::string::c_str(&var_248));
        std::string::~string(&var_248);
    }
    system("rmdir regs");
    std::ifstream::~ifstream(&var_228);
    if (rax == *(fsbase + 0x28))
    {
        return 0;
    }
#  --- snip ---
int64_t interpret(class std::istream* arg1)
#  --- snip ---
case 0x42:
{
    remove("regs/reg2");
    std::ofstream::ofstream(&var_228, "regs/reg2");
    std::ostream::put(&var_228, 0x30);
    std::ofstream::~ofstream(&var_228);
    goto label_3aae;
}
case 0x43:
{
    remove("regs/reg3");
    std::ofstream::ofstream(&var_228, "regs/reg3");
    std::ostream::put(&var_228, 0x30);
    std::ofstream::~ofstream(&var_228);
    goto label_3aae;
}
#  --- snip ---
```

I confirmed this by running the binary, which prompts for a flag.

```bash
┌──(k1f0@kali)-[/tmp/fsvm]
└─$ ./vm bytecode
flag:
```

Looking at the folder in which I ran this, we can indeed see the `regs/` directory I observed in the decompilation

```bash
┌──(k1f0@kali)-[/tmp/fsvm]
└─$ ls regs/
reg0  reg1  reg2  reg3  reg4  reg5  reg6  reg7
```

Ok, so we now know that the binary tweaks these files at runtime, but it would be interesting to see what gets changed. So I searched for a tool that could report to me when a file in a specific tool is modified and found `inotifywait`.

```bash
┌──(k1f0@kali)-[/tmp/fsvm]
└─$ inotifywait --timeout 2 -m regs/ | tee -a logfile.log | wc -l
Setting up watches.
Watches established.
3688
```

The above command is a combination with `wc` to just give an overview of how many operations are actually done. And thats a lot but in general we would only be interested in what actually gets written. Additionally lets print out what is modifiedm, so we can observe the data.

So in one terminal i ran the `vm` binary, and in the other the tweaked `inotifywait` command to fire at `close_write`.

```bash
┌──(k1f0@kali)-[/tmp/fsvm]
└─$ while inotifywait -e close_write -r regs/; do cat regs/*; done
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
regs/ CLOSE_WRITE,CLOSE reg0
411flag:09952100140000Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
regs/ CLOSE_WRITE,CLOSE reg1
111-1flag:cat: regs/reg5: No such file or directory
11111210111069678367123115117112101114101971151211181099952101565599521001400000000000000000000000000000Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
```

I only got this longer output after trying a few times, sometimes I would get nothing or only a few number.

But after I got it, immediately obvious is the long sequence of numbers, which i promptly threw into [CyberChef](https://cyberchef.org/) and it suggested the `From Decimal` option. Based on the fact that this binary seemingly wants the actual flag as an input, we can assume the prefix is also wanted, which is `openECSC` for this CTF.

Sure enough, when tweaking the sequence a bit, we actually get our prefix pretty fast.

```t
# initial string
11111210111069678367

# split it
111 112 101 110 69 67 83 67

# and this actually relates
111 -> o
112 -> p
101 -> e
110 -> n
69 -> E
67 -> C
83 -> S
67 -> C
```

So I repeated this for the other numbers until the sequence that is seemingly a null termination of some sort. And in (at least readable characters) that gives me

```t
openECSC{supereasyvmc4e87c4d
```

Which confused me a bit at first because it seems random and the last `}` is missing, but sure enough when just adding it and trying it out, we get confirmation.

```bash
┌──(k1f0@kali)-[/tmp/fsvm]
└─$ ./vm bytecode
flag:
openECSC{supereasyvmc4e87c4d}
ok
```

And that is a solve.
