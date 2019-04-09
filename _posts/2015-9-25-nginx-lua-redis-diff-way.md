---
layout: default
title:  nginx调用redis的几种方式
date: 2015-09-25
description: nginx调用redis的几种方式
img: nginx_redis.jpg
tags: [tech, nginx, redis]
---

## 直接调用redis module
用的是httpredis2module的原始方式。 
新建一个upstream：

    upstream redis_server{ server 127.0.0.1:6379 weight=1; }

    location /foo {
        set $value 'first';
        redis2_query set one $value;
        redis2_query get one;
        redis2_query set one two;
        redis2_query get one;
        redis2_pass redis_server;
    }
这里用了管道，将几条命令一起发过去。 
这里没有用到lua,所以不能实现逻辑，返回的是redis的原始数据。
优点是简单直接，直接在配置就能一目了然，如果是简单的逻辑，如设置upstream的hash值的时候可能会用到。

##subrequest的方式
在nginx.conf里面新建一个location

    location = /redis2 {
    internal;
    redis2_raw_queries $args $echo_request_body;
    redis2_pass redis_server ;
     }
然后就是插入lua脚本，

    local parser = require "redis.parser"
    ngx.req.read_body()
    local args = ngx.req.get_post_args()
    local key=ngx.var.arg_key;
    local value=nil;
        local reqs = {
            {"get", key}
        }
    
        local raw_reqs = {}
        for i, req in ipairs(reqs) do
            table.insert(raw_reqs, parser.build_query(req))
        end
    
        local res = ngx.location.capture("/redis2?" .. #reqs,
            { body = table.concat(raw_reqs, "") })
        if res.status ~= 200 or not res.body then
            ngx.log(ngx.ERR, "failed to query redis")
            ngx.exit(500)
        end
    
       local replies = parser.parse_replies(res.body, #reqs)
        for i, reply in ipairs(replies) do
            if(reply[1] ~= nil) then
                    ngx.say(reply[1])
            end
        end     
用的是ngx.location.capture调用redis module后再进行处理，这是一种常用的方式。好处是能利用subrequest的异步请求和upstream模块的各种功能。在性能上要比下面这种方式要好很多。

## 在lua内部的连接方式
用lua-resty-redis的方式，相对于前面几种方式的好处是你不是用proxy_pass， 所以你可以对连接（redis:new(),redis:connect()）等方式在一个文件中处理，从逻辑上清晰很多，而且对连接的状态（如一次连接的操作次数get_reused_times） 进行获取。不足就是利用不了upstream模块的功能。当然你可以自己写一个轮询算法对几个server进行访问。不过感觉这样就没什么意思了。

        local redis = require "resty.redis" local red = redis:new()
        
                red:set_timeout(1000) -- 1 sec
        
                -- or connect to a unix domain socket file listened
                -- by a redis server:
                --     local ok, err = red:connect("unix:/path/to/redis.sock")
        
                local ok, err = red:connect("127.0.0.1", 6379)
                if not ok then
                    ngx.say("failed to connect: ", err)
                    return
                end
        
                ok, err = red:set("dog", "an aniaml")
                if not ok then
                    ngx.say("failed to set dog: ", err)
                    return
                end
        
                ngx.say("set result: ", res)
        
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
                -- with 0 idle timeout
                local ok, err = red:set_keepalive(0, 100)
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




