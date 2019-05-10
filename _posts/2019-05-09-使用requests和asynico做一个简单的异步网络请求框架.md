---
layout:     post                    
title:      使用requests和asynico做一个简单的异步网络请求框架              
subtitle:   Hello World
date:       2019-05-09              
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
首先定义一个基本的请求头，如果使用的时候没有传入该参数，就默认使用它：
```py
headers = {
  'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
  'Accept-Language': 'en',
  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.110 Safari/537.36'
}
```
接下来定义一个`Reuqet`类，并定义一个类属性`user_agents`，作为一种简单的反爬应对措施。
```py
class Request(object):
    user_agents = [
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0',
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.103 Safari/537.36',
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.140 Safari/537.36 Edge/18.17763',
        'Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 10.0; WOW64; Trident/7.0; .NET4.0C; .NET4.0E; .NET CLR 2.0.50727; .NET CLR 3.0.30729; .NET CLR 3.5.30729)'
    ]
```
然后是构造函数，它接收一个必要参数`url`，其它的都可以选择性传入：
```py
	def __init__(self, url, headers=None, retry_times=3,
	                 timeout=5, callback=None, meta=None,
	                 cookies=None, proxies=None):
	        self.url = url # 请求地址
	        self._headers = headers # 请求头
	        self.retry_times = retry_times # 重试次数
	        self.timeout = timeout # 超时
	        self.callback = callback # 回调函数
	        self.meta = meta or dict() # 携带数据，提供给回调函数使用
	        self.cookies = cookies # cookies
	        self.proxies = proxies # 代理
```
接下来定义一个属性`headers`，如果使用时没有传入请求头，它会随机从类属性`user_agents`中获取一个浏览器代理并返回包装好的请求头：
```py
    @property
    def headers(self):
        if self._headers is None:
            headers['User-Agent'] = random.choice(self.user_agents)
            return headers
        return self._headers
```
这样一个简单的`Request`对象就定义好了，接下来开始定义`Crawler`类。

##### 2.定义`Crawler`类
`Crawler`主要作用就是发起网络请求，并在成功后自动执行回调函数，失败了则重试。所以逻辑也很简单，首先导入需要的`requests`、`asyncio`库，并使用`logging`来记录请求的情况，同时还需要导入偏函数`partial`用来给requests传入参数：
```py
import asyncio
import logging
import requests
from functools import partial

class Crawler(object):

    def __init__(self, requests): 
        self.requests = requests # Request实例
        self.loop = asyncio.get_event_loop() # 实例化一个event_loop

    @property
    def logger(self):
    	'''定义logger'''
        return logging.getLogger('Crawler')
```
接下来定义网络请求函数：
```py
	async def get_html(self, request):
	        self.logger.debug('crawling %s' % request.url)
	        # 传入参数等待执行
	        future = self.loop.run_in_executor(None,
	                                           partial(requests.get,
	                                                   request.url,
	                                                   headers=request.headers,
	                                                   timeout=request.timeout,
	                                                   cookies=request.cookies,
	                                                   proxies=request.proxies
	                                                   ))
	        # 开始请求，如果失败则重试，并将重试次数减1
	        while request.retry_times >= 0:
	            try:
	                r = await future
	                r.meta = request.meta
	                break
	            except Exception as e:
	                self.logger.info('Error happen when crawling %s' % request.url)
	                self.logger.error(e)
	                request.retry_times -= 1
	                self.logger.info('Retrying %s' % request.url)
	        else:
	            # 重试都失败了则放弃，并返回一个空的Response对象，设置状态码为404
	            self.logger.info('Gave up retry %s' % (request.url))
	            r = requests.Response()
	            r.status_code, r.url = 404, request.url
	        # 指定回调函数
	        request.callback(r)
```
最后定义一个启动`event_loop`函数和关闭`event_loop`函数：
```py
	def run(self):
        tasks = [self.get_html(req) for req in self.requests]
        self.loop.run_until_complete(asyncio.gather(*tasks))

    def stop(self):
        self.loop.close()
        self.logger.debug('crawler stopped')
```

OK，到此这个简单的框架也就做完了，简单使用一下：
```py
urls = ['https://www.baidu.com/', 'https://cn.bing.com/', 'https://www.google.com/']

def parse(response):
    print(response.status_code)

reqs = [Request(url, callback=parse) for url in urls]
c = Crawler(reqs)
c.run()
c.stop()
```
结果简直完美：
```
2019-05-05 18:12:49 [Crawler] DEBUG crawling https://www.baidu.com/
2019-05-05 18:12:49 [Crawler] DEBUG crawling https://cn.bing.com/
2019-05-05 18:12:49 [Crawler] DEBUG crawling https://www.google.com/
200
200
2019-05-05 18:12:54 [Crawler] INFO Error happen when crawling https://www.google.com/
2019-05-05 18:12:54 [Crawler] ERROR HTTPSConnectionPool(host='www.google.com', port=443): Max retries exceeded with url: / (Caused by ConnectTimeoutError(<requests.packages.urllib3.connection.VerifiedHTTPSConnection object at 0x00000240121FF630>, 'Connection to www.google.com timed out. (connect timeout=5)'))
2019-05-05 18:12:54 [Crawler] INFO Retrying https://www.google.com/
2019-05-05 18:12:54 [Crawler] INFO Error happen when crawling https://www.google.com/
404
2019-05-05 18:12:54 [Crawler] ERROR HTTPSConnectionPool(host='www.google.com', port=443): Max retries exceeded with url: / (Caused by ConnectTimeoutError(<requests.packages.urllib3.connection.VerifiedHTTPSConnection object at 0x00000240121FF630>, 'Connection to www.google.com timed out. (connect timeout=5)'))
2019-05-05 18:12:54 [Crawler] INFO Retrying https://www.google.com/
2019-05-05 18:12:54 [Crawler] INFO Error happen when crawling https://www.google.com/
2019-05-05 18:12:54 [Crawler] ERROR HTTPSConnectionPool(host='www.google.com', port=443): Max retries exceeded with url: / (Caused by ConnectTimeoutError(<requests.packages.urllib3.connection.VerifiedHTTPSConnection object at 0x00000240121FF630>, 'Connection to www.google.com timed out. (connect timeout=5)'))
2019-05-05 18:12:54 [Crawler] INFO Retrying https://www.google.com/
2019-05-05 18:12:54 [Crawler] INFO Gave up retry https://www.google.com/
2019-05-05 18:12:54 [Crawler] DEBUG crawler stopped
```