---
title: lua的json操作
date: 2014-10-21 15:07:00 +0800
tags:
- lua
- cjson
- safe
- decode
- encode
---

## 安装cjson
Yichun "agentzh" Zhang (章亦春) 推荐的cjson库

    cd /path/to/you/want/
    wget http://www.kyne.com.au/~mark/software/download/lua-cjson-2.1.0.tar.gz
    tar zxvf lua-cjson-2.1.0.tar.gz && cd lua-cjson-2.1.0

    ## 下面这个sed操作可根据需要才操作，当系统的lua库不是在/usr/local/include路径下时就需要更改
    ## 就是更改Makefile文件中的 LUA_INCLUDE_DIR 这个变量的路径值
    ## 本文是在ubuntu系统下，而lua库路径为/usr/include/lua5.1/下，所以命令如下，其他情况请根据需要修改
    sed -i 's/^LUA_INCLUDE_DIR = .*$/LUA_INCLUDE_DIR = \/usr\/include\/lua5.1\//;' Makefile

    make
    mv cjson.so /path/to/you/want




## 解码decode

新建一测试lua代码文件，名为 t.lua 。代码如下：

    local cjson = require("cjson")
    local str = '{"a":"a", "b":"b"}'
    local j = cjson_safe.decode(str)
    for k, v in pairs(j) do 
        print(k..":"..v)
    end

运行结果如下:

    a:a
    b:b


## 编码encode

代码如下：
    
    local cjson = require("cjson")
    local str = '{"a":"a", "b":"b"}'
    local j_str = cjson_safe.encode(str)
    print(j_str)

运行结果：

    {"a":"a", "b":"b"}

# 解码出错时如何处理？
   
上面的解码都是在正常的json字符串时候的情况，如果非正常json字符串呢？这会导致出错抛出异常并终止执行，这样的行为并不是我们想要的，所以还有一个 cjson.safe 可以使用，其decode与encode返回两个值，第一个值为正常返回对象，若出错则为nil；第二个值为出错时的描述信息，正常时为nil。

## cjson.safe的解码

示例代码：

    -- 非法的json字符串
    local str = '{"a":"a", "b":"b"} c'
    
    -- 使用 cjson.safe
    local cjson = require("cjson.safe")
    
    j, msg = cjson.decode(str)
    if j ~= nil then
        for k, v in pairs(j) do 
            print(k..":"..v)
        end
    else
        print(msg)
    end

运行结果：

    Expected the end but found invalid token at character 20


