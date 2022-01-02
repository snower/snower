---
title: 使用slock进行最大并发限流
date: 2022-01-02 15:58:48
tags: slock
---

# 实现原理

使用最大锁定次数来控制最大并发数，请求前先进行Lock操作，请求完成后进行Unlock操作，超过最大锁定次数会进入等待，等待超时则出错返回，此时可以达到控制最大并发请求的目的。

<!-- more -->

# openresty配置代码

```conf
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
            local flow_key = ngx.var.flow_key or "flow:maxconcurrent"
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

# java中使用

```java
package main;

import io.github.snower.jaslock.Client;
import io.github.snower.jaslock.MaxConcurrentFlow;
import io.github.snower.jaslock.exceptions.SlockException;
import java.nio.charset.StandardCharsets;

public class App {
    public static void useCustomerCode() {

    }

    public static void main(String[] args) {
        Client slock = new Client("localhost", 5658);
        byte[] flowKey = "test".getBytes(StandardCharsets.UTF_8);
        MaxConcurrentFlow flow = slock.newMaxConcurrentFlow(flowKey, (short) 10, 5, 120); //最多10并发
        try {
            flow.acquire();
            try {
                useCustomerCode();
            } finally {
                try {
                    flow.release();
                } catch (SlockException e) {
                    e.printStackTrace();
                }
            }
        } catch (SlockException e) {
            e.printStackTrace();
        } finally {
            slock.close();
        }
    }
}
```

# 参数详解

### flow_key 限流key

字节类型，长度超过16字节时会用MD5做digest变为16字节，不足16字节时再前补0x00到16字节。

### count 最大并发数

最大数值不可超过2字节整型大小。

### timeout 最大等待时间

单位秒，最大数值不可超过2字节整型大小，设置较长时间有较好削峰平谷，但注意可能超过客户端请求超时时间，设置较短时间可进行fallback逻辑，设置0表示不等待。

### expreid acquire后过期时间

单位秒，最大数值不可超过2字节整型大小，注意此时间一般会在异常情况下，请求完成后未正常release导致该并发一致被占用时，可在过期时间到后自动修复，但也要注意设置时间小于绝大部分请求时间时，会导致请求释放并发使得限流失效。