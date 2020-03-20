---
title: 高性能千万级定时任务管理服务forsun扩展开发之整合celery
date: 2020-03-20 10:58:48
tags: forsun
---

forsun是一个高性能高精度定时服务，可以轻松管理千万级定时任务。
项目地址： https://github.com/snower/forsun

forsun内置支持 shell、http、redis、thrift、beanstalk、mysql 六种到时触发回调执行器，但是很多时候自己的项目需求千奇百怪，单一的内置执行器并不能很好的在自己的项目中整合，所以forsun也支持通过扩展Extension开发的方式将自己编写的触发执行器Action注册进去。

那么我们就来轻松愉快实现一个整合celery的扩展吧。



# 示例(实现一个celery执行器扩展)

### 添加代码celery_extension.py

```python
# -*- coding: utf-8 -*-

import json
import logging
from concurrent.futures import ThreadPoolExecutor
from celery import Celery
from forsun.extension import Extension
from forsun.action.action import Action

app = Celery('hello', broker='amqp://guest@localhost//')
executor = ThreadPoolExecutor()


@app.task
def hello():
    return 'hello world'


@app.task
def add(x, y):
    return x + y


class CeleryAction(Action):
    METHODS = {
        "hello": hello,
        "add": add,
    }

    async def execute(self, *args, **kwargs):
        method = self.params.get("method", '')
        args = json.loads(self.params.get('args', '[]'))
        kwargs = json.loads(self.params.get('kwargs', '{}'))
        if method not in self.METHODS:
            logging.info("celery action execute unknow method %s", method)
            return
        executor.submit(self.METHODS[method], *tuple(args), **kwargs)
        logging.info("celery action execute %s", method)


class CeleryExtension(Extension):
    name = "celery"

    def register(self):
        self.register_action("celery", CeleryAction)
```

可以看出实现一个扩展非常简单，定义一个扩展类CeleryExtension继承自forsun.extension.Extension，添加一个执行器CeleryAction继承自forsun.extension.Action，起个名字，在扩展类register函数中注册执行器CeleryAction，搞定。

当创建的定时任务到期触发时会自动调用CeleryAction的execute函数，其中当前Action实例的ts属性保存着任务触发时间，params即为创建定时的params参数，提取参数解析继续完成即可。

需要注意的是因为整个forsun服务使用tornado异步IO实现，所以Action的execute会使用异步调用，如果你需要做同步阻塞调用时，推荐将需要执行的方法放到ThreadPoolExecutor去执行，这样性能会更好哦。



### 添加启动参数加载扩展

```bash
forsund --bind=0.0.0.0 --port=6458 --http=0.0.0.0:8001 --log=/var/log/forsun.log --log-level=INFO --driver=mem --driver-mem-store-file=/var/lib/fousun/forsun.session --extension-path=./ --extension=celery_extension.CeleryExtension
```

如若使用conf文件配置时，那么也在conf文件中添加扩展加载参数即可。

```
# 扩展配置
[extension]

# 扩展目录
path=./
# 载入扩展，已;分隔
extensions=celery_extension.CeleryExtension
```

这时候查看日志输出，你会发现扩展已经成功加载了。

```
2020-03-20 14:09:20,650 1022 INFO register extension path ./
2020-03-20 14:09:20,762 1022 INFO load extension celery_extension.CeleryExtension <class 'celery_extension.CeleryExtension'>
2020-03-20 14:09:20,762 1022 INFO action register celery <class 'celery_extension.CeleryAction'>
```

再通过info名称查看下当前状态信息，可以发现在支持的actions列表里已经有celery支持了，非常棒，现在愉快的开始自己的项目之旅吧。

```bash
forsun info

python_version: 3.6.9 (default, Nov  7 2019, 10:44:02)  [GCC 8.3.0]
forsun_version: 0.1.3
start_time:     2020-03-20 14:31:27.538081+08:00
cpu_user:       0.18
cpu_system:     0.1
mem_rss:        28.06M
mem_vms:        122.46M
current_time:   2020-03-20 14:31:38+08:00

stores: mem;redis
current_store:  mem
actions:        shell;http;redis;thrift;celery
bind_port:      0.0.0.0:6458
http_bind_port:
extensions:     celery_extension.CeleryExtension
```



### Http请求测试一下

```bash
curl -X PUT -H 'Content-Type: application/json' -d '{"key": "test", "seconds": 5, "minute": 0, "hour": 0, "day": 0, "month": 0, "count": 1, "action": "celery", "params": {"method": "hello"}}' http://127.0.0.1:8001/v1/plan

{"errcode": 0, "errmsg": "", "data": {"key": "test", "second": 5, "minute": 0, "hour": 0, "day": 0, "month": 0, "week": -1, "status": 0, "count": 0, "is_time_out": true, "next_time": 1584657610, "current_count": 0, "last_timeout": 0, "created_time": 1584657605.0, "action": "celery", "params": {"method": "hello"}}}
```

等待5秒，你就会看到celery成功调用。



# 最后

forsun除了可以通过Extension添加自定义执行器Action外，当然也可以通过Extension自定义实现持久化存储，这个后面咱有时间再介绍。

在实际工程中我们有非常多类似订单支付超时、配送超时之类的量大而又需要可编程控制的定时调度需求，一个方便管理而又高性能精准的定时任务调度服务显然可以大大节省我们的时间，游戏那么好玩为啥不能多一点呢。