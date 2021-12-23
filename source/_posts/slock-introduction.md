---
title: 分布式高性能状态与原子操作数据库slock简介
date: 2021-12-23 10:58:48
tags: slock
---

# 概述

项目地址：<https://github.com/snower/slock>

何为状态与原子操作数据库？区别于redis主要用于保存数据，可在多节点多系统间高效统同步数据，slock则是设计为只用于保存同步状态，几乎不能携带数据，高性能的异步二进制协议也保证了在状态达成时高效的主动触发等待系统。区别于redis被动检查的过期时间，slock的等待超时时间及锁定过期时间都是精确主动触发的。多核支持、更简单的系统结构也保证了其拥有远超redis的性能及延时，这也更符合状态同步需求中更高性能更低延时的需求。

秒杀为何难做？其问题就是我们需要在很短时间内完成大量的无效请求中夹杂仅很少的有效请求处理，进一步简化就是需要完成超高并发下海量请求间的状态同步的过程，slock高QPS可以快速解决过滤大量无效请求的问题，高性能的原子操作又可以很好的解决抢库存的逻辑。

随着nodejs的使用，异步IO相关框架也越来越成熟，使用也越来越方便，多线程同步IO模式下，某些场景很多时候我们需要转化为队列处理然后再推送结果，但异步IO就完全不需要这么复杂，直接加分布式锁等待可用就行，整个过程完全回到了单机多线程编程的逻辑，更简单也更容易理解和维护了，比如下单请求需要操作很多，在高并发下可能需要发送到队列中处理完成再推送结果，但用异步IO的加分布式锁话，仔细看异步IO加锁其实又组成了一个更大的分布式队列，大大简化了实现步骤。

# 特性

- 超高性能，在Intel i5-4590上超过200万QPS
- 高性能二进制异步协议简单稳定可靠，也可用redis同步文本协议
- 多核多线程支持
- 4级AOF持久化
	- 不持久化直接返回
	- 超过过期时间百分比时间后持久化后返回
	- 超过AOF时间持久化后返回
	- 立刻异步持久化后返回
	- 需整个集群活跃节点都成功并持久化后返回
- 高可用集群模式，自动迁移、自动代理
- 精确到毫秒、秒、分钟超时、过期时间，可单独订阅超时、过期时间
- 多次锁定支持，重入锁定支持
- 遗言命令

# 场景示例

## 分布式锁

整个协议只有两头指令，Lock和Unlock，分布式锁也即是最常用场景，和redis实现分布式锁区别除了性能更好延时也更低外，等待超时及锁定超时过期时间时精确主动触发的，所以有wait机制，redis实现的分布式锁一般则需要client主动延时重试来检查。

```java
package main;

import io.github.snower.jaslock.Client;
import io.github.snower.jaslock.Event;
import io.github.snower.jaslock.Lock;
import io.github.snower.jaslock.ReplsetClient;
import io.github.snower.jaslock.exceptions.SlockException;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class App {
    public static void main(String[] args) {
        ReplsetClient replsetClient = new ReplsetClient(new String[]{"172.27.214.150:5658"});
        try {
            replsetClient.open();
            Lock lock = replsetClient.newLock("test".getBytes(StandardCharsets.UTF_8), 5, 5);
            lock.acquire();
            lock.release();
        } catch (SlockException e) {
            e.printStackTrace();
        } finally {
            replsetClient.close();
        }
    }
}
```

## nginx & openresty限流

openresty使用此服务完成限流可以很方便的完成跨节点，同时因为使用高性能异步二进制协议，每个work只需要一个和server的连接，高并发下不会产生内部连接耗尽的问题，server主节点变更的时候work可自动使用新可用主节点，实现高可用。

#### 最大并发数限流

每个key可以设置最大锁定次数，使用该逻辑可以非常方便的实现最大并发限流。

```lua
lua_package_path "lib/resty/slock.lua;";

init_worker_by_lua_block {
        local slock = require("slock")
        slock:connect("lock1", "127.0.0.1", 5658)
}

server {
	listen 80;

	location /flow/maxconcurrent {
          access_by_lua_block {
                  local slock = require("slock")
                  local client = slock:get("lock1")
                  local flow_key = "flow:maxconcurrent"
                  local args = ngx.req.get_uri_args()
                  for key, val in pairs(args) do
                          if key == "flow_key" then
                                  flow_key = val
                          end
                  end
                  local lock = client:newMaxConcurrentFlow(flow_key, 10, 5, 60)
                  local ok, err = lock:acquire()
                  if not ok then
                          ngx.say("acquire error:" .. err)
                          ngx.exit(ngx.HTTP_OK)
                  else
                          ngx.ctx.lock1 = lock
                  end
          }

          echo "hello world";

          log_by_lua_block {
                  local lock = ngx.ctx.lock1
                  if lock ~= nil then
                          local ok, err = lock:release()
                          if not ok then
                                  ngx.log(ngx.ERR, "slock release error:" .. err)
                          end
                  end
          }
	}
}

```

#### 令牌桶限流

每个key可以设置最大锁定次数，并设置为在令牌到期时过期，即可实现令牌桶限流，使用毫秒级过期时间的时候也可以从此方式来完成削峰平衡流量。

```lua
lua_package_path "lib/resty/?.lua;";

init_worker_by_lua_block {
        local slock = require("slock")
        slock:connect("lock1", "127.0.0.1", 5658)
}

server {
	listen 80;

	location /flow/tokenbucket {
                access_by_lua_block {
                        local slock = require("slock")
                        local client = slock:get("lock1")
                        local flow_key = "flow:tokenbucket"
                        local args = ngx.req.get_uri_args()
                        for key, val in pairs(args) do
                                if key == "flow_key" then
                                        flow_key = val
                                end
                        end
                        local lock = client:newTokenBucketFlow(flow_key, 10, 5, 60)
                        local ok, err = lock:acquire()
                        if not ok then
                                ngx.say("acquire error:" .. err)
                                ngx.exit(ngx.HTTP_OK)
                        end
                }

                echo "hello world";
        }
}
```

## 其它可用场景

- 分布式Event，一个常用场景如扫码登录，二维码这边需等待扫码状态。
- 分布式Semaphore，这个即是更通用的限流，此外也可以用于异步任务结果通知。
- 分布式读写锁。
- 秒杀场景，秒杀场景是典型的请求数很高但有效请求数十分少的场景，原子操作的特性可以很好支持抢库存的逻辑，超高的并发支持也可以很好解决天量无效请求的问题。
- 异步结果通知，网页直接实现的功能又需要后台定时任务执行，此时完全可以网络也调用异步任务，然后通过分布式Event等待执行完成即可。

以上这些使用场景都可以在openresty完成对外接口，再有内部系统完成触发即可，openresty的高性能高并发完全可以很容易的解决很多之前需要用队列需要长连接推送的需求。