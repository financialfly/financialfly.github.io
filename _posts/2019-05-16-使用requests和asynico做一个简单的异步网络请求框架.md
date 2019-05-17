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

当面对小型的数据采集任务时，采用诸如`scrapy`等框架会觉得臃肿，而使用`requests`这样的阻塞型请求库又会觉得太耗时。所以自定义一个小巧而实用网络请求工具就很有必要了——它可以随时调用，速度也还行，并且代码也不多。         

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

    def __init__(self, url, params=None, headers=None, retry_times=3,
                 timeout=5, callback=None, meta=None,
                 cookies=None, proxies=None, method='GET', **kwargs):
        '''
        初始化
        '''
        self.url = url
        self._headers = headers
        self.retry_times = retry_times
        self.callback = callback
        self.meta = meta or dict()
        # 因为是基于requests的，所以这些参数都是直接作为关键字参数传递给requests.request方法
        self.params = dict(
            url=url,
            params=params,
            headers=self.headers,
            method=method,
            timeout=timeout,
            cookies=cookies,
            proxies=proxies,
            **kwargs
        )

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
    
    def __init__(self, url, data=None, json=None, method='POST',
             callback=None, meta=None, retry_times=3, headers=None, **kwargs):
        super().__init__(url=url, method=method, callback=callback,
                         meta=meta, retry_times=retry_times, headers=headers, **kwargs)
        self.params = dict(
            url=url,
            method=method,
            headers=self.headers,
            data=data,
            json=json,
            **kwargs
        )
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

logger = logging.getLogger('async_request.Crawler')

class Crawler(object):

    def __init__(self, requests, result_callback=None, logger=None):
        '''
        初始化crawler
        :param requests: Request请求列表
        :param result_callback: 请求结束后的结果处理回调函数
        '''
        self.requests = requests
        self.loop = asyncio.get_event_loop()
        self.result_callback = result_callback

    async def get_html(self, request):
        logger.debug('Crawling {}'.format(request.url))
        # 使用偏函数传递request参数
        future = self.loop.run_in_executor(None, partial(requests.request, **request.params))
        # 开始请求，如果失败则重试，并将重试次数减1
        while request.retry_times >= 0:
            try:
                response = await future
                break
            except Exception as e:
                logger.info('Error happen when crawling %s' % request.url)
                logger.error(e)
                request.retry_times -= 1
                logger.info('Retrying %s' % request.url)
        else:
            logger.info('Gave up retry %s, total retry %d times' % (request.url, request.retry_times + 1))
            # 重试都失败了则放弃，并返回一个空的Response对象，设置状态码为404
            response = requests.Response()
            response.status_code, response.url = 404, request.url

        logger.debug('[%d] Scraped from %s' % (response.status_code, response.url))
        # 传递meta
        response.meta = request.meta
        # 执行回调
        try:
            results = request.callback(response)
        except Exception as e:
            logger.error(e)
            return
        # 如果不可迭代则返回
        if not isinstance(results, types.GeneratorType):
            return
        # 检测结果，如果是Request，则添加到requests列表中准备继续请求，否则执行结果回调函数
        for x in results:
            if isinstance(x, Request):
                self.requests.append(x)
            elif self.result_callback:
                self.result_callback(x)

    def run(self, close_eventloop=True):
        '''启动函数'''
        try:
            # 如果requests列表中还有Request实例，则继续请求
            while self.requests:
                tasks = [self.get_html(req) for req in self.requests]
                # 清空请求列表
                self.requests.clear()
                self.loop.run_until_complete(asyncio.gather(*tasks))
        finally:
            if close_eventloop:
                self.loop.close()
                logger.debug('crawler stopped')
```
最后定义下启动`event_loop`的方法：
```py
    def run(self, close_eventloop=True):
        '''启动函数'''
        try:
            # 如果requests列表中还有Request实例，则继续请求
            while self.requests:
                tasks = [self.get_html(req) for req in self.requests]
                # 清空请求列表
                self.requests.clear()
                self.loop.run_until_complete(asyncio.gather(*tasks))
        finally:
            if close_eventloop:
                self.loop.close()
                logger.debug('crawler stopped')
```

##### 更新`xpath`解析
`xpath`是数据采集中常用到的解析规则，作为一个轻量级框架，虽然功能不能做太多，但是封装一个`xpath`功能还是可以的，下面就基于`lxml`着手定义一个极简的`XpathSelector`吧，为了顺手，方法名字就参照`scrapy`来了：
```py
from lxml import etree

class XpathSelector(object):

    def __init__(self, raw_text=None):
        self.html = None
        self._text = raw_text

    def get(self):
        '''获取一个结果'''
        try:
            return self.html.xpath(self.syntax)[0]
        except IndexError:
            return None

    def getall(self):
        '''获取所有结果'''
        return self.html.xpath(self.syntax)

    def __call__(self, syntax):
        '''只有传入解析规则的时候才解析网页，减少性能消耗'''
        self.syntax = syntax
        if self.html is None::
            self.html = etree.HTML(self._text)
        return self
```
如果你愿意，可以在请求完成后将`XpathSelector`作为属性赋值给`response`，那么就可以在回调方法中直接使用`response.xpath('...').get()`这样的方法了，但是请注意这样会稍微影响性能。具体操作可以在`Crawler`类的`get_html`方法中更新如下代码：
```py
        ...
        # 传递meta
        r.meta = request.meta
        # 创建XpathSelecto实例并绑定给response
        r.xpath = XpathSelector(raw_text=r.text)
        ...
```
OK，到此这个简单的框架也就做完了，最后再来封装一个启动函数，它接收一个`Request`请求列表，用来创建`Crawler`实例并运行`run`方法：
```py
def crawl(requests, result_callback=None, close_eventloop=True):
    c = Crawler(requests=requests, result_callback=result_callback)
    c.run(close_eventloop=close_eventloop)
```
##### 测试
最后简单使用一下：
```py
import async_request as ar

def parse_baidu(response):
    print(response.url, response.status_code)
    yield ar.Request('https://cn.bing.com/', callback=parse_bing)

def parse_bing(response):
    print(response.url, response.status_code)
    print(response.xpath('//a/@href').get())
    yield ar.Request('https://github.com/financialfly/async-request', callback=parse_github)

def parse_github(response):
    print(response.url, response.status_code)
    yield {'hello': 'github'}

def process_result(result):
    print(result)

request_list = [ar.Request(url='https://www.baidu.com', callback=parse_baidu)]
ar.crawl(request_list, result_callback=process_result)
```
结果正如预期：
```
https://www.baidu.com/ 200
https://cn.bing.com/ 200
javascript:void(0)
https://github.com/financialfly/async-request 200
{'hello': 'github'}
```

源码直达：[financialfly/async-request: 基于asyncio和requests做的轻量级异步网络请求工具](https://github.com/financialfly/async-request)