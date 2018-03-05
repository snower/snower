---
title: 高性能千万级定时任务管理服务forsun使用详解
---

Forsun高性能高精度定时服务，轻松管理千万级定时任务。
项目地址： https://github.com/snower/forsun

* 使用 linux 系统定时器提供精确到秒级的定时调度，长时间运行保证无误差。
* 支持本地内存存储和 redis 持久化存储，使用 redis 可轻松管理数千万定时任务。
* 支持命令行创建删除任务，支持Apache Thrift接口调用创建和删除更新定时任务信息。
* 支持 shell、http、redis、thrift、beanstalk、mysql 六种到时触发回调方式，并可以通过扩展轻松自定义回调器。

# 背景

在实际项目中，存在大量需要定时或是延时触发的任务，比如电商中，延时需要检查订单是否支付成功，是否配送成功，定时给用户推送提醒等等，常规做法是用 crontab 每分钟扫码数据看是否到达时间，繁琐且扩展性伸缩性较差。

使用 forsun 服务，可以简单的针对每一个订单创建一个定时任务，配合异步队列，可以轻松实现扩展性伸缩性，Apache Thrift 的编程接口也可以很容易的和 celery、laravel 配合。 

其他场景下，比如失败延时重试，使用 forsun 定时服务也可以很简单就可以实现。

# 安装启动

使用pip自动安装

```
pip install forsun
```

帮助

```
forsund -h
usage: forsund [-h] [--bind BIND_HOST] [--port BIND_PORT] [--demon DEMON]
               [--log LOG_FILE] [--log-level LOG_LEVEL] [--driver DRIVER]
               [--driver-mem-store-file STORE_MEM_STORE_FILE]
               [--driver-redis-host DRIVER_REDIS_HOST]
               [--driver-redis-port DRIVER_REDIS_PORT]
               [--driver-redis-db DRIVER_REDIS_DB]
               [--driver-redis-prefix DRIVER_REDIS_PREFIX]
               [--driver-redis-server-id DRIVER_REDIS_SERVER_ID]
               [--extension-path EXTENSION_PATH] [--extension EXTENSIONS]

High-performance timing scheduling service

optional arguments:
  -h, --help            show this help message and exit
  --bind BIND_HOST      bind host (default: 0.0.0.0)
  --port BIND_PORT      bind port (default: 6458)
  --demon DEMON         run demon mode
  --log LOG_FILE        log file
  --log-level LOG_LEVEL
                        log level (defaul: INFO)
  --driver DRIVER       store driver mem or redis (defaul: mem)
  --driver-mem-store-file STORE_MEM_STORE_FILE
                        store mem driver store file (defaul:
                        /tmp/forsun.session)
  --driver-redis-host DRIVER_REDIS_HOST
                        store reids driver host (defaul: 127.0.0.1)
  --driver-redis-port DRIVER_REDIS_PORT
                        store reids driver port (defaul: 6379)
  --driver-redis-db DRIVER_REDIS_DB
                        store reids driver db (defaul: 0)
  --driver-redis-prefix DRIVER_REDIS_PREFIX
                        store reids driver key prefix (defaul: forsun)
  --driver-redis-server-id DRIVER_REDIS_SERVER_ID
                        store reids driver server id (defaul: 0)
  --extension-path EXTENSION_PATH
                        extension path
  --extension EXTENSIONS
                        extension name
```

使用内存持久化存储启动：

```
forsund --bind=0.0.0.0 --log=/var/log/forsun.log --log-level=INFO --driver=mem --driver-mem-store-file=/var/lib/fousun/forsun.session --demon
```

使用redis持久化存储启动：

```
forsund --bind=0.0.0.0 --log=/var/log/forsun.log --log-level=INFO --driver=redis --driver-redis-host=127.0.0.1 --driver-redis-db=1 --demon
```

注意：使用mem存储时只有进程正常退出时才会序列化任务保存到本地文件，下次启动时可能会丢失任务，建议使用性能更高的redis存储方式。

# 使用示例

命令行帮助

```
forsun -h
usage: forsun [-h] [--host HOST] [--port PORT] [--exe EXECUTE] [cmd]

High-performance timing scheduling service

positional arguments:
  cmd            execute cmd (default: )

optional arguments:
  -h, --help     show this help message and exit
  --host HOST    host (default: 127.0.0.1)
  --port PORT    port (default: 6458)
  --exe EXECUTE  execute cmd (default: )
```

延时运行示例

```
＃每五秒运行redis命令，共运行一次
forsun "set redis */5/1 * * * * * redis 'host=172.16.0.2;command=\'SET b 1 EX 300\'"
＃每五秒运行shell命令，共运行二次
forsun "set shell */5/2 * * * * * shell 'cmd=ls"
＃每五秒运行beanstalk命令，共运行一次
forsun "set beanstalk */5/1 * * * * * beanstalk 'host=10.4.14.14;name=etask;body={}'"
＃每五秒请求thrift接口，重复运行
forsun "set thrift */5/0 * * * * * thrift 'host=10.4.14.14;port=4220"
＃每五秒请求http接口，共运行一次
forsun "set http */5/1 * * * * * http 'url=\'http://www.baidu.com\''"
＃每五秒运行mysql命令，共运行一次
forsun "set mysql */5/1 * * * * * mysql 'host=172.16.0.2;user=root;passwd=123456;db=test;sql=\'update test set created_at=now() where id=1\'"
```

定时运行示例

```
#于每天16:32:00运行redis命令
forsun "set redis 0 32 16 * * * redis 'host=172.16.0.2;command=\'SET b 1 EX 300\'"
#于每小时32:00运行shell命令
forsun "set shell 0 32 ＊ * * * shell 'cmd=ls"
#于每分钟1秒时运行beanstalk命令
forsun "set beanstalk 1 * * * * * beanstalk 'host=10.4.14.14;name=etask;body={}'"
#于每月3日16:32:00请求thrift接口
forsun "set thrift 0 32 16 3 * * thrift 'host=10.4.14.14;port=4220"
#于每天16:32:00请求http接口
forsun "set http 0 32 16 * * * http 'url=\'http://www.baidu.com\''"
#于每天16:32:00运行mysql命令
forsun "set mysql 32 16 * * * mysql 'host=172.16.0.2;user=root;passwd=123456;db=test;sql=\'update test set created_at=now() where id=1\'"
```

# Apache Thrift接口文件定义

```
exception ForsunPlanError{
    1:i16 code,
    2:string message
}

struct ForsunPlan {
    1: required bool is_time_out,
    2: required string key,
    3: required i16 second,
    4: i16 minute = -1,
    5: i16 hour = -1,
    6: i16 day = -1,
    7: i16 month = -1,
    8: i16 week = -1,
    9: required i32 next_time,
    10: i16 status = 0,
    11: i16 count = 0,
    12: i16 current_count = 0,
    13: i32 last_timeout = 0,
    14:string action = "shell",
    15:map<string, string> params = {}
}

service Forsun{
    i16 ping(),
    #创建固定时间运行任务
    ForsunPlan create(1:string key, 2:i16 second, 3:i16 minute = -1, 4:i16 hour = -1, 5:i16 day = -1, 6:i16 month = -1, 7:i16 week = -1, 8:string action="shell", 9:map<string, string> params={}) throws (1:ForsunPlanError err),
    ＃创建延时运行任务
    ForsunPlan createTimeout(1:string key, 2:i16 second, 3:i16 minute = -1, 4:i16 hour = -1, 5:i16 day = -1, 6:i16 month = -1, 7:i16 week = -1, 8:i16 count=1, 9:string action="shell", 10:map<string, string> params={}) throws (1:ForsunPlanError err),
    ＃删除任务
    ForsunPlan remove(1:string key) throws (1:ForsunPlanError err),
    ＃获取任务信息
    ForsunPlan get(1:string key) throws (1:ForsunPlanError err),
    ＃获取当前即将运行任务列表
    list<ForsunPlan> getCurrent(),
    ＃获取某个时间运行任务列表
    list<ForsunPlan> getTime(1:i32 timestamp),
    ＃查询某些key前缀任务列表
    list<string> getKeys(1:string prefix),
    ＃thrift回调器请求函数定义
    void forsun_call(1:string key, 2:i32 ts, 3:map<string, string> params)
}
```

# 回调器参数详解

回调器参数为create和createTimeout最后一个参数params key和value的map。

## shell参数

* cmd shell命令
* cwd 工作目录
* env 环境变量，以;分割＝号连接的字符串，如：a=1;b=c

## http参数

* url 请求接口URL字符串
* method 请求方法，只支持get,post,put,delete,head五种方法
* body 请求体字符串
* header_ 以header_为前缀的key都会放到请求header中
* auth_username 校验用户名
* auth_password 校验密码
* auth_mode 校验方法
* user_agent 请求User－Agent
* connect_timeout 连接超时时间，默认5秒
* request_timeout 请求超时时间，默认60秒

## redis参数

* host redis服务器地址，默认127.0.0.1
* port redis服务器端口，默认6379
* selected_db redis运行命令db
* max_connections 连接redis服务器最大连接数，第一次连接时的命令中的值有效
* command 需要执行的命令，多条命令以;分割

## mysql参数

* host mysql服务器地址，默认127.0.0.1
* port mysql服务器端口，默认3306
* db mysql运行命令db，默认mysql
* user mysql登陆用户名，默认root
* passwd mysql登陆密码，默认空字符串
* max_connections 连接redis服务器最大连接数，第一次连接时的命令中的值有效
* sql 需要执行的sql

## beanstalk参数

* host beanstalk服务器地址，默认127.0.0.1
* port beanstalk服务器端口，默认11300
* name 队列名称，默认default
* body 推送的消息体

## thrift参数

回调thrift接口时，固定请求void forsun_call(1:string key, 2:i32 ts, 3:map<string, string> params)该函数，第三个params参数即为任务定义时的params值。

* host thrift服务器地址，默认127.0.0.1
* port thrift服务器端口，默认5643
* max_connections 连接thrift服务器最大连接数，第一次连接时的命令中的值有效

