# nokia

## Description

```
So you have a very security-concerned friend who does not like any smartphones and only uses his original Nokia phone. He constantly receives some secret SMS, but you've never figured out what he got, as when he showed you his messages, no new texts were there. One day, you decide to whip out our old HackRF and sniff all GSM traffic until your friend receives another SMS. After some fiddling with the signals, you recover three SMS messages, which you'll find attached. Can you extract the secret message sent to your friend?

Flagformat: lowercase string which has to be put in dach2025{<string>}
```

## Categories

- `misc`

## Provided Files

- `nokia.zip`

## Solve Process

The raw amount of googling and searching that I did for this challenge may be not only a little bit insane. I was at the point of installing some very sketchy Logo Manager from days where I wasn't even born yet.

For those curious here it is: [LogoManager](https://www.chip.de/downloads/LogoManager-Classic_12996628.html) (yes I did run this in a VM, it's a CHIP installer bro).

But in the end I resorted back to raw Python to solve this one. The raw output the challenge provides contains the following

```text
//SCKL1583 0B05041583000000030103013000480E01FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFBFFF9FFFFF3FFC71FFFFFFDFFFEFBFFDED0F32
//SCKL1583 0B050415830000000301030273DFFCC18FF8EDB7BDE1DFFB6FB7FB71B7BAEFDFFB6DB7FB7B8F12718FFCF30FF8E3BFFFFFFFFFFFFFFFFF1FFFFFFFFFFFFFFFFF
//SCKL1583 0B0504158300000003010303FFFFFFFC0FFFFC0FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF00
```

The sources I found state:

> "SCKL messages consits of a header and user data. The headers specifies to which application port the message have to be send, and includes data for message segmentation and reassembly."

```text
Bytes	Description
6       //SCKL string
4       Destination Port number in HEX
4       Source Port number in HEX
2       Message Reference number
2       Number of Segments
2       Number of this segment
```

> "0x1583 Group Logo: A Group logo is a small icon that will be associated with certain groups of people. When a member of a caller group rings you, this icon is displayed in the display of the mobile phone. The size of the picture is 72x14 pixels, and has to be black and white."

Armed with this knowledge and the basic understanding that this is just creating a black/white image in the correct aspect ratio, we can seperate the userdata correctly from the SMS messages. this gives us 3 hex strings.

```text
3000480E01FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFBFFF9FFFFF3FFC71FFFFFFDFFFEFBFFDED0F32
73DFFCC18FF8EDB7BDE1DFFB6FB7FB71B7BAEFDFFB6DB7FB7B8F12718FFCF30FF8E3BFFFFFFFFFFFFFFFFF1FFFFFFFFFFFFFFFFF
FFFFFFFC0FFFFC0FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF00
```

This initally didn't work and I don't know why but the first part of the initial hex string seems to be some other specifier that Nokia phones parse, so I cut the `3000480E01` part which seemed more reasonable as we now start with `FF`. I also cut the trailing nullbytes of the last hex string.

Now I admit, I am not quite well-versed with image manipulation in Python, so my trusty QwenCoder LLM helped me out on this a bit. But essentially, we just read the hex strings in and load it as raw binary data which we then apply pixel by pixel to a flat 72x14 image, as per specification. This yields us a mirrored and upside down image initially, which we can correct using a few additional statements.

```py
from PIL import Image, ImageOps

# http://justsolve.archiveteam.org/wiki/Nokia_Operator_Logo
# https://www.smssolutions.net/tutorials/smart/sckl/

payloads = [
    'FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFBFFF9FFFFF3FFC71FFFFFFDFFFEFBFFDED0F32',
    '73DFFCC18FF8EDB7BDE1DFFB6FB7FB71B7BAEFDFFB6DB7FB7B8F12718FFCF30FF8E3BFFFFFFFFFFFFFFFFF1FFFFFFFFFFFFFFFFF',
    'FFFFFFFC0FFFFC0FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF'
]

full_content = ''
for i in payloads:
    byte_data = bytes.fromhex(i)
    binary_data = ''.join(format(byte, '08b') for byte in byte_data)
    full_content += binary_data
    print(len(binary_data))

flat_pixel_data = []
for bit in full_content:
    flat_pixel_data.insert(0, int(bit))

print(f"Pixels: {len(flat_pixel_data)}/{72*14}")

image = Image.new('1', (72, 14))
image.putdata(flat_pixel_data)
image = ImageOps.flip(image)
image = ImageOps.mirror(image)
image.show()
```

And surely enough, this Nokia SMS-GroupLogo shows us the flag in plain-sight, which gets us another solve.
