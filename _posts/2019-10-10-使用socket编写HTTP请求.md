---
layout:     post                    
title:      使用socket编写HTTP请求              
subtitle:   裸写socket
date:       2019-10-10              
author:     financialfly                     
header-img: img/post-bg-os-metro.jpg    
catalog: true                       
tags:                               
    - python
---

### 前言

在做爬虫的时候，经常使用的是`requests`等高级模块进行操作，虽然很方便，但是仍然不免要想这样的方式是如何实现的呢？当然，不用想也知道一定会用到`socket`模块。在此不妨使用`socket`来实现一下简单的网络请求。大概思路如下：
```
    1. 写一个解析 URL 的类  用来解析并保存请求协议、地址、端口等信息
    2. 写一个请求类和响应类 分别用来包装请求地址、请求头、请求体和响应头、响应体等信息
    3. 写一个客户端类用于创建连接、发送请求、返回数据等
```


### 解析URL
地址解析可以直接调用`urlparse`方法，不过仍需要保存协议以及根据协议判定请求端口(如果未指定的话)等。`UrlParser`只是用来保存`url`解析结果，用于请求类的使用。
```py
from urllib.parse import urlparse


class UrlParser(object):

    def __init__(self, url):
        result = urlparse(url)

        if result.scheme == 'https':
            self.port = 443
            self.ssl = True
        elif result.scheme == 'http':
            self.port = 80
            self.ssl = False

        if result.port:
            self.port = result.port

        self.path = result.path or '/'
        if result.query:
            self.path += '?' + result.query

        self._r = result

    def __getattr__(self, item):
        return getattr(self._r, item)
```

### 封装Request和Response
在发起请求前，我们需要两个对象用来存储请求信息和响应信息：
```py
class Request(object):
    """请求对象"""
    def __init__(self, url, method='GET', headers=None, body=None):
        self.url = url
        self.method = method
        self.body = body
        self.headers = headers or {}


class Response(object):
    """响应对象"""
    def __init__(self, request, headers=None, body=None, status=None):
        self.request = request
        self.headers = headers
        self.body = body
        self.status = status
```


### 网络请求
当然，网络请求才是重点。如何将发送的数据组装为正确的格式，可以参考`HTTP权威指南`。在与服务器建立连接时，首先应看地址是否指定了端口号，如果没有则需要根据是否为`HTTPS`连接来判定，一般`HTTP`连接默认的端口是`80`，而`HTTPS`的则为`443`。如果是`HTTPS`，我们还需要对`socket`连接包装一下才行(通过`ssl`模块提供的`wrap_socket`方法)。    
建立连接后，就需要发送请求头和请求体(如果有)信息了。请求头每行以`\r\n`结束，且与请求体相隔一个`\r\n`。如果有请求体，还需要传入`Content-length`和`Content-Type`来指定请求体长度和类型。具体的请求体类型可以参考这篇文章：[四种常见的 POST 提交数据方式 | JerryQu 的小站](https://imququ.com/post/four-ways-to-post-data-in-http.html)，这里为了方便，就没有具体指定类型和长度了。请求体最后以`\r\n\r\n`结束，注意请求头和请求体的数据都是`bytes`类型。    
具体代码如下：
```py
import socket
import ssl as _ssl

from collections import defaultdict
from urllib.parse import urlparse
from http.client import HTTPResponse


class ClientSession(object):

    def __init__(self):
        self._sk = None

    def _make_sk(self, ssl=True):
        """创建socket对象"""
        sk = socket.socket()
        if ssl:
            sk = _ssl.wrap_socket(sk)
        self._sk = sk

    def _make_buffer(self, request, url):
        """组装请求头和请求体"""
        buffer = []
        buffer.append(f'{request.method} {url.path} HTTP/1.1')

        if not 'host' in request.headers:
            buffer.append(f'host: {url.hostname}')

        for k, v in request.headers.items():
            buffer.append(f'{k}: {v}')

        body = request.body
        if body is not None:
            buffer.append('\r\n')
            if isinstance(body, dict):
                buffer.append('&'.join(f'{k}={v}' for k, v in body.items()))
            elif isinstance(body, list):
                buffer.append('&'.join(f'{k}={v}' for k, v in body))
            elif isinstance(body, str):
                buffer.append(body)
            else:
                pass
        buffer.append('\r\n')
        return bytes('\r\n'.join(buffer), 'utf8')

    def _make_response(self, request):
        """封装为Response对象并返回"""
        r = HTTPResponse(self._sk, method=request.method)
        r.begin()

        headers = defaultdict(str)
        for k, v in r.headers.items():
            headers[k] += v

        return Response(request, r.headers, body=r.read(), status=r.status)

    def request(self, request):
        """发起请求"""
        url = UrlParser(request.url)
        if self._sk is None:
            self._make_sk(url.ssl)
        self._sk.connect((url.hostname, url.port))
        self._sk.send(self._make_buffer(request, url))
        return self._make_response(request)

    def close(self):
        if self._sk is not None:
            self._sk.close()
```
最后在封装`Response`对象时，偷懒就没有自己做解析了，而是引用了标准库的`http.client`模块中的`HTTPResponse`对象。该对象初始化时接收一个`socket`实例，调用`begin`时它就会自动解析服务器返回的响应头数据。而`read`方法则会读取并返回响应体数据。如果要手动接收这些数据，调用`socket`的`recv`方法即可，注意该方法是阻塞的，它会在以下三种情况返回：
```
    1. 接收到了服务器的数据；
    2. 服务器关闭了连接；
    3. 网络发生了错误。
```
如果使用死循环来不断接收数据，当没有数据时就退出循环，比如下面的例子。
```py
while True:
    data = self._sk.recv(1024)
    if not data:
        break
```
这样的操作可能不会按照预期进行。原因是`HTTP1.1`中的`Connection`默认为`Keep-Alive`，即使服务端已经发送完数据，仍然会等到`5`秒后再断开连接，而客户端不断调用`recv`则会阻塞到服务器关闭连接或网络发生错误。解决的办法有：

1. 在请求头中加入`Connection: close`方法告诉服务端发送完数据后主动关闭连接。
2. 根据服务端返回的请求头中的`Content-Length`来判断数据接收长度，如果达到指定长度则主动退出。
3. 使用非阻塞`socket`。

### 测试
最后测试一下写的代码：
```py
>>> headers = {
...    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
...    'Accept-Language': 'en',
...    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36'
...}
>>> request = Request('http://www.jszyfw.com/techInfoCtr/static/toapptech.html?tdsourcetag=s_pcqq_aiomsg', headers=headers)
>>> s = ClientSession()
>>> r = s.request(request)
>>> r.status
200
>>> r.headers
Server: nginx/1.13.7
Date: Thu, 10 Oct 2019 06:26:27 GMT
Content-Type: text/html;charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Content-Language: en
>>> s.close()
```

### 参考
1. [http.client --- HTTP 协议客户端 — Python 3.7.5rc1 文档](https://docs.python.org/zh-cn/3/library/http.client.html)
2. [socket --- 底层网络接口 — Python 3.7.5rc1 文档](https://docs.python.org/zh-cn/3/library/socket.html)