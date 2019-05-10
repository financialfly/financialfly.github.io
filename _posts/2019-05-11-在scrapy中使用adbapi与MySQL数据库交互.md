---
layout:     post                    
title:      在scrapy中使用adbapi与MySQL数据库交互              
subtitle:   更好的发挥scrapy的异步性能
date:       2019-05-11              
author:     financialfly                     
header-img: img/post-bg-os-metro.jpg    
catalog: true                       
tags:                               
    - python
    - scrapy
---

#### 简介
Twisted 是一个异步网络框架，不幸的是大部分数据库api实现只有阻塞式接口，twisted.enterprise.adbapi为此产生，它是DB-API 2.0 API的非阻塞接口，可以访问各种关系数据库。而scrapy正是基于twisted实现的异步爬虫框架，所以在使用scrapy的pipeline存储数据时，使用adbapi无疑能更好的利用异步的性能，减少爬虫阻塞。

#### 配置数据库接口
首先在项目的settings文件中配置如下字段：
```py
MYSQL_HOST = 'localhost' # 连接地址
MYSQL_PORT = 3306 # 端口号
MYSQL_USER = 'root' # 用户名
MYSQL_PWD = 'root' # 密码
MYSQL_DATABASE = 'test' # 数据库名称
```

#### 定义构造方法和`from_settings`方法
接下来在pipeline中定义构造方法来绑定连接池，并定义`from_settings`方法用来读取数据库配置：
```py
import logging
from twisted.enterprise import adbapi
from twisted.internet import defer

logger = logging.getLogger(__name__)

class TestPipeline(object):

    table = 'test' # 存储数据的表

    @classmethod
    def from_settings(cls, settings):
        '''创建对象时读取settings配置'''
        db_params = dict(
            host=settings['MYSQL_HOST'],
            port=settings['MYSQL_PORT'],
            user=settings['MYSQL_USER'],
            password=settings['MYSQL_PWD'],
            database=settings['MYSQL_DATABASE']
        )
        # 创建连接池，并实例化对象
        dbpool = adbapi.ConnectionPool('pymysql', **db_params)
        return cls(dbpool)

    def __init__(self, dbpool):
        self.dbpool = dbpool
```

#### 使用`dbpool`
`adbapi`提供了`runQuery`方法用来查询数据库，`runInteraction`方法用来调用方法更新/插入数据，一般情况下，只需要用到`runInteraction`，但有时候或许也需要查询结果获取相关数据之类的操作，所以这里用分别介绍两种方法的使用。
          
需要注意的是，在定义异步操作数据库时，需要使用`defer.inlineCallbacks`装饰器，以便接受执行结果的返回值和操作数据库。所以我们可以将与数据库打交道的操作封装到一个函数里面，并返回`item`，这样扩展的时候就方便多了。具体使用示例如下：
```py
    def process_item(self, item, spider):
        # do something or nothing
        return self.go_to_db(item)

    @defer.inlineCallbacks
    def go_to_db(self, item):
        # 获取查询结果
        result = yield self.query_something('test_field')
        if result:
            item['query_result'] = result
        # runInteraction调用执行存储数据的方法，并传入item
        yield self.dbpool.runInteraction(self.result, item)
        # 完成后使用returnValue返回item
        defer.returnValue(item)

    def query_something(self, field):
        # 查询方法，runQuery实际上就是返回的pymysql中cursor的fetchall结果
        # 不过返回的是一个defer对象
        query = 'SELECT {} FROM test LIMIT 1'.format(field)
        return self.dbpool.runQuery(query)

    def save(self, tx, item):
        # 存储item，参数tx即cursor，不用显示的传入，但必须定义，用于执行sql语句
        # 它会自动提交
        keys = '{}'.format(tuple(k for k, v in item.items() if v))
        values = tuple(v for v in item.values() if v)
        query = 'INSERT INTO {} {} VALUES {}'.format(self.table, keys, values)
        try:
            tx.execute(query)
            logger.info('Item saved.')
        except Exception as e:
            logger.error('Save error. reason: %s, query: %s, item: %s' % (e, query, item))
```

#### 参考
1.[官方接口文档](https://twistedmatrix.com/documents/18.7.0/api/twisted.enterprise.adbapi.ConnectionPool.html#runQuery)            
2.[scrapy pipelines中使用示例](https://www.programcreek.com/python/example/95823/twisted.enterprise.adbapi.ConnectionPool)       
3.[官方用法示例](https://twistedmatrix.com/documents/current/core/howto/rdbms.html)              
4.[Twisted Klein: Database Usage (runQuery返回值及其他相关用法)](https://notoriousno.blogspot.com/2016/08/twisted-klein-database-usage.html)    