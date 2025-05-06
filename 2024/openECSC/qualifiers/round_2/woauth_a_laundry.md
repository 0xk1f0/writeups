# WOauth a Laundry

## Description

```
Welcome to our innovative business, the only ONE Laundry capable of completely sanitize your clothing by removing 100% of bacteria and viruses.

Flag is in /flag.txt.

Site: http://woauthalaundry.challs.open.ecsc2024.it
```

## Categories

- `web`

## Provided Files

- None

## Solve Process

Examining the signup process, we get a `GET` request to `/openid/authentication`. It is pretty long and has some notable values in it.

```text
http://woauthalaundry.challs.open.ecsc2024.it/openid/authentication?response_type=token%20id_token&client_id=ELX4Gr0HSRZx&scope=openid%20laundry%20amenities&redirect_uri=http://localhost:5173/&grant_type=implicit&nonce=nonce
```

We can filter out a few things, since it is some basic OAuth things you can look up. As of now, nothing really struck my interest specifically.

But after looking at the other requests, I noticed they were all getting the data from the same basepath, namely `/api/v1/`

```text
# Amenities
POST -> http://woauthalaundry.challs.open.ecsc2024.it/api/v1/amenities
# Laundry
POST -> http://woauthalaundry.challs.open.ecsc2024.it/api/v1/laundry
```

These yield a `JSON` response each. So why not poke around in that endpoint a bit. I tried a few paths like `/api/v1/users`, `/api/v1/openid` and so on.

After a bit of tinkering I found that `/api/v1/admin`, yielded not a `404`, but a `500`. Ok now this is interesting and led me to look back at the scope of the `/openid/authentication` request.

```
&scope=openid%20laundry%20amenities
```

This states the scope for the two `POST` endpoints mentioned above, but `/admin` is seemingly missing. So let's add it! Which I did using burpsuite to intercept the request made and it went through withput error. Since I don't really like BurpSuite that much, I proceeded with curl and simply grabbed the `Bearer` token that the auth request provided me with.

When trying to access the `/admin` endpoint now using a curl command:

```bash
curl -v -H "Authorization: Bearer 3520e56d3f6444e88185fd1ba967e0cd" "http://woauthalaundry.challs.open.ecsc2024.it/api/v1/admin"
```

We get a response!

```json
{
    "admin_endpoints": [
        {
            "exampleBody": {
                "requiredBy": "John Doe"
            },
            "methods": [
                "POST"
            ],
            "path": "/generate_report"
        }
    ]
}
```

Very informative, so we now know of the exisitance of another API endpoint, including its location and a sample body. How nice!

What was not so nice however is that I unsuccefully tried to access `/api/v1/generate_report`, `/generate_report` and `/admin/generate_report` without success.

Later on I tried replacing the `_` with an `-`, and that was indeed the problem. Since now `/api/v1/generate-report` when used with:

```bash
curl -X POST -v -d '{"requiredBy": "admin" }' -H "Authorization: Bearer 3520e56d3f6444e88185fd1ba967e0cd" -H "Content-Type: application/json" "http://woauthalaundry.challs.open.ecsc2024.it/api/v1/generate-report" --output out.pdf
```

Sent a response, namely a data stream which was of the `.pdf` format.

This PDF file contained the name provided by `requiredBy`. This led me to believe that I could surely somehow exploit this using some complex PDF trickery. But it was actually easier than that.

Apparently the `requiredBy` is directly interpreted HTML, which means you can pass `<script>alert()</script` and it will pass that successfully (although this example yields no output in the pdf).

Some quick googling later revealed that one can use an `iframe` tag to read a file. Since we know where the flag resides (`/flag.txt`), let's try to read that. So our payload would be the following.

```JSON
{
    "requiredBy": "<iframe id=\"textfile\" src=\"/flag.txt\"></iframe>"
}
```

And we can send that using the `curl` command above, and it actually yields us a PDF with an embedded `iframe` that reads the contents of the flag.

So we can also consider this challenge as solved.
