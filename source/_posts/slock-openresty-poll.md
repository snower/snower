---
title: 在openresty上基于是lock和redis快速搭建高性能long polling推送服务
date: 2021-12-28 10:58:48
tags: slock
---

# 为啥需要？

在实际开发中我们经常会遇到需要长时间等待后台事件的情况，例如较为常见的扫码登录功能，二维码界面需等待后台扫码登录成功的事件，再如导入导出等需要较长时间才能处理完成的任务，此时需要把任务放到后台由异步任务进行处理，完成后再给前台界面推送完成事件，以上需求我们需要用长连接才能完成推送，但长连接推送状态管理复杂，且需要部署独立系统，系统流程复杂且横向水平扩展困难，此时选择更简单long polling等待是一个更好的选择，http请求直接等待返回，显然逻辑更简单，可用性可维护性也会更高。

openresty是一个构建在nginx上的高性能能系统，一般情况下我们也需要在自身服务前部署nginx作为网关，那么选择openresty来构建一个高性能的long polling服务显然是一个好选择。slock是高性能的状态及原子操作数据库，redis则是高性能的内存缓存数据库，使用下边nginx配置文件即可快速基于slock和redis构建一个高性能高可用long polling服务。同时构建的此long polling服务是一个通用服务，即可用于扫码登录这样的需求完成状态推送，也可用于像消息系统、私信系统等的消息推送。

slock项目地址：<https://github.com/snower/slock>

slock简介可看：<https://segmentfault.com/a/1190000041148625>

<!-- more -->

# 快速配置构建

首先需在安装好的openresty服务中安装slock的lua client包。

项目地址：https://github.com/snower/slock-lua-nginx

安装方式即把slock-lua-nginx中slock.lua复制到openresty目录中的lualib/中，然后添加以下nginx配置文件修改相关参数即可。

```conf
init_worker_by_lua_block {
    local slock = require "slock"
    slock:connect("server1", "127.0.0.1", 5658)
}

server {
    listen 8081;
    default_type application/json;

    location /poll/event {
        content_by_lua_block {
            local cjson = require "cjson"
            local slock = require "slock"
            local slock_client = slock:get("server1")
            local default_type = ngx.var.arg_default_type or "clear"
            local wait_type = ngx.var.arg_wait_type or ""
            local event_key = ngx.var.arg_event or ""
            local wait_timeout = tonumber(ngx.var.arg_timeout) or 60

            local sendResult = function(err_code, err_message)
                ngx.say(cjson.encode({
                    err_code = err_code,
                    err_message = err_message,
                }))
            end

            if event_key == "" then
                return sendResult(400, "event key is empty")
            end

            local event = nil
            if default_type == "set" then
                event = slock_client:newDefaultSetEvent(event_key, 5, wait_timeout * 2)
            else
                event = slock_client:newDefaultClearEvent(event_key, 5, wait_timeout * 2)
            end

            if wait_type == "reset" then
                local ok, err = event:waitAndTimeoutRetryClear(wait_timeout)
                if not ok then
                    return sendResult(504, "wait event timeout")
                end
                return sendResult(0, "succed")
            end

            local ok, err = event:wait(wait_timeout)
            if not ok then
                return sendResult(504, "wait event timeout")
            end
            return sendResult(0, "succed")
        }
    }

    location /poll/message {
        content_by_lua_block {
            local cjson = require "cjson"
            local redis = require "resty.redis"
            local slock = require "slock"
            local redis_client = redis:new()
            local slock_client = slock:get("server1")

            local default_type = ngx.var.arg_default_type or "clear"
            local wait_type = ngx.var.arg_wait_type or ""
            local event_key = ngx.var.arg_event or ""
            local wait_timeout = tonumber(ngx.var.arg_timeout) or 60

            local sendResult = function(err_code, err_message, data)
                ngx.say(cjson.encode({
                    err_code = err_code,
                    err_message = err_message,
                    data = data,
                }))
            end

            if event_key == "" then
                return sendResult(400, "event key is empty")
            end

            redis_client:set_timeouts(5000, wait_timeout * 500, wait_timeout * 500)
            local ok, err = redis_client:connect("10.10.10.251", 6379)
            if not ok then
                return sendResult(502, "redis connect fail")
            end
            local message, err = redis_client:lpop(event_key)
            if err ~= nil then
                return sendResult(500, "redis lpop fail")
            end
            if message ~= ngx.null then
                redis_client:set_keepalive(7200000, 16)
                return sendResult(0, "", message)
            end

            local event = nil
            if default_type == "set" then
                event = slock_client:newDefaultSetEvent(event_key, 5, wait_timeout * 2)
            else
                event = slock_client:newDefaultClearEvent(event_key, 5, wait_timeout * 2)
            end

            if wait_type == "reset" then
                local ok, err = event:waitAndTimeoutRetryClear(wait_timeout)
                if not ok then
                    return sendResult(504, "wait timeout")
                end

                local message, err = redis_client:lpop(event_key)
                if err ~= nil then
                    return sendResult(500, "redis lpop fail")
                end
                redis_client:set_keepalive(7200000, 16)
                return sendResult(0, "succed", message)
            end

            local ok, err = event:wait(wait_timeout)
            if not ok then
                return sendResult(504, "wait timeout")
            end

            local message, err = redis_client:lpop(event_key)
            if err ~= nil then
                return sendResult(500, "redis lpop fail")
            end
            redis_client:set_keepalive(7200000, 16)
            return sendResult(0, "succed", message)
        }
    }
}
```
/poll/event 接口只等待事件触发，不返回数据。

/poll/message 则是先从redis中获取数据，成功则返回，否则等待事件触发，再从redis获取数据返回。

接口Query String参数：

- default_type 创建Event的初始状态是seted还是cleared，wait等待seted状态触发，即如果使用初始是seted则需其它系统先执行clear操作，可选值：set、clear，默认值clear
- wait_type 事件触发后是否重置Event状态，不重置可保证在过期时间内可重入，重置则可用于私信系统的循环获取消息或事件，设置为reset为重置，默认值空字符不重置
- event 等待的事件key，不可为空，redis也使用该key在List数据结构中保存消息
- timeout 数字，等待超时时间，单位秒，默认等待60秒


### 特别注意：
- openresty使用单一连接到slock的tcp连接处理所有请求，nginx只有init_worker_by_lua_block创建的socket才可在整个worker生命周期中保持存在，所以slock connect需在init_worker_by_lua_block完成，第一个参数为连接名称，后续可用该名称获取该连接使用。

- slock配置为replset模式时，可用replset方式连接，使用该模式连接时，nginx会自动跟踪可用节点，保持高可用, 如：{% raw %}slock:connectReplset("server1", {{"127.0.0.1", 5658}, {"127.0.0.1", 5659}}){% endraw %}

配置完成后执行下方shell会处于等待返回状态：
```bash
curl "http://localhost:8081/poll/message?event=test&default_type=clear"
```

# 其它系统如何推送事件？

## 使用slock java client推送事件

java client项目地址：https://github.com/snower/jaslock

```java
package main;

import io.github.snower.jaslock.Client;
import io.github.snower.jaslock.Event;
import io.github.snower.jaslock.exceptions.SlockException;
import redis.clients.jedis.Jedis;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class App {
    public static void main(String[] args) {
        Client slock = new Client("localhost", 5658);
        Jedis jedis = new Jedis("10.10.10.251", 6379);
        try {
            byte[] eventKey = "test".getBytes(StandardCharsets.UTF_8);
            slock.open();
            jedis.rpush(eventKey, "hello".getBytes(StandardCharsets.UTF_8));
            jedis.expire(eventKey, 120);
            Event event = slock.newEvent(eventKey, 5, 120, false);
            event.set();
        } catch (IOException | SlockException e) {
            e.printStackTrace();
        } finally {
            slock.close();
            jedis.close();
        }
    }
}
```

newEvent参数：

- eventKey 事件名称，和前端请求一致
- timeout set操作超时事件
- expried 如果初始状态时cleared，则表示set之后状态保持时间，如果初始状态是seted，则表示clear之后状态保持时间，超过改时间都将被自动回收
- defaultSeted ture表示初始是seted状态，false为cleared状态，需和前端传参一致

注：只推送事件时去除redis操作即可。

php、python、golang操作类似。

php client项目地址：https://github.com/snower/pyslock

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Redis;
use Snower\Phslock\Laravel\Facades\Phslock;

class TestSlockEvent extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'test:slock-event';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Command description';

    /**
     * Create a new command instance.
     *
     * @return void
     */
    public function __construct()
    {
        parent::__construct();
    }

    /**
     * Execute the console command.
     *
     * @return int
     */
    public function handle()
    {
        Redis::rpush("test", "hello"); //需禁用prefix
        Redis::expire("test", 120);
        $event = Phslock::Event("test", 5, 120, false);
        $event->set();
        return 0;
    }
}
```

python client项目地址：https://github.com/snower/phslock

```python
import redis
import pyslock

redis_client = redis.Redis("10.10.10.251")
slock_client = pyslock.Client("localhost")

redis_client.rpush("test", "hello")
redis_client.expire("test", 120)
event = slock_client.Event("test", 5, 120, False)
event.set()
```

## 使用redis自定义命令推送事件

也可用redis自定义命令来执行slock Event的set和clear操作完成事件触发。

### 初始是seted时：
```bash
#clear操作
lock ${EVENT_KEY} lock_id ${EVENT_KEY} flag 2 timeout ${TIMEOUT} expried ${EXPRIED}
#如 lock test lock_id test flag 2 timeout 5 expried 120


#set操作
unlock ${EVENT_KEY} lock_id ${EVENT_KEY}
#如 unlock test lock_id test
```

### 初始是cleared时：
```bash
#clear操作
unlock ${EVENT_KEY} lock_id ${EVENT_KEY}
#如 unlock test lock_id test


#set操作
lock ${EVENT_KEY} lock_id ${EVENT_KEY} flag 2 timeout ${TIMEOUT} expried ${EXPRIED} count 2
#如 lock test lock_id test flag 2 timeout 5 expried 120 count 2
#用redis-cli -p 5658连接slock后执行该示例命令，即可看到上方等待curl命令成功返回。
```

# 关于高可用与扩展

关于高可用，slock支持配置为集群模式，在集权模式运行时主节点异常时可自动选择新主节点，此时如果openresty使用resplset模式连接时，可自动使用新可用节点，保证高可用。

关于水平扩展，slock良好的多核支持，百万级qps，保证了无需过多考虑水平扩展问题，而openresty则依然保持了web服务常规无状态特性，可按照web常规水平扩展方式扩容即可不断提高系统承载性能。