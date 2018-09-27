---
title: PyMOTW-3 --- Cryptography
categories: language
tags: PyMOTW-3
---

加密可以保护消息，验证准确性并防止被拦截。Python的加密支持包括使用标准算法（如MD5和SHA）生成消息内容签名的hashlib，以及验证消息在传输中未被更改的hmac。

## hashlib — Cryptographic Hashing

目的：加密的哈希和消息摘要  
hashlib模块定义了用于访问不同加密散列算法的API。要使用特定的哈希算法，请使用适当的构造函数或new（）来创建哈希对象。从那里开始，无论使用何种算法，对象都使用相同的API。

### Hash Algorithms

由于hashlib由OpenSSL“支持”，因此该库提供的所有算法都可用，包括：

+ md5
+ sha1
+ sha224
+ sha256
+ sha384
+ sha512

有些算法可用于所有平台，有些算法依赖于底层库。有关每个列表，请分别查看`algorithms_guaranteed`和`algorithms_available`。

```python
#hashlib_algorithms.py
import hashlib

print('Guaranteed:\n{}\n'.format(', '.join(sorted(hashlib.algorithms_guaranteed))))
print('Available:\n{}'.format(', '.join(sorted(hashlib.algorithms_available))))
```
```
$ python3 hashlib_algorithms.py

Guaranteed:
blake2b, blake2s, md5, sha1, sha224, sha256, sha384, sha3_224,
sha3_256, sha3_384, sha3_512, sha512, shake_128, shake_256

Available:
DSA, DSA-SHA, MD4, MD5, RIPEMD160, SHA, SHA1, SHA224, SHA256,
SHA384, SHA512, blake2b, blake2s, dsaEncryption, dsaWithSHA,
ecdsa-with-SHA1, md4, md5, ripemd160, sha, sha1, sha224, sha256,
sha384, sha3_224, sha3_256, sha3_384, sha3_512, sha512,
shake_128, shake_256, whirlpool
```

### Sample Data

本节中的所有示例都使用相同的示例数据：

```python
#hashlib_data.py
import hashlib

lorem = '''Lorem ipsum dolor sit amet, consectetur adipisicing
elit, sed do eiusmod tempor incididunt ut labore et dolore magna
aliqua. Ut enim ad minim veniam, quis nostrud exercitation
ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis
aute irure dolor in reprehenderit in voluptate velit esse cillum
dolore eu fugiat nulla pariatur. Excepteur sint occaecat
cupidatat non proident, sunt in culpa qui officia deserunt
mollit anim id est laborum.'''
```

### MD5 Example

要计算数据块（这里unicode字符串被转换为字节字符串）的MD5哈希值或摘要，请先创建哈希对象，
然后添加数据并调用digest（）或hexdigest（）。

```python
#hashlib_md5.py
import hashlib

from hashlib_data import lorem

h = hashlib.md5()
h.update(lorem.encode('utf-8'))
print(h.hexdigest())
```
此示例使用hexdigest（）方法而不是digest（），因为输出已格式化，因此可以清晰地打印。 如果二进制摘要值可以接受，请使用digest（）。

```
$ python3 hashlib_md5.py

3f2fd2c9e25d60fb0fa5d593b802b7a8
```

### SHA1 Example

SHA1摘要以相同的方式计算。

```python
#hashlib_sha1.py
import hashlib

from hashlib_data import lorem

h = hashlib.sha1()
h.update(lorem.encode('utf-8'))
print(h.hexdigest())
```
摘要值在此示例中是不同的，因为算法从MD5更改为SHA1。

```
$ python3 hashlib_sha1.py

ea360b288b3dd178fe2625f55b2959bf1dba6eef
```

### Creating a Hash by Name

有时，使用字符串名称引用算法比通过直接使用构造函数更方便。例如，能够将哈希类型存储在配置文件中是有用的。
在此例中，使用new（）创建哈希计算器。

```python
#hashlib_new.py
import argparse
import hashlib
import sys

from hashlib_data import lorem

parser = argparse.ArgumentParser('hashlib demo')
parser.add_argument(
    'hash_name',
    choices=hashlib.algorithms_available,
    help='the name of the hash algorithm to use',
)
parser.add_argument(
    'data',
    nargs='?',
    default=lorem,
    help='the input data to hash, defaults to lorem ipsum',
)
args = parser.parse_args()

h = hashlib.new(args.hash_name)
h.update(args.data.encode('utf-8'))
print(h.hexdigest())
```
使用各种参数运行：

```
$ python3 hashlib_new.py sha1

ea360b288b3dd178fe2625f55b2959bf1dba6eef

$ python3 hashlib_new.py sha256

3c887cc71c67949df29568119cc646f46b9cd2c2b39d456065646bc2fc09ffd8

$ python3 hashlib_new.py sha512

a7e53384eb9bb4251a19571450465d51809e0b7046101b87c4faef96b9bc904cf7f90
035f444952dfd9f6084eeee2457433f3ade614712f42f80960b2fca43ff

$ python3 hashlib_new.py md5

3f2fd2c9e25d60fb0fa5d593b802b7a8
```

### Incremental Updates

可以重复调用哈希计算器的update（）方法。每次，摘要都会根据输入的附加文本进行更新。逐步更新比将整个文件读入内存更有效率，并产生相同的结果。

```python
#hashlib_update.py
import hashlib

from hashlib_data import lorem

h = hashlib.md5()
h.update(lorem.encode('utf-8'))
all_at_once = h.hexdigest()

def chunkize(size, text):
    "Return parts of the text in size-based increments."
    start = 0
    while start < len(text):
        chunk = text[start:start + size]
        yield chunk
        start += size
    return

h = hashlib.md5()
for chunk in chunkize(64, lorem.encode('utf-8')):
    h.update(chunk)
line_by_line = h.hexdigest()

print('All at once :', all_at_once)
print('Line by line:', line_by_line)
print('Same        :', (all_at_once == line_by_line))
```
此示例演示如何在读取或以其他方式生成数据时递增更新摘要。

```
$ python3 hashlib_update.py

All at once : 3f2fd2c9e25d60fb0fa5d593b802b7a8
Line by line: 3f2fd2c9e25d60fb0fa5d593b802b7a8
Same        : True
```

## hmac — Cryptographic Message Signing and Verification

目的：hmac模块实现用于消息验证的密钥散列，如RFC 2104中所述。  
HMAC算法可用于验证在应用程序之间传递或存储在可能易受攻击位置的信息的完整性。基本思想是生成“与共享密钥组合的”实际数据的加密哈希。然后，可以使用所得到的散列来检查所发送或存储的消息以确定信任级别，而不发送密钥。

>警告  
免责声明：我不是安全专家。有关HMAC的完整详细信息，请查看RFC 2104。

### Signing Messages

new（）函数创建一个用于计算消息签名的新对象。此示例使用默认的MD5哈希算法。

```python
#hmac_simple.py
import hmac

digest_maker = hmac.new(b'secret-shared-key-goes-here')

with open('lorem.txt', 'rb') as f:
    while True:
        block = f.read(1024)
        if not block:
            break
        digest_maker.update(block)

digest = digest_maker.hexdigest()
print(digest)
```
运行时，代码读取数据文件并为其计算HMAC签名。

```
$ python3 hmac_simple.py

4bcb287e284f8c21e87e14ba2dc40b16
```

### Alternate Digest Types

虽然hmac的默认加密算法是MD5，但这不是最安全的方法。MD5哈希有一些弱点，例如冲突（两个不同的消息产生相同的哈希）。 SHA-1算法被认为是更强的，应该使用它。

```python
#hmac_sha.py
import hmac
import hashlib

digest_maker = hmac.new(
    b'secret-shared-key-goes-here',
    b'',
    hashlib.sha1,
)

with open('hmac_sha.py', 'rb') as f:
    while True:
        block = f.read(1024)
        if not block:
            break
        digest_maker.update(block)

digest = digest_maker.hexdigest()
print(digest)
```
new（）函数有三个参数。第一个是密钥，应该在正在通信的两个端点之间共享，因此两端可以使用相同的值。第二个值是初始消息。如果需要进行身份验证的消息内容很小，例如时间戳或HTTP POST，则可以将消息的整个主体传递给new（），而不使用update（）方法。 最后一个参数是要使用的摘要模块。默认值为hashlib.md5。此示例传递'sha1'，导致hmac使用hashlib.sha1

```
$ python3 hmac_sha.py

dcee20eeee9ef8a453453f510d9b6765921cf099
```

### Binary Digests

前面的示例使用hexdigest（）方法生成可打印的摘要。 hexdigest是digest（）方法计算的值的不同表示，digest（）方法是一个二进制值，可能包含不可打印的字符，包括NUL。 某些网络服务（Google Checkout，Amazon S3）使用二进制摘要的base64编码版本而不是hexdigest。

```python
#hmac_base64.py
import base64
import hmac
import hashlib

with open('lorem.txt', 'rb') as f:
    body = f.read()

hash = hmac.new(
    b'secret-shared-key-goes-here',
    body,
    hashlib.sha1,
)

digest = hash.digest()
print(base64.encodestring(digest))
```
base64编码的字符串以换行符结尾，当将字符串嵌入到http标头或其他格式敏感的上下文中时，经常需要将换行符删除。

```
$ python3 hmac_base64.py

b'olW2DoXHGJEKGU0aE9fOwSVE/o4=\n'
```

### Applications of Message Signatures

HMAC身份验证应该用于任何公共网络服务，和数据存储在安全性很重要的地方的任何时候。例如，当通过管道或套接字发送数据时，应该对该数据进行签名，然后在使用数据之前测试签名。此处给出的扩展示例位于文件hmac_pickle.py中。  
第一步是建立一个函数来计算字符串的摘要，以及一个简单的类，用于实例化并通过通信通道传递。

```python
#hmac_pickle.py
import hashlib
import hmac
import io
import pickle
import pprint

def make_digest(message):
    "Return a digest for the message."
    hash = hmac.new(
        b'secret-shared-key-goes-here',
        message,
        hashlib.sha1,
    )
    return hash.hexdigest().encode('utf-8')

class SimpleObject:
    """Demonstrate checking digests before unpickling.
    """

    def __init__(self, name):
        self.name = name

    def __str__(self):
        return self.name
```
接下来，创建一个BytesIO缓冲区来模仿套接字或管道。该示例使用简单但易于解析的数据流格式。写入摘要和数据长度，跟着一个换行符。接着是用pickle生成的对象的序列化表示。真实系统不希望依赖于长度值，因为如果摘要错误，则长度也可能是错误的。某些不太可能出现在实际数据中的终结符序列会更合适。

然后，示例程序将两个对象写入流。第一个是使用正确的摘要值写入的。

```python
# Simulate a writable socket or pipe with a buffer
out_s = io.BytesIO()

# Write a valid object to the stream:
#  digest\nlength\npickle
o = SimpleObject('digest matches')
pickled_data = pickle.dumps(o)
digest = make_digest(pickled_data)
header = b'%s %d\n' % (digest, len(pickled_data))
print('WRITING: {}'.format(header))
out_s.write(header)
out_s.write(pickled_data)
```
使用无效摘要将第二个对象写入流中，通过计算某些其他数据的摘要而不是pickle生成的数据。

```python
# Write an invalid object to the stream
o = SimpleObject('digest does not match')
pickled_data = pickle.dumps(o)
digest = make_digest(b'not the pickled data at all')
header = b'%s %d\n' % (digest, len(pickled_data))
print('\nWRITING: {}'.format(header))
out_s.write(header)
out_s.write(pickled_data)

out_s.flush()
```
现在数据在BytesIO缓冲区中，可以再次读回。首先读取带有摘要和数据长度的数据行。然后使用长度值读取剩余数据。 pickle.load（）可以直接从流中读取，但是它假设有一个受信任的数据流，而这个数据可能没有足够的信任以unpikle它。 从流中读取pickle作为字符串，而不实际unpickle对象，更安全。

```python
# Simulate a readable socket or pipe with a buffer
in_s = io.BytesIO(out_s.getvalue())

# Read the data
while True:
    first_line = in_s.readline()
    if not first_line:
        break
    incoming_digest, incoming_length = first_line.split(b' ')
    incoming_length = int(incoming_length.decode('utf-8'))
    print('\nREAD:', incoming_digest, incoming_length)
```
一旦pickle数据在内存中，就可以重新计算摘要值并使用compare_digest（）将其与读取的摘要进行比较。 如果摘要匹配，则可以安全地信任数据并unpickle它。

```python
    incoming_pickled_data = in_s.read(incoming_length)

    actual_digest = make_digest(incoming_pickled_data)
    print('ACTUAL:', actual_digest)

    if hmac.compare_digest(actual_digest, incoming_digest):
        obj = pickle.loads(incoming_pickled_data)
        print('OK:', obj)
    else:
        print('WARNING: Data corruption')
```
输出显示第一个对象已经过验证，第二个对象被视为“已损坏”，如预期的那样。

```
$ python3 hmac_pickle.py

WRITING: b'f49cd2bf7922911129e8df37f76f95485a0b52ca 69\n'

WRITING: b'b01b209e28d7e053408ebe23b90fe5c33bc6a0ec 76\n'

READ: b'f49cd2bf7922911129e8df37f76f95485a0b52ca' 69
ACTUAL: b'f49cd2bf7922911129e8df37f76f95485a0b52ca'
OK: digest matches

READ: b'b01b209e28d7e053408ebe23b90fe5c33bc6a0ec' 76
ACTUAL: b'2ab061f9a9f749b8dd6f175bf57292e02e95c119'
WARNING: Data corruption
```
可以在定时攻击(timing attack)中使用简单的字符串比较或字节比较来比较两个摘要，以通过传递不同长度的摘要来暴露部分或全部密钥。 compare_digest（）实现快速但恒定时间的比较功能，以防止定时攻击。
