名称
====

lua-resty-redis - 基于cosocket API为ngx_lua 开发的 lua连接redis 的链接库。

目录列表
=================

* [名称](#名称)
* [状态](#状态)
* [介绍](#介绍)
* [用法](#用法)
* [方法](#方法)
    * [new](#new)
    * [connect](#connect)
    * [set_timeout](#set_timeout)
    * [set_keepalive](#set_keepalive)
    * [get_reused_times](#get_reused_times)
    * [close](#close)
    * [init_pipeline](#init_pipeline)
    * [commit_pipeline](#commit_pipeline)
    * [cancel_pipeline](#cancel_pipeline)
    * [hmset](#hmset)
    * [array_to_hash](#array_to_hash)
    * [read_reply](#read_reply)
    * [add_commands](#add_commands)
* [Redis 认证](#Redis 认证)
* [Redis 事务](#Redis 事务)
* [负载均衡和故障转移](#负载均衡和故障转移)
* [调试](#调试)
* [自动记录错误](#自动记录错误)
* [问题列表](#问题列表)
* [限制](#限制)
* [安装](#安装)
* [TODO](#todo)
* [社区](#社区)
    * [英文邮件列表](#english-mailing-list)
    * [中文邮件列表](#chinese-mailing-list)
* [Bugs and Patches](#bugs-and-patches)
* [作者](#作者)
* [版权和协议](#版权和协议)
* [查看更多](#查看更多)

状态
======

可以被应用到生产环境。

介绍
===========
这个库是为nginx的拓展 ngx_lua 写的，用于通过lua操作redis的库。

http://wiki.nginx.org/HttpLuaModule 
//nginx_lua_module地址。

受益于ngx_lua 的 cosocket API，这个库能够在完全非阻塞的模式下运行。



版本支持： 需要[ngx_lua 0.5.14](https://github.com/chaoslawful/lua-nginx-module/tags)以上
            或
         需要[ngx_openresty 1.2.1.14](http://openresty.org/#Download) 以上

用法
========

```lua
    # 如果使用ngx_openresty可不写下面这一行
    lua_package_path "/path/to/lua-resty-redis/lib/?.lua;;";

    server {
        location /test {
            content_by_lua '
                local redis = require "resty.redis"
                local red = redis:new()

                red:set_timeout(1000) -- 1 sec

                -- socket监听模式的写法
                --     local ok, err = red:connect("unix:/path/to/redis.sock")

                local ok, err = red:connect("127.0.0.1", 6379)
                if not ok then
                    ngx.say("failed to connect: ", err)
                    return
                end

                ok, err = red:set("dog", "an animal")
                if not ok then
                    ngx.say("failed to set dog: ", err)
                    return
                end

                ngx.say("set result: ", ok)

                local res, err = red:get("dog")
                if not res then
                    ngx.say("failed to get dog: ", err)
                    return
                end

                if res == ngx.null then
                    ngx.say("dog not found.")
                    return
                end

                ngx.say("dog: ", res)

                red:init_pipeline()
                red:set("cat", "Marry")
                red:set("horse", "Bob")
                red:get("cat")
                red:get("horse")
                local results, err = red:commit_pipeline()
                if not results then
                    ngx.say("failed to commit the pipelined requests: ", err)
                    return
                end

                for i, res in ipairs(results) do
                    if type(res) == "table" then
                        if not res[1] then
                            ngx.say("failed to run command ", i, ": ", res[2])
                        else
                            -- process the table value
                        end
                    else
                        -- process the scalar value
                    end
                end

                -- put it into the connection pool of size 100,
                -- with 10 seconds max idle time
                local ok, err = red:set_keepalive(10000, 100)
                if not ok then
                    ngx.say("failed to set keepalive: ", err)
                    return
                end

                -- or just close the connection right away:
                -- local ok, err = red:close()
                -- if not ok then
                --     ngx.say("failed to close: ", err)
                --     return
                -- end
            ';
        }
    }
```

[返回顶部](#目录列表)

方法
=======

支持所有的redis命令，除非命令是全部小写的。
可以在下面的链接找到redis的所有命令
http://redis.io/commands

你需要从这个命令参考中找到redis的哪些命令支持哪些参数。

REDIS的命令调用可以直接映射到这里的方法调用。例如   redis的GET命令接受一个参数，你就可以像下面这样调用REDIS的GET方法：
```lua
    local res, err = red:get("key")
```

类似的，REDIS的命令LRANGE命令接受三个参数，你可以像下面这样调用这个方法：
```lua
    local res, err = red:lrange("nokey", 0, 1)
```

例如，"SET","GET","LRANGE" 和 "BLPOP" 命令映射过来就是"set", "get", "lrange", 和 "blpop" 方法.
下面是更多的例子：

```lua
    -- HMGET myhash field1 field2 nofield
    local res, err = red:hmget("myhash", "field1", "field2", "nofield")
```

```lua
    -- HMSET myhash field1 "Hello" field2 "World"
    local res, err = red:hmset("myhash", "field1", "Hello", "field2", "World")
```


所有的命令都能够在正常的时候得到一个正常的返回值或者在有什么错误发生时返回 `nil` 和一个字符串形式的错误描述信息。


一个REDIS的 "status reply" 返回值会返回一个去掉了"+"的返回值。


一个REDIS的 "integer reply" 返回值会返回一个lua的number类型值。


一个REDIS的 "error reply"返回值会返回一个 false *和* 字符串类型的错误描述


一个REDIS的 非空"bulk replay"返回类型 在lua里面使用字符串返回，而空的"bulk replay" 在lua中将以 `ngx.null`来返回。


一个非空的REDIS "multi-bulk replay" 返回值在lua里面使用关联数组保存并返回所有可能的值。如果任何一个值包含一个redis错误信息，将返回一个如下格式的二维数组 `{false,err}`


一个空的 REDIS"multi-bulk" 返回值在lua里面将会返回`ngx.null`.

参看 http://redis.io/topics/protocol  去了解redis的返回值类型

除了上述方法外，还支持以下方法。


[返回顶部](#目录列表)

new
---
`syntax: red, err = redis:new()`

创建一个到redis的连接对象。如果失败返回`nil`和一个字符串形式的错误描述。

[返回顶部](#目录列表)

connect
-------
`syntax: ok, err = red:connect(host, port, options_table?)`

`syntax: ok, err = red:connect("unix:/path/to/unix.sock", options_table?)`

尝试通过指定的主机名和端口或通过一个redis的本地unix套接字连接一个正在监听的redis主机。


在解析远程主机并尝试连接之前，这个方法会去尝试查找由上次连接打开的在连接池中的可用空闲连接。


一个lua数组方式的参数可以被指定为最后一个参数。用来指定不同的连接选项。

* `pool`


    为当前连接指定一个自定义的名称，如果省略，讲使用连接信息的字面含义表示。如`<host>:<port>` 或 `<unix-socket-path>`.

[返回顶部](#目录列表)

set_timeout
----------
`syntax: red:set_timeout(time)`

设置超时时间（单位ms） 来保证后续操作。包括`connect` 方法。

[返回顶部](#目录列表)

set_keepalive
------------
`syntax: ok, err = red:set_keepalive(max_idle_timeout, pool_size)`

立即将当前的Redis连接放入ngx_lua的cosocket连接池。

当连接在连接池中时可以针对单个nginx 进程指定空闲等待时间。？？不确定。

成功时返回`1`，失败时返回`nil`和字符串类型的错误描述信息。

当你希望调用`close`指令关闭一个连接的时候可以用这个方法来替代。调用这个方法将立即将当前的REDIS连接对象置为关闭状态，任何后续的非`connect()`请求都将返回`closed`错误。
[返回顶部](#目录列表)

get_reused_times
----------------
`syntax: times, err = red:get_reused_times()`

这个方法返回当前连接被重用的次数。如果出错，返回`nil`和一个字符串形式的错误描述信息。

如果当前的连接并非来自内置的连接池，将返回`0` 这意味着这个连接并没有被重用过。

如果当前连接来自连接池，那么这个方法的返回值将是一个非零的整数。所有此方法可以用来判断当前连接是不是来自连接池。

[返回顶部](#目录列表)

close
-----
`syntax: ok, err = red:close()`

关闭当前的REDIS连接并且返回状态。

如果成功返回`1`，失败时返回`nil`和一个错误描述信息。

[返回顶部](#目录列表)

init_pipeline
-------------
`syntax: red:init_pipeline()`

`syntax: red:init_pipeline(n)`

支持REDIS的命令队列模式，执行后，后续的命令将被缓存，直到调用`commit_pipeline`方法，或调用`cancel_pipeline`方法放弃执行。

此命令始终成功。

如果REDIS对象已经运行在命令队列模式，调用这个方法讲使得命令队列中已经存在的命令被执行。


参数`n` 近似的指定添加到命令队列中的数量，这个参数可以少许提高性能。

[返回顶部](#目录列表)

commit_pipeline
---------------
`syntax: results, err = red:commit_pipeline()`

退出命令队列模式，并提交所有已经被缓存的redis命令。所有命令的返回值将被自动收集起来并且以最高优先级返回一个 `multi-bulk`。

发生错误时，该方法返回一个`nil`和一个lua string形式的错误描述信息

[返回顶部](#目录列表)

cancel_pipeline
---------------
`syntax: red:cancel_pipeline()`

放弃所有从上次调用`init_pipeline`方法开始的已经被缓存的所有redis命令，并且退出命令队列模式。

此方法始终成功。

如果redis对象并非运行在命令队列模式，此方法是一个空操作。

[返回顶部](#目录列表)

hmset
-----
`syntax: red:hmset(myhash, field1, value1, field2, value2, ...)`

`syntax: red:hmset(myhash, { field1 = value1, field2 = value2, ... })`

Special wrapper for the Redis "hmset" command.

对REDIS hmset命令的特殊封装。

如果调用此方法时只传递了3个参数（包括redis实例这个参数），强制要求最后的参数是一个包含所有键值对的LUA数组。

[返回顶部](#目录列表)

array_to_hash
-------------
`syntax: hash = red:array_to_hash(array)`

一个辅助方法，将lua里面的数组转化为一个hash表。

最早可用版本`v0.11`。

[返回顶部](#目录列表)

read_reply
----------
`syntax: res, err = red:read_reply()`

从redis服务器读取返回值。这个方法大多数用于[Redis Pub/Sub API](http://redis.io/topics/pubsub/)，例如：


```lua
    local cjson = require "cjson"
    local redis = require "resty.redis"

    local red = redis:new()
    local red2 = redis:new()

    red:set_timeout(1000) -- 1 sec
    red2:set_timeout(1000) -- 1 sec

    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("1: failed to connect: ", err)
        return
    end

    ok, err = red2:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("2: failed to connect: ", err)
        return
    end

    local res, err = red:subscribe("dog")
    if not res then
        ngx.say("1: failed to subscribe: ", err)
        return
    end

    ngx.say("1: subscribe: ", cjson.encode(res))

    res, err = red2:publish("dog", "Hello")
    if not res then
        ngx.say("2: failed to publish: ", err)
        return
    end

    ngx.say("2: publish: ", cjson.encode(res))

    res, err = red:read_reply()
    if not res then
        ngx.say("1: failed to read reply: ", err)
        return
    end

    ngx.say("1: receive: ", cjson.encode(res))

    red:close()
    red2:close()
```


得到的结果类似下面：

    1: subscribe: ["subscribe","dog",1]
    2: publish: 1
    1: receive: ["message","dog","Hello"]

下面的方法是被保护的：

[返回顶部](#目录列表)

add_commands
------------
`syntax: hash = redis.add_commands(cmd_name1, cmd_name2, ...)`

在`resty.redis`类中注册一个新的redis命令。 下面是一些例子:

```lua
    local redis = require "resty.redis"

    redis.add_commands("foo", "bar")

    local red = redis:new()

    red:set_timeout(1000) -- 1 sec

    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("failed to connect: ", err)
        return
    end

    local res, err = red:foo("a")
    if not res then
        ngx.say("failed to foo: ", err)
    end

    res, err = red:bar()
    if not res then
        ngx.say("failed to bar: ", err)
    end
```

[返回顶部](#目录列表)

Redis 认证
====================

Redis使用`AUTH`命令进行认证。

相比redis的`GET` 和 `SET` 命令，这个命令并没有什么特殊，所以可以使用 `auth`方法进行认证。举例如下：

```lua
    local redis = require "resty.redis"
    local red = redis:new()

    red:set_timeout(1000) -- 1 sec

    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("failed to connect: ", err)
        return
    end

    local res, err = red:auth("foobared")
    if not res then
        ngx.say("failed to authenticate: ", err)
        return
    end
```

此处我们假设在redis的 `redis.conf` 里面已经配置了 `foobared`密码。

    requirepass foobared

如果指定的密码错误，HTTP连接请求会返回如下信息：

    failed to authenticate: ERR invalid password

[返回顶部](#目录列表)

Redis 事务
==================


这个库支持redis 事务 [Redis transactions](http://redis.io/topics/transactions/). 下面是例子：

```lua
    local cjson = require "cjson"
    local redis = require "resty.redis"
    local red = redis:new()

    red:set_timeout(1000) -- 1 sec

    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("failed to connect: ", err)
        return
    end

    local ok, err = red:multi()
    if not ok then
        ngx.say("failed to run multi: ", err)
        return
    end
    ngx.say("multi ans: ", cjson.encode(ok))

    local ans, err = red:set("a", "abc")
    if not ans then
        ngx.say("failed to run sort: ", err)
        return
    end
    ngx.say("set ans: ", cjson.encode(ans))

    local ans, err = red:lpop("a")
    if not ans then
        ngx.say("failed to run sort: ", err)
        return
    end
    ngx.say("set ans: ", cjson.encode(ans))

    ans, err = red:exec()
    ngx.say("exec ans: ", cjson.encode(ans))

    red:close()
```

输出结果如下：

    multi ans: "OK"
    set ans: "QUEUED"
    set ans: "QUEUED"
    exec ans: ["OK",[false,"ERR Operation against a key holding the wrong kind of value"]]

[返回顶部](#目录列表)

负载均衡和故障转移
===========================


你可以简单的在lua脚本里面实现负载均衡。例如创建一个数组来存储所有的主机信息，并且在每次请求来临的时候通过一定的规则（随机或者基于关键key的hash）选择一个合适的服务器来响应这次请求。你可以跟踪当前规则下的lua_module的相关数据，参见：http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker


同样的，也可以通过lua脚本很容易的实现自动故障转移。

[返回顶部](#目录列表)

调试
=========


通常情况下为了方便都会使用[lua-cjson](http://www.kyne.com.au/~mark/software/lua-cjson.php) 库去将返回值编码成json数据。例如：


```lua
    local cjson = require "cjson"
    ...
    local res, err = red:mget("h1234", "h5678")
    if res then
        print("res: ", cjson.encode(res))
    end
```

[返回顶部](#目录列表)

自动记录错误
=======================


底层的 [ngx_lua](http://wiki.nginx.org/HttpLuaModule) 模块已经在socket错误的时候进行了日志记录，如果你已经在你的lua脚本里面正确的处理了错误信息。那么推荐关闭 [ngx_lua](http://wiki.nginx.org/HttpLuaModule)模块的错误记录功能，操作如下：

```nginx
    lua_socket_log_errors off;
```

[返回顶部](#目录列表)

问题列表
=====================

1. Ensure you configure the connection pool size properly in the [set_keepalive](#set_keepalive) . Basically if your NGINX handle `n` concurrent requests and your NGINX has `m` workers, then the connection pool size should be configured as `n/m`. For example, if your NGINX usually handles 1000 concurrent requests and you have 10 NGINX workers, then the connection pool size should be 100.

1，确保你[set_keepalive](#set_keepalive)设定的连接池大小是合适的。基本上如果你的nginx能够支撑`n`个并发连接并且你的nginx有`m`个wokers。你的连接池大小应该设置成 `n/m` 。例如，如果你的nginx在通常情况下启动10个workers进程能够承载1000并发，那么你的连接池大小应该设置成100。


2. Ensure the backlog setting on the Redis side is large enough. For Redis 2.8+, you can directly tune the `tcp-backlog` parameter in the `redis.conf` file (and also tune the kernel parameter `SOMAXCONN` accordingly at least on Linux). You may also want to tune the `maxclients` parameter in `redis.conf`.

2，确保Redis中的队列长度设置的足够大，在Redis2.8以上版本中，可以在`redis.conf`文件中调整参数`tcp-backlog` 并且调整内核的`SOMAXCONN`参数。（默认情况下redis.conf中的tcp-backlog参数不会大于系统的SOMAXCONN数。当redis的客户端并发数量比较高并且响应缓慢的时候需要考虑调整这两个参数。）



3. Ensure you are not using too short timeout setting in the [set_timeout](#set_timeout) method. If you have to, try redoing the operation upon timeout and turning off [automatic error logging](#automatic-error-logging) (because you are already doing proper error handling in your own Lua code).

3，确保你通过[set_timeout](#set_timeout)方法设定了足够长的超时时间。试着去调整超时时间并且关闭[automatic error logging](#automatic-error-logging) （因为你已经在你的lua脚本里面正确处理了错误信息。）


4. If your NGINX worker processes' CPU usage is very high under load, then the NGINX event loop might be blocked by the CPU computation too much. Try sampling a [C-land on-CPU Flame Graph](https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt) and [Lua-land on-CPU Flame Graph](https://github.com/agentzh/stapxx#ngx-lj-lua-stacks) for a typical NGINX worker process. You can optimize the CPU-bound things according to these Flame Graphs.

4，如果你的NGINX进程的CPU使用率很高，可能是因为nginx的事件池被过多的CPU计算所阻塞。可以通过[C-land on-CPU Flame Graph](https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt) 和 [Lua-land on-CPU Flame Graph](https://github.com/agentzh/stapxx#ngx-lj-lua-stacks) 去追踪一个典型的nginx进程。去优化CPU的使用。

5. If your NGINX worker processes' CPU usage is very low under load, then the NGINX event loop might be blocked by some blocking system calls (like file IO system calls). You can confirm the issue by running the [epoll-loop-blocking-distr](https://github.com/agentzh/stapxx#epoll-loop-blocking-distr) tool against a typical NGINX worker process. If it is indeed the case, then you can further sample a [C-land off-CPU Flame Graph](https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt-off-cpu) for a NGINX worker process to analyze the actual blockers.

5，如果你的NGINX进程的CPU使用率很低，你的NGINX事件池可能被一些系统调用所阻塞（例如文件IO)。可以通过[epoll-loop-blocking-distr](https://github.com/agentzh/stapxx#epoll-loop-blocking-distr) 去跟踪一个典型的进程。如果确实如此  ，还可以通过[C-land off-CPU Flame Graph](https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt-off-cpu)确定到底什么地方出了问题。

6. If your `redis-server` process is running near 100% CPU usage, then you should consider scale your Redis backend by multiple nodes or use the [C-land on-CPU Flame Graph tool](https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt) to analyze the internal bottlenecks within the Redis server process.

6，如果你的`redis-server` 进程的CPU使用率将近100% 。应该考虑拓展你的redis后端服务。或使用[C-land on-CPU Flame Graph tool](https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt) 分析redis的内部瓶颈。


[返回顶部](#目录列表)

限制
===========

* This library cannot be used in code contexts like init_by_lua*, set_by_lua*, log_by_lua*, and
header_filter_by_lua* where the ngx_lua cosocket API is not available.

这个库不能被init_by_lua*, set_by_lua*, log_by_lua*, 和header_filter_by_lua*等不支持ngx_lua cosocket API方法所使用。

* The `resty.redis` object instance cannot be stored in a Lua variable at the Lua module level,
because it will then be shared by all the concurrent requests handled by the same nginx
 worker process (see
http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker ) and
result in bad race conditions when concurrent requests are trying to use the same `resty.redis` instance
(you would see the "bad request" or "socket busy" error to be returned from the method calls).

在lua module层面上不能将resty.redis对象保存在一个lua变量中。因为这样做会使得这个对象被同一个进程下的所有并发请求所共享。从而造成这些并发请求争抢同一个resty.redis实例的情况。
参看http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker
（请求结果讲出现"bad request" 或者 "socket busy"等错误。）


You should always initiate `resty.redis` objects in function local
variables or in the `ngx.ctx` table. These places all have their own data copies for
each request.

你应该在函数的范围内或者在ngx.ctx中声明resty.redis对象。这些地方在每个独立的请求中有自己的变量副本。

[返回顶部](#目录列表)

安装
============

If you are using the ngx_openresty bundle (http://openresty.org ), then
you do not need to do anything because it already includes and enables
lua-resty-redis by default. And you can just use it in your Lua code,
as in

如果你安装了ngx_openresty包，那么你可以直接使用此库，因为默认情况直接包含lua-resty-redis、或者你可以通过lua像下面这样引用：



```lua
    local redis = require "resty.redis"
    ...
```

If you are using your own nginx + ngx_lua build, then you need to configure
the lua_package_path directive to add the path of your lua-resty-redis source
tree to ngx_lua's LUA_PATH search path, as in

如果你使用自己搭建的nginx+ngx_lua环境。需要通过配置 `lua_package_path`指令添加 lua-resty-redis 的路径到LUA_PATH。

```nginx
    # nginx.conf
    http {
        lua_package_path "/path/to/lua-resty-redis/lib/?.lua;;";
        ...
    }
```

Ensure that the system account running your Nginx ''worker'' proceses have
enough permission to read the `.lua` file.

确保nginx worker进程的权限能够访问lua文件。

[返回顶部](#目录列表)

TODO
====

[返回顶部](#目录列表)

社区
=========

[返回顶部](#目录列表)

英文邮件列表
--------------------

The [openresty-en](https://groups.google.com/group/openresty-en) mailing list is for English speakers.

[返回顶部](#目录列表)

中文邮件列表
--------------------

The [openresty](https://groups.google.com/group/openresty) mailing list is for Chinese speakers.

[返回顶部](#目录列表)

Bugs and Patches
================

Please report bugs or submit patches by

您可以通过以下途径提交建议或者反馈bug

1.在github上注册并跟踪 [GitHub Issue Tracker](http://github.com/agentzh/lua-resty-redis/issues),

2. 提交到openresty 社区 [OpenResty 社区](#社区).

[返回顶部](#目录列表)
作者
======

Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

[返回顶部](#目录列表)

版权和协议
=====================

This module is licensed under the BSD license.

Copyright (C) 2012-2014, by Yichun Zhang (agentzh) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

[返回顶部](#目录列表)

查看更多
========
* ngx_lua 模块: http://wiki.nginx.org/HttpLuaModule
* REDIS线上协议规范: http://redis.io/topics/protocol
* [lua-resty-memcached](https://github.com/agentzh/lua-resty-memcached) library
* [lua-resty-mysql](https://github.com/agentzh/lua-resty-mysql) library

[返回顶部](#目录列表)

