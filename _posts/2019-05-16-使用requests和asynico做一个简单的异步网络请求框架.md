---
layout:     post                    
title:      使用requests和asynico做一个简单的异步网络请求框架              
subtitle:   一个类scrapy的mini框架
date:       2019-05-16              
author:     financialfly                     
header-img: img/post-bg-2015.jpg    
catalog: true                       
tags:                               
    - python
---

当面对小型的数据采集任务时，采用诸如scrapy等框架会觉得臃肿，而使用requests这样的阻塞型请求库又会觉得太耗时。所以自定义一个小巧而实用网络请求工具就很有必要了——它可以随时调用，速度也还行，并且代码也不多。         

整体设计分为两部分：一个是`Request`类，它包含要请求的地址、请求头、重试次数、超时、回调函数等等(基本上就是参照scrapy的Reuqest类)；另一个就是`Crawler`类，它接收一个或多个`Request`实例，负责运行它们并在成功后自动执行回调函数。      
具体步骤如下：
##### 1.定义`Request`类
首先建立`Request`对象，用来存储`url`，`headers`，`callback`，`meta`，`cookies`等信息，同时由于是基于`requests`做的，请求的时候会调用`requests`的`request`方法，所以还要指定一个`method`参数，默认为`GET`，同时可以再定义一个`FormRequest`来作为`POST`请求。
```py
import random
# 请求代理列表
USER_AGENTS = [
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0',
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.103 Safari/537.36',
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.140 Safari/537.36 Edge/18.17763',
        'Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 10.0; WOW64; Trident/7.0; .NET4.0C; .NET4.0E; .NET CLR 2.0.50727; .NET CLR 3.0.30729; .NET CLR 3.5.30729)',
        'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/30.0.1599.101',
        'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/38.0.2125.122',
        'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.71',
        'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.95',
        'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.71',
        'Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; QQDownload 732; .NET4.0C; .NET4.0E)',
        'Mozilla/5.0 (Windows NT 5.1; U; en; rv:1.8.1) Gecko/20061208 Firefox/2.0.0 Opera 9.50',
        'Mozilla/5.0 (Windows NT 6.1; WOW64; rv:34.0) Gecko/20100101 Firefox/34.0',
    ]

class Request(object):

    def __init__(self, url, headers=None, retry_times=3,
                 timeout=5, callback=None, meta=None,
                 cookies=None, proxies=None, data=None, json=None, method='GET'):
        '''
        因为是基于requests的，所以这些参数都是直接作为关键字参数传递给requests.request方法
        '''
        self.url = url
        self._headers = headers
        self.retry_times = retry_times
        self.timeout = timeout
        self.callback = callback
        self.meta = meta or dict()
        self.cookies = cookies
        self.proxies = proxies
        self.data = data
        self.json = json
        self.method = method

    @property
    def headers(self):
        '''如果没有传递headers参数，则默认从USER_AGENTS中选择一个'''
        if self._headers is None:
            return {
                'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
                'Accept-Language': 'en',
                'User-Agent': random.choice(USER_AGENTS)
            }
        return self._headers
    
class FormRquest(Request):
    
    def __init__(self, url, method='POST', *args, **kwargs):
        super().__init__(url=url, method=method, *args, **kwargs)
```
这样一个简单的`Request`对象就定义好了，接下来开始定义`Crawler`类。

##### 2.定义`Crawler`类
`Crawler`主要作用就是发起网络请求，并在成功后自动执行回调方法，失败了则重试。所以逻辑也很简单，首先导入需要的`requests`、`asyncio`库，并使用`logging`来记录请求的情况，同时还需要导入偏函数`partial`用来给`requests`传入参数，而如果想要自动处理采集的结果，则需要将处理结果的方法传入`result_callback`参数：
```py
import asyncio
import logging
import requests
import types
from functools import partial
from .request import Request

logging.basicConfig(level='DEBUG')

class Crawler(object):

    def __init__(self, requests, result_callback=None):
        '''
        初始化crawler
        :param requests: Request请求列表
        :param result_callback: 请求结束后的结果处理回调函数
        '''
        self.requests = requests
        self.loop = asyncio.get_event_loop()
        self.result_callback = result_callback

    async def get_html(self, request):
        logging.debug('Crawling {}'.format(request.url))
        # 传入参数等待执行
        future = self.loop.run_in_executor(None,
                                           partial(requests.request,
                                                   method=request.method,
                                                   url=request.url,
                                                   headers=request.headers,
                                                   timeout=request.timeout,
                                                   cookies=request.cookies,
                                                   proxies=request.proxies,
                                                   data=request.data,
                                                   json=request.json
                                                   ))
        # 开始请求，如果失败则重试，并将重试次数减1
        while request.retry_times >= 0:
            try:
                r = await future
                break
            except Exception as e:
                logging.info('Error happen when crawling %s' % request.url)
                logging.error(e)
                request.retry_times -= 1
                logging.info('Retrying %s' % request.url)
        else:
            logging.info('Gave up retry %s, total retry %d times' % (request.url, request.retry_times + 1))
            # 重试都失败了则放弃，并返回一个空的Response对象，设置状态码为404
            r = requests.Response()
            r.status_code, r.url = 404, request.url

        logging.debug('[%d] Scraped from %s' % (r.status_code, r.url))
        # 传递meta
        r.meta = request.meta
        results = request.callback(r)
        if not isinstance(results, types.GeneratorType):
            return
        # 检测结果，如果是Request，则添加到requests列表中准备继续请求，否则执行结果回调函数
        for x in results:
            if isinstance(x, Request):
                self.requests.append(x)
            elif self.result_callback:
                self.result_callback(x)
```
最后定义下启动`event_loop`和关闭`event_loop`方法：
```py
    def run(self):
        # 如果requests列表中还有Request实例，则继续请求
        while self.requests:
            tasks = [self.get_html(req) for req in self.requests]
            # 重置为空列表
            self.requests = list()
            self.loop.run_until_complete(asyncio.gather(*tasks))

    def stop(self):
        self.loop.close()
        self.logger.debug('crawler stopped')
```

OK，到此这个简单的框架也就做完了，简单使用一下：
```py
from async_request import Request, Crawler

def parse_baidu(response):
    print(response.url, response.status_code)
    yield Request('https://cn.bing.com/', callback=parse_bing)

def parse_bing(response):
    print(response.url, response.status_code)
    yield Request('https://github.com/financialfly/async-request', callback=parse_github)

def parse_github(response):
    print(response.url, response.status_code)
    yield {'hello': 'github'}

def process_result(result):
    print(result)

c = Crawler([Request(url='https://www.baidu.com', callback=parse_baidu)], result_callback=process_result)
c.run()
c.stop()
```
结果正如预期：
```
DEBUG:asyncio:Using selector: SelectSelector
DEBUG:Crawler:Crawling https://www.baidu.com
DEBUG:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.baidu.com
DEBUG:requests.packages.urllib3.connectionpool:https://www.baidu.com:443 "GET / HTTP/1.1" 200 None
https://www.baidu.com/ 200
DEBUG:Crawler:[200] Scraped from https://www.baidu.com/
DEBUG:Crawler:Crawling https://cn.bing.com/
DEBUG:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): cn.bing.com
DEBUG:requests.packages.urllib3.connectionpool:https://cn.bing.com:443 "GET / HTTP/1.1" 200 46546
https://cn.bing.com/ 200
DEBUG:Crawler:[200] Scraped from https://cn.bing.com/
DEBUG:Crawler:Crawling https://github.com/financialfly/async-request
DEBUG:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): github.com
DEBUG:requests.packages.urllib3.connectionpool:https://github.com:443 "GET /financialfly/async-request HTTP/1.1" 200 None
DEBUG:Crawler:[200] Scraped from https://github.com/financialfly/async-request
https://github.com/financialfly/async-request 200
{'hello': 'github'}
DEBUG:Crawler:crawler stopped
```

源码直达：[financialfly/async-request: 基于asyncio和requests做的轻量级异步网络请求工具](https://github.com/financialfly/async-request)