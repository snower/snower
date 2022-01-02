---
title: 使用slock进行令牌桶限流
date: 2022-01-02 15:58:48
tags: slock
---

# 实现原理

设置最大锁定数为令牌数，初始即为所有令牌都可用，过期时间设置为全部令牌发放的周期，请求前先进行Lock操作，请求完成后不进行Unlock操作，请求前进行的Lock会在过期后重置，此操作可认为令牌重置，此时该令牌可被先请求使用，保证时间周期内只能完成指定请求个数。

在较长的时间周期中如需完成时间对齐，可先进行一次超时时间设置为0，过期时间设置为距离下次重置时间的剩余时间，如果进行Lock操作返回超时，则表示当前时间周期令牌已用尽，如等待超时时间还大于剩余时间，可再次进行Lock操作，此时超时时间设置为等待超时的时间，过期时间设置为令牌周期，即可完成令牌时间周期对齐。

<!-- more -->

# openresty配置代码

```conf
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
            local flow_key = ngx.var.flow_key or "flow:tokenbucket"
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

# java中使用

```java
package main;

import io.github.snower.jaslock.Client;
import io.github.snower.jaslock.TokenBucketFlow;
import io.github.snower.jaslock.exceptions.SlockException;
import java.nio.charset.StandardCharsets;

public class App {
    public static void useCustomerCode() {

    }

    public static void main(String[] args) {
        Client slock = new Client("localhost", 5658);
        byte[] flowKey = "test".getBytes(StandardCharsets.UTF_8);
        TokenBucketFlow flow = slock.newTokenBucketFlow(flowKey, (short) 10, 5, (double) 0.1); //每秒最多100次请求
        try {
            flow.acquire();
            useCustomerCode();
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

### count 每个周期令牌数

最大数值不可超过2字节整型大小。

### timeout 最大等待时间

单位秒，最大数值不可超过2字节整型大小，设置较长时间有较好削峰平谷，但注意可能超过客户端请求超时时间，设置较短时间可进行fallback逻辑，设置0表示不等待。

### period 令牌周期

单位秒，最大数值不可超过2字节整型大小，acquire成功后会在此时间到达时自动释放令牌。