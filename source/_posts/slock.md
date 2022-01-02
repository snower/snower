---
title: slock-分布式高性能状态与原子操作数据库
date: 2022-01-02 14:58:48
tags: slock
---

高性能分布式状态同步与原子操作数据库。通过锁队列、高性能异步二进制网络协议提供良好的多核支持。 可用于尖峰、同步、事件通知、并发控制等。支持Redis客户端。 

[https://github.com/snower/slock](https://github.com/snower/slock)

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

## Client库

- java https://github.com/snower/jaslock
- python https://github.com/snower/pyslock
- php https://github.com/snower/phslock
- openresty https://github.com/snower/slock-lua-nginx
- golang https://github.com/snower/slock


## 文章

- [分布式高性能状态与原子操作数据库slock简介](/slock-introduction/)
- [在openresty上基于是lock和redis快速搭建高性能long polling推送服务](/slock-openresty-poll/)
- [使用slock进行最大并发限流](/slock-flow-maxconcurrent/)
- [使用slock进行令牌桶限流](/slock-flow-tokenbucket/)