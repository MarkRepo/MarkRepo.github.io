---
title: PyMOTW-3 --- Email
categories: language
tags: PyMOTW-3
---

电子邮件是最古老的数字通信形式之一，但它仍然是最受欢迎的形式之一。Python的标准库包括用于发送，接收和存储电子邮件消息的模块。

smtplib与邮件服务器通信以传递消息。smtpd可用于创建自定义邮件服务器，并提供类用于调试其他应用程序中的电子邮件传输。

imaplib使用IMAP协议来处理存储在服务器上的消息。它为IMAP客户端提供低级API，可以查询，检索，移动和删除消息。

mailbox可以使用多种标准格式（包括许多电子邮件客户端程序使用的流行的mbox和Maildir格式）创建和修改本地消息存档。

## smtplib — Simple Mail Transfer Protocol Client

目的：与SMTP服务器交互，包括发送电子邮件。  
smtplib包括SMTP类，可用于与邮件服务器通信以发送邮件。

>注意  
以下示例中的电子邮件地址，主机名和IP地址已被隐藏，但其他情况下，传输脚本准确地说明了命令和响应序列。

### Sending an Email Message

SMTP的最常见用途是连接到邮件服务器并发送消息。邮件服务器主机名和端口可以传递给构造函数，或者可以显式调用connect（）。连接后，使用信封(envelope)参数和消息正文调用sendmail（）。消息文本应该完全形成（formed）并遵守RFC5322，因为smtplib根本不会修改内容或标头（headers）。这意味着调用者需要添加From和To 标头（header）。

```python
#smtplib_sendmail.py
import smtplib
import email.utils
from email.mime.text import MIMEText

# Create the message
msg = MIMEText('This is the body of the message.')
msg['To'] = email.utils.formataddr(('Recipient', 'recipient@example.com'))
msg['From'] = email.utils.formataddr(('Author', 'author@example.com'))
msg['Subject'] = 'Simple test message'

server = smtplib.SMTP('localhost', 1025)
server.set_debuglevel(True)  # show communication with the server
try:
    server.sendmail('author@example.com', ['recipient@example.com'], msg.as_string())
finally:
    server.quit()
```
在此示例中，还打开了调试以显示客户端和服务器之间的通信。 否则，该示例根本不会产生任何输出。

```
$ python3 smtplib_sendmail.py

send: 'ehlo 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\r\n'
reply: b'250-1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250 HELP\r\n'
reply: retcode (250); Msg: b'1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.
0.0.0.0.0.ip6.arpa\nSIZE 33554432\nHELP'
send: 'mail FROM:<author@example.com> size=236\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt TO:<recipient@example.com>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'Content-Type: text/plain; charset="us-ascii"\r\nMIME-Ver
sion: 1.0\r\nContent-Transfer-Encoding: 7bit\r\nTo: Recipient <r
ecipient@example.com>\r\nFrom: Author <author@example.com>\r\nSu
bject: Simple test message\r\n\r\nThis is the body of the messag
e.\r\n.\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
data: (250, b'OK')
send: 'quit\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'
```
sendmail（）的第二个参数（收件人）作为列表传递。列表中可以包含任意数量的地址，以便依次将消息传递给每个地址。 由于信封信息与消息头是分开的，因此可以通过将其包含在方法参数中而不是在消息头中来盲目抄送（BCC blind carbon-copy,即该收件人对其他收件人是不可见的）给某人。

### Authentication and Encryption

当服务器支持时，SMTP类还处理身份验证和TLS（传输层安全性）加密。要确定服务器是否支持TLS，请直接调用ehlo（）以向服务器标识客户端，
并询问它可用的扩展。然后调用`has_extn（）`来检查结果。启动TLS后，必须在进行身份验证之前再次调用ehlo（）。
许多邮件托管提供商现在只支持基于TLS的连接。 要与这些服务器通信，请使用SMTP_SSL以加密连接开始。

```python
#smtplib_authenticated.py
import smtplib
import email.utils
from email.mime.text import MIMEText
import getpass

# Prompt the user for connection info
to_email = input('Recipient: ')
servername = input('Mail server name: ')
serverport = input('Server port: ')
if serverport:
    serverport = int(serverport)
else:
    serverport = 25
use_tls = input('Use TLS? (yes/no): ').lower()
username = input('Mail username: ')
password = getpass.getpass("%s's password: " % username)

# Create the message
msg = MIMEText('Test message from PyMOTW.')
msg.set_unixfrom('author')
msg['To'] = email.utils.formataddr(('Recipient', to_email))
msg['From'] = email.utils.formataddr(('Author', 'author@example.com'))
msg['Subject'] = 'Test from PyMOTW'

if use_tls == 'yes':
    print('starting with a secure connection')
    server = smtplib.SMTP_SSL(servername, serverport)
else:
    print('starting with an insecure connection')
    server = smtplib.SMTP(servername, serverport)
try:
    server.set_debuglevel(True)

    # identify ourselves, prompting server for supported features
    server.ehlo()

    # If we can encrypt this session, do it
    if server.has_extn('STARTTLS'):
        print('(starting TLS)')
        server.starttls()
        server.ehlo()  # reidentify ourselves over TLS connection
    else:
        print('(no STARTTLS)')

    if server.has_extn('AUTH'):
        print('(logging in)')
        server.login(username, password)
    else:
        print('(no AUTH)')

    server.sendmail('author@example.com', [to_email], msg.as_string())
finally:
    server.quit()
```
启用TLS后，STARTTLS扩展名不会出现在对EHLO的回复中。

```
$ python3 source/smtplib/smtplib_authenticated.py
Recipient: doug@pymotw.com
Mail server name: localhost
Server port: 1025
Use TLS? (yes/no): no
Mail username: test
test's password:
starting with an insecure connection
send: 'ehlo 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\r\n'
reply: b'250-1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250 HELP\r\n'
reply: retcode (250); Msg: b'1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.
0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\nSIZE 33554432\nHELP'
(no STARTTLS)
(no AUTH)
send: 'mail FROM:<author@example.com> size=220\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt TO:<doug@pymotw.com>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'Content-Type: text/plain; charset="us-ascii"\r\n
MIME-Version: 1.0\r\nContent-Transfer-Encoding: 7bit\r\nTo:
Recipient <doug@pymotw.com>\r\nFrom: Author <author@example.com>
\r\nSubject: Test from PyMOTW\r\n\r\nTest message from PyMOTW.
\r\n.\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
data: (250, b'OK')
send: 'quit\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'

$ python3 source/smtplib/smtplib_authenticated.py
Recipient: doug@pymotw.com
Mail server name: mail.isp.net
Server port: 465
Use TLS? (yes/no): yes
Mail username: doughellmann@isp.net
doughellmann@isp.net's password:
starting with a secure connection
send: 'ehlo 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\r\n'
reply: b'250-mail.isp.net\r\n'
reply: b'250-PIPELINING\r\n'
reply: b'250-SIZE 71000000\r\n'
reply: b'250-ENHANCEDSTATUSCODES\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-AUTH PLAIN LOGIN\r\n'
reply: b'250 AUTH=PLAIN LOGIN\r\n'
reply: retcode (250); Msg: b'mail.isp.net\nPIPELINING\nSIZE
71000000\nENHANCEDSTATUSCODES\n8BITMIME\nAUTH PLAIN LOGIN\n
AUTH=PLAIN LOGIN'
(no STARTTLS)
(logging in)
send: 'AUTH PLAIN AGRvdWdoZWxsbWFubkBmYXN0bWFpbC5mbQBUTUZ3MDBmZmFzdG1haWw=\r\n'
reply: b'235 2.0.0 OK\r\n'
reply: retcode (235); Msg: b'2.0.0 OK'
send: 'mail FROM:<author@example.com> size=220\r\n'
reply: b'250 2.1.0 Ok\r\n'
reply: retcode (250); Msg: b'2.1.0 Ok'
send: 'rcpt TO:<doug@pymotw.com>\r\n'
reply: b'250 2.1.5 Ok\r\n'
reply: retcode (250); Msg: b'2.1.5 Ok'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'Content-Type: text/plain; charset="us-ascii"\r\n
MIME-Version: 1.0\r\nContent-Transfer-Encoding: 7bit\r\nTo:
Recipient <doug@pymotw.com>\r\nFrom: Author <author@example.com>
\r\nSubject: Test from PyMOTW\r\n\r\nTest message from PyMOTW.
\r\n.\r\n'
reply: b'250 2.0.0 Ok: queued as A0EF7F2983\r\n'
reply: retcode (250); Msg: b'2.0.0 Ok: queued as A0EF7F2983'
data: (250, b'2.0.0 Ok: queued as A0EF7F2983')
send: 'quit\r\n'
reply: b'221 2.0.0 Bye\r\n'
reply: retcode (221); Msg: b'2.0.0 Bye'
```

### Verifying an Email Address

SMTP协议包括一个命令用于向服务器询问地址是否有效。通常禁用VRFY以防止垃圾邮件发送者查找合法的电子邮件地址，但如果启用了VRFY，则客户端可以向服务器询问地址并接收指示有效性的状态码以及用户的全名（如果可用）。

```python
#smtplib_verify.py
import smtplib

server = smtplib.SMTP('mail')
server.set_debuglevel(True)  # show communication with the server
try:
    dhellmann_result = server.verify('dhellmann')
    notthere_result = server.verify('notthere')
finally:
    server.quit()

print('dhellmann:', dhellmann_result)
print('notthere :', notthere_result)
```
正如这里显示的最后两行输出所示，地址dhellmann是有效的，但notthere不是。

```
$ python3 smtplib_verify.py

send: 'vrfy <dhellmann>\r\n'
reply: '250 2.1.5 Doug Hellmann <dhellmann@mail>\r\n'
reply: retcode (250); Msg: 2.1.5 Doug Hellmann <dhellmann@mail>
send: 'vrfy <notthere>\r\n'
reply: '550 5.1.1 <notthere>... User unknown\r\n'
reply: retcode (550); Msg: 5.1.1 <notthere>... User unknown
send: 'quit\r\n'
reply: '221 2.0.0 mail closing connection\r\n'
reply: retcode (221); Msg: 2.0.0 mail closing connection
dhellmann: (250, '2.1.5 Doug Hellmann <dhellmann@mail>')
notthere : (550, '5.1.1 <notthere>... User unknown')
```

## smtpd — Sample Mail Servers

用途：包括用于实现SMTP服务器的类。  
smtpd模块包括用于构建简单邮件传输协议服务器的类。它是smtplib使用的协议的服务器端。

### Mail Server Base Class

所有提供的示例服务器的基类都是SMTPServer。它处理与客户端的通信，接收传入的数据，并提供一个方便的挂钩（hook）来覆盖（override），以便在消息完全可用后处理它。构造函数参数是侦听连接的本地地址和被代理的消息应发往的远程地址。方法process_message（）作为钩子提供，以便被派生类覆盖。在完全接收到消息时调用它，并给出以下参数：

+ peer 
客户端的地址，包含IP和传入端口的元组。
+ MAILFROM  
邮件信封中的“发件人”信息，在邮件传递时由客户端提供给服务器。不总是与From标头匹配。
+ rcpttos  
邮件信封中的收件人列表。同样，这并不总是与To标题匹配，特别是如果收件人被盲目复制。
+ data  
完整的RFC 5322 消息体.

process_message（）的默认实现会抛出NotImplementedError。下一个示例定义了一个子类，该子类重写该方法以打印有关其接收的消息的信息。

```python
#smtpd_custom.py
import smtpd
import asyncore

class CustomSMTPServer(smtpd.SMTPServer):

    def process_message(self, peer, mailfrom, rcpttos, data):
        print('Receiving message from:', peer)
        print('Message addressed from:', mailfrom)
        print('Message addressed to  :', rcpttos)
        print('Message length        :', len(data))

server = CustomSMTPServer(('127.0.0.1', 1025), None)

asyncore.loop()
```
SMTPServer使用asyncore，因此调用asyncore.loop（）运行服务器。  
需要客户端来演示服务器。可以调整smtplib部分中的一个示例来创建客户端，以将数据发送到在端口1025上本地运行的测试服务器。

```python
#smtpd_senddata.py
import smtplib
import email.utils
from email.mime.text import MIMEText

# Create the message
msg = MIMEText('This is the body of the message.')
msg['To'] = email.utils.formataddr(('Recipient', 'recipient@example.com'))
msg['From'] = email.utils.formataddr(('Author', 'author@example.com'))
msg['Subject'] = 'Simple test message'

server = smtplib.SMTP('127.0.0.1', 1025)
server.set_debuglevel(True)  # show communication with the server
try:
    server.sendmail('author@example.com', ['recipient@example.com'], msg.as_string())
finally:
    server.quit()
```
要测试程序，请在一个终端中运行`smtpd_custom.py`，在另一个终端中运行`smtpd_senddata.py`。

```
$ python3 smtpd_custom.py

Receiving message from: ('127.0.0.1', 58541)
Message addressed from: author@example.com
Message addressed to  : ['recipient@example.com']
Message length        : 229
```
smtpd_senddata.py的调试输出显示与服务器的所有通信。

```
$ python3 smtpd_senddata.py

send: 'ehlo 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\r\n'
reply: b'250-1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250 HELP\r\n'
reply: retcode (250); Msg: b'1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0
.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa\nSIZE 33554432\nHELP'
send: 'mail FROM:<author@example.com> size=236\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt TO:<recipient@example.com>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'Content-Type: text/plain; charset="us-ascii"\r\nMIME-Version: 
1.0\r\nContent-Transfer-Encoding: 7bit\r\nTo: Recipient <recipient@example.com>\r\n
From: Author <author@example.com>\r\nSubject: Simple test message\r\n\r\n
This is the body of the message.\r\n.\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
data: (250, b'OK')
send: 'quit\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'
```
要停止服务器, 按Ctrl-C.

### Debugging Server

前面的示例显示了process_message（）的参数，但smtpd还包括专门为更完整的调试而设计的服务器，称为DebuggingServer。 它将整个传入消息打印到控制台，然后停止处理（它不会将消息代理到真实的邮件服务器）。

```python
#smtpd_debug.py
import smtpd
import asyncore

server = smtpd.DebuggingServer(('127.0.0.1', 1025), None)

asyncore.loop()
```
使用之前的smtpd_senddata.py客户端程序，DebuggingServer的输出是：

```
---------- MESSAGE FOLLOWS ----------
Content-Type: text/plain; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
To: Recipient <recipient@example.com>
From: Author <author@example.com>
Subject: Simple test message
X-Peer: 127.0.0.1

This is the body of the message.
------------ END MESSAGE ------------
```

### Proxy Server

PureProxy类实现了一个简单的代理服务器。传入消息被向上游转发到作为构造函数参数给出的服务器。

>警告  
smtpd的标准库文档说，“运行它有很好的机会让你进入一个开放的中继，所以请小心。”

设置代理服务器的步骤与调试服务器类似。

```python
#smtpd_proxy.py
import smtpd
import asyncore

server = smtpd.PureProxy(('127.0.0.1', 1025), ('mail', 25))

asyncore.loop()
```
但它不打印任何输出，因此要验证它是否正常工作，请查看邮件服务器日志。

```
Aug 20 19:16:34 homer sendmail[6785]: m9JNGXJb006785:
from=<author@example.com>, size=248, class=0, nrcpts=1,
msgid=<200810192316.m9JNGXJb006785@homer.example.com>,
proto=ESMTP, daemon=MTA, relay=[192.168.1.17]
```

## mailbox — Manipulate Email Archives

目的：使用各种本地文件格式的电子邮件。  
mailbox模块定义了一个通用API，用于访问以本地磁盘格式存储的电子邮件，包括：

+ Maildir
+ mbox
+ MH
+ Babyl
+ MMDF

Mailbox和Message有基类，每个mailbox格式都包含一对相应的子类，以实现该格式的细节。

### mbox

mbox是用于在文档中显示的最简单的格式，因为它完全是纯文本。每个mailbox的所有邮件连接在一起存储为单个文件。
每次遇到以“From ”（“From”后跟单个空格）开头的行时，它将被视为新消息的开头。
只要这些字符出现在邮件正文中一行的开头，就会通过在行前面加上“>”来转义它们。

#### Creating an mbox Mailbox

通过将文件名传递给构造函数来实例化mbox类。如果该文件不存在，则当用add（）追加消息时创建该文件。

```python
#mailbox_mbox_create.py
import mailbox
import email.utils

from_addr = email.utils.formataddr(('Author', 'author@example.com'))
to_addr = email.utils.formataddr(('Recipient', 'recipient@example.com'))

payload = '''This is the body.
From (will not be escaped).
There are 3 lines.
'''

mbox = mailbox.mbox('example.mbox')
mbox.lock()
try:
    msg = mailbox.mboxMessage()
    msg.set_unixfrom('author Sat Feb  7 01:05:34 2009')
    msg['From'] = from_addr
    msg['To'] = to_addr
    msg['Subject'] = 'Sample message 1'
    msg.set_payload(payload)
    mbox.add(msg)
    mbox.flush()

    msg = mailbox.mboxMessage()
    msg.set_unixfrom('author')
    msg['From'] = from_addr
    msg['To'] = to_addr
    msg['Subject'] = 'Sample message 2'
    msg.set_payload('This is the second body.\n')
    mbox.add(msg)
    mbox.flush()
finally:
    mbox.unlock()

print(open('example.mbox', 'r').read())
```
此脚本的结果是一个包含两封电子邮件的新邮箱文件。

```
$ python3 mailbox_mbox_create.py

From MAILER-DAEMON Sun Mar 18 20:20:59 2018
From: Author <author@example.com>
To: Recipient <recipient@example.com>
Subject: Sample message 1

This is the body.
>From (will not be escaped).
There are 3 lines.

From MAILER-DAEMON Sun Mar 18 20:20:59 2018
From: Author <author@example.com>
To: Recipient <recipient@example.com>
Subject: Sample message 2

This is the second body.
```

#### Reading an mbox Mailbox

要读取现有邮箱（mailbox），请将其打开并像字典一样对待mbox对象。它的key是由邮箱实例定义的任意值，除了作为邮件对象的内部标识符之外，它们没有必需的含义。

```python
#mailbox_mbox_read.py
import mailbox

mbox = mailbox.mbox('example.mbox')
for message in mbox:
    print(message['subject'])
```
open mailbox支持迭代器协议，但与真正的字典对象不同，邮箱的默认迭代器使用值而不是键。

```
$ python3 mailbox_mbox_read.py

Sample message 1
Sample message 2
```

#### Removing Messages from an mbox Mailbox

要从mbox文件中删除现有邮件，请将其key与remove（）一起使用或使用del。

```python
#mailbox_mbox_remove.py
import mailbox

mbox = mailbox.mbox('example.mbox')
mbox.lock()
try:
    to_remove = []
    for key, msg in mbox.iteritems():
        if '2' in msg['subject']:
            print('Removing:', key)
            to_remove.append(key)
    for key in to_remove:
        mbox.remove(key)
finally:
    mbox.flush()
    mbox.close()

print(open('example.mbox', 'r').read())
```
lock（）和unlock（）方法用于防止同时访问文件的问题，flush（）强制将更改写入磁盘。

```
$ python3 mailbox_mbox_remove.py

Removing: 1
From MAILER-DAEMON Sun Mar 18 20:20:59 2018
From: Author <author@example.com>
To: Recipient <recipient@example.com>
Subject: Sample message 1

This is the body.
>From (will not be escaped).
There are 3 lines.
```

### Maildir

创建Maildir格式是为了消除对mbox文件进行并发修改的问题。邮箱被组织为目录，而不是使用单个文件，其中每个邮件都包含在其自己的文件中。 这也允许嵌套邮箱，因此扩展了Maildir邮箱的API提供方法用于处理子文件夹。

#### Creating a Maildir Mailbox

创建Maildir和mbox之间唯一真正的区别是构造函数的参数是目录名而不是文件名。和以前一样，如果邮箱不存在，则在添加邮件时创建邮箱。

```python
#mailbox_maildir_create.py
import mailbox
import email.utils
import os

from_addr = email.utils.formataddr(('Author',  'author@example.com'))
to_addr = email.utils.formataddr(('Recipient', 'recipient@example.com'))

payload = '''This is the body.
From (will not be escaped).
There are 3 lines.
'''

mbox = mailbox.Maildir('Example')
mbox.lock()
try:
    msg = mailbox.mboxMessage()
    msg.set_unixfrom('author Sat Feb  7 01:05:34 2009')
    msg['From'] = from_addr
    msg['To'] = to_addr
    msg['Subject'] = 'Sample message 1'
    msg.set_payload(payload)
    mbox.add(msg)
    mbox.flush()

    msg = mailbox.mboxMessage()
    msg.set_unixfrom('author Sat Feb  7 01:05:34 2009')
    msg['From'] = from_addr
    msg['To'] = to_addr
    msg['Subject'] = 'Sample message 2'
    msg.set_payload('This is the second body.\n')
    mbox.add(msg)
    mbox.flush()
finally:
    mbox.unlock()

for dirname, subdirs, files in os.walk('Example'):
    print(dirname)
    print('  Directories:', subdirs)
    for name in files:
        fullname = os.path.join(dirname, name)
        print('\n***', fullname)
        print(open(fullname).read())
        print('*' * 20)
```
将邮件添加到邮箱后，它们将转到new子目录。

>警告  
虽然从多个进程写入同一maildir是安全的，但add（）不是线程安全的。使用信号量或其他锁定设备可防止同一进程的多个线程同时修改邮箱。

```
$ python3 mailbox_maildir_create.py

Example
  Directories: ['new', 'cur', 'tmp']
Example/new
  Directories: []

*** Example/new/1521404460.M306174P41689Q2.hubert.local
From: Author <author@example.com>
To: Recipient <recipient@example.com>
Subject: Sample message 2

This is the second body.

********************

*** Example/new/1521404460.M303200P41689Q1.hubert.local
From: Author <author@example.com>
To: Recipient <recipient@example.com>
Subject: Sample message 1

This is the body.
From (will not be escaped).
There are 3 lines.

********************
Example/cur
  Directories: []
Example/tmp
  Directories: []
```
读取它们之后，客户端可以使用MaildirMessage的set_subdir（）方法将它们移动到cur子目录。

```python
#mailbox_maildir_set_subdir.py
import mailbox
import os

print('Before:')
mbox = mailbox.Maildir('Example')
mbox.lock()
try:
    for message_id, message in mbox.iteritems():
        print('{:6} "{}"'.format(message.get_subdir(), message['subject']))
        message.set_subdir('cur')
        # Tell the mailbox to update the message.
        mbox[message_id] = message
finally:
    mbox.flush()
    mbox.close()

print('\nAfter:')
mbox = mailbox.Maildir('Example')
for message in mbox:
    print('{:6} "{}"'.format(message.get_subdir(), message['subject']))

print()
for dirname, subdirs, files in os.walk('Example'):
    print(dirname)
    print('  Directories:', subdirs)
    for name in files:
        fullname = os.path.join(dirname, name)
        print(fullname)
```
虽然maildir包含一个“tmp”目录，但set_subdir（）的唯一有效参数是“cur”和“new”。

```
$ python3 mailbox_maildir_set_subdir.py

Before:
new    "Sample message 2"
new    "Sample message 1"

After:
cur    "Sample message 2"
cur    "Sample message 1"

Example
  Directories: ['new', 'cur', 'tmp']
Example/new
  Directories: []
Example/cur
  Directories: []
Example/cur/1521404460.M306174P41689Q2.hubert.local
Example/cur/1521404460.M303200P41689Q1.hubert.local
Example/tmp
  Directories: []
```

#### Reading a Maildir Mailbox

从现有的Maildir邮箱中读取就像mbox邮箱一样。

```python
#mailbox_maildir_read.py
import mailbox

mbox = mailbox.Maildir('Example')
for message in mbox:
    print(message['subject'])
```
不保证以任何特定顺序读取消息。

```
$ python3 mailbox_maildir_read.py

Sample message 2
Sample message 1
```

#### Removing Messages from a Maildir Mailbox

要从Maildir邮箱中删除现有邮件，请将其key传递给remove（）或使用del。

```python
#mailbox_maildir_remove.py
import mailbox
import os

mbox = mailbox.Maildir('Example')
mbox.lock()
try:
    to_remove = []
    for key, msg in mbox.iteritems():
        if '2' in msg['subject']:
            print('Removing:', key)
            to_remove.append(key)
    for key in to_remove:
        mbox.remove(key)
finally:
    mbox.flush()
    mbox.close()

for dirname, subdirs, files in os.walk('Example'):
    print(dirname)
    print('  Directories:', subdirs)
    for name in files:
        fullname = os.path.join(dirname, name)
        print('\n***', fullname)
        print(open(fullname).read())
        print('*' * 20)
```
无法计算邮件的key，因此请使用items（）或iteritems（）同时从邮箱中检索key和邮件对象。

```
$ python3 mailbox_maildir_remove.py

Removing: 1521404460.M306174P41689Q2.hubert.local
Example
  Directories: ['new', 'cur', 'tmp']
Example/new
  Directories: []
Example/cur
  Directories: []

*** Example/cur/1521404460.M303200P41689Q1.hubert.local
From: Author <author@example.com>
To: Recipient <recipient@example.com>
Subject: Sample message 1

This is the body.
From (will not be escaped).
There are 3 lines.

********************
Example/tmp
  Directories: []
```

#### Maildir folders

可以通过Maildir类的方法直接管理Maildir邮箱的子目录或文件夹。调用者可以列出，检索，创建和删除给定邮箱的子文件夹。

```python
#mailbox_maildir_folders.py
import mailbox
import os

def show_maildir(name):
    os.system('find {} -print'.format(name))

mbox = mailbox.Maildir('Example')
print('Before:', mbox.list_folders())
show_maildir('Example')

print('\n{:#^30}\n'.format(''))

mbox.add_folder('subfolder')
print('subfolder created:', mbox.list_folders())
show_maildir('Example')

subfolder = mbox.get_folder('subfolder')
print('subfolder contents:', subfolder.list_folders())

print('\n{:#^30}\n'.format(''))

subfolder.add_folder('second_level')
print('second_level created:', subfolder.list_folders())
show_maildir('Example')

print('\n{:#^30}\n'.format(''))

subfolder.remove_folder('second_level')
print('second_level removed:', subfolder.list_folders())
show_maildir('Example')
```
通过在文件夹名称前加上句点（`.`）来构造文件夹的目录名称。

```
$ python3 mailbox_maildir_folders.py

Example
Example/new
Example/cur
Example/cur/1521404460.M303200P41689Q1.hubert.local
Example/tmp
Example
Example/.subfolder
Example/.subfolder/maildirfolder
Example/.subfolder/new
Example/.subfolder/cur
Example/.subfolder/tmp
Example/new
Example/cur
Example/cur/1521404460.M303200P41689Q1.hubert.local
Example/tmp
Example
Example/.subfolder
Example/.subfolder/.second_level
Example/.subfolder/.second_level/maildirfolder
Example/.subfolder/.second_level/new
Example/.subfolder/.second_level/cur
Example/.subfolder/.second_level/tmp
Example/.subfolder/maildirfolder
Example/.subfolder/new
Example/.subfolder/cur
Example/.subfolder/tmp
Example/new
Example/cur
Example/cur/1521404460.M303200P41689Q1.hubert.local
Example/tmp
Example
Example/.subfolder
Example/.subfolder/maildirfolder
Example/.subfolder/new
Example/.subfolder/cur
Example/.subfolder/tmp
Example/new
Example/cur
Example/cur/1521404460.M303200P41689Q1.hubert.local
Example/tmp
Before: []

##############################

subfolder created: ['subfolder']
subfolder contents: []

##############################

second_level created: ['second_level']

##############################

second_level removed: []
```

### Message Flags

邮箱中的邮件具有用于跟踪方面的标志，例如邮件是否已被读取，是否被读者标记为重要，或标记为稍后删除。标志存储为一系列特定于格式的字母代码，Message类具有检索和更改标志值的方法。此示例在添加标志以指示邮件被认为是重要的之前显示Example maildir中邮件的标志。

```python
#mailbox_maildir_add_flag.py
import mailbox

print('Before:')
mbox = mailbox.Maildir('Example')
mbox.lock()
try:
    for message_id, message in mbox.iteritems():
        print('{:6} "{}"'.format(message.get_flags(), message['subject']))
        message.add_flag('F')
        # Tell the mailbox to update the message.
        mbox[message_id] = message
finally:
    mbox.flush()
    mbox.close()

print('\nAfter:')
mbox = mailbox.Maildir('Example')
for message in mbox:
    print('{:6} "{}"'.format(message.get_flags(), message['subject']))
```
默认情况下，邮件没有标志。添加标志会更改内存中的邮件，但不会更新磁盘上的邮件。
要更新磁盘上的邮件，请使用其现有标识符存储邮箱中的邮件对象。

```
$ python3 mailbox_maildir_add_flag.py

Before:
       "Sample message 1"

After:
F      "Sample message 1"
```
使用`add_flag（）`添加标志会保留任何现有标志。使用set_flags（）写入会覆盖任何现有标志集，将其替换为传递给该方法的新值。

```python
#mailbox_maildir_set_flags.py
import mailbox

print('Before:')
mbox = mailbox.Maildir('Example')
mbox.lock()
try:
    for message_id, message in mbox.iteritems():
        print('{:6} "{}"'.format(message.get_flags(), message['subject']))
        message.set_flags('S')
        # Tell the mailbox to update the message.
        mbox[message_id] = message
finally:
    mbox.flush()
    mbox.close()

print('\nAfter:')
mbox = mailbox.Maildir('Example')
for message in mbox:
    print('{:6} "{}"'.format(message.get_flags(), message['subject']))
```
在此示例中，当set_flags（）用S替换标志时，前一示例添加的F标志将丢失。

```
$ python3 mailbox_maildir_set_flags.py

Before:
F      "Sample message 1"

After:
S      "Sample message 1"
```

### Other Formats

mailbox支持其他一些格式，但没有一种像mbox或Maildir那样流行。MH是一些邮件处理程序使用的另一种多文件邮箱格式。 Babyl和MMDF是单文件格式，具有与mbox不同的消息分隔符。单文件格式支持与mbox相同的API，MH包含Maildir类中的文件夹相关方法。

## imaplib — IMAP4 Client Library

目的：用于IMAP4通信的客户端库。  
imaplib实现了与Internet邮件访问协议（IMAP）版本4服务器通信的客户端。IMAP协议定义了一组命令用于发送到服务器，并将响应传递回客户端。 大多数命令可用作用于与服务器通信的IMAP4对象的方法。  
这些示例讨论了IMAP协议的一部分，但并不完整。有关完整的详细信息，请参阅RFC 3501。

### Variations

有三种客户端类用于使用各种机制与服务器通信。第一个是IMAP4，使用明文套接字; `IMAP4_SSL`使用SSL套接字上的加密通信; 和`IMAP4_stream`使用外部命令的标准输入和标准输出。这里的所有示例都将使用IMAP4_SSL，但其他类的API类似。

### Connecting to a Server

与IMAP服务器建立连接有两个步骤。第一步，设置套接字连接本身。第二步，在服务器上作为具有帐户的用户进行身份验证。以下示例代码将从配置文件中读取服务器和用户信息。

```python
#imaplib_connect.py
import imaplib
import configparser
import os

def open_connection(verbose=False):
    # Read the config file
    config = configparser.ConfigParser()
    config.read([os.path.expanduser('~/.pymotw')])

    # Connect to the server
    hostname = config.get('server', 'hostname')
    if verbose:
        print('Connecting to', hostname)
    connection = imaplib.IMAP4_SSL(hostname)

    # Login to our account
    username = config.get('account', 'username')
    password = config.get('account', 'password')
    if verbose:
        print('Logging in as', username)
    connection.login(username, password)
    return connection

if __name__ == '__main__':
    with open_connection(verbose=True) as c:
        print(c)
```
运行时，`open_connection（）`从用户主目录中的文件中读取配置信息，然后打开IMAP4_SSL连接并进行身份验证。

```
$ python3 imaplib_connect.py

Connecting to pymotw.hellfly.net
Logging in as example
<imaplib.IMAP4_SSL object at 0x10421e320>
```
本节中的其他示例重用此模块，以避免重复代码。

#### Authentication Failure

如果建立连接但身份验证失败，则会引发异常。

```python
#imaplib_connect_fail.py
import imaplib
import configparser
import os

# Read the config file
config = configparser.ConfigParser()
config.read([os.path.expanduser('~/.pymotw')])

# Connect to the server
hostname = config.get('server', 'hostname')
print('Connecting to', hostname)
connection = imaplib.IMAP4_SSL(hostname)

# Login to our account
username = config.get('account', 'username')
password = 'this_is_the_wrong_password'
print('Logging in as', username)
try:
    connection.login(username, password)
except Exception as err:
    print('ERROR:', err)
```
此示例使用错误的密码来触发异常。

```
$ python3 imaplib_connect_fail.py

Connecting to pymotw.hellfly.net
Logging in as example
ERROR: b'[AUTHENTICATIONFAILED] Authentication failed.'
```

### Example Configuration

示例帐户有多个邮箱，层次结构如下：

+ INBOX
+ Deleted Messages
+ Archive
+ Example
	+ 2016

INBOX文件夹中有一条未读消息，Example/2016中有一条已读消息。

### Listing Mailboxes

要检索一个帐户可用的邮箱，请使用list（）方法。

```python
#imaplib_list.py
import imaplib
from pprint import pprint
from imaplib_connect import open_connection

with open_connection() as c:
    typ, data = c.list()
    print('Response code:', typ)
    print('Response:')
    pprint(data)
```
返回值是一个包含响应代码和服务器返回的数据的元组。响应代码是OK，除非出现错误。 list（）的数据是一系列字符串，包含每个邮箱的标志，层次结构分隔符和邮箱名称。

```
$ python3 imaplib_list.py

Response code: OK
Response:
[b'(\\HasChildren) "." Example',
 b'(\\HasNoChildren) "." Example.2016',
 b'(\\HasNoChildren) "." Archive',
 b'(\\HasNoChildren) "." "Deleted Messages"',
 b'(\\HasNoChildren) "." INBOX']
```
可以使用re或csv将每个响应字符串拆分为三个部分（有关使用csv的示例，请参阅本节末尾的参考中的IMAP Backup Script）。

```python
#imaplib_list_parse.py
import imaplib
import re
from imaplib_connect import open_connection

list_response_pattern = re.compile( r'\((?P<flags>.*?)\) "(?P<delimiter>.*)" (?P<name>.*)' )

def parse_list_response(line):
    match = list_response_pattern.match(line.decode('utf-8'))
    flags, delimiter, mailbox_name = match.groups()
    mailbox_name = mailbox_name.strip('"')
    return (flags, delimiter, mailbox_name)

with open_connection() as c:
    typ, data = c.list()
print('Response code:', typ)

for line in data:
    print('Server response:', line)
    flags, delimiter, mailbox_name = parse_list_response(line)
    print('Parsed response:', (flags, delimiter, mailbox_name))
```
如果邮箱名称包含空格，则服务器会用引号包含该邮箱名称，但需要删除这些引号，以便稍后在其他调用中使用邮箱名称返回给服务器。

```
$ python3 imaplib_list_parse.py

Response code: OK
Server response: b'(\\HasChildren) "." Example'
Parsed response: ('\\HasChildren', '.', 'Example')
Server response: b'(\\HasNoChildren) "." Example.2016'
Parsed response: ('\\HasNoChildren', '.', 'Example.2016')
Server response: b'(\\HasNoChildren) "." Archive'
Parsed response: ('\\HasNoChildren', '.', 'Archive')
Server response: b'(\\HasNoChildren) "." "Deleted Messages"'
Parsed response: ('\\HasNoChildren', '.', 'Deleted Messages')
Server response: b'(\\HasNoChildren) "." INBOX'
Parsed response: ('\\HasNoChildren', '.', 'INBOX')
```
list（）接受参数以层次结构的一部分指定邮箱。例如，要列出Example的子文件夹，请将“Example”作为directory参数传递。

```python
#imaplib_list_subfolders.py
import imaplib
from imaplib_connect import open_connection

with open_connection() as c:
    typ, data = c.list(directory='Example')

print('Response code:', typ)

for line in data:
    print('Server response:', line)
```
父目录和子文件夹被返回

```
$ python3 imaplib_list_subfolders.py

Response code: OK
Server response: b'(\\HasChildren) "." Example'
Server response: b'(\\HasNoChildren) "." Example.2016'
```
或者，传递pattern参数以列出与模式匹配的文件夹。

```python
#imaplib_list_pattern.py
import imaplib
from imaplib_connect import open_connection

with open_connection() as c:
    typ, data = c.list(pattern='*Example*')

print('Response code:', typ)

for line in data:
    print('Server response:', line)
```
在这种情况下，Example和Example.2016都包含在响应中。

```
$ python3 imaplib_list_pattern.py

Response code: OK
Server response: b'(\\HasChildren) "." Example'
Server response: b'(\\HasNoChildren) "." Example.2016'
```

### Mailbox Status

使用status（）询问有关内容的汇总信息。下表列出了标准定义的状态条件。  
IMAP 4 Mailbox Status Conditions

Condition	| Meaning
:---		| :---
MESSAGES	| 邮箱中的邮件数。
RECENT		| 设置了\Recent标志的邮件数。
UIDNEXT		| 邮箱的下一个唯一标识符值。
UIDVALIDITY	| 邮箱的唯一标识符有效性值。
UNSEEN		| 没有设置\Seen标志的邮件数。

状态条件必须格式化为括在括号中的空格分隔字符串，即IMAP4规范中“列表”的编码。
邮箱名称包含在双引号`"`内，如果名称包含空格或其他会引发解析器异常的字符。

```python
#imaplib_status.py
import imaplib
import re
from imaplib_connect import open_connection
from imaplib_list_parse import parse_list_response

with open_connection() as c:
    typ, data = c.list()
    for line in data:
        flags, delimiter, mailbox = parse_list_response(line)
        print('Mailbox:', mailbox)
        status = c.status(
            '"{}"'.format(mailbox),
            '(MESSAGES RECENT UIDNEXT UIDVALIDITY UNSEEN)',
        )
        print(status)
```
返回值是通常的元组，包含响应码和从服务器返回的信息列表。 在此例中，列表包含一个字符串，格式化为：使用引号包含的邮箱名称，然后是括号中的状态条件和值。

```
$ python3 imaplib_status.py

Response code: OK
Server response: b'(\\HasChildren) "." Example'
Parsed response: ('\\HasChildren', '.', 'Example')
Server response: b'(\\HasNoChildren) "." Example.2016'
Parsed response: ('\\HasNoChildren', '.', 'Example.2016')
Server response: b'(\\HasNoChildren) "." Archive'
Parsed response: ('\\HasNoChildren', '.', 'Archive')
Server response: b'(\\HasNoChildren) "." "Deleted Messages"'
Parsed response: ('\\HasNoChildren', '.', 'Deleted Messages')
Server response: b'(\\HasNoChildren) "." INBOX'
Parsed response: ('\\HasNoChildren', '.', 'INBOX')
Mailbox: Example
('OK', [b'Example (MESSAGES 0 RECENT 0 UIDNEXT 2 UIDVALIDITY 1457297771 UNSEEN 0)'])
Mailbox: Example.2016
('OK', [b'Example.2016 (MESSAGES 1 RECENT 0 UIDNEXT 3 UIDVALIDITY 1457297772 UNSEEN 0)'])
Mailbox: Archive
('OK', [b'Archive (MESSAGES 0 RECENT 0 UIDNEXT 1 UIDVALIDITY 1457297770 UNSEEN 0)'])
Mailbox: Deleted Messages
('OK', [b'"Deleted Messages" (MESSAGES 3 RECENT 0 UIDNEXT 4 UIDVALIDITY 1457297773 UNSEEN 0)'])
Mailbox: INBOX
('OK', [b'INBOX (MESSAGES 2 RECENT 0 UIDNEXT 6 UIDVALIDITY 1457297769 UNSEEN 1)'])
```

### Selecting a Mailbox

一旦客户端通过身份验证，基本操作模式是选择邮箱，然后询问服务器有关邮箱中的邮件的信息。 连接是有状态的，因此在选择邮箱后，所有命令都会对该邮箱中的邮件进行操作，直到选中新邮箱为止。

```python
#imaplib_select.py
import imaplib
import imaplib_connect

with imaplib_connect.open_connection() as c:
    typ, data = c.select('INBOX')
    print(typ, data)
    num_msgs = int(data[0])
    print('There are {} messages in INBOX'.format(num_msgs))
```
响应数据包含邮箱中的邮件总数。

```
$ python3 imaplib_select.py

OK [b'1']
There are 1 messages in INBOX
```
如果指定了无效的邮箱，则响应码为NO。

```python
#imaplib_select_invalid.py
import imaplib
import imaplib_connect

with imaplib_connect.open_connection() as c:
    typ, data = c.select('Does-Not-Exist')
    print(typ, data)
```
数据包含描述问题的错误消息。

```
$ python3 imaplib_select_invalid.py

NO [b"Mailbox doesn't exist: Does-Not-Exist"]
```

### Searching for Messages

选择邮箱后，使用search（）检索邮箱中的邮件ID。

```python
#imaplib_search_all.py
import imaplib
import imaplib_connect
from imaplib_list_parse import parse_list_response

with imaplib_connect.open_connection() as c:
    typ, mbox_data = c.list()
    for line in mbox_data:
        flags, delimiter, mbox_name = parse_list_response(line)
        c.select('"{}"'.format(mbox_name), readonly=True)
        typ, msg_ids = c.search(None, 'ALL')
        print(mbox_name, typ, msg_ids)
```
邮件ID由服务器分配，并且取决于实现。IMAP4协议区分了事务期间给定时间点的邮件的顺序ID和邮件的UID标识符，但并非所有服务器都实现这两者。

```
$ python3 imaplib_search_all.py

Response code: OK
Server response: b'(\\HasChildren) "." Example'
Parsed response: ('\\HasChildren', '.', 'Example')
Server response: b'(\\HasNoChildren) "." Example.2016'
Parsed response: ('\\HasNoChildren', '.', 'Example.2016')
Server response: b'(\\HasNoChildren) "." Archive'
Parsed response: ('\\HasNoChildren', '.', 'Archive')
Server response: b'(\\HasNoChildren) "." "Deleted Messages"'
Parsed response: ('\\HasNoChildren', '.', 'Deleted Messages')
Server response: b'(\\HasNoChildren) "." INBOX'
Parsed response: ('\\HasNoChildren', '.', 'INBOX')
Example OK [b'']
Example.2016 OK [b'1']
Archive OK [b'']
Deleted Messages OK [b'']
INBOX OK [b'1']
```
在此例中，INBOX和Example.2016各自具有id为1的不同消息。其他邮箱为空。

### Search Criteria

可以使用各种其他搜索条件，包括查看邮件的日志，标志和其他标头。请参阅第6.4.4节。 RFC 3501的完整细节。  
要查找带有主题“Example message 2”的邮件，搜索条件应构造为：

```
(SUBJECT "Example message 2")
```
此示例在所有邮箱中查找标题为“Example message 2”的所有邮件：

```python
#imaplib_search_subject.py
import imaplib
import imaplib_connect
from imaplib_list_parse import parse_list_response

with imaplib_connect.open_connection() as c:
    typ, mbox_data = c.list()
    for line in mbox_data:
        flags, delimiter, mbox_name = parse_list_response(line)
        c.select('"{}"'.format(mbox_name), readonly=True)
        typ, msg_ids = c.search( None, '(SUBJECT "Example message 2")', )
        print(mbox_name, typ, msg_ids)
```
帐户中只有一条此类消息，它位于INBOX中。

```
$ python3 imaplib_search_subject.py

Response code: OK
Server response: b'(\\HasChildren) "." Example'
Parsed response: ('\\HasChildren', '.', 'Example')
Server response: b'(\\HasNoChildren) "." Example.2016'
Parsed response: ('\\HasNoChildren', '.', 'Example.2016')
Server response: b'(\\HasNoChildren) "." Archive'
Parsed response: ('\\HasNoChildren', '.', 'Archive')
Server response: b'(\\HasNoChildren) "." "Deleted Messages"'
Parsed response: ('\\HasNoChildren', '.', 'Deleted Messages')
Server response: b'(\\HasNoChildren) "." INBOX'
Parsed response: ('\\HasNoChildren', '.', 'INBOX')
Example OK [b'']
Example.2016 OK [b'']
Archive OK [b'']
Deleted Messages OK [b'']
INBOX OK [b'1']
```
搜索条件也可以组合。

```python
#imaplib_search_from.py
import imaplib
import imaplib_connect
from imaplib_list_parse import parse_list_response

with imaplib_connect.open_connection() as c:
    typ, mbox_data = c.list()
    for line in mbox_data:
        flags, delimiter, mbox_name = parse_list_response(line)
        c.select('"{}"'.format(mbox_name), readonly=True)
        typ, msg_ids = c.search( None, '(FROM "Doug" SUBJECT "Example message 2")',)
        print(mbox_name, typ, msg_ids)
```
使用逻辑and操作组合搜索条件

```
$ python3 imaplib_search_from.py

Response code: OK
Server response: b'(\\HasChildren) "." Example'
Parsed response: ('\\HasChildren', '.', 'Example')
Server response: b'(\\HasNoChildren) "." Example.2016'
Parsed response: ('\\HasNoChildren', '.', 'Example.2016')
Server response: b'(\\HasNoChildren) "." Archive'
Parsed response: ('\\HasNoChildren', '.', 'Archive')
Server response: b'(\\HasNoChildren) "." "Deleted Messages"'
Parsed response: ('\\HasNoChildren', '.', 'Deleted Messages')
Server response: b'(\\HasNoChildren) "." INBOX'
Parsed response: ('\\HasNoChildren', '.', 'INBOX')
Example OK [b'']
Example.2016 OK [b'']
Archive OK [b'']
Deleted Messages OK [b'']
INBOX OK [b'1']
```

### Fetching Messages

search（）返回的标识符用于检索邮件的内容或部分内容，以便使用fetch（）方法进一步处理。它需要两个参数，即要获取的邮件ID和要检索的邮件的部分。`message_ids`参数是逗号分隔的id列表（例如，“1”，“1,2”）或ID范围（例如，1：2）。`message_parts`参数是邮件段名称的IMAP列表。与search（）的搜索条件一样，IMAP协议指定了命名的邮件段，因此客户端可以高效地只检索它们实际需要的邮件部分。例如，要检索邮箱中邮件的标头，请使用带有参数`BODY.PEEK[HEADER]`的fetch（）。

>注意  
获取标头的另一种方法是BODY[HEADERS]，但是该形式具有隐式将邮件标记为已读的副作用，这在许多情况下是不想这样的。

```python
#imaplib_fetch_raw.py
import imaplib
import pprint
import imaplib_connect

imaplib.Debug = 4
with imaplib_connect.open_connection() as c:
    c.select('INBOX', readonly=True)
    typ, msg_data = c.fetch('1', '(BODY.PEEK[HEADER] FLAGS)')
    pprint.pprint(msg_data)
```
fetch（）的返回值已被部分解析，因此使用它比使用list（）的返回值更难。打开调试显示客户端和服务器之间的完整交互，以了解其原因。

```
$ python3 imaplib_fetch_raw.py

  19:40.68 imaplib version 2.58
  19:40.68 new IMAP4 connection, tag=b'IIEN'
  19:40.70 < b'* OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE AUTH=PLAIN] Dovecot (Ubuntu) ready.'
  19:40.70 > b'IIEN0 CAPABILITY'
  19:40.73 < b'* CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE AUTH=PLAIN'
  19:40.73 < b'IIEN0 OK Pre-login capabilities listed, post-login capabilities have more.'
  19:40.73 CAPABILITIES: ('IMAP4REV1', 'LITERAL+', 'SASL-IR', 'LOGIN-REFERRALS', 'ID', 'ENABLE', 'IDLE', 'AUTH=PLAIN')
  19:40.73 > b'IIEN1 LOGIN example "TMFw00fpymotw"'
  19:40.79 < b'* CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REF
ERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD
=REFS THREAD=ORDEREDSUBJECT MULTIAPPEND URL-PARTIAL CATENATE UNS
ELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDS
TORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-
STATUS SPECIAL-USE BINARY MOVE'
  19:40.79 < b'IIEN1 OK Logged in'
  19:40.79 > b'IIEN2 EXAMINE INBOX'
  19:40.82 < b'* FLAGS (\\Answered \\Flagged \\Deleted \\Seen \\Draft)'
  19:40.82 < b'* OK [PERMANENTFLAGS ()] Read-only mailbox.'
  19:40.82 < b'* 2 EXISTS'
  19:40.82 < b'* 0 RECENT'
  19:40.82 < b'* OK [UNSEEN 1] First unseen.'
  19:40.82 < b'* OK [UIDVALIDITY 1457297769] UIDs valid'
  19:40.82 < b'* OK [UIDNEXT 6] Predicted next UID'
  19:40.82 < b'* OK [HIGHESTMODSEQ 20] Highest'
  19:40.82 < b'IIEN2 OK [READ-ONLY] Examine completed (0.000 secs).'
  19:40.82 > b'IIEN3 FETCH 1 (BODY.PEEK[HEADER] FLAGS)'
  19:40.86 < b'* 1 FETCH (FLAGS () BODY[HEADER] {3108}'
  19:40.86 read literal size 3108
  19:40.86 < b')'
  19:40.89 < b'IIEN3 OK Fetch completed.'
  19:40.89 > b'IIEN4 LOGOUT'
  19:40.93 < b'* BYE Logging out'
  19:40.93 BYE response: b'Logging out'
[(b'1 (FLAGS () BODY[HEADER] {3108}',
  b'Return-Path: <doug@doughellmann.com>\r\nReceived: from compute4.internal ('
  b'compute4.nyi.internal [10.202.2.44])\r\n\t by sloti26t01 (Cyrus 3.0.0-beta1'
  b'-git-fastmail-12410) with LMTPA;\r\n\t Sun, 06 Mar 2016 16:16:03 -0500\r'
  b'\nX-Sieve: CMU Sieve 2.4\r\nX-Spam-known-sender: yes, fadd1cf2-dc3a-4984-a0'
  b'8b-02cef3cf1221="doug",\r\n  ea349ad0-9299-47b5-b632-6ff1e394cc7d="both he'
  b'llfly"\r\nX-Spam-score: 0.0\r\nX-Spam-hits: ALL_TRUSTED -1,BAYES_00 -1.'
  b'9, LANGUAGES unknown, BAYES_USED global,\r\n  SA_VERSION 3.3.2\r\nX-Spam'
  b"-source: IP='127.0.0.1', Host='unk', Country='unk', FromHeader='com',\r\n "
  b" MailFrom='com'\r\nX-Spam-charsets: plain='us-ascii'\r\nX-Resolved-to: d"
  b'oughellmann@fastmail.fm\r\nX-Delivered-to: doug@doughellmann.com\r\nX-Ma'
  b'il-from: doug@doughellmann.com\r\nReceived: from mx5 ([10.202.2.204])\r'
  b'\n  by compute4.internal (LMTPProxy); Sun, 06 Mar 2016 16:16:03 -0500\r\nRe'
  b'ceived: from mx5.nyi.internal (localhost [127.0.0.1])\r\n\tby mx5.nyi.inter'
  b'nal (Postfix) with ESMTP id 47CBA280DB3\r\n\tfor <doug@doughellmann.com>; S'
  b'un,  6 Mar 2016 16:16:03 -0500 (EST)\r\nReceived: from mx5.nyi.internal (l'
  b'ocalhost [127.0.0.1])\r\n    by mx5.nyi.internal (Authentication Milter) w'
  b'ith ESMTP\r\n    id A717886846E.30BA4280D81;\r\n    Sun, 6 Mar 2016 16:1'
  b'6:03 -0500\r\nAuthentication-Results: mx5.nyi.internal;\r\n   dkim=pass'
  b' (1024-bit rsa key) header.d=messagingengine.com header.i=@messagingengi'
  b'ne.com header.b=Jrsm+pCo;\r\n    x-local-ip=pass\r\nReceived: from mailo'
  b'ut.nyi.internal (gateway1.nyi.internal [10.202.2.221])\r\n\t(using TLSv1.2 '
  b'with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\r\n\t(No client cer'
  b'tificate requested)\r\n\tby mx5.nyi.internal (Postfix) withESMTPS id 30BA4'
  b'280D81\r\n\tfor <doug@doughellmann.com>; Sun,  6 Mar 2016 16:16:03 -0500 (E'
  b'ST)\r\nReceived: from compute2.internal (compute2.nyi.internal [10.202.2.4'
  b'2])\r\n\tby mailout.nyi.internal (Postfix) with ESMTP id 1740420D0A\r\n\tf'
  b'or <doug@doughellmann.com>; Sun,  6 Mar 2016 16:16:03 -0500(EST)\r\nRecei'
  b'ved: from frontend2 ([10.202.2.161])\r\n  by compute2.internal (MEProxy); '
  b'Sun, 06 Mar 2016 16:16:03 -0500\r\nDKIM-Signature: v=1; a=rsa-sha1; c=rela'
  b'xed/relaxed; d=\r\n\tmessagingengine.com; h=content-transfer-encoding:conte'
  b'nt-type\r\n\t:date:from:message-id:mime-version:subject:to:x-sasl-enc\r\n'
  b'\t:x-sasl-enc; s=smtpout; bh=P98NTsEo015suwJ4gk71knAWLa4=; b=Jrsm+\r\n\t'
  b'pCovRIoQIRyp8Fl0L6JHOI8sbZy2obx7O28JF2iTlTWmX33Rhlq9403XRklwN3JA\r\n\t7KSPq'
  b'MTp30Qdx6yIUaADwQqlO+QMuQq/QxBHdjeebmdhgVfjhqxrzTbSMww/ZNhL\r\n\tYwv/QM/oDH'
  b'bXiLSUlB3Qrg+9wsE/0jU/EOisiU=\r\nX-Sasl-enc: 8ZJ+4ZRE8AGPzdLRWQFivGymJb8pa'
  b'4G9JGcb7k4xKn+I 1457298962\r\nReceived: from [192.168.1.14](75-137-1-34.d'
  b'hcp.nwnn.ga.charter.com [75.137.1.34])\r\n\tby mail.messagingengine.com (Po'
  b'stfix) with ESMTPA id C0B366801CD\r\n\tfor <doug@doughellmann.com>; Sun,  6'
  b' Mar 2016 16:16:02 -0500 (EST)\r\nFrom: Doug Hellmann <doug@doughellmann.c'
  b'om>\r\nContent-Type: text/plain; charset=us-ascii\r\nContent-Transfer-En'
  b'coding: 7bit\r\nSubject: PyMOTW Example message 2\r\nMessage-Id: <00ABCD'
  b'46-DADA-4912-A451-D27165BC3A2F@doughellmann.com>\r\nDate: Sun, 6 Mar 2016 '
  b'16:16:02 -0500\r\nTo: Doug Hellmann <doug@doughellmann.com>\r\nMime-Vers'
  b'ion: 1.0 (Mac OS X Mail 9.2 \\(3112\\))\r\nX-Mailer: Apple Mail (2.3112)'
  b'\r\n\r\n'), b')']
```
FETCH 命令的响应以标志开始，然后指示有595个字节的标头数据。客户端使用邮件响应构造一个元组，然后在fetch响应的尾部，使用服务器发送的包含右括号的单个字符串关闭序列。由于这种格式化，可能更容易单独获取不同的信息，或重新组合响应并在客户端中解析它。

```python
#imaplib_fetch_separately.py
import imaplib
import pprint
import imaplib_connect

with imaplib_connect.open_connection() as c:
    c.select('INBOX', readonly=True)

    print('HEADER:')
    typ, msg_data = c.fetch('1', '(BODY.PEEK[HEADER])')
    for response_part in msg_data:
        if isinstance(response_part, tuple):
            print(response_part[1])

    print('\nBODY TEXT:')
    typ, msg_data = c.fetch('1', '(BODY.PEEK[TEXT])')
    for response_part in msg_data:
        if isinstance(response_part, tuple):
            print(response_part[1])

    print('\nFLAGS:')
    typ, msg_data = c.fetch('1', '(FLAGS)')
    for response_part in msg_data:
        print(response_part)
        print(imaplib.ParseFlags(response_part))
```
单独获取值有一个额外的好处，就是可以很容易地使用ParseFlags（）来解析响应中的标志。

```
$ python3 imaplib_fetch_separately.py

HEADER:
b'Return-Path: <doug@doughellmann.com>\r\nReceived: from compute
4.internal (compute4.nyi.internal [10.202.2.44])\r\n\t by sloti2
6t01 (Cyrus 3.0.0-beta1-git-fastmail-12410) with LMTPA;\r\n\t Su
n, 06 Mar 2016 16:16:03 -0500\r\nX-Sieve: CMU Sieve 2.4\r\nX-Spa
m-known-sender: yes, fadd1cf2-dc3a-4984-a08b-02cef3cf1221="doug"
,\r\n  ea349ad0-9299-47b5-b632-6ff1e394cc7d="both hellfly"\r\nX-
Spam-score: 0.0\r\nX-Spam-hits: ALL_TRUSTED -1, BAYES_00 -1.9, L
ANGUAGES unknown, BAYES_USED global,\r\n  SA_VERSION 3.3.2\r\nX-
Spam-source: IP=\'127.0.0.1\', Host=\'unk\', Country=\'unk\', Fr
omHeader=\'com\',\r\n  MailFrom=\'com\'\r\nX-Spam-charsets: plai
n=\'us-ascii\'\r\nX-Resolved-to: doughellmann@fastmail.fm\r\nX-D
elivered-to: doug@doughellmann.com\r\nX-Mail-from: doug@doughell
mann.com\r\nReceived: from mx5 ([10.202.2.204])\r\n  by compute4
.internal (LMTPProxy); Sun, 06 Mar 2016 16:16:03 -0500\r\nReceiv
ed: from mx5.nyi.internal (localhost [127.0.0.1])\r\n\tby mx5.ny
i.internal (Postfix) with ESMTP id 47CBA280DB3\r\n\tfor <doug@do
ughellmann.com>; Sun,  6 Mar 2016 16:16:03 -0500 (EST)\r\nReceiv
ed: from mx5.nyi.internal (localhost [127.0.0.1])\r\n    by mx5.
nyi.internal (Authentication Milter) with ESMTP\r\n    id A71788
6846E.30BA4280D81;\r\n    Sun, 6 Mar 2016 16:16:03 -0500\r\nAuth
entication-Results: mx5.nyi.internal;\r\n    dkim=pass (1024-bit
 rsa key) header.d=messagingengine.com header.i=@messagingengine
.com header.b=Jrsm+pCo;\r\n    x-local-ip=pass\r\nReceived: from
 mailout.nyi.internal (gateway1.nyi.internal [10.202.2.221])\r\n
\t(using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/25
6 bits))\r\n\t(No client certificate requested)\r\n\tby mx5.nyi.
internal (Postfix) with ESMTPS id 30BA4280D81\r\n\tfor <doug@dou
ghellmann.com>; Sun,  6 Mar 2016 16:16:03 -0500 (EST)\r\nReceive
d: from compute2.internal (compute2.nyi.internal [10.202.2.42])\
r\n\tby mailout.nyi.internal (Postfix) with ESMTP id 1740420D0A\
r\n\tfor <doug@doughellmann.com>; Sun,  6 Mar 2016 16:16:03 -050
0 (EST)\r\nReceived: from frontend2 ([10.202.2.161])\r\n  by com
pute2.internal (MEProxy); Sun, 06 Mar 2016 16:16:03 -0500\r\nDKI
M-Signature: v=1; a=rsa-sha1; c=relaxed/relaxed; d=\r\n\tmessagi
ngengine.com; h=content-transfer-encoding:content-type\r\n\t:dat
e:from:message-id:mime-version:subject:to:x-sasl-enc\r\n\t:x-sas
l-enc; s=smtpout; bh=P98NTsEo015suwJ4gk71knAWLa4=; b=Jrsm+\r\n\t
pCovRIoQIRyp8Fl0L6JHOI8sbZy2obx7O28JF2iTlTWmX33Rhlq9403XRklwN3JA
\r\n\t7KSPqMTp30Qdx6yIUaADwQqlO+QMuQq/QxBHdjeebmdhgVfjhqxrzTbSMw
w/ZNhL\r\n\tYwv/QM/oDHbXiLSUlB3Qrg+9wsE/0jU/EOisiU=\r\nX-Sasl-en
c: 8ZJ+4ZRE8AGPzdLRWQFivGymJb8pa4G9JGcb7k4xKn+I 1457298962\r\nRe
ceived: from [192.168.1.14] (75-137-1-34.dhcp.nwnn.ga.charter.co
m [75.137.1.34])\r\n\tby mail.messagingengine.com (Postfix) with
 ESMTPA id C0B366801CD\r\n\tfor <doug@doughellmann.com>; Sun,  6
 Mar 2016 16:16:02 -0500 (EST)\r\nFrom: Doug Hellmann <doug@doug
hellmann.com>\r\nContent-Type: text/plain; charset=us-ascii\r\nC
ontent-Transfer-Encoding: 7bit\r\nSubject: PyMOTW Example messag
e 2\r\nMessage-Id: <00ABCD46-DADA-4912-A451-D27165BC3A2F@doughel
lmann.com>\r\nDate: Sun, 6 Mar 2016 16:16:02 -0500\r\nTo: Doug H
ellmann <doug@doughellmann.com>\r\nMime-Version: 1.0 (Mac OS X M
ail 9.2 \\(3112\\))\r\nX-Mailer: Apple Mail (2.3112)\r\n\r\n'

BODY TEXT:
b'This is the second example message.\r\n'

FLAGS:
b'1 (FLAGS ())'
()
```

### Whole Messages

如前所述，客户端可以分别向服务器询问邮件的各个部分。还可以将整个邮件检索为RFC 822格式的邮件消息，并使用email模块中的类对其进行解析。

```python
#imaplib_fetch_rfc822.py
import imaplib
import email
import email.parser
import imaplib_connect

with imaplib_connect.open_connection() as c:
    c.select('INBOX', readonly=True)

    typ, msg_data = c.fetch('1', '(RFC822)')
    for response_part in msg_data:
        if isinstance(response_part, tuple):
            email_parser = email.parser.BytesFeedParser()
            email_parser.feed(response_part[1])
            msg = email_parser.close()
            for header in ['subject', 'to', 'from']:
                print('{:^8}: {}'.format(
                    header.upper(), msg[header]))
```
email模块中的解析器使访问和操作消息变得非常容易。 此示例仅打印每条消息的一些标头。

```
$ python3 imaplib_fetch_rfc822.py

SUBJECT : PyMOTW Example message 2
   TO   : Doug Hellmann <doug@doughellmann.com>
  FROM  : Doug Hellmann <doug@doughellmann.com>
```

### Uploading Messages

要向邮箱添加新邮件，请构造Message实例并将其传递给append（）方法以及传递邮件的时间戳。

```python
#imaplib_append.py
import imaplib
import time
import email.message
import imaplib_connect

new_message = email.message.Message()
new_message.set_unixfrom('pymotw')
new_message['Subject'] = 'subject goes here'
new_message['From'] = 'pymotw@example.com'
new_message['To'] = 'example@example.com'
new_message.set_payload('This is the body of the message.\n')

print(new_message)

with imaplib_connect.open_connection() as c:
    c.append('INBOX', '', 
    	imaplib.Time2Internaldate(time.time()), 
    	str(new_message).encode('utf-8'))

    # Show the headers for all messages in the mailbox
    c.select('INBOX')
    typ, [msg_ids] = c.search(None, 'ALL')
    for num in msg_ids.split():
        typ, msg_data = c.fetch(num, '(BODY.PEEK[HEADER])')
        for response_part in msg_data:
            if isinstance(response_part, tuple):
                print('\n{}:'.format(num))
                print(response_part[1])
```
此示例中使用的payload是一个简单的纯文本电子邮件正文。Message还支持MIME编码的多部分（multi-part）消息。

```
$ python3 imaplib_append.py

Subject: subject goes here
From: pymotw@example.com
To: example@example.com

This is the body of the message.

b'1':
b'Return-Path: <doug@doughellmann.com>\r\nReceived: from compute
4.internal (compute4.nyi.internal [10.202.2.44])\r\n\t by sloti2
6t01 (Cyrus 3.0.0-beta1-git-fastmail-12410) with LMTPA;\r\n\t Su
n, 06 Mar 2016 16:16:03 -0500\r\nX-Sieve: CMU Sieve 2.4\r\nX-Spa
m-known-sender: yes, fadd1cf2-dc3a-4984-a08b-02cef3cf1221="doug"
,\r\n  ea349ad0-9299-47b5-b632-6ff1e394cc7d="both hellfly"\r\nX-
Spam-score: 0.0\r\nX-Spam-hits: ALL_TRUSTED -1, BAYES_00 -1.9, L
ANGUAGES unknown, BAYES_USED global,\r\n  SA_VERSION 3.3.2\r\nX-
Spam-source: IP=\'127.0.0.1\', Host=\'unk\', Country=\'unk\', Fr
omHeader=\'com\',\r\n  MailFrom=\'com\'\r\nX-Spam-charsets: plai
n=\'us-ascii\'\r\nX-Resolved-to: doughellmann@fastmail.fm\r\nX-D
elivered-to: doug@doughellmann.com\r\nX-Mail-from: doug@doughell
mann.com\r\nReceived: from mx5 ([10.202.2.204])\r\n  by compute4
.internal (LMTPProxy); Sun, 06 Mar 2016 16:16:03 -0500\r\nReceiv
ed: from mx5.nyi.internal (localhost [127.0.0.1])\r\n\tby mx5.ny
i.internal (Postfix) with ESMTP id 47CBA280DB3\r\n\tfor <doug@do
ughellmann.com>; Sun,  6 Mar 2016 16:16:03 -0500 (EST)\r\nReceiv
ed: from mx5.nyi.internal (localhost [127.0.0.1])\r\n    by mx5.
nyi.internal (Authentication Milter) with ESMTP\r\n    id A71788
6846E.30BA4280D81;\r\n    Sun, 6 Mar 2016 16:16:03 -0500\r\nAuth
entication-Results: mx5.nyi.internal;\r\n    dkim=pass (1024-bit
 rsa key) header.d=messagingengine.com header.i=@messagingengine
.com header.b=Jrsm+pCo;\r\n    x-local-ip=pass\r\nReceived: from
 mailout.nyi.internal (gateway1.nyi.internal [10.202.2.221])\r\n
\t(using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/25
6 bits))\r\n\t(No client certificate requested)\r\n\tby mx5.nyi.
internal (Postfix) with ESMTPS id 30BA4280D81\r\n\tfor <doug@dou
ghellmann.com>; Sun,  6 Mar 2016 16:16:03 -0500 (EST)\r\nReceive
d: from compute2.internal (compute2.nyi.internal [10.202.2.42])\
r\n\tby mailout.nyi.internal (Postfix) with ESMTP id 1740420D0A\
r\n\tfor <doug@doughellmann.com>; Sun,  6 Mar 2016 16:16:03 -050
0 (EST)\r\nReceived: from frontend2 ([10.202.2.161])\r\n  by com
pute2.internal (MEProxy); Sun, 06 Mar 2016 16:16:03 -0500\r\nDKI
M-Signature: v=1; a=rsa-sha1; c=relaxed/relaxed; d=\r\n\tmessagi
ngengine.com; h=content-transfer-encoding:content-type\r\n\t:dat
e:from:message-id:mime-version:subject:to:x-sasl-enc\r\n\t:x-sas
l-enc; s=smtpout; bh=P98NTsEo015suwJ4gk71knAWLa4=; b=Jrsm+\r\n\t
pCovRIoQIRyp8Fl0L6JHOI8sbZy2obx7O28JF2iTlTWmX33Rhlq9403XRklwN3JA
\r\n\t7KSPqMTp30Qdx6yIUaADwQqlO+QMuQq/QxBHdjeebmdhgVfjhqxrzTbSMw
w/ZNhL\r\n\tYwv/QM/oDHbXiLSUlB3Qrg+9wsE/0jU/EOisiU=\r\nX-Sasl-en
c: 8ZJ+4ZRE8AGPzdLRWQFivGymJb8pa4G9JGcb7k4xKn+I 1457298962\r\nRe
ceived: from [192.168.1.14] (75-137-1-34.dhcp.nwnn.ga.charter.co
m [75.137.1.34])\r\n\tby mail.messagingengine.com (Postfix) with
 ESMTPA id C0B366801CD\r\n\tfor <doug@doughellmann.com>; Sun,  6
 Mar 2016 16:16:02 -0500 (EST)\r\nFrom: Doug Hellmann <doug@doug
hellmann.com>\r\nContent-Type: text/plain; charset=us-ascii\r\nC
ontent-Transfer-Encoding: 7bit\r\nSubject: PyMOTW Example messag
e 2\r\nMessage-Id: <00ABCD46-DADA-4912-A451-D27165BC3A2F@doughel
lmann.com>\r\nDate: Sun, 6 Mar 2016 16:16:02 -0500\r\nTo: Doug H
ellmann <doug@doughellmann.com>\r\nMime-Version: 1.0 (Mac OS X M
ail 9.2 \\(3112\\))\r\nX-Mailer: Apple Mail (2.3112)\r\n\r\n'

b'2':
b'Subject: subject goes here\r\nFrom: pymotw@example.com\r\nTo:
example@example.com\r\n\r\n'
```

### Moving and Copying Messages

一旦消息在服务器上，就可以使用move（）或copy（）移动或复制它，而无需下载它。这些方法对消息id范围进行操作，就像fetch（）一样。

```python
#imaplib_archive_read.py
import imaplib
import imaplib_connect

with imaplib_connect.open_connection() as c:
    # Find the "SEEN" messages in INBOX
    c.select('INBOX')
    typ, [response] = c.search(None, 'SEEN')
    if typ != 'OK':
        raise RuntimeError(response)
    msg_ids = ','.join(response.decode('utf-8').split(' '))

    # Create a new mailbox, "Example.Today"
    typ, create_response = c.create('Example.Today')
    print('CREATED Example.Today:', create_response)

    # Copy the messages
    print('COPYING:', msg_ids)
    c.copy(msg_ids, 'Example.Today')

    # Look at the results
    c.select('Example.Today')
    typ, [response] = c.search(None, 'ALL')
    print('COPIED:', response)
```
此示例脚本在Example下创建一个新邮箱，并将INBOX中已读的邮件复制到其中。

```
$ python3 imaplib_archive_read.py

CREATED Example.Today: [b'Completed']
COPYING: 2
COPIED: b'1'
```
再次运行相同的脚本以说明检查返回码的重要性。 调用create（）以使新邮箱报告邮箱已存在，而不是引发异常。

```
$ python3 imaplib_archive_read.py

CREATED Example.Today: [b'[ALREADYEXISTS] Mailbox already exists']
COPYING: 2
COPIED: b'1 2'
```

### Deleting Messages

尽管许多现代邮件客户端使用“垃圾文件夹”模型来处理已删除的邮件，但邮件通常不会移动到实际文件夹中。相反，他们的标志被更新以添加\Deleted。
“清空”垃圾箱的操作是通过EXPUNGE命令实现的。
此示例脚本查找在主题中带有“Lorem ipsum”的已归档邮件，设置已删除标志，然后通过再次查询服务器显示邮件仍然存在于该文件夹中。

```python
#imaplib_delete_messages.py
import imaplib
import imaplib_connect
from imaplib_list_parse import parse_list_response

with imaplib_connect.open_connection() as c:
    c.select('Example.Today')

    # What ids are in the mailbox?
    typ, [msg_ids] = c.search(None, 'ALL')
    print('Starting messages:', msg_ids)

    # Find the message(s)
    typ, [msg_ids] = c.search(None,'(SUBJECT "subject goes here")',)
    msg_ids = ','.join(msg_ids.decode('utf-8').split(' '))
    print('Matching messages:', msg_ids)

    # What are the current flags?
    typ, response = c.fetch(msg_ids, '(FLAGS)')
    print('Flags before:', response)

    # Change the Deleted flag
    typ, response = c.store(msg_ids, '+FLAGS', r'(\Deleted)')

    # What are the flags now?
    typ, response = c.fetch(msg_ids, '(FLAGS)')
    print('Flags after:', response)

    # Really delete the message.
    typ, response = c.expunge()
    print('Expunged:', response)

    # What ids are left in the mailbox?
    typ, [msg_ids] = c.search(None, 'ALL')
    print('Remaining messages:', msg_ids)
```
显式调用expunge（）会删除邮件，但调用close（）会产生相同的效果。不同之处在于调用close（）时客户端不会收到有关删除的通知。

```
$ python3 imaplib_delete_messages.py

Response code: OK
Server response: b'(\\HasChildren) "." Example'
Parsed response: ('\\HasChildren', '.', 'Example')
Server response: b'(\\HasNoChildren) "." Example.Today'
Parsed response: ('\\HasNoChildren', '.', 'Example.Today')
Server response: b'(\\HasNoChildren) "." Example.2016'
Parsed response: ('\\HasNoChildren', '.', 'Example.2016')
Server response: b'(\\HasNoChildren) "." Archive'
Parsed response: ('\\HasNoChildren', '.', 'Archive')
Server response: b'(\\HasNoChildren) "." "Deleted Messages"'
Parsed response: ('\\HasNoChildren', '.', 'Deleted Messages')
Server response: b'(\\HasNoChildren) "." INBOX'
Parsed response: ('\\HasNoChildren', '.', 'INBOX')
Starting messages: b'1 2'
Matching messages: 1,2
Flags before: [b'1 (FLAGS (\\Seen))', b'2 (FLAGS (\\Seen))']
Flags after: [b'1 (FLAGS (\\Deleted \\Seen))', b'2 (FLAGS (\\Del
eted \\Seen))']
Expunged: [b'2', b'1']
Remaining messages: b''
```
