---
title: PyMOTW-3 --- The Internet
categories: language
tags: PyMOTW-3
---

互联网是现代计算的普遍方面。即使是小型的一次性脚本也经常与远程服务交互以发送或接收数据。Pythond用于处理Web协议的丰富工具集使其非常适合编写基于Web的应用程序，无论是作为客户端还是服务器。

urllib.parse模块操纵URL字符串，拆分和组合它们的组件，在客户端和服务器中都很有用。

urllib.request模块实现了一个用于检索远程内容的API。

HTTP POST请求通常使用urllib进行“表单编码”。通过POST发送的二进制数据应首先使用base64进行编码，以符合消息格式标准。

作为蜘蛛或爬虫访问许多站点的行为良好的客户端应该使用urllib.robotparser来确保他们在将重负载放在远程服务器上之前具有权限。

要使用Python创建自定义Web服务器，而不需要任何外部框架，请使用http.server作为起点。它处理HTTP协议，因此唯一需要的自定义是用于响应传入请求的应用程序代码。

服务器中的会话状态可以通过http.cookies模块创建和解析的cookie进行管理。完全支持到期，路径，域和其他cookie设置，可以轻松配置会话。

uuid模块用于为需要唯一值的资源生成标识符。UUID适用于自动生成统一资源名称（URN）值，其中资源的名称必须是唯一的，但不需要传达任何含义。

Python的标准库包括对两个基于Web的远程过程调用机制的支持。 AJAX通信和REST API中使用的JavaScript Object Notation（JSON）编码方案在json中实现。它在客户端或服务器中同样有效。完整的XML-RPC客户端和服务器库也分别包含在xmlrpc.client和xmlrpc.server中。


## urllib.parse — Split URLs into Components

目的：将URL拆分为组件  
urllib.parse模块提供了操作URL及其组成部分的功能，可以将其分解或构建它们。

### Parsing

urlparse（）函数的返回值是一个ParseResult对象，其行为类似于具有六个元素的元组。

```python
#urllib_parse_urlparse.py
from urllib.parse import urlparse

url = 'http://netloc/path;param?query=arg#frag'
parsed = urlparse(url)
print(parsed)
```
通过元组接口可用的URL部分是：方案（scheme），网络位置（network location），路径（path），路径段参数（path segment parameters）（通过分号与路径分开），查询（query）和片段（fragment）。

```
$ python3 urllib_parse_urlparse.py

ParseResult(scheme='http', netloc='netloc', path='/path',
params='param', query='query=arg', fragment='frag')
```
尽管返回值的行为类似于元组，但它实际上基于一个namedtuple，它是tuple的子类，支持通过命名属性和索引访问URL的各个部分。 除了更容易为程序员使用外，属性API还提供对tuple API中不可用的多个值的访问。

```python
#urllib_parse_urlparseattrs.py
from urllib.parse import urlparse

url = 'http://user:pwd@NetLoc:80/path;param?query=arg#frag'
parsed = urlparse(url)
print('scheme  :', parsed.scheme)
print('netloc  :', parsed.netloc)
print('path    :', parsed.path)
print('params  :', parsed.params)
print('query   :', parsed.query)
print('fragment:', parsed.fragment)
print('username:', parsed.username)
print('password:', parsed.password)
print('hostname:', parsed.hostname)
print('port    :', parsed.port)
```
如果输入URL中存在用户名和密码，则username和password可用，否则设置为“None”。 hostname与netloc的值相同，全部为小写，并且端口值已剥离。 如果端口在存在则转换为整数，否则为None。

```
$ python3 urllib_parse_urlparseattrs.py

scheme  : http
netloc  : user:pwd@NetLoc:80
path    : /path
params  : param
query   : query=arg
fragment: frag
username: user
password: pwd
hostname: netloc
port    : 80
```
urlsplit（）函数是urlparse（）的替代方法。 它的行为略有不同，因为它不会从URL中拆分参数。 这对于遵循RFC 2396的URL很有用，它支持每路径段参数。

```python
#urllib_parse_urlsplit.py
from urllib.parse import urlsplit

url = 'http://user:pwd@NetLoc:80/p1;para/p2;para?query=arg#frag'
parsed = urlsplit(url)
print(parsed)
print('scheme  :', parsed.scheme)
print('netloc  :', parsed.netloc)
print('path    :', parsed.path)
print('query   :', parsed.query)
print('fragment:', parsed.fragment)
print('username:', parsed.username)
print('password:', parsed.password)
print('hostname:', parsed.hostname)
print('port    :', parsed.port)
```
由于参数未拆分，元组API将显示五个元素而不是六个元素，并且没有params属性。

```
$ python3 urllib_parse_urlsplit.py

SplitResult(scheme='http', netloc='user:pwd@NetLoc:80',
path='/p1;para/p2;para', query='query=arg', fragment='frag')
scheme  : http
netloc  : user:pwd@NetLoc:80
path    : /p1;para/p2;para
query   : query=arg
fragment: frag
username: user
password: pwd
hostname: netloc
port    : 80
```
要简单地从URL中剥离fragment标识符，例如从URL中查找基页（base page）名称时，请使用urldefrag（）。

```python
#urllib_parse_urldefrag.py
from urllib.parse import urldefrag

original = 'http://netloc/path;param?query=arg#frag'
print('original:', original)
d = urldefrag(original)
print('url     :', d.url)
print('fragment:', d.fragment)
```
返回值是一个DefragResult，基于namedtuple，包含基本URL和fragment。

```
$ python3 urllib_parse_urldefrag.py

original: http://netloc/path;param?query=arg#frag
url     : http://netloc/path;param?query=arg
fragment: frag
```

### Unparsing

有几种方法可以将拆分URL的各个部分组合在一起形成一个字符串。 解析的URL对象具有geturl（）方法。

```python
#urllib_parse_geturl.py
from urllib.parse import urlparse

original = 'http://netloc/path;param?query=arg#frag'
print('ORIG  :', original)
parsed = urlparse(original)
print('PARSED:', parsed.geturl())
```
geturl（）仅适用于urlparse（）或urlsplit（）返回的对象。

```
$ python3 urllib_parse_geturl.py

ORIG  : http://netloc/path;param?query=arg#frag
PARSED: http://netloc/path;param?query=arg#frag
```
包含字符串的常规元组可以使用urlunparse（）组合成一个URL。

```python
#urllib_parse_urlunparse.py
from urllib.parse import urlparse, urlunparse

original = 'http://netloc/path;param?query=arg#frag'
print('ORIG  :', original)
parsed = urlparse(original)
print('PARSED:', type(parsed), parsed)
t = parsed[:]
print('TUPLE :', type(t), t)
print('NEW   :', urlunparse(t))
```
虽然urlparse（）返回的ParseResult可以用作元组，但是这个示例显式创建了一个新元组，以显示urlunparse（）也可以使用普通元组。

```
$ python3 urllib_parse_urlunparse.py

ORIG  : http://netloc/path;param?query=arg#frag
PARSED: <class 'urllib.parse.ParseResult'> ParseResult(scheme='http', netloc='netloc', path='/path',
params='param', query='query=arg', fragment='frag')
TUPLE : <class 'tuple'> ('http', 'netloc', '/path', 'param', 'query=arg', 'frag')
NEW   : http://netloc/path;param?query=arg#frag
```
如果输入的URL包含多余的部分，则可以从重建的URL中删除这些部分。

```python
#urllib_parse_urlunparseextra.py
from urllib.parse import urlparse, urlunparse

original = 'http://netloc/path;?#'
print('ORIG  :', original)
parsed = urlparse(original)
print('PARSED:', type(parsed), parsed)
t = parsed[:]
print('TUPLE :', type(t), t)
print('NEW   :', urlunparse(t))
```
在这个例子中，原始URL中缺少parameters，query 和 fragment。 新URL与原始URL看起来不一样，但根据标准是等效的。

```
$ python3 urllib_parse_urlunparseextra.py

ORIG  : http://netloc/path;?#
PARSED: <class 'urllib.parse.ParseResult'>
ParseResult(scheme='http', netloc='netloc', path='/path',
params='', query='', fragment='')
TUPLE : <class 'tuple'> ('http', 'netloc', '/path', '', '', '')
NEW   : http://netloc/path
```

### Joining

除了解析URL之外，urlparse还包括urljoin（），用于从相对片段(fragment)构造绝对(absolute) URL。

```python
#urllib_parse_urljoin.py
from urllib.parse import urljoin

print(urljoin('http://www.example.com/path/file.html', 'anotherfile.html'))
print(urljoin('http://www.example.com/path/file.html', '../anotherfile.html'))
```
在该示例中，在计算第二URL时考虑路径的相对部分（“../”）。

```
$ python3 urllib_parse_urljoin.py

http://www.example.com/path/anotherfile.html
http://www.example.com/anotherfile.html
```
非相对路径的处理方式与os.path.join（）相同。

```python
#urllib_parse_urljoin_with_path.py
from urllib.parse import urljoin

print(urljoin('http://www.example.com/path/', '/subpath/file.html'))
print(urljoin('http://www.example.com/path/', 'subpath/file.html'))
```
如果加入URL的路径以斜杠（/）开头，则会将URL的路径重置为顶级。 如果它不是以斜杠开头，则会将其附加到URL的路径末尾。

```
$ python3 urllib_parse_urljoin_with_path.py

http://www.example.com/subpath/file.html
http://www.example.com/path/subpath/file.html
```

### Encoding Query Arguments

在将参数添加到URL之前，需要对它们进行编码。

```python
#urllib_parse_urlencode.py
from urllib.parse import urlencode

query_args = {
    'q': 'query string',
    'foo': 'bar',
}
encoded_args = urlencode(query_args)
print('Encoded:', encoded_args)
```
编码替换空格等特殊字符，以确保它们使用符合标准的格式传递给服务器。

```
$ python3 urllib_parse_urlencode.py

Encoded: q=query+string&foo=bar
```
要在查询字符串中使用单独出现的变量传递一系列值，请在调用urlencode（）时将doseq设置为True。

```python
#urllib_parse_urlencode_doseq.py
from urllib.parse import urlencode

query_args = {
    'foo': ['foo1', 'foo2'],
}
print('Single  :', urlencode(query_args))
print('Sequence:', urlencode(query_args, doseq=True))
```
结果是一个查询字符串，其中包含多个与同一名称关联的值。

```
$ python3 urllib_parse_urlencode_doseq.py

Single  : foo=%5B%27foo1%27%2C+%27foo2%27%5D
Sequence: foo=foo1&foo=foo2
```
要解码查询字符串，请使用`parse_qs（）`或`parse_qsl（）`。

```python
#urllib_parse_parse_qs.py
from urllib.parse import parse_qs, parse_qsl

encoded = 'foo=foo1&foo=foo2'

print('parse_qs :', parse_qs(encoded))
print('parse_qsl:', parse_qsl(encoded))
```
`parse_qs（）`的返回值是将名称映射到值的字典，而`parse_qsl（）`返回包含名称和值的元组列表。

```
$ python3 urllib_parse_parse_qs.py

parse_qs : {'foo': ['foo1', 'foo2']}
parse_qsl: [('foo', 'foo1'), ('foo', 'foo2')]
```
查询参数中“可能导致服务器端URL解析问题”的特殊字符在传递给urlencode（）时会被“引用”。 要在本地引用它们以生成字符串的安全版本，请直接使用quote（）或quote_plus（）函数。

```python
#urllib_parse_quote.py
from urllib.parse import quote, quote_plus, urlencode

url = 'http://localhost:8080/~hellmann/'
print('urlencode() :', urlencode({'url': url}))
print('quote()     :', quote(url))
print('quote_plus():', quote_plus(url))
```
quote_plus（）中的引用实现对它所替换的字符更积极。

```
$ python3 urllib_parse_quote.py

urlencode() : url=http%3A%2F%2Flocalhost%3A8080%2F%7Ehellmann%2F
quote()     : http%3A//localhost%3A8080/%7Ehellmann/
quote_plus(): http%3A%2F%2Flocalhost%3A8080%2F%7Ehellmann%2F
```
要反转引用操作，请根据需要使用unquote（）或unquote_plus（）。

```python
#urllib_parse_unquote.py
from urllib.parse import unquote, unquote_plus

print(unquote('http%3A//localhost%3A8080/%7Ehellmann/'))
print(unquote_plus('http%3A%2F%2Flocalhost%3A8080%2F%7Ehellmann%2F'))
```
编码值将转换回普通字符串URL。

```
$ python3 urllib_parse_unquote.py

http://localhost:8080/~hellmann/
http://localhost:8080/~hellmann/
```

## urllib.request — Network Resource Access

目的：用于打开（opening）URL的库，可以通过定义自定义协议处理程序来扩展URL。  
urllib.request模块提供了一个API，用于使用URL标识的Internet资源。它旨在通过各个应用程序进行扩展，以支持新协议或为现有协议添加变体（例如处理HTTP基本身份验证）。

### HTTP GET

>注意  
这些示例的测试服务器位于http.server_GET.py中，来自http.server模块的示例。 在一个终端窗口中启动服务器，然后在另一个窗口中运行这些示例。

HTTP GET操作是urllib.request的最简单用法。 将URL传递给urlopen（）以获取远程数据的“类文件”句柄。

```python
#urllib_request_urlopen.py
from urllib import request

response = request.urlopen('http://localhost:8080/')
print('RESPONSE:', response)
print('URL     :', response.geturl())

headers = response.info()
print('DATE    :', headers['date'])
print('HEADERS :')
print('---------')
print(headers)

data = response.read().decode('utf-8')
print('LENGTH  :', len(data))
print('DATA    :')
print('---------')
print(data)
```
示例服务器接受传入的值并格式化为一个纯文本响应以发回。 urlopen（）的返回值允许通过info（）方法访问HTTP服务器的头，以及通过·read（）和readlines（）等方法访问远程资源的数据。

```
$ python3 urllib_request_urlopen.py

RESPONSE: <http.client.HTTPResponse object at 0x101744d68>
URL     : http://localhost:8080/
DATE    : Sat, 08 Oct 2016 18:08:54 GMT
HEADERS :
---------
Server: BaseHTTP/0.6 Python/3.5.2
Date: Sat, 08 Oct 2016 18:08:54 GMT
Content-Type: text/plain; charset=utf-8


LENGTH  : 349
DATA    :
---------
CLIENT VALUES:
client_address=('127.0.0.1', 58420) (127.0.0.1)
command=GET
path=/
real path=/
query=
request_version=HTTP/1.1

SERVER VALUES:
server_version=BaseHTTP/0.6
sys_version=Python/3.5.2
protocol_version=HTTP/1.0

HEADERS RECEIVED:
Accept-Encoding=identity
Connection=close
Host=localhost:8080
User-Agent=Python-urllib/3.5
```
urlopen（）返回的类文件对象是可迭代的：

```python
#urllib_request_urlopen_iterator.py
from urllib import request

response = request.urlopen('http://localhost:8080/')
for line in response:
    print(line.decode('utf-8').rstrip())
```
此示例在打印输出之前删除尾随换行符和回车符。

```
$ python3 urllib_request_urlopen_iterator.py

CLIENT VALUES:
client_address=('127.0.0.1', 58444) (127.0.0.1)
command=GET
path=/
real path=/
query=
request_version=HTTP/1.1

SERVER VALUES:
server_version=BaseHTTP/0.6
sys_version=Python/3.5.2
protocol_version=HTTP/1.0

HEADERS RECEIVED:
Accept-Encoding=identity
Connection=close
Host=localhost:8080
User-Agent=Python-urllib/3.5
```

### Encoding Arguments

可以通过使用urllib.parse.urlencode（）对参数进行编码并将它们附加到URL来将参数传递给服务器。

```python
#urllib_request_http_get_args.py
from urllib import parse
from urllib import request

query_args = {'q': 'query string', 'foo': 'bar'}
encoded_args = parse.urlencode(query_args)
print('Encoded:', encoded_args)

url = 'http://localhost:8080/?' + encoded_args
print(request.urlopen(url).read().decode('utf-8'))
```
示例输出中返回的CLIENT VALUES列表包含已编码的查询参数。

```
$ python urllib_request_http_get_args.py
Encoded: q=query+string&foo=bar
CLIENT VALUES:
client_address=('127.0.0.1', 58455) (127.0.0.1)
command=GET
path=/?q=query+string&foo=bar
real path=/
query=q=query+string&foo=bar
request_version=HTTP/1.1

SERVER VALUES:
server_version=BaseHTTP/0.6
sys_version=Python/3.5.2
protocol_version=HTTP/1.0

HEADERS RECEIVED:
Accept-Encoding=identity
Connection=close
Host=localhost:8080
User-Agent=Python-urllib/3.5
```

### HTTP POST

>注意  
这些示例的测试服务器位于http.server_POST.py中，来自http.server模块的示例。 在一个终端窗口中启动服务器，然后在另一个窗口中运行这些示例。

要将表单编码数据发送到远程服务器使用POST而不是GET，请将已编码的查询参数作为数据传递给urlopen（）。

```python
#urllib_request_urlopen_post.py
from urllib import parse
from urllib import request

query_args = {'q': 'query string', 'foo': 'bar'}
encoded_args = parse.urlencode(query_args).encode('utf-8')
url = 'http://localhost:8080/'
print(request.urlopen(url, encoded_args).read().decode('utf-8'))
```
服务器可以解码表单数据并按名称访问各个值。

```
$ python3 urllib_request_urlopen_post.py

Client: ('127.0.0.1', 58568)
User-agent: Python-urllib/3.5
Path: /
Form data:
    foo=bar
    q=query string
```

### Adding Outgoing Headers

urlopen（）是一个便利函数，它隐藏了如何生成和处理请求的一些细节。 通过直接使用Request实例，可以实现更精确的控制。 例如，可以将自定义headers添加到传出请求以控制返回数据的格式，指定本地缓存的文档的版本，并告知远程服务器与其通信的软件客户端的名称。

正如前面示例中的输出所示，默认的User-agent header值由常量Python-urllib组成，后跟Python解释器版本。 在创建将访问其他人拥有的Web资源的应用程序时，在请求中包含真实用户代理信息是有礼貌的，因此他们可以更轻松地识别命中源。 使用自定义代理还允许他们使用robots.txt文件控制抓取工具（请参阅http.robotparser模块）。

```python
#urllib_request_request_header.py
from urllib import request

r = request.Request('http://localhost:8080/')
r.add_header(
    'User-agent',
    'PyMOTW (https://pymotw.com/)',
)

response = request.urlopen(r)
data = response.read().decode('utf-8')
print(data)
```
创建Request对象后，使用add_header（）在打开请求之前设置用户代理值。 输出的最后一行显示自定义值。

```
$ python3 urllib_request_request_header.py

CLIENT VALUES:
client_address=('127.0.0.1', 58585) (127.0.0.1)
command=GET
path=/
real path=/
query=
request_version=HTTP/1.1

SERVER VALUES:
server_version=BaseHTTP/0.6
sys_version=Python/3.5.2
protocol_version=HTTP/1.0

HEADERS RECEIVED:
Accept-Encoding=identity
Connection=close
Host=localhost:8080
User-Agent=PyMOTW (https://pymotw.com/)
```

### Posting Form Data from a Request

在构建Request以将其提交（post）到服务器时，可以指定传出数据。

```python
#urllib_request_request_post.py
from urllib import parse
from urllib import request

query_args = {'q': 'query string', 'foo': 'bar'}

r = request.Request(
    url='http://localhost:8080/',
    data=parse.urlencode(query_args).encode('utf-8'),
)
print('Request method :', r.get_method())
r.add_header(
    'User-agent',
    'PyMOTW (https://pymotw.com/)',
)

print()
print('OUTGOING DATA:')
print(r.data)

print()
print('SERVER RESPONSE:')
print(request.urlopen(r).read().decode('utf-8'))
```
添加数据后，请求使用的HTTP方法会自动从GET更改为POST。

```
$ python3 urllib_request_request_post.py

Request method : POST

OUTGOING DATA:
b'q=query+string&foo=bar'

SERVER RESPONSE:
Client: ('127.0.0.1', 58613)
User-agent: PyMOTW (https://pymotw.com/)
Path: /
Form data:
    foo=bar
    q=query string
```

### Uploading Files

编码上传文件需要比简单表单更多的工作。需要在请求正文（body）中构造完整的MIME消息，以便服务器可以区分传入的表单字段和上传的文件。

```python
#urllib_request_upload_files.py
import io
import mimetypes
from urllib import request
import uuid

class MultiPartForm:
    """Accumulate the data to be used when posting a form."""

    def __init__(self):
        self.form_fields = []
        self.files = []
        # Use a large random byte string to separate
        # parts of the MIME data.
        self.boundary = uuid.uuid4().hex.encode('utf-8')
        return

    def get_content_type(self):
        return 'multipart/form-data; boundary={}'.format(
            self.boundary.decode('utf-8'))

    def add_field(self, name, value):
        """Add a simple field to the form data."""
        self.form_fields.append((name, value))

    def add_file(self, fieldname, filename, fileHandle,
                 mimetype=None):
        """Add a file to be uploaded."""
        body = fileHandle.read()
        if mimetype is None:
            mimetype = (
                mimetypes.guess_type(filename)[0] or
                'application/octet-stream'
            )
        self.files.append((fieldname, filename, mimetype, body))
        return

    @staticmethod
    def _form_data(name):
        return ('Content-Disposition: form-data; '
                'name="{}"\r\n').format(name).encode('utf-8')

    @staticmethod
    def _attached_file(name, filename):
        return ('Content-Disposition: file; '
                'name="{}"; filename="{}"\r\n').format(
                    name, filename).encode('utf-8')

    @staticmethod
    def _content_type(ct):
        return 'Content-Type: {}\r\n'.format(ct).encode('utf-8')

    def __bytes__(self):
        """Return a byte-string representing the form data,
        including attached files.
        """
        buffer = io.BytesIO()
        boundary = b'--' + self.boundary + b'\r\n'

        # Add the form fields
        for name, value in self.form_fields:
            buffer.write(boundary)
            buffer.write(self._form_data(name))
            buffer.write(b'\r\n')
            buffer.write(value.encode('utf-8'))
            buffer.write(b'\r\n')

        # Add the files to upload
        for f_name, filename, f_content_type, body in self.files:
            buffer.write(boundary)
            buffer.write(self._attached_file(f_name, filename))
            buffer.write(self._content_type(f_content_type))
            buffer.write(b'\r\n')
            buffer.write(body)
            buffer.write(b'\r\n')

        buffer.write(b'--' + self.boundary + b'--\r\n')
        return buffer.getvalue()


if __name__ == '__main__':
    # Create the form with simple fields
    form = MultiPartForm()
    form.add_field('firstname', 'Doug')
    form.add_field('lastname', 'Hellmann')

    # Add a fake file
    form.add_file(
        'biography', 'bio.txt',
        fileHandle=io.BytesIO(b'Python developer and blogger.'))

    # Build the request, including the byte-string
    # for the data to be posted.
    data = bytes(form)
    r = request.Request('http://localhost:8080/', data=data)
    r.add_header(
        'User-agent',
        'PyMOTW (https://pymotw.com/)',
    )
    r.add_header('Content-type', form.get_content_type())
    r.add_header('Content-length', len(data))

    print()
    print('OUTGOING DATA:')
    for name, value in r.header_items():
        print('{}: {}'.format(name, value))
    print()
    print(r.data.decode('utf-8'))

    print()
    print('SERVER RESPONSE:')
    print(request.urlopen(r).read().decode('utf-8'))
```
MultiPartForm类可以将任意形式（form）表示为包含附加文件的多部分MIME消息。

```
$ python3 urllib_request_upload_files.py

OUTGOING DATA:
User-agent: PyMOTW (https://pymotw.com/)
Content-type: multipart/form-data; boundary=d99b5dc60871491b9d63352eb24972b4
Content-length: 389

--d99b5dc60871491b9d63352eb24972b4
Content-Disposition: form-data; name="firstname"

Doug
--d99b5dc60871491b9d63352eb24972b4
Content-Disposition: form-data; name="lastname"

Hellmann
--d99b5dc60871491b9d63352eb24972b4
Content-Disposition: file; name="biography";
    filename="bio.txt"
Content-Type: text/plain

Python developer and blogger.
--d99b5dc60871491b9d63352eb24972b4--


SERVER RESPONSE:
Client: ('127.0.0.1', 59310)
User-agent: PyMOTW (https://pymotw.com/)
Path: /
Form data:
    Uploaded biography as 'bio.txt' (29 bytes)
    firstname=Doug
    lastname=Hellmann
```

### Creating Custom Protocol Handlers

urllib.request内置支持HTTP（S），FTP和本地文件访问。 要添加对其他URL类型的支持，请注册另一个协议处理程序。 例如，要支持指向远程NFS服务器上任意文件的URL，而不要求用户在访问文件之前挂载路径，请创建一个派生自BaseHandler的类，实现nfs_open（）。

特定于协议的open（）方法被赋予一个参数，即Request实例，它应该返回一个带有read（）方法的对象，该方法可以用来读取数据，一个info（）方法返回响应头， 和geturl（）返回正在读取的文件的实际URL。实现这一目标的一种简单方法是创建urllib.response.addinfourl的实例，将headers，URL和打开文件句柄传递给构造函数。

```python
#urllib_request_nfs_handler.py
import io
import mimetypes
import os
import tempfile
from urllib import request
from urllib import response

class NFSFile:

    def __init__(self, tempdir, filename):
        self.tempdir = tempdir
        self.filename = filename
        with open(os.path.join(tempdir, filename), 'rb') as f:
            self.buffer = io.BytesIO(f.read())

    def read(self, *args):
        return self.buffer.read(*args)

    def readline(self, *args):
        return self.buffer.readline(*args)

    def close(self):
        print('\nNFSFile:')
        print('  unmounting {}'.format(os.path.basename(self.tempdir)))
        print('  when {} is closed'.format(os.path.basename(self.filename)))

class FauxNFSHandler(request.BaseHandler):

    def __init__(self, tempdir):
        self.tempdir = tempdir
        super().__init__()

    def nfs_open(self, req):
        url = req.full_url
        directory_name, file_name = os.path.split(url)
        server_name = req.host
        print('FauxNFSHandler simulating mount:')
        print('  Remote path: {}'.format(directory_name))
        print('  Server     : {}'.format(server_name))
        print('  Local path : {}'.format(os.path.basename(tempdir)))
        print('  Filename   : {}'.format(file_name))
        local_file = os.path.join(tempdir, file_name)
        fp = NFSFile(tempdir, file_name)
        content_type = (
            mimetypes.guess_type(file_name)[0] or
            'application/octet-stream'
        )
        stats = os.stat(local_file)
        size = stats.st_size
        headers = {
            'Content-type': content_type,
            'Content-length': size,
        }
        return response.addinfourl(fp, headers, req.get_full_url())

if __name__ == '__main__':
    with tempfile.TemporaryDirectory() as tempdir:
        # Populate the temporary file for the simulation
        filename = os.path.join(tempdir, 'file.txt')
        with open(filename, 'w', encoding='utf-8') as f:
            f.write('Contents of file.txt')

        # Construct an opener with our NFS handler
        # and register it as the default opener.
        opener = request.build_opener(FauxNFSHandler(tempdir))
        request.install_opener(opener)

        # Open the file through a URL.
        resp = request.urlopen(
            'nfs://remote_server/path/to/the/file.txt'
        )
        print()
        print('READ CONTENTS:', resp.read())
        print('URL          :', resp.geturl())
        print('HEADERS:')
        for name, value in sorted(resp.info().items()):
            print('  {:<15} = {}'.format(name, value))
        resp.close()
```
FauxNFSHandler和NFSFile类打印消息，以说明实际实现将添加挂载和卸载调用的位置。 由于这仅仅是一个模拟，因此FauxNFSHandler将使用临时目录的名称进行准备，该目录应该查找其所有文件。

```
$ python3 urllib_request_nfs_handler.py

FauxNFSHandler simulating mount:
  Remote path: nfs://remote_server/path/to/the
  Server     : remote_server
  Local path : tmprucom5sb
  Filename   : file.txt

READ CONTENTS: b'Contents of file.txt'
URL          : nfs://remote_server/path/to/the/file.txt
HEADERS:
  Content-length  = 20
  Content-type    = text/plain

NFSFile:
  unmounting tmprucom5sb
  when file.txt is closed
```

## urllib.robotparser — Internet Spider Access Control

目的：解析用于控制网络蜘蛛的robots.txt文件  
robotparser为robots.txt的文件格式实现了一个解析器，包括一个检查给定用户代理是否可以访问资源的函数。 它适用于行为良好的蜘蛛或其他“需要调节或其他限制的”爬虫应用程序。

### robots.txt

robots.txt文件格式是一个简单的基于文本的访问控制系统，用于自动访问Web资源（“蜘蛛”，“爬虫”等）的计算机程序。 该文件由记录组成，这些记录指定程序的用户代理标识符，后跟代理可能无法访问的URL（或URL前缀）列表。  
这是`https://pymotw.com/`的robots.txt文件：

```
#robots.txt

Sitemap: https://pymotw.com/sitemap.xml
User-agent: *
Disallow: /admin/
Disallow: /downloads/
Disallow: /media/
Disallow: /static/
Disallow: /codehosting/
```
它可以阻止访问网站中某些计算成本高昂的部分，如果搜索引擎试图对其进行索引，则会使服务器超载。 有关robots.txt的更完整示例，请参阅[The Web Robots Page](http://www.robotstxt.org/orig.html)。

### Testing Access Permissions

使用前面介绍的数据，一个简单的爬虫可以测试是否允许使用RobotFileParser.can_fetch（）下载页面。

```python
#urllib_robotparser_simple.py
from urllib import parse
from urllib import robotparser

AGENT_NAME = 'PyMOTW'
URL_BASE = 'https://pymotw.com/'
parser = robotparser.RobotFileParser()
parser.set_url(parse.urljoin(URL_BASE, 'robots.txt'))
parser.read()

PATHS = [
    '/',
    '/PyMOTW/',
    '/admin/',
    '/downloads/PyMOTW-1.92.tar.gz',
]

for path in PATHS:
    print('{!r:>6} : {}'.format(parser.can_fetch(AGENT_NAME, path), path))
    url = parse.urljoin(URL_BASE, path)
    print('{!r:>6} : {}'.format(parser.can_fetch(AGENT_NAME, url), url))
    print()
```
can_fetch（）的URL参数可以是相对于站点根目录的路径，也可以是完整URL。

```
$ python3 urllib_robotparser_simple.py

  True : /
  True : https://pymotw.com/

  True : /PyMOTW/
  True : https://pymotw.com/PyMOTW/

 False : /admin/
 False : https://pymotw.com/admin/

 False : /downloads/PyMOTW-1.92.tar.gz
 False : https://pymotw.com/downloads/PyMOTW-1.92.tar.gz
```

### Long-lived Spiders

处理下载的资源需要花费很长时间或由于限制在下载之间暂停的应用程序应根据已下载的内容的年龄定期检查新的robots.txt文件。 年龄不是自动管理的，但有一些方便的方法可以使跟踪变得更容易。

```python
#urllib_robotparser_longlived.py
from urllib import robotparser
import time

AGENT_NAME = 'PyMOTW'
parser = robotparser.RobotFileParser()
# Using the local copy
parser.set_url('file:robots.txt')
parser.read()
parser.modified()

PATHS = [
    '/',
    '/PyMOTW/',
    '/admin/',
    '/downloads/PyMOTW-1.92.tar.gz',
]

for path in PATHS:
    age = int(time.time() - parser.mtime())
    print('age:', age, end=' ')
    if age > 1:
        print('rereading robots.txt')
        parser.read()
        parser.modified()
    else:
        print()
    print('{!r:>6} : {}'.format(
        parser.can_fetch(AGENT_NAME, path), path))
    # Simulate a delay in processing
    time.sleep(1)
    print()
```
这个极端的例子下载一个新的robots.txt文件，如果它拥有的文件更旧超过一秒。

```
$ python3 urllib_robotparser_longlived.py

age: 0
  True : /

age: 1
  True : /PyMOTW/

age: 2 rereading robots.txt
 False : /admin/

age: 1
 False : /downloads/PyMOTW-1.92.tar.gz
```
一个更好的长期应用程序版本可能会在下载整个文件之前请求文件的修改时间。 另一方面，robots.txt文件通常相当小，因此再次检索整个文档并不是那么昂贵。

## base64 — Encode Binary Data with ASCII

目的：base64模块包含将二进制数据转换为“适合使用明文协议传输的”ASCII子集的功能。  
base64，base32，base16和base85编码将8位字节转换为适合ASCII可打印字符范围内的值，交换更多位以表示数据，以便与仅支持ASCII数据的系统（如SMTP）兼容。 基值对应于每个编码中使用的字母表的长度。 还有原始编码的URL安全变体，使用略有不同的字母表。

### Base 64 Encoding

这是编码某些文本的基本示例。

```python
#base64_b64encode.py
import base64
import textwrap

# Load this source file and strip the header.
with open(__file__, 'r', encoding='utf-8') as input:
    raw = input.read()
    initial_data = raw.split('#end_pymotw_header')[1]

byte_string = initial_data.encode('utf-8')
encoded_data = base64.b64encode(byte_string)

num_initial = len(byte_string)

# There will never be more than 2 padding bytes.
padding = 3 - (num_initial % 3)

print('{} bytes before encoding'.format(num_initial))
print('Expect {} padding bytes'.format(padding))
print('{} bytes after encoding\n'.format(len(encoded_data)))
print(encoded_data)
```
输入必须是字节字符串，因此unicode字符串首先编码为UTF-8。 输出显示UTF-8源的185个字节在编码后扩展到248个字节。
>注意  
库生成的编码数据中没有回车符，但输出已经人工包装，以使其更适合页面。

```
$ python3 base64_b64encode.py

184 bytes before encoding
Expect 2 padding bytes
248 bytes after encoding

b'CmltcG9ydCBiYXNlNjQKaW1wb3J0IHRleHR3cmFwCgojIExvYWQgdGhpcyBzb3
VyY2UgZmlsZSBhbmQgc3RyaXAgdGhlIGhlYWRlci4Kd2l0aCBvcGVuKF9fZmlsZV
9fLCAncicsIGVuY29kaW5nPSd1dGYtOCcpIGFzIGlucHV0OgogICAgcmF3ID0gaW
5wdXQucmVhZCgpCiAgICBpbml0aWFsX2RhdGEgPSByYXcuc3BsaXQoJw=='
```

### Base 64 Decoding

b64decode（）通过使用查找表将四个字节转换为原始三个字节，将编码后的字符串转换回原始格式。

```python
#base64_b64decode.py
import base64

encoded_data = b'VGhpcyBpcyB0aGUgZGF0YSwgaW4gdGhlIGNsZWFyLg=='
decoded_data = base64.b64decode(encoded_data)
print('Encoded :', encoded_data)
print('Decoded :', decoded_data)
```
编码过程查看输入中的每个24位序列（三个字节），并将相同的24位编码成输出中的四个字节。 输出末尾的等号是插入的填充，因为在这个例子中，原始字符串中的位数不能被24整除。

```
$ python3 base64_b64decode.py

Encoded : b'VGhpcyBpcyB0aGUgZGF0YSwgaW4gdGhlIGNsZWFyLg=='
Decoded : b'This is the data, in the clear.'
```
从b64decode（）返回的值是一个字节字符串。 如果已知内容是文本，则可以将字节字符串转换为unicode对象。 但是，使用base 64编码的目的是能够传输二进制数据，因此假设解码值是文本并不总是安全的。

### URL-safe Variations

由于默认的base64字母表可能使用+和/，并且这两个字符用于URL，因此通常需要使用替代编码来替换这些字符。

```python
#base64_urlsafe.py
import base64

encodes_with_pluses = b'\xfb\xef'
encodes_with_slashes = b'\xff\xff'

for original in [encodes_with_pluses, encodes_with_slashes]:
    print('Original         :', repr(original))
    print('Standard encoding:',
          base64.standard_b64encode(original))
    print('URL-safe encoding:',
          base64.urlsafe_b64encode(original))
    print()
```
`+`被替换为 `-` ，而`/`被替换为下划线（`_`）。 否则，字母表是相同的。

```
$ python3 base64_urlsafe.py

Original         : b'\xfb\xef'
Standard encoding: b'++8='
URL-safe encoding: b'--8='

Original         : b'\xff\xff'
Standard encoding: b'//8='
URL-safe encoding: b'__8='
```

### Other Encodings

除了Base64之外，该模块还提供了使用Base85，Base32和Base16（十六进制）编码数据的功能。

```python
#base64_base32.py
import base64

original_data = b'This is the data, in the clear.'
print('Original:', original_data)

encoded_data = base64.b32encode(original_data)
print('Encoded :', encoded_data)

decoded_data = base64.b32decode(encoded_data)
print('Decoded :', decoded_data)
```
Base32字母表包括ASCII集中的26个大写字母和数字2到7。

```
$ python3 base64_base32.py

Original: b'This is the data, in the clear.'
Encoded : b'KRUGS4ZANFZSA5DIMUQGIYLUMEWCA2LOEB2GQZJAMNWGKYLSFY==
===='
Decoded : b'This is the data, in the clear.'
```
Base16函数使用十六进制字母表。

```python
#base64_base16.py
import base64

original_data = b'This is the data, in the clear.'
print('Original:', original_data)

encoded_data = base64.b16encode(original_data)
print('Encoded :', encoded_data)

decoded_data = base64.b16decode(encoded_data)
print('Decoded :', decoded_data)
```
每次编码位数下降时，编码格式的输出占用更多空间。

```
$ python3 base64_base16.py

Original: b'This is the data, in the clear.'
Encoded : b'546869732069732074686520646174612C20696E207468652063
6C6561722E'
Decoded : b'This is the data, in the clear.'
```
Base85函数使用扩展字母表，比base 64更节省空间。

```python
#base64_base85.py
import base64

original_data = b'This is the data, in the clear.'
print('Original    : {} bytes {!r}'.format(
    len(original_data), original_data))

b64_data = base64.b64encode(original_data)
print('b64 Encoded : {} bytes {!r}'.format(
    len(b64_data), b64_data))

b85_data = base64.b85encode(original_data)
print('b85 Encoded : {} bytes {!r}'.format(
    len(b85_data), b85_data))

a85_data = base64.a85encode(original_data)
print('a85 Encoded : {} bytes {!r}'.format(
    len(a85_data), a85_data))
```
有几种Base85编码，Mercurial，git和PDF文件格式使用不同的变体。 Python包含两个实现，b85encode（）实现Git和Mercurial中使用的版本，而a85encode（）实现PDF文件使用的Ascii85变体。

```
$ python3 base64_base85.py

Original    : 31 bytes b'This is the data, in the clear.'
b64 Encoded : 44 bytes b'VGhpcyBpcyB0aGUgZGF0YSwgaW4gdGhlIGNsZWF
yLg=='
b85 Encoded : 39 bytes b'RA^~)AZc?TbZBKDWMOn+EFfuaAarPDAY*K0VR9}
'
a85 Encoded : 39 bytes b'<+oue+DGm>FD,5.A79Rg/0JYE+EV:.+Cf5!@<*t
'
```

## http.server — Base Classes for Implementing Web Servers

目的：http.server包含可以构成Web服务器基础的类。  
http.server使用socketserver中的类来创建“用于创建HTTP服务器的”基类。 HTTPServer可以直接使用，但BaseHTTPRequestHandler旨在扩展以处理每种协议方法（GET，POST等）。

### HTTP GET

要在请求处理程序类中添加对HTTP方法的支持，请实现方法`do_METHOD（）`，将METHOD替换为HTTP方法的名称（例如，`do_GET（）`，`do_POST（）`等）。 为了保持一致性，请求处理程序方法不带参数。 请求的所有参数都由BaseHTTPRequestHandler解析并存储为请求实例的实例属性。

此示例请求处理程序说明了如何将响应返回给客户端，以及一些可用于构建响应的本地属性。

```python
#http_server_GET.py
from http.server import BaseHTTPRequestHandler
from urllib import parse

class GetHandler(BaseHTTPRequestHandler):

    def do_GET(self):
        parsed_path = parse.urlparse(self.path)
        message_parts = [
            'CLIENT VALUES:',
            'client_address={} ({})'.format(self.client_address, self.address_string()),
            'command={}'.format(self.command),
            'path={}'.format(self.path),
            'real path={}'.format(parsed_path.path),
            'query={}'.format(parsed_path.query),
            'request_version={}'.format(self.request_version),
            '',
            'SERVER VALUES:',
            'server_version={}'.format(self.server_version),
            'sys_version={}'.format(self.sys_version),
            'protocol_version={}'.format(self.protocol_version),
            '',
            'HEADERS RECEIVED:',
        ]
        for name, value in sorted(self.headers.items()):
            message_parts.append(
                '{}={}'.format(name, value.rstrip())
            )
        message_parts.append('')
        message = '\r\n'.join(message_parts)
        self.send_response(200)
        self.send_header('Content-Type', 'text/plain; charset=utf-8')
        self.end_headers()
        self.wfile.write(message.encode('utf-8'))

if __name__ == '__main__':
    from http.server import HTTPServer
    server = HTTPServer(('localhost', 8080), GetHandler)
    print('Starting server, use <Ctrl-C> to stop')
    server.serve_forever()
```
消息文本被聚合，然后写入wfile，文件句柄包装响应套接字。 每个响应都需要一个响应码，通过send_response（）设置。 如果使用了错误码（404,501等），则header中会包含相应的默认错误消息，或者消息可以和错误码一起传送。  
要在服务器中运行请求处理程序，请将其传递给HTTPServer的构造函数，如示例脚本的`__main__`处理部分中所示。  
然后启动服务器：

```
$ python3 http_server_GET.py

Starting server, use <Ctrl-C> to stop
```
在单独的终端中，使用curl访问它：

```
$ curl -v -i http://127.0.0.1:8080/?foo=bar

*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 8080 (#0)
> GET /?foo=bar HTTP/1.1
> Host: 127.0.0.1:8080
> User-Agent: curl/7.43.0
> Accept: */*
>
HTTP/1.0 200 OK
Content-Type: text/plain; charset=utf-8
Server: BaseHTTP/0.6 Python/3.5.2
Date: Thu, 06 Oct 2016 20:44:11 GMT

CLIENT VALUES:
client_address=('127.0.0.1', 52934) (127.0.0.1)
command=GET
path=/?foo=bar
real path=/
query=foo=bar
request_version=HTTP/1.1

SERVER VALUES:
server_version=BaseHTTP/0.6
sys_version=Python/3.5.2
protocol_version=HTTP/1.0

HEADERS RECEIVED:
Accept=*/*
Host=127.0.0.1:8080
User-Agent=curl/7.43.0
* Connection #0 to host 127.0.0.1 left intact
```
>注意  
不同版本的curl产生的输出可能会有所不同。 如果运行示例产生不同的输出，请检查curl报告的版本号。

### HTTP POST

支持POST请求需要更多工作，因为基类不会自动解析表单数据。 cgi模块提供FieldStorage类，如果给出正确的输入，它将知道如何解析表单。

```python
#http_server_POST.py
import cgi
from http.server import BaseHTTPRequestHandler
import io

class PostHandler(BaseHTTPRequestHandler):

    def do_POST(self):
        # Parse the form data posted
        form = cgi.FieldStorage(
            fp=self.rfile,
            headers=self.headers,
            environ={
                'REQUEST_METHOD': 'POST',
                'CONTENT_TYPE': self.headers['Content-Type'],
            }
        )

        # Begin the response
        self.send_response(200)
        self.send_header('Content-Type', 'text/plain; charset=utf-8')
        self.end_headers()

        out = io.TextIOWrapper(
            self.wfile,
            encoding='utf-8',
            line_buffering=False,
            write_through=True,
        )

        out.write('Client: {}\n'.format(self.client_address))
        out.write('User-agent: {}\n'.format(self.headers['user-agent']))
        out.write('Path: {}\n'.format(self.path))
        out.write('Form data:\n')

        # Echo back information about what was posted in the form
        for field in form.keys():
            field_item = form[field]
            if field_item.filename:
                # The field contains an uploaded file
                file_data = field_item.file.read()
                file_len = len(file_data)
                del file_data
                out.write(
                    '\tUploaded {} as {!r} ({} bytes)\n'.format(
                        field, field_item.filename, file_len)
                )
            else:
                # Regular form value
                out.write('\t{}={}\n'.format(
                    field, form[field].value))

        # Disconnect our encoding wrapper from the underlying
        # buffer so that deleting the wrapper doesn't close
        # the socket, which is still being used by the server.
        out.detach()

if __name__ == '__main__':
    from http.server import HTTPServer
    server = HTTPServer(('localhost', 8080), PostHandler)
    print('Starting server, use <Ctrl-C> to stop')
    server.serve_forever()
```
在一个窗口中运行服务器：

```
$ python3 http_server_POST.py

Starting server, use <Ctrl-C> to stop
```
curl使用`-F`选项，设置提交到服务器的表单数据。 最后一个参数`-F datafile=@http_server_GET.py`提交文件`http_server_GET.py`的内容，以说明从表单中读取文件数据。

```
$ curl -v http://127.0.0.1:8080/ -F name=dhellmann -F foo=bar \
-F datafile=@http_server_GET.py

*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 8080 (#0)
> POST / HTTP/1.1
> Host: 127.0.0.1:8080
> User-Agent: curl/7.43.0
> Accept: */*
> Content-Length: 1974
> Expect: 100-continue
> Content-Type: multipart/form-data;
boundary=------------------------a2b3c7485cf8def2
>
* Done waiting for 100-continue
HTTP/1.0 200 OK
Content-Type: text/plain; charset=utf-8
Server: BaseHTTP/0.6 Python/3.5.2
Date: Thu, 06 Oct 2016 20:53:48 GMT

Client: ('127.0.0.1', 53121)
User-agent: curl/7.43.0
Path: /
Form data:
    name=dhellmann
    Uploaded datafile as 'http_server_GET.py' (1612 bytes)
    foo=bar
* Connection #0 to host 127.0.0.1 left intact
```

### Threading and Forking

HTTPServer是socketserver.TCPServer的简单子类，不使用多个线程或进程来处理请求。 要添加线程或分支进程（forking），请使用socketserver中的相应混合（mix-in）创建一个新类。

```python
#http_server_threads.py
from http.server import HTTPServer, BaseHTTPRequestHandler
from socketserver import ThreadingMixIn
import threading

class Handler(BaseHTTPRequestHandler):

    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-Type',
                         'text/plain; charset=utf-8')
        self.end_headers()
        message = threading.currentThread().getName()
        self.wfile.write(message.encode('utf-8'))
        self.wfile.write(b'\n')


class ThreadedHTTPServer(ThreadingMixIn, HTTPServer):
    """Handle requests in a separate thread."""

if __name__ == '__main__':
    server = ThreadedHTTPServer(('localhost', 8080), Handler)
    print('Starting server, use <Ctrl-C> to stop')
    server.serve_forever()
```
以与其他示例相同的方式运行服务器。

```
$ python3 http_server_threads.py

Starting server, use <Ctrl-C> to stop
```
每次服务器收到请求时，它都会启动一个新的线程或进程来处理它：

```
$ curl http://127.0.0.1:8080/

Thread-1

$ curl http://127.0.0.1:8080/

Thread-2

$ curl http://127.0.0.1:8080/

Thread-3
```
使用ForkingMixIn 代替 ThreadingMixIn将使用单独的进程而不是线程来实现类似的结果。

### Handling Errors

通过调用send_error（）处理错误，传递相应的错误码和可选的错误消息。 整个响应（包含headers，status code和body）会自动生成。

```python
#http_server_errors.py
from http.server import BaseHTTPRequestHandler

class ErrorHandler(BaseHTTPRequestHandler):

    def do_GET(self):
        self.send_error(404)

if __name__ == '__main__':
    from http.server import HTTPServer
    server = HTTPServer(('localhost', 8080), ErrorHandler)
    print('Starting server, use <Ctrl-C> to stop')
    server.serve_forever()
```
在这种情况下，始终返回404错误。

```
$ python3 http_server_errors.py

Starting server, use <Ctrl-C> to stop
```
使用HTML文档向客户端报告错误消息以及header指示错误码。

```
$ curl -i http://127.0.0.1:8080/

HTTP/1.0 404 Not Found
Server: BaseHTTP/0.6 Python/3.5.2
Date: Thu, 06 Oct 2016 20:58:08 GMT
Connection: close
Content-Type: text/html;charset=utf-8
Content-Length: 447

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN"
        "http://www.w3.org/TR/html4/strict.dtd">
<html>
    <head>
        <meta http-equiv="Content-Type"
        content="text/html;charset=utf-8">
        <title>Error response</title>
    </head>
    <body>
        <h1>Error response</h1>
        <p>Error code: 404</p>
        <p>Message: Not Found.</p>
        <p>Error code explanation: 404 - Nothing matches the
        given URI.</p>
    </body>
</html>
```

### Setting Headers

send_header方法将header数据添加到HTTP响应中。 它有两个参数：header的名称和值。

```python
#http_server_send_header.py
from http.server import BaseHTTPRequestHandler
import time

class GetHandler(BaseHTTPRequestHandler):

    def do_GET(self):
        self.send_response(200)
        self.send_header(
            'Content-Type',
            'text/plain; charset=utf-8',
        )
        self.send_header(
            'Last-Modified',
            self.date_time_string(time.time())
        )
        self.end_headers()
        self.wfile.write('Response body\n'.encode('utf-8'))

if __name__ == '__main__':
    from http.server import HTTPServer
    server = HTTPServer(('localhost', 8080), GetHandler)
    print('Starting server, use <Ctrl-C> to stop')
    server.serve_forever()
```
此示例将Last-Modified标头设置为当前时间戳，根据RFC 7231格式化。

```
$ curl -i http://127.0.0.1:8080/

HTTP/1.0 200 OK
Server: BaseHTTP/0.6 Python/3.5.2
Date: Thu, 06 Oct 2016 21:00:54 GMT
Content-Type: text/plain; charset=utf-8
Last-Modified: Thu, 06 Oct 2016 21:00:54 GMT

Response body
```
服务器将请求记录到终端，就像其他示例一样。

```
$ python3 http_server_send_header.py

Starting server, use <Ctrl-C> to stop
127.0.0.1 - - [06/Oct/2016 17:00:54] "GET / HTTP/1.1" 200 -
```

### Command Line Use

http.server包含一个内置服务器，用于从本地文件系统提供文件。 使用Python解释器的`-m`选项从命令行启动它。

```
$ python3 -m http.server 8080

Serving HTTP on 0.0.0.0 port 8080 ...
127.0.0.1 - - [06/Oct/2016 17:12:48] "HEAD /index.rst HTTP/1.1" 200 -
```
服务器的根目录是启动服务器的工作目录。

```
$ curl -I http://127.0.0.1:8080/index.rst

HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.5.2
Date: Thu, 06 Oct 2016 21:12:48 GMT
Content-type: application/octet-stream
Content-Length: 8285
Last-Modified: Thu, 06 Oct 2016 21:12:10 GMT
```

## http.cookies — HTTP Cookies

目的：定义用于解析和创建HTTP cookie headers的类。  
http.cookies模块实现了一个大致符合RFC 2109的cookie解析器。 实现不如标准严格，因为MSIE 3.0x不支持整个标准。

### Creating and Setting a Cookie

Cookie用作基于浏览器的应用程序的状态管理，因此通常由服务器设置，由客户端存储和返回。 创建cookie的最简单示例是设置单个`名称-值`对。

```python
#http_cookies_setheaders.py
from http import cookies

c = cookies.SimpleCookie()
c['mycookie'] = 'cookie_value'
print(c)
```
输出是一个有效的Set-Cookie header，可以作为HTTP响应的一部分传递给客户端。

```
$ python3 http_cookies_setheaders.py

Set-Cookie: mycookie=cookie_value
```

### Morsels

还可以控制cookie的其他方面，例如到期、路径和域。 事实上，cookie的所有RFC属性都可以通过表示cookie值的Morsel对象进行管理。

```python
#http_cookies_Morsel.py
from http import cookies
import datetime

def show_cookie(c):
    print(c)
    for key, morsel in c.items():
        print()
        print('key =', morsel.key)
        print('  value =', morsel.value)
        print('  coded_value =', morsel.coded_value)
        for name in morsel.keys():
            if morsel[name]:
                print('  {} = {}'.format(name, morsel[name]))

c = cookies.SimpleCookie()

# A cookie with a value that has to be encoded
# to fit into the header
c['encoded_value_cookie'] = '"cookie,value;"'
c['encoded_value_cookie']['comment'] = 'Has escaped punctuation'

# A cookie that only applies to part of a site
c['restricted_cookie'] = 'cookie_value'
c['restricted_cookie']['path'] = '/sub/path'
c['restricted_cookie']['domain'] = 'PyMOTW'
c['restricted_cookie']['secure'] = True

# A cookie that expires in 5 minutes
c['with_max_age'] = 'expires in 5 minutes'
c['with_max_age']['max-age'] = 300  # seconds

# A cookie that expires at a specific time
c['expires_at_time'] = 'cookie_value'
time_to_live = datetime.timedelta(hours=1)
expires = (datetime.datetime(2009, 2, 14, 18, 30, 14) + time_to_live)

# Date format: Wdy, DD-Mon-YY HH:MM:SS GMT
expires_at_time = expires.strftime('%a, %d %b %Y %H:%M:%S')
c['expires_at_time']['expires'] = expires_at_time

show_cookie(c)
```
此示例包括两种不同的方法，用于设置存储Cookie的过期。一个将max-age设置为秒数，另一个设置将到达应该丢弃cookie的日期和时间。

```
$ python3 http_cookies_Morsel.py

Set-Cookie: encoded_value_cookie="\"cookie\054value\073\"";
Comment=Has escaped punctuation
Set-Cookie: expires_at_time=cookie_value; expires=Sat, 14 Feb 2009 19:30:14
Set-Cookie: restricted_cookie=cookie_value; Domain=PyMOTW;
Path=/sub/path; Secure
Set-Cookie: with_max_age="expires in 5 minutes"; Max-Age=300

key = encoded_value_cookie
  value = "cookie,value;"
  coded_value = "\"cookie\054value\073\""
  comment = Has escaped punctuation

key = restricted_cookie
  value = cookie_value
  coded_value = cookie_value
  path = /sub/path
  domain = PyMOTW
  secure = True

key = with_max_age
  value = expires in 5 minutes
  coded_value = "expires in 5 minutes"
  max-age = 300

key = expires_at_time
  value = cookie_value
  coded_value = cookie_value
  expires = Sat, 14 Feb 2009 19:30:14
```
Cookie和Morsel对象都像字典一样。 一个Morsel响应一组固定的键：

+ expires
+ path
+ comment
+ domain
+ max-age
+ secure
+ version

Cookie实例的键是存储的各个cookie的名称。 该信息也可以从Morsel的key属性获得。

### Encoded Values

cookie header需要对值进行编码，以便可以正确解析它们。

```python
#http_cookies_coded_value.py
from http import cookies

c = cookies.SimpleCookie()
c['integer'] = 5
c['with_quotes'] = 'He said, "Hello, World!"'

for name in ['integer', 'with_quotes']:
    print(c[name].key)
    print('  {}'.format(c[name]))
    print('  value={!r}'.format(c[name].value))
    print('  coded_value={!r}'.format(c[name].coded_value))
    print()
```
Morsel.value始终是cookie的解码值，而Morsel.coded_value始终是用于将值传输到客户端的表示。 两个值都是字符串。 保存到cookie的值如果不是字符串将被自动转换。

```
$ python3 http_cookies_coded_value.py

integer
  Set-Cookie: integer=5
  value='5'
  coded_value='5'

with_quotes
  Set-Cookie: with_quotes="He said\054 \"Hello\054 World!\""
  value='He said, "Hello, World!"'
  coded_value='"He said\\054 \\"Hello\\054 World!\\""'
```

### Receiving and Parsing Cookie Headers

一旦客户端收到Set-Cookie header，它将使用Cookie header在后续请求中将这些cookie返回给服务器。 传入的Cookie header字符串可能包含多个cookie值，以分号（`;`）分隔。

```
Cookie: integer=5; with_quotes="He said, \"Hello, World!\""
```
根据Web服务器和框架，cookie可以直接从headers或HTTP_COOKIE环境变量中获得。

```python
#http_cookies_parse.py
from http import cookies

HTTP_COOKIE = '; '.join([
    r'integer=5',
    r'with_quotes="He said, \"Hello, World!\""',
])

print('From constructor:')
c = cookies.SimpleCookie(HTTP_COOKIE)
print(c)

print()
print('From load():')
c = cookies.SimpleCookie()
c.load(HTTP_COOKIE)
print(c)
```
要对它们进行解码，请在实例化时将没有header前缀的字符串传递给SimpleCookie，或使用load（）方法。

```
$ python3 http_cookies_parse.py

From constructor:
Set-Cookie: integer=5
Set-Cookie: with_quotes="He said, \"Hello, World!\""

From load():
Set-Cookie: integer=5
Set-Cookie: with_quotes="He said, \"Hello, World!\""
```

### Alternative Output Formats

除了使用Set-Cookie header之外，服务器还可以提供将cookie添加到客户端的JavaScript。 SimpleCookie和Morsel通过js_output（）方法提供JavaScript输出。

```python
#http_cookies_js_output.py
from http import cookies
import textwrap

c = cookies.SimpleCookie()
c['mycookie'] = 'cookie_value'
c['another_cookie'] = 'second value'
js_text = c.js_output()
print(textwrap.dedent(js_text).lstrip())
```
结果是一个完整的script标记，其中包含用于设置cookie的语句。

```
$ python3 http_cookies_js_output.py

<script type="text/javascript">
<!-- begin hiding
document.cookie = "another_cookie=\"second value\"";
// end hiding -->
</script>

<script type="text/javascript">
<!-- begin hiding
document.cookie = "mycookie=cookie_value";
// end hiding -->
</script>
```

## webbrowser — Displays web pages

目的：使用webbrowser模块向用户显示网页。  
webbrowser模块包括在交互式浏览器应用程序中打开URL的功能。 该模块包括可用浏览器的注册表，以防系统上有多个选项。 它也可以使用BROWSER环境变量进行控制。

### Simple Example

要在浏览器中打开页面，请使用open（）函数。

```python
#webbrowser_open.py
import webbrowser

webbrowser.open(
    'https://docs.python.org/3/library/webbrowser.html'
)
```
URL在浏览器窗口中打开，该窗口被提升到窗口堆栈的顶部。 文档中说如果可能，将重用现有窗口，但实际行为可能取决于浏览器的设置。 在Mac OS X上使用Firefox，始终会创建一个新窗口。

### Windows vs. Tabs

如果您总是想要使用新窗口，请使用open_new（）。

```python
#webbrowser_open_new.py
import webbrowser

webbrowser.open_new(
    'https://docs.python.org/3/library/webbrowser.html'
)
```
如果您希望创建新选项卡，请改用`open_new_tab（）`。


### Using a specific browser

如果由于某种原因您的应用程序需要使用特定的浏览器，您可以使用get（）函数访问已注册的浏览器控制器集。 浏览器控制器具有`open（）`，`open_new（）`和`open_new_tab（）`的方法。 此示例强制使用lynx浏览器：

```python
#webbrowser_get.py
import webbrowser

b = webbrowser.get('lynx')
b.open('https://docs.python.org/3/library/webbrowser.html')
```
有关可用浏览器类型的列表，请参阅模块文档。

### BROWSER variable

用户可以通过将环境变量BROWSER设置为要尝试的浏览器名称或命令来从应用程序外部控制模块。 该值应包含一系列由`os.pathsep`分隔的浏览器名称。 如果名称包含％s，则该名称将被解释为文字命令，并使用URL替换％s后直接执行。 否则，将名称传递给get（）以从注册表获取控制器对象。  
例如，无论其他什么浏览器被注册，此命令都会在lynx中打开网页（假设它可用）。

$ BROWSER=lynx python3 webbrowser_open.py
If none of the names in BROWSER work, webbrowser falls back to its default behavior.

### Command Line Interface

webbrowser模块的所有功能都可以通过命令行以及从Python程序中获得。

```
$ python3 -m webbrowser

Usage: .../lib/python3.6/webbrowser.py [-n | -t] url
    -n: open new window
    -t: open new tab

```

## uuid — Universally Unique Identifiers

目的：uuid模块实现了RFC 4122中描述的通用唯一标识符。   
RFC 4122定义了一种系统，用于以不需要中央注册器的方式为资源创建通用唯一标识符。 UUID值为128位长，正如参考指南所说，“可以保证跨空间和时间的唯一性。”它们可用于为文档，主机，应用程序客户端以及需要唯一值的其他情况生成标识符。 RFC专门用于创建统一资源名称命名空间，并涵盖三个主要算法：

+ 使用IEEE 802 MAC地址作为唯一性的来源
+ 使用伪随机数
+ 使用众所周知的字符串与加密散列相结合

在所有情况下，种子值与系统时钟和时钟序列值组合，用于在时钟向后设置时保持唯一性。

### UUID 1 - IEEE 802 MAC Address

UUID版本1使用主机的MAC地址计算值。 uuid模块使用getnode（）来检索当前系统的MAC值。

```python
#uuid_getnode.py
import uuid

print(hex(uuid.getnode()))
```
如果系统具有多个网卡，并且具有多个MAC，则可以返回任何一个值。

```
$ python3 uuid_getnode.py

0xa860b60304d5
```
要为主机生成UUID（由其MAC地址标识），请使用uuid1（）函数。 节点标识符参数是可选的; 将该字段留空以使用getnode（）返回的值。

```python
#uuid_uuid1.py
import uuid

u = uuid.uuid1()

print(u)
print(type(u))
print('bytes   :', repr(u.bytes))
print('hex     :', u.hex)
print('int     :', u.int)
print('urn     :', u.urn)
print('variant :', u.variant)
print('version :', u.version)
print('fields  :', u.fields)
print('  time_low            : ', u.time_low)
print('  time_mid            : ', u.time_mid)
print('  time_hi_version     : ', u.time_hi_version)
print('  clock_seq_hi_variant: ', u.clock_seq_hi_variant)
print('  clock_seq_low       : ', u.clock_seq_low)
print('  node                : ', u.node)
print('  time                : ', u.time)
print('  clock_seq           : ', u.clock_seq)
```
返回的UUID对象的组件可以通过只读实例属性访问。 某些属性（如hex，int和urn）是UUID值的不同表示形式。

```
$ python3 uuid_uuid1.py

38332b62-2aea-11e8-b103-a860b60304d5
<class 'uuid.UUID'>
bytes   : b'83+b*\xea\x11\xe8\xb1\x03\xa8`\xb6\x03\x04\xd5'
hex     : 38332b622aea11e8b103a860b60304d5
int     : 74702454824994792138317938288475964629
urn     : urn:uuid:38332b62-2aea-11e8-b103-a860b60304d5
variant : specified in RFC 4122
version : 1
fields  : (942877538, 10986, 4584, 177, 3, 185133323977941)
  time_low            :  942877538
  time_mid            :  10986
  time_hi_version     :  4584
  clock_seq_hi_variant:  177
  clock_seq_low       :  3
  node                :  185133323977941
  time                :  137406974088391522
  clock_seq           :  12547
```
由于时间组件，每次调用uuid1（）都会返回一个新值。

```python
#uuid_uuid1_repeat.py
import uuid

for i in range(3):
    print(uuid.uuid1())
```
在此输出中，只有时间组件（在字符串的开头）发生更改。

```
$ python3 uuid_uuid1_repeat.py

3842ca28-2aea-11e8-8fec-a860b60304d5
3844cd18-2aea-11e8-aca3-a860b60304d5
3844cdf4-2aea-11e8-ac38-a860b60304d5
```
由于每台计算机都有不同的MAC地址，因此在不同系统上运行示例程序将产生完全不同的值。 此示例显式传递节点ID以模拟在不同主机上运行。

```python
#uuid_uuid1_othermac.py
import uuid

for node in [0x1ec200d9e0, 0x1e5274040e]:
    print(uuid.uuid1(node), hex(node))
```
除了不同的时间值之外，UUID末尾的节点标识符也会改变。

```
$ python3 uuid_uuid1_othermac.py

3851ea50-2aea-11e8-936d-001ec200d9e0 0x1ec200d9e0
3852caa6-2aea-11e8-a805-001e5274040e 0x1e5274040e
```

### UUID 3 and 5 - Name-Based Values

在某些上下文中，从名称而不是随机或基于时间的值创建UUID值也很有用。 UUID规范的版本3和5使用加密哈希值（分别为MD5或SHA-1）将名称空间特定的种子值与名称组合在一起。 有几个众所周知的命名空间，由预定义的UUID值标识，用于处理DNS，URL，ISO OID和X.500可分辨名称。 可以通过生成和保存UUID值来定义新的特定于应用程序的命名空间。

```python
#uuid_uuid3_uuid5.py
import uuid

hostnames = ['www.doughellmann.com', 'blog.doughellmann.com']

for name in hostnames:
    print(name)
    print('  MD5   :', uuid.uuid3(uuid.NAMESPACE_DNS, name))
    print('  SHA-1 :', uuid.uuid5(uuid.NAMESPACE_DNS, name))
    print()
```
要从DNS名称创建UUID，请将`uuid.NAMESPACE_DNS`作为名称空间参数传递给uuid3（）或uuid5（）：

```
$ python3 uuid_uuid3_uuid5.py

www.doughellmann.com
  MD5   : bcd02e22-68f0-3046-a512-327cca9def8f
  SHA-1 : e3329b12-30b7-57c4-8117-c2cd34a87ce9

blog.doughellmann.com
  MD5   : 9bdabfce-dfd6-37ab-8a3f-7f7293bcf111
  SHA-1 : fa829736-7ef8-5239-9906-b4775a5abacb
```
无论何时何地计算，命名空间中给定名称的UUID值始终相同。

```python
#uuid_uuid3_repeat.py
import uuid

namespace_types = sorted(
    n
    for n in dir(uuid)
    if n.startswith('NAMESPACE_')
)
name = 'www.doughellmann.com'

for namespace_type in namespace_types:
    print(namespace_type)
    namespace_uuid = getattr(uuid, namespace_type)
    print(' ', uuid.uuid3(namespace_uuid, name))
    print(' ', uuid.uuid3(namespace_uuid, name))
    print()
```
名称空间中相同名称的值是不同的。

```
$ python3 uuid_uuid3_repeat.py

NAMESPACE_DNS
  bcd02e22-68f0-3046-a512-327cca9def8f
  bcd02e22-68f0-3046-a512-327cca9def8f

NAMESPACE_OID
  e7043ac1-4382-3c45-8271-d5c083e41723
  e7043ac1-4382-3c45-8271-d5c083e41723

NAMESPACE_URL
  5d0fdaa9-eafd-365e-b4d7-652500dd1208
  5d0fdaa9-eafd-365e-b4d7-652500dd1208

NAMESPACE_X500
  4a54d6e7-ce68-37fb-b0ba-09acc87cabb7
  4a54d6e7-ce68-37fb-b0ba-09acc87cabb7
```

### UUID 4 - Random Values

有时基于主机和基于名称空间的UUID值不是“足够差异”的。例如，在UUID旨在用作散列键的情况下，为了避免哈希表中的冲突，需要具有更多区分的更随机的值序列。 具有较少公共数字的值也使得在日志文件中更容易找到它们。 要在UUID中添加更大的区别，请使用uuid4（）使用随机输入值生成它们。

```python
#uuid_uuid4.py
import uuid

for i in range(3):
    print(uuid.uuid4())
```
随机源取决于导入uuid时可用的C库。 如果可以加载libuuid（或uuid.dll）并且它包含用于生成随机值的函数，则使用它。 否则使用os.urandom（）或random模块。

```
$ python3 uuid_uuid4.py

74695723-65ed-4170-af77-b9f22608535d
db199e25-e292-41cd-b488-80a8f99d163a
196750b3-bbb9-488e-b3ec-62ec0e468bbc
```

### Working with UUID Objects

除了生成新的UUID值之外，还可以解析标准格式的字符串以创建UUID对象，从而更容易处理比较和排序操作。

```python
#uuid_uuid_objects.py
import uuid

def show(msg, l):
    print(msg)
    for v in l:
        print(' ', v)
    print()

input_values = [
    'urn:uuid:f2f84497-b3bf-493a-bba9-7c68e6def80b',
    '{417a5ebb-01f7-4ed5-aeac-3d56cd5037b0}',
    '2115773a-5bf1-11dd-ab48-001ec200d9e0',
]

show('input_values', input_values)

uuids = [uuid.UUID(s) for s in input_values]
show('converted to uuids', uuids)

uuids.sort()
show('sorted', uuids)
```
从输入中删除周围的花括号，短划线（`-`）。 如果字符串的前缀包含`urn：`和`/`或`uuid：`，则也会将其删除。 其余文本必须是16个十六进制数字的字符串，然后将其解释为UUID值。

```
$ python3 uuid_uuid_objects.py

input_values
  urn:uuid:f2f84497-b3bf-493a-bba9-7c68e6def80b
  {417a5ebb-01f7-4ed5-aeac-3d56cd5037b0}
  2115773a-5bf1-11dd-ab48-001ec200d9e0

converted to uuids
  f2f84497-b3bf-493a-bba9-7c68e6def80b
  417a5ebb-01f7-4ed5-aeac-3d56cd5037b0
  2115773a-5bf1-11dd-ab48-001ec200d9e0

sorted
  2115773a-5bf1-11dd-ab48-001ec200d9e0
  417a5ebb-01f7-4ed5-aeac-3d56cd5037b0
  f2f84497-b3bf-493a-bba9-7c68e6def80b

```

## json — JavaScript Object Notation

目的：将Python对象编码为JSON字符串，并将JSON字符串解码为Python对象。  
json模块提供类似于pickle的API，用于将内存中的Python对象转换为称为JavaScript Object Notation（JSON）的序列化表示。 与pickle不同，JSON具有以多种语言（尤其是JavaScript）实现的优点。 它最广泛地用于REST API中的Web服务器和客户端之间的通信，但对于其他应用程序间通信需求也很有用。

### Encoding and Decoding Simple Data Types

编码器默认情况下理解Python的本机类型（str，int，float，list，tuple和dict）。

```python
#json_simple_types.py
import json

data = [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
print('DATA:', repr(data))

data_string = json.dumps(data)
print('JSON:', data_string)
```
值以类似于Python的repr（）输出的方式编码。

```
$ python3 json_simple_types.py

DATA: [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
JSON: [{"a": "A", "b": [2, 4], "c": 3.0}]
```
编码，然后重新解码可能不会给出完全相同类型的对象。

```python
#json_simple_types_decode.py
import json

data = [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
print('DATA   :', data)

data_string = json.dumps(data)
print('ENCODED:', data_string)

decoded = json.loads(data_string)
print('DECODED:', decoded)

print('ORIGINAL:', type(data[0]['b']))
print('DECODED :', type(decoded[0]['b']))
```
特别是，元组成为列表。

```
$ python3 json_simple_types_decode.py

DATA   : [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
ENCODED: [{"a": "A", "b": [2, 4], "c": 3.0}]
DECODED: [{'a': 'A', 'b': [2, 4], 'c': 3.0}]
ORIGINAL: <class 'tuple'>
DECODED : <class 'list'>
```

### Human-consumable vs. Compact Output

JSON相对于pickle的另一个好处是结果是人类可读的。 dumps（）函数接受几个参数以使输出更好。 例如，`sort_keys`标志告诉编码器以排序而非随机的顺序输出字典的键。

```python
#json_sort_keys.py
import json

data = [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
print('DATA:', repr(data))

unsorted = json.dumps(data)
print('JSON:', json.dumps(data))
print('SORT:', json.dumps(data, sort_keys=True))

first = json.dumps(data, sort_keys=True)
second = json.dumps(data, sort_keys=True)

print('UNSORTED MATCH:', unsorted == first)
print('SORTED MATCH  :', first == second)
```
排序使得通过眼睛扫描结果更容易，并且还可以在测试中比较JSON输出。

```
$ python3 json_sort_keys.py

DATA: [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
JSON: [{"a": "A", "b": [2, 4], "c": 3.0}]
SORT: [{"a": "A", "b": [2, 4], "c": 3.0}]
UNSORTED MATCH: True
SORTED MATCH  : True
```
对于高度嵌套的数据结构，请为缩进(indent)指定一个值，以便输出格式也很好。

```python
#json_indent.py
import json

data = [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
print('DATA:', repr(data))

print('NORMAL:', json.dumps(data, sort_keys=True))
print('INDENT:', json.dumps(data, sort_keys=True, indent=2))
```
当indent是非负整数时，输出更接近于pprint的输出，数据结构的每个级别的前导空格与缩进级别匹配。

```
$ python3 json_indent.py

DATA: [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
NORMAL: [{"a": "A", "b": [2, 4], "c": 3.0}]
INDENT: [
  {
    "a": "A",
    "b": [
      2,
      4
    ],
    "c": 3.0
  }
]
```
但是，这样的详细输出会增加传输相同数据量所需的字节数，因此不适合在生产环境中使用。 事实上，可以调整编码输出中分隔符的设置，使其比默认设置更紧凑。

```python
#json_compact_encoding.py
import json

data = [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
print('DATA:', repr(data))

print('repr(data)             :', len(repr(data)))

plain_dump = json.dumps(data)
print('dumps(data)            :', len(plain_dump))

small_indent = json.dumps(data, indent=2)
print('dumps(data, indent=2)  :', len(small_indent))

with_separators = json.dumps(data, separators=(',', ':'))
print('dumps(data, separators):', len(with_separators))
```
dumps（）的separators参数是一个元组，其中包含用于将列表中的项和字典中的键值分开的字符串。 默认值为（'，'，'：'）。 通过移除空白，可以产生更紧凑的输出。

```
$ python3 json_compact_encoding.py

DATA: [{'a': 'A', 'b': (2, 4), 'c': 3.0}]
repr(data)             : 35
dumps(data)            : 35
dumps(data, indent=2)  : 73
dumps(data, separators): 29
```

### Encoding Dictionaries

JSON格式要求字典的键是字符串。 尝试编码具有非字符串键的字典会产生TypeError。 解决该限制的一种方法是告诉编码器使用skipkeys参数跳过非字符串键：

```python
#json_skipkeys.py
import json

data = [{'a': 'A', 'b': (2, 4), 'c': 3.0, ('d',): 'D tuple'}]

print('First attempt')
try:
    print(json.dumps(data))
except TypeError as err:
    print('ERROR:', err)

print()
print('Second attempt')
print(json.dumps(data, skipkeys=True))
```
不会引发异常，而是忽略非字符串键。

```
$ python3 json_skipkeys.py

First attempt
ERROR: keys must be a string

Second attempt
[{"a": "A", "b": [2, 4], "c": 3.0}]
```

### Working with Custom Types

到目前为止，所有的例子都使用了Pythons内置类型，因为json原生支持这些类型。 通常需要对自定义类进行编码，有两种方法可以做到这一点。  
给出要编码的类：

```python
#json_myobj.py

class MyObj:

    def __init__(self, s):
        self.s = s

    def __repr__(self):
        return '<MyObj({})>'.format(self.s)
```
编码MyObj实例的简单方法是定义一个将未知类型转换为已知类型的函数。 它不需要进行编码，因此它应该只将一个对象转换为另一个对象。

```python
#json_dump_default.py
import json
import json_myobj

obj = json_myobj.MyObj('instance value goes here')

print('First attempt')
try:
    print(json.dumps(obj))
except TypeError as err:
    print('ERROR:', err)

def convert_to_builtin_type(obj):
    print('default(', repr(obj), ')')
    # Convert objects to a dictionary of their representation
    d = {
        '__class__': obj.__class__.__name__,
        '__module__': obj.__module__,
    }
    d.update(obj.__dict__)
    return d

print()
print('With default')
print(json.dumps(obj, default=convert_to_builtin_type))
```
在`convert_to_builtin_type（）`中，如果程序可以访问必要的Python模块，则json无法识别的类的实例将转换为具有足够信息的字典，以重新创建对象。

```
$ python3 json_dump_default.py

First attempt
ERROR: Object of type 'MyObj' is not JSON serializable

With default
default( <MyObj(instance value goes here)> )
{"__class__": "MyObj", "__module__": "json_myobj", "s": "instance value goes here"}
```

要解码结果并创建MyObj（）实例，请使用`object_hook`参数调用load（）以绑定到解码器，以便可以从模块导入类并用于创建实例。
调用object_hook解码传入数据流中的每个字典，从而提供将字典转换为另一种类型的对象的机会。 钩子函数应该返回调用程序应该接收的对象而不是字典。

```python
#json_load_object_hook.py
import json

def dict_to_object(d):
    if '__class__' in d:
        class_name = d.pop('__class__')
        module_name = d.pop('__module__')
        module = __import__(module_name)
        print('MODULE:', module.__name__)
        class_ = getattr(module, class_name)
        print('CLASS:', class_)
        args = {
            key: value
            for key, value in d.items()
        }
        print('INSTANCE ARGS:', args)
        inst = class_(**args)
    else:
        inst = d
    return inst

encoded_object = '''
    [{"s": "instance value goes here",
      "__module__": "json_myobj", "__class__": "MyObj"}]
    '''

myobj_instance = json.loads(
    encoded_object,
    object_hook=dict_to_object,
)
print(myobj_instance)
```
由于json将字符串值转换为unicode对象，因此需要将它们重新编码为ASCII字符串，然后才能将它们用作类构造函数的关键字参数。

```
$ python3 json_load_object_hook.py

MODULE: json_myobj
CLASS: <class 'json_myobj.MyObj'>
INSTANCE ARGS: {'s': 'instance value goes here'}
[<MyObj(instance value goes here)>]
```
类似的钩子可用于内置类型整数（`parse_int`），浮点数（`parse_float`）和常量（`parse_constant`）。

### Encoder and Decoder Classes

除了已经涵盖的便利功能外，json模块还提供用于编码和解码的类。 直接使用这些类可以访问额外的API以自定义其行为。
JSONEncoder使用可迭代接口来生成编码数据的“块”，从而更容易写入文件或网络套接字，而无需在内存中表示整个数据结构。

```python
#json_encoder_iterable.py
import json

encoder = json.JSONEncoder()
data = [{'a': 'A', 'b': (2, 4), 'c': 3.0}]

for part in encoder.iterencode(data):
    print('PART:', part)
```
输出以逻辑单元生成，而不是基于任何大小值。

```
$ python3 json_encoder_iterable.py

PART: [
PART: {
PART: "a"
PART: :
PART: "A"
PART: ,
PART: "b"
PART: :
PART: [2
PART: , 4
PART: ]
PART: ,
PART: "c"
PART: :
PART: 3.0
PART: }
PART: ]
```
encode（）方法基本上等同于`''.join（encoder.iterencode（））`，前面有一些额外的错误检查。  
要编码任意对象，请使用类似于`convert_to_builtin_type（）`中使用的实现覆盖default（）方法。

```python
#json_encoder_default.py
import json
import json_myobj

class MyEncoder(json.JSONEncoder):

    def default(self, obj):
        print('default(', repr(obj), ')')
        # Convert objects to a dictionary of their representation
        d = {
            '__class__': obj.__class__.__name__,
            '__module__': obj.__module__,
        }
        d.update(obj.__dict__)
        return d

obj = json_myobj.MyObj('internal data')
print(obj)
print(MyEncoder().encode(obj))
```
输出与先前的实现相同。

```
$ python3 json_encoder_default.py

<MyObj(internal data)>
default( <MyObj(internal data)> )
{"__class__": "MyObj", "__module__": "json_myobj", "s": "internal data"}
```
解码文本，然后将字典转换为对象需要比先前的实现更多的工作，但不多。

```python
#json_decoder_object_hook.py
import json

class MyDecoder(json.JSONDecoder):

    def __init__(self):
        json.JSONDecoder.__init__(
            self,
            object_hook=self.dict_to_object,
        )

    def dict_to_object(self, d):
        if '__class__' in d:
            class_name = d.pop('__class__')
            module_name = d.pop('__module__')
            module = __import__(module_name)
            print('MODULE:', module.__name__)
            class_ = getattr(module, class_name)
            print('CLASS:', class_)
            args = {
                key: value
                for key, value in d.items()
            }
            print('INSTANCE ARGS:', args)
            inst = class_(**args)
        else:
            inst = d
        return inst

encoded_object = '''
[{"s": "instance value goes here",
  "__module__": "json_myobj", "__class__": "MyObj"}]
'''

myobj_instance = MyDecoder().decode(encoded_object)
print(myobj_instance)
```
输出与前面的例子相同。

```
$ python3 json_decoder_object_hook.py

MODULE: json_myobj
CLASS: <class 'json_myobj.MyObj'>
INSTANCE ARGS: {'s': 'instance value goes here'}
[<MyObj(instance value goes here)>]
```

### Working with Streams and Files

到目前为止的所有示例都假设整个数据结构的编码版本可以一次保存在存储器中。 对于大型数据结构，最好将编码直接写入类文件对象。 便利函数load（）和dump（）接受对类文件对象的引用，以用于读取或写入。

```python
#json_dump_file.py
import io
import json

data = [{'a': 'A', 'b': (2, 4), 'c': 3.0}]

f = io.StringIO()
json.dump(data, f)

print(f.getvalue())
```
套接字或普通文件句柄的工作方式与本示例中使用的StringIO缓冲区的工作方式相同。

```
$ python3 json_dump_file.py

[{"a": "A", "b": [2, 4], "c": 3.0}]
```
虽然它没有被优化为一次只读取部分数据，但load（）函数仍然提供了“封装从输入流生成对象的逻辑的”好处。

```python
#json_load_file.py
import io
import json

f = io.StringIO('[{"a": "A", "c": 3.0, "b": [2, 4]}]')
print(json.load(f))
```
就像dump（）一样，任何类文件对象都可以传递给load（）。

```
$ python3 json_load_file.py

[{'a': 'A', 'c': 3.0, 'b': [2, 4]}]
```

### Mixed Data Streams

JSONDecoder包括`raw_decode（）`，一种用于解码数据结构的方法，后跟更多数据，例如带有尾随文本的JSON数据。 返回值是通过解码输入数据而创建的对象，以及指示解码停止的位置的索引。

```python
#json_mixed_data.py
import json

decoder = json.JSONDecoder()

def get_decoded_and_remainder(input_data):
    obj, end = decoder.raw_decode(input_data)
    remaining = input_data[end:]
    return (obj, end, remaining)

encoded_object = '[{"a": "A", "c": 3.0, "b": [2, 4]}]'
extra_text = 'This text is not JSON.'

print('JSON first:')
data = ' '.join([encoded_object, extra_text])
obj, end, remaining = get_decoded_and_remainder(data)

print('Object              :', obj)
print('End of parsed input :', end)
print('Remaining text      :', repr(remaining))

print()
print('JSON embedded:')
try:
    data = ' '.join([extra_text, encoded_object, extra_text])
    obj, end, remaining = get_decoded_and_remainder(data)
except ValueError as err:
    print('ERROR:', err)
```
不幸的是，这仅在对象出现在输入的开头时才有效。

```
$ python3 json_mixed_data.py

JSON first:
Object              : [{'a': 'A', 'c': 3.0, 'b': [2, 4]}]
End of parsed input : 35
Remaining text      : ' This text is not JSON.'

JSON embedded:
ERROR: Expecting value: line 1 column 1 (char 0)
```

### JSON at the Command Line

json.tool模块实现了一个命令行程序，用于重新格式化JSON数据以便于阅读。

```
[{"a": "A", "c": 3.0, "b": [2, 4]}]
```
输入文件example.json包含一个不按字母顺序排列的键的映射。 下面的第一个示例显示了按顺序重新格式化的数据，第二个示例使用`--sort-keys`在打印输出之前对映射键进行排序。

```
$ python3 -m json.tool example.json

[
    {
        "a": "A",
        "c": 3.0,
        "b": [
            2,
            4
        ]
    }
]

$ python3 -m json.tool --sort-keys example.json

[
    {
        "a": "A",
        "b": [
            2,
            4
        ],
        "c": 3.0
    }
]
```

## xmlrpc.client — Client Library for XML-RPC

目的：用于XML-RPC通信的客户端库。  
XML-RPC是一种基于HTTP和XML构建的轻量级远程过程调用协议。 xmlrpclib模块允许Python程序与以任何语言编写的XML-RPC服务器通信。
本节中的所有示例都使用xmlrpc_server.py中定义的服务器，该服务器在源码发布中提供，并包含在此处以供参考。

```python
#xmlrpc_server.py
from xmlrpc.server import SimpleXMLRPCServer
from xmlrpc.client import Binary
import datetime

class ExampleService:

    def ping(self):
        """Simple function to respond when called
        to demonstrate connectivity.
        """
        return True

    def now(self):
        """Returns the server current date and time."""
        return datetime.datetime.now()

    def show_type(self, arg):
        """Illustrates how types are passed in and out of
        server methods.

        Accepts one argument of any type.

        Returns a tuple with string representation of the value,
        the name of the type, and the value itself.

        """
        return (str(arg), str(type(arg)), arg)

    def raises_exception(self, msg):
        "Always raises a RuntimeError with the message passed in"
        raise RuntimeError(msg)

    def send_back_binary(self, bin):
        """Accepts single Binary argument, and unpacks and
        repacks it to return it."""
        data = bin.data
        print('send_back_binary({!r})'.format(data))
        response = Binary(data)
        return response

if __name__ == '__main__':
    server = SimpleXMLRPCServer(('localhost', 9000),
                                logRequests=True,
                                allow_none=True)
    server.register_introspection_functions()
    server.register_multicall_functions()

    server.register_instance(ExampleService())

    try:
        print('Use Control-C to exit')
        server.serve_forever()
    except KeyboardInterrupt:
        print('Exiting')
```

### Connecting to a Server

将客户端连接到服务器的最简单方法是实例化ServerProxy对象，为其提供服务器的URI。 例如，demo服务器在localhost的端口9000上运行。

```python
#xmlrpc_ServerProxy.py
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://localhost:9000')
print('Ping:', server.ping())
```
在这种情况下，服务的ping（）方法不带参数并返回单个布尔值。

```
$ python3 xmlrpc_ServerProxy.py

Ping: True
```
其他选项可用于支持备用传输。 HTTP和HTTPS都支持开箱即用，都具有基本身份验证功能。 要实现新的通信通道，只需要一个新的传输类。 这是一项有趣的练习，例如，通过SMTP实现XML-RPC可能。

```python
#xmlrpc_ServerProxy_verbose.py
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://localhost:9000',
                                   verbose=True)
print('Ping:', server.ping())
```
verbose选项提供有助于解决通信错误的调试信息。

```
$ python3 xmlrpc_ServerProxy_verbose.py

send: b'POST /RPC2 HTTP/1.1\r\nHost: localhost:9000\r\n
Accept-Encoding: gzip\r\nContent-Type: text/xml\r\n
User-Agent: Python-xmlrpc/3.5\r\nContent-Length: 98\r\n\r\n'
send: b"<?xml version='1.0'?>\n<methodCall>\n<methodName>
ping</methodName>\n<params>\n</params>\n</methodCall>\n"
reply: 'HTTP/1.0 200 OK\r\n'
header: Server header: Date header: Content-type header:
Content-length body: b"<?xml version='1.0'?>\n<methodResponse>\n
<params>\n<param>\n<value><boolean>1</boolean></value>\n</param>
\n</params>\n</methodResponse>\n"
Ping: True
```
如果需要备用系统，则可以从UTF-8更改默认编码。

```python
#xmlrpc_ServerProxy_encoding.py
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://localhost:9000',
                                   encoding='ISO-8859-1')
print('Ping:', server.ping())
```
服务器自动检测正确的编码。

```
$ python3 xmlrpc_ServerProxy_encoding.py

Ping: True
```
`allow_none`选项控制Python的None值是自动转换为nil值还是产生一个错误。

```python
#xmlrpc_ServerProxy_allow_none.py
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://localhost:9000',
                                   allow_none=False)
try:
    server.show_type(None)
except TypeError as err:
    print('ERROR:', err)

server = xmlrpc.client.ServerProxy('http://localhost:9000',
                                   allow_none=True)
print('Allowed:', server.show_type(None))
```
如果客户端不允许None，则会在本地引发错误；但如果服务端未配置为允许None，则也可以从服务器内引发错误。

```
$ python3 xmlrpc_ServerProxy_allow_none.py

ERROR: cannot marshal None unless allow_none is enabled
Allowed: ['None', "<class 'NoneType'>", None]
```

### Data Types

XML-RPC协议识别一组有限的常见数据类型。 这些类型可以作为参数传递或返回值并组合以创建更复杂的数据结构。

```python
#xmlrpc_types.py
import xmlrpc.client
import datetime

server = xmlrpc.client.ServerProxy('http://localhost:9000')

data = [
    ('boolean', True),
    ('integer', 1),
    ('float', 2.5),
    ('string', 'some text'),
    ('datetime', datetime.datetime.now()),
    ('array', ['a', 'list']),
    ('array', ('a', 'tuple')),
    ('structure', {'a': 'dictionary'}),
]

for t, v in data:
    as_string, type_name, value = server.show_type(v)
    print('{:<12}: {}'.format(t, as_string))
    print('{:12}  {}'.format('', type_name))
    print('{:12}  {}'.format('', value))
```
The simple types are

```
$ python3 xmlrpc_types.py

boolean     : True
              <class 'bool'>
              True
integer     : 1
              <class 'int'>
              1
float       : 2.5
              <class 'float'>
              2.5
string      : some text
              <class 'str'>
              some text
datetime    : 20160618T19:31:47
              <class 'xmlrpc.client.DateTime'>
              20160618T19:31:47
array       : ['a', 'list']
              <class 'list'>
              ['a', 'list']
array       : ['a', 'tuple']
              <class 'list'>
              ['a', 'tuple']
structure   : {'a': 'dictionary'}
              <class 'dict'>
              {'a': 'dictionary'}
```
可以嵌套支持的类型以创建任意复杂度的值。

```python
#xmlrpc_types_nested.py
import xmlrpc.client
import datetime
import pprint

server = xmlrpc.client.ServerProxy('http://localhost:9000')

data = {
    'boolean': True,
    'integer': 1,
    'floating-point number': 2.5,
    'string': 'some text',
    'datetime': datetime.datetime.now(),
    'array1': ['a', 'list'],
    'array2': ('a', 'tuple'),
    'structure': {'a': 'dictionary'},
}
arg = []
for i in range(3):
    d = {}
    d.update(data)
    d['integer'] = i
    arg.append(d)

print('Before:')
pprint.pprint(arg, width=40)

print('\nAfter:')
pprint.pprint(server.show_type(arg)[-1], width=40)
```
此程序将包含所有受支持类型的字典列表传递给样本服务器，后者返回数据。 元组转换为列表，datetime实例转换为DateTime对象，其他数据不变。

```
$ python3 xmlrpc_types_nested.py

Before:
[{'array': ('a', 'tuple'),
  'boolean': True,
  'datetime': datetime.datetime(2016, 6, 18, 19, 27, 30, 45333),
  'floating-point number': 2.5,
  'integer': 0,
  'string': 'some text',
  'structure': {'a': 'dictionary'}},
 {'array': ('a', 'tuple'),
  'boolean': True,
  'datetime': datetime.datetime(2016, 6, 18, 19, 27, 30, 45333),
  'floating-point number': 2.5,
  'integer': 1,
  'string': 'some text',
  'structure': {'a': 'dictionary'}},
 {'array': ('a', 'tuple'),
  'boolean': True,
  'datetime': datetime.datetime(2016, 6, 18, 19, 27, 30, 45333),
  'floating-point number': 2.5,
  'integer': 2,
  'string': 'some text',
  'structure': {'a': 'dictionary'}}]

After:
[{'array': ['a', 'tuple'],
  'boolean': True,
  'datetime': <DateTime '20160618T19:27:30' at 0x101ecfac8>,
  'floating-point number': 2.5,
  'integer': 0,
  'string': 'some text',
  'structure': {'a': 'dictionary'}},
 {'array': ['a', 'tuple'],
  'boolean': True,
  'datetime': <DateTime '20160618T19:27:30' at 0x101ecfcc0>,
  'floating-point number': 2.5,
  'integer': 1,
  'string': 'some text',
  'structure': {'a': 'dictionary'}},
 {'array': ['a', 'tuple'],
  'boolean': True,
  'datetime': <DateTime '20160618T19:27:30' at 0x101ecfe10>,
  'floating-point number': 2.5,
  'integer': 2,
  'string': 'some text',
  'structure': {'a': 'dictionary'}}]
```
XML-RPC支持将日期作为本机类型，xmlrpclib可以使用两个类中的一个来表示传出代理中的日期值或从服务器接收的日期值。

```python
#xmlrpc_ServerProxy_use_datetime.py
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://localhost:9000',
                                   use_datetime=True)
now = server.now()
print('With:', now, type(now), now.__class__.__name__)

server = xmlrpc.client.ServerProxy('http://localhost:9000',
                                   use_datetime=False)
now = server.now()
print('Without:', now, type(now), now.__class__.__name__)
```
默认情况下，使用DateTime的内部版本，但`use_datetime`选项打开了对使用datetime模块中的类的支持。

```
$ python3 source/xmlrpc.client/xmlrpc_ServerProxy_use_datetime.py

With: 2016-06-18 19:18:31 <class 'datetime.datetime'> datetime
Without: 20160618T19:18:31 <class 'xmlrpc.client.DateTime'> DateTime
```

### Passing Objects

Python类的实例被视为结构并作为字典传递，对象的属性作为字典中的值。

```python
#xmlrpc_types_object.py
import xmlrpc.client
import pprint

class MyObj:

    def __init__(self, a, b):
        self.a = a
        self.b = b

    def __repr__(self):
        return 'MyObj({!r}, {!r})'.format(self.a, self.b)

server = xmlrpc.client.ServerProxy('http://localhost:9000')

o = MyObj(1, 'b goes here')
print('o  :', o)
pprint.pprint(server.show_type(o))

o2 = MyObj(2, o)
print('\no2 :', o2)
pprint.pprint(server.show_type(o2))
```
当值从服务器发送回客户端时，结果是客户端上的字典，因为值中没有任何编码信息可以告诉服务器（或客户端）它应该作为类的一部分进行实例化。

```
$ python3 xmlrpc_types_object.py

o  : MyObj(1, 'b goes here')
["{'b': 'b goes here', 'a': 1}", "<class 'dict'>",
{'a': 1, 'b': 'b goes here'}]

o2 : MyObj(2, MyObj(1, 'b goes here'))
["{'b': {'b': 'b goes here', 'a': 1}, 'a': 2}",
 "<class 'dict'>",
 {'a': 2, 'b': {'a': 1, 'b': 'b goes here'}}]
```

### Binary Data

传递给服务器的所有值都将自动编码和转义。 但是，某些数据类型可能包含非有效XML的字符。 例如，二进制图像数据可以包括ASCII控制范围0到31中的字节值。要传递二进制数据，最好使用Binary类对其进行编码以进行传输。

```python
#xmlrpc_Binary.py
import xmlrpc.client
import xml.parsers.expat

server = xmlrpc.client.ServerProxy('http://localhost:9000')

s = b'This is a string with control characters\x00'
print('Local string:', s)

data = xmlrpc.client.Binary(s)
response = server.send_back_binary(data)
print('As binary:', response.data)

try:
    print('As string:', server.show_type(s))
except xml.parsers.expat.ExpatError as err:
    print('\nERROR:', err)
```
如果将包含NULL字节的字符串传递给show_type（），则在处理响应时会在XML解析器中引发异常。

```
$ python3 xmlrpc_Binary.py

Local string: b'This is a string with control characters\x00'
As binary: b'This is a string with control characters\x00'

ERROR: not well-formed (invalid token): line 6, column 55
```
Binary对象也可用于使用pickle发送对象。 与通过网线发送多少可执行代码相关的正常安全问题在此处适用（即，除非通信信道是安全的，否则不执行此操作）。

```python
import xmlrpc.client
import pickle
import pprint

class MyObj:

    def __init__(self, a, b):
        self.a = a
        self.b = b

    def __repr__(self):
        return 'MyObj({!r}, {!r})'.format(self.a, self.b)

server = xmlrpc.client.ServerProxy('http://localhost:9000')

o = MyObj(1, 'b goes here')
print('Local:', id(o))
print(o)

print('\nAs object:')
pprint.pprint(server.show_type(o))

p = pickle.dumps(o)
b = xmlrpc.client.Binary(p)
r = server.send_back_binary(b)

o2 = pickle.loads(r.data)
print('\nFrom pickle:', id(o2))
pprint.pprint(o2)
```
Binary实例的data属性包含对象的pickle版本，因此在使用之前必须进行unpickled。 这会导致不同的对象（具有新的id值）。

```
$ python3 xmlrpc_Binary_pickle.py

Local: 4327262304
MyObj(1, 'b goes here')

As object:
["{'a': 1, 'b': 'b goes here'}", "<class 'dict'>",
{'a': 1, 'b': 'b goes here'}]

From pickle: 4327262472
MyObj(1, 'b goes here')
```

### Exception Handling

由于XML-RPC服务器可能使用任何语言编写，因此无法直接传输异常类。 相反，服务器中引发的异常将转换为Fault对象，并在客户端本地引发异常。

```python
#xmlrpc_exception.py
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://localhost:9000')
try:
    server.raises_exception('A message')
except Exception as err:
    print('Fault code:', err.faultCode)
    print('Message   :', err.faultString)
```
原始错误消息保存在faultString属性中，faultCode设置为XML-RPC错误号。

```
$ python3 xmlrpc_exception.py

Fault code: 1
Message   : <class 'RuntimeError'>:A message
```

### Combining Calls Into One Message

Multicall是XML-RPC协议的扩展，允许同时发送多个调用，响应被收集并返回给调用者。

```python
#xmlrpc_MultiCall.py
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://localhost:9000')

multicall = xmlrpc.client.MultiCall(server)
multicall.ping()
multicall.show_type(1)
multicall.show_type('string')

for i, r in enumerate(multicall()):
    print(i, r)
```
要使用MultiCall实例，请像ServerProxy一样调用其上的方法，然后调用不带参数的对象来实际运行远程函数。 返回值是一个迭代器，它产生（yield）所有调用的结果。

```
$ python3 xmlrpc_MultiCall.py

0 True
1 ['1', "<class 'int'>", 1]
2 ['string', "<class 'str'>", 'string']
```
如果其中一个调用导致Fault，则从迭代器生成结果时会引发异常并且没有更多结果可用。

```python
#xmlrpc_MultiCall_exception.py
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://localhost:9000')

multicall = xmlrpc.client.MultiCall(server)
multicall.ping()
multicall.show_type(1)
multicall.raises_exception('Next to last call stops execution')
multicall.show_type('string')

try:
    for i, r in enumerate(multicall()):
        print(i, r)
except xmlrpc.client.Fault as err:
    print('ERROR:', err)
```
由于第三个响应（来自`raises_exception（）`）生成异常，因此无法访问`show_type（）`的响应。

```
$ python3 xmlrpc_MultiCall_exception.py

0 True
1 ['1', "<class 'int'>", 1]
ERROR: <Fault 1: "<class 'RuntimeError'>:Next to last call stops execution">
```

## xmlrpc.server — An XML-RPC server

目的：实现XML-RPC服务器。  
xmlrpc.server模块包含用于使用XML-RPC协议创建跨平台，与语言无关的服务器的类。 除了Python之外，还存在许多其他语言的客户端库，使得XML-RPC成为构建RPC样式服务的简单选择。

>注意  
此处提供的所有示例都包括客户端模块以及与演示服务器交互。 要运行这些示例，请使用两个单独的shell窗口，一个用于服务器，另一个用于客户端。

### A Simple Server

这个简单的服务器示例公开了一个接收目录名称并返回其内容的函数。 第一步是创建SimpleXMLRPCServer实例并告诉它在哪里监听传入请求（在本例中为'localhost'端口9000）。 然后将函数定义为服务的一部分，并进行注册，以便服务器知道如何调用它。 最后一步是将服务器置于接收和响应请求的无限循环中。

>警告  
此实现具有明显的安全隐患。 不要在开放Internet上的服务器上或安全性可能存在问题的任何环境中运行它。

```python
#xmlrpc_function.py
from xmlrpc.server import SimpleXMLRPCServer
import logging
import os

# Set up logging
logging.basicConfig(level=logging.INFO)

server = SimpleXMLRPCServer(
    ('localhost', 9000),
    logRequests=True,
)

# Expose a function
def list_contents(dir_name):
    logging.info('list_contents(%s)', dir_name)
    return os.listdir(dir_name)

server.register_function(list_contents)

# Start the server
try:
    print('Use Control-C to exit')
    server.serve_forever()
except KeyboardInterrupt:
    print('Exiting')
```
可以使用xmlrpc.client在URL `http：//localhost：9000`访问服务器。 此客户端代码说明了如何从Python调用list_contents（）服务。

```python
#xmlrpc_function_client.py
import xmlrpc.client

proxy = xmlrpc.client.ServerProxy('http://localhost:9000')
print(proxy.list_contents('/tmp'))
```
ServerProxy使用其基本URL连接到服务器，然后直接在代理上调用方法。 在代理上调用的每个方法都被转换为对服务器的请求。 参数使用XML格式化，然后通过POST消息发送到服务器。 服务器解压缩XML并根据从客户端调用的方法名称确定要调用的函数。 参数传递给函数，返回值被转换回XML以返回给客户端。

启动服务器会提供以下输出。

```
$ python3 xmlrpc_function.py

Use Control-C to exit
```
在第二个窗口中运行客户端显示`/tmp`目录的内容。

```
$ python3 xmlrpc_function_client.py

['com.apple.launchd.aoGXonn8nV', 'com.apple.launchd.ilryIaQugf',
'example.db.db',
'KSOutOfProcessFetcher.501.ppfIhqX0vjaTSb8AJYobDV7Cu68=',
'pymotw_import_example.shelve.db']
```
请求完成后，日志输出将显示在服务器窗口中。

```
$ python3 xmlrpc_function.py

Use Control-C to exit
INFO:root:list_contents(/tmp)
127.0.0.1 - - [18/Jun/2016 19:54:54] "POST /RPC2 HTTP/1.1" 200 -
```
第一行输出来自`list_contents（）`内的logging.info（）调用。 第二行来自服务器记录请求，因为logRequests为True。

### Alternate API Names

有时，模块或库中使用的函数名称不是在外部使用的API名称。名称可能会更改，因为加载了特定于平台的实现，服务API是基于配置文件动态构建的，或者实际函数可以替换为存根（stubs）以进行测试。要使用替换名称注册函数，请将替换名称作为第二个参数传递给register_function（）。

```python
#xmlrpc_alternate_name.py
from xmlrpc.server import SimpleXMLRPCServer
import os

server = SimpleXMLRPCServer(('localhost', 9000))

def list_contents(dir_name):
    "Expose a function with an alternate name"
    return os.listdir(dir_name)

server.register_function(list_contents, 'dir')

try:
    print('Use Control-C to exit')
    server.serve_forever()
except KeyboardInterrupt:
    print('Exiting')
```
客户端现在应该使用名称`dir（）`而不是`list_contents（）`。

```python
#xmlrpc_alternate_name_client.py
import xmlrpc.client

proxy = xmlrpc.client.ServerProxy('http://localhost:9000')
print('dir():', proxy.dir('/tmp'))
try:
    print('\nlist_contents():', proxy.list_contents('/tmp'))
except xmlrpc.client.Fault as err:
    print('\nERROR:', err)
```
调用`list_contents（）`会导致错误，因为服务器不再具有由该名称注册的处理程序。

```
$ python3 xmlrpc_alternate_name_client.py

dir(): ['com.apple.launchd.aoGXonn8nV',
'com.apple.launchd.ilryIaQugf', 'example.db.db',
'KSOutOfProcessFetcher.501.ppfIhqX0vjaTSb8AJYobDV7Cu68=',
'pymotw_import_example.shelve.db']

ERROR: <Fault 1: '<class \'Exception\'>:method "list_contents"
is not supported'>
```

### Dotted API Names

可以使用通常对Python标识符不合法的名称注册单个函数。 例如，可以在名称中包含句点（`.`）以分隔服务中的名称空间。 下一个示例扩展“目录”服务以添加“创建”和“删除”调用。所有功能都使用前缀`“dir.”`进行注册，以便同一服务器可以使用不同的前缀提供其他服务。此示例中的另一个不同之处是某些函数返回None，因此必须告知服务器将None值转换为nil值。

```python
#xmlrpc_dotted_name.py
from xmlrpc.server import SimpleXMLRPCServer
import os

server = SimpleXMLRPCServer(('localhost', 9000), allow_none=True)

server.register_function(os.listdir, 'dir.list')
server.register_function(os.mkdir, 'dir.create')
server.register_function(os.rmdir, 'dir.remove')

try:
    print('Use Control-C to exit')
    server.serve_forever()
except KeyboardInterrupt:
    print('Exiting')
```
要在客户端中调用服务功能，只需使用带点名称引用它们即可。

```python
#xmlrpc_dotted_name_client.py
import xmlrpc.client

proxy = xmlrpc.client.ServerProxy('http://localhost:9000')
print('BEFORE       :', 'EXAMPLE' in proxy.dir.list('/tmp'))
print('CREATE       :', proxy.dir.create('/tmp/EXAMPLE'))
print('SHOULD EXIST :', 'EXAMPLE' in proxy.dir.list('/tmp'))
print('REMOVE       :', proxy.dir.remove('/tmp/EXAMPLE'))
print('AFTER        :', 'EXAMPLE' in proxy.dir.list('/tmp'))
```
假设当前系统上没有`/tmp/EXAMPLE`文件，则示例客户端脚本的输出如下所示。

```
$ python3 xmlrpc_dotted_name_client.py

BEFORE       : False
CREATE       : None
SHOULD EXIST : True
REMOVE       : None
AFTER        : False
```

### Arbitrary API Names

另一个有趣的功能是能够使用其他无效的Python对象属性名称来注册函数。 此示例服务注册名为“multiply args”的函数。

```python
#xmlrpc_arbitrary_name.py
from xmlrpc.server import SimpleXMLRPCServer

server = SimpleXMLRPCServer(('localhost', 9000))

def my_function(a, b):
    return a * b

server.register_function(my_function, 'multiply args')

try:
    print('Use Control-C to exit')
    server.serve_forever()
except KeyboardInterrupt:
    print('Exiting')
```
由于注册名称包含空格，因此不能使用点表示法直接从proxy访问它。 但是，使用getattr（）可以。

```python
#xmlrpc_arbitrary_name_client.py
import xmlrpc.client

proxy = xmlrpc.client.ServerProxy('http://localhost:9000')
print(getattr(proxy, 'multiply args')(5, 5))
```
但是，应避免使用这样的名称创建服务。提供此示例不是因为它是一个好主意，而是因为存在具有任意名称的现有服务，并且新程序可能需要能够调用它们。

```
$ python3 xmlrpc_arbitrary_name_client.py

25
```

### Exposing Methods of Objects

前面的部分讨论了使用良好的命名约定和命名空间来建立API的技术。将命名空间合并到API中的另一种方法是使用类的实例并公开它们的方法。 可以使用具有单个方法的实例重新创建第一个示例。

```python
#xmlrpc_instance.py
from xmlrpc.server import SimpleXMLRPCServer
import os
import inspect

server = SimpleXMLRPCServer(
    ('localhost', 9000),
    logRequests=True,
)

class DirectoryService:
    def list(self, dir_name):
        return os.listdir(dir_name)

server.register_instance(DirectoryService())

try:
    print('Use Control-C to exit')
    server.serve_forever()
except KeyboardInterrupt:
    print('Exiting')
```
客户端可以直接调用该方法。

```python
#xmlrpc_instance_client.py
import xmlrpc.client

proxy = xmlrpc.client.ServerProxy('http://localhost:9000')
print(proxy.list('/tmp'))
```
输出显示目录的内容。

```
$ python3 xmlrpc_instance_client.py

['com.apple.launchd.aoGXonn8nV', 'com.apple.launchd.ilryIaQugf',
'example.db.db',
'KSOutOfProcessFetcher.501.ppfIhqX0vjaTSb8AJYobDV7Cu68=',
'pymotw_import_example.shelve.db']
```
但是，服务的`“dir.”`前缀已经丢失。 可以通过定义类来设置可以从客户端调用的服务树来恢复它。

```python
#xmlrpc_instance_dotted_names.py
from xmlrpc.server import SimpleXMLRPCServer
import os
import inspect

server = SimpleXMLRPCServer(
    ('localhost', 9000),
    logRequests=True,
)

class ServiceRoot:
    pass

class DirectoryService:
    def list(self, dir_name):
        return os.listdir(dir_name)

root = ServiceRoot()
root.dir = DirectoryService()

server.register_instance(root, allow_dotted_names=True)

try:
    print('Use Control-C to exit')
    server.serve_forever()
except KeyboardInterrupt:
    print('Exiting')
```
通过在启用了`allow_dotted_names`的情况下注册ServiceRoot实例，服务器有权在请求进入时遍历对象树以使用getattr（）查找指定的方法。

```python
#xmlrpc_instance_dotted_names_client.py
import xmlrpc.client

proxy = xmlrpc.client.ServerProxy('http://localhost:9000')
print(proxy.dir.list('/tmp'))
```
dir.list（）的输出与之前的实现相同。

```
$ python3 xmlrpc_instance_dotted_names_client.py

['com.apple.launchd.aoGXonn8nV', 'com.apple.launchd.ilryIaQugf',
'example.db.db',
'KSOutOfProcessFetcher.501.ppfIhqX0vjaTSb8AJYobDV7Cu68=',
'pymotw_import_example.shelve.db']
```

### Dispatching Calls

默认情况下，`register_instance（）`查找名称不以下划线（`_`）开头的实例的所有可调用属性，并使用其名称注册它们。 为了更加小心暴露的方法，可以使用自定义调度逻辑。

```python
#xmlrpc_instance_with_prefix.py
from xmlrpc.server import SimpleXMLRPCServer
import os
import inspect

server = SimpleXMLRPCServer( ('localhost', 9000), logRequests=True,)

def expose(f):
    "Decorator to set exposed flag on a function."
    f.exposed = True
    return f

def is_exposed(f):
    "Test whether another function should be publicly exposed."
    return getattr(f, 'exposed', False)

class MyService:
    PREFIX = 'prefix'

    def _dispatch(self, method, params):
        # Remove our prefix from the method name
        if not method.startswith(self.PREFIX + '.'):
            raise Exception(
                'method "{}" is not supported'.format(method)
            )

        method_name = method.partition('.')[2]
        func = getattr(self, method_name)
        if not is_exposed(func):
            raise Exception(
                'method "{}" is not supported'.format(method)
            )

        return func(*params)

    @expose
    def public(self):
        return 'This is public'

    def private(self):
        return 'This is private'

server.register_instance(MyService())

try:
    print('Use Control-C to exit')
    server.serve_forever()
except KeyboardInterrupt:
    print('Exiting')
```
MyService的public（）方法被标记为暴露给XML-RPC服务，而private（）则没有。当客户端尝试访问属于MyService的函数时，将调用`_dispatch（）`方法。 它首先强制使用前缀（在本例中为`“prefix.”`，可以使用任何字符串）。 然后它需要函数具有一个名为exposed的属性，其值为true。 为方便起见，使用装饰器在函数上设置公开标志。  
以下示例包含一些示例客户端调用。

```python
#xmlrpc_instance_with_prefix_client.py
import xmlrpc.client

proxy = xmlrpc.client.ServerProxy('http://localhost:9000')
print('public():', proxy.prefix.public())
try:
    print('private():', proxy.prefix.private())
except Exception as err:
    print('\nERROR:', err)
try:
    print('public() without prefix:', proxy.public())
except Exception as err:
    print('\nERROR:', err)
```
结果输出以及捕获和报告的预期错误消息如下。

```
$ python3 xmlrpc_instance_with_prefix_client.py

public(): This is public

ERROR: <Fault 1: '<class \'Exception\'>:method "prefix.private" is
not supported'>

ERROR: <Fault 1: '<class \'Exception\'>:method "public" is not
supported'>
```
还有其他几种方法可以覆盖调度机制，包括直接从SimpleXMLRPCServer进行子类化。 有关更多详细信息，请参阅模块文档。

### Introspection API

与许多网络服务一样，可以查询XML-RPC服务器以询问它支持哪些方法并学习如何使用它们。SimpleXMLRPCServer包含一组用于执行此自省（introspection）的公共方法。 默认情况下，它们处于关闭状态，但可以使用`register_introspection_functions（）`启用。 通过在服务类上定义`_listMethods（）`和`_methodHelp（）`，可以将对`system.listMethods（）`和`system.methodHelp（）`的支持添加到服务中。

```python
#xmlrpc_introspection.py
from xmlrpc.server import (SimpleXMLRPCServer, list_public_methods)
import os
import inspect

server = SimpleXMLRPCServer(
    ('localhost', 9000),
    logRequests=True,
)
server.register_introspection_functions()

class DirectoryService:

    def _listMethods(self):
        return list_public_methods(self)

    def _methodHelp(self, method):
        f = getattr(self, method)
        return inspect.getdoc(f)

    def list(self, dir_name):
        """list(dir_name) => [<filenames>]

        Returns a list containing the contents of
        the named directory.

        """
        return os.listdir(dir_name)

server.register_instance(DirectoryService())

try:
    print('Use Control-C to exit')
    server.serve_forever()
except KeyboardInterrupt:
    print('Exiting')
```
在这种情况下，便捷函数`list_public_methods（）`扫描实例以返回不以下划线（`_`）开头的可调用属性的名称。 重新定义`_listMethods（）`以应用所需的任何规则。类似地，对于此基本示例，`_methodHelp（）`返回函数的docstring，但可以重新编写以从其他源(source)构建帮助字符串。  
此客户端查询服务器并报告所有可公开调用的方法。

```python
#xmlrpc_introspection_client.py
import xmlrpc.client

proxy = xmlrpc.client.ServerProxy('http://localhost:9000')
for method_name in proxy.system.listMethods():
    print('=' * 60)
    print(method_name)
    print('-' * 60)
    print(proxy.system.methodHelp(method_name))
    print()
```
系统方法包含在结果中。

```
$ python3 xmlrpc_introspection_client.py

============================================================
list
------------------------------------------------------------
list(dir_name) => [<filenames>]

Returns a list containing the contents of
the named directory.

============================================================
system.listMethods
------------------------------------------------------------
system.listMethods() => ['add', 'subtract', 'multiple']

Returns a list of the methods supported by the server.

============================================================
system.methodHelp
------------------------------------------------------------
system.methodHelp('add') => "Adds two integers together"

Returns a string containing documentation for the specified method.

============================================================
system.methodSignature
------------------------------------------------------------
system.methodSignature('add') => [double, int, int]

Returns a list describing the signature of the method. In the
above example, the add method takes two integers as arguments
and returns a double result.

This server does NOT support system.methodSignature.
```
