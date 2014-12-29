---
title: nginx与lua的安装教程
date: 2014-11-14 12:02:00 +0800
tags:
- nginx
- lua
- luajit
---

* toc
{:toc}


## 准备安装文件

### ngx_devel_kit
    
    wget https://github.com/simpl/ngx_devel_kit/archive/v0.2.19.tar.gz -O ngx_devel_kit-v0.2.19.tar.gz

### lua-nginx-module

    wget https://github.com/openresty/lua-nginx-module/archive/v0.9.13.tar.gz -O lua-nginx-module-v0.9.13.tar.gz

### nginx

    wget http://nginx.org/download/nginx-1.7.9.tar.gz

### lua

**本文采用LuaJIT。**

    wget http://luajit.org/download/LuaJIT-2.0.3.tar.gz






## 安装LuaJIT

    tar zxvf LuaJIT-2.0.3.tar.gz 
    cd LuaJIT-2.0.3/
    make PREFIX=/path/to/luajit-2.0.3
    make install PREFIX=/path/to/luajit-2.0.3

**两次的PREFIX的值必须一致，或都不使用，即默认安装到/usr/local下。**

建立软链接，以便后面nginx启动时可以找到所需文件：

    sudo ln -s /path/to/luajit-2.0.3/lib/libluajit-5.1.so.2 /usr/lib/libluajit-5.1.so.2

## 安装nginx

编译前，先安装需要用到的基础模块，我用的是ubuntu12.04 server LTS版，所以用以下命令安装：

    sudo apt-get install libpcre3-dev libssl-dev

告诉nginx编译时到哪里找到luajit

    export LUAJIT_LIB=/path/to/luajit-2.0.3/lib
    export LUAJIT_INC=/path/to/luajit-2.0.3/include/luajit-2.0


解压缩两个nginx编译时需要用到的模块源代码

    tar zxvf ngx_devel_kit-v0.2.19.tar.gz 
    tar zxvf lua-nginx-module-v0.9.13.tar.gz

编译安装nginx

    ./configure --prefix=/path/to/nginx --add-module=/path/to/ngx_devel_kit-0.2.19 --add-module=/path/to/lua-nginx-module-0.9.13 --with-http_ssl_module
    make -j2
    make install

## 安装lua常用的库

lua一些常用的库，还是需要安装上的，这样方便以后随时使用，库的存放位置，这里在安装好后的nginx目录下新建一个`lua_so`目录：

    cd /path/to/nginx
    mkdir lua_so

### redis库

下载

    wget https://raw.githubusercontent.com/openresty/lua-resty-redis/master/lib/resty/redis.lua

无需编译，拷贝到文件夹中：

    cp redis.lua /path/to/lua_so

### struct pack

下载
    
    wget http://www.inf.puc-rio.br/~roberto/struct/struct-0.2.tar.gz
    mkdir struct-0.2
    tar zxvf struct-0.2.tar.gz -C struct-0.2
    cd struct-0.2/

修改makefile中的`LUADIR`变量如下：

    LUADIR = /path/to/luajit-2.0.3/include/luajit-2.0/

编译，并移到文件夹：

    make
    cp struct.so ~/env/nginx/lua_so/

### cjson

下载cjson，并解压

    wget http://www.kyne.com.au/~mark/software/download/lua-cjson-2.1.0.tar.gz
    tar zxf lua-cjson-2.1.0.tar.gz
    cd lua-cjson-2.1.0

修改`MakeFile`文件中关于`LUA_INCLUDE_DIR`的设置，如下：

    #LUA_INCLUDE_DIR =   $(PREFIX)/include
    LUA_INCLUDE_DIR =   /home/work/env/lua/default/include/luajit-2.0

编译，并移动到上面所说的`lua_so`文件夹中：

    make
    cp cjson.so /path/to/lua_so

### lua-zlib

下载lua-zlib

    git clone https://github.com/brimworks/lua-zlib.git
    cd lua-zlib

建立一个liblua.so的软链接，不然编译不成功：

    sudo ln -s /path/to/luajit-2.0.3/lib/libluajit-5.1.so /usr/lib/liblua.so   

修改`Makefile`文件，使以下几个变量值如下：

    LUAPATH  ?= /path/to/luajit-2.0.3
    LUACPATH ?= /path/to/luajit-2.0.3/lib
    INCDIR   ?= -I/path/to/luajit-2.0.3/include/luajit-2.0
    LIBDIR   ?= -L/usr/lib -L/path/to/luajit-2.0.3/lib

编译，并拷贝至`lua_so`文件夹：

    make linux
    cp zlib.so /path/to/lua_so


### mysql, bitop

下载mysql.lua，mysql不需要编译，直接使用mysql.lua文件即可，不过会用到bit.so，所以要编译bit.so

    wget https://raw.githubusercontent.com/openresty/lua-resty-mysql/master/lib/resty/mysql.lua
    cp mysql.lua /path/to/lua_so

下载bitop

    wget http://bitop.luajit.org/download/LuaBitOp-1.0.2.tar.gz
    tar zxvf LuaBitOp-1.0.2.tar.gz
    cd LuaBitOp-1.0.2

修改LuaBitOp中的Makefile文件前面中的INCLUDES变量如下：

    INCLUDES= -I/path/to/luajit-2.0.3/include/luajit-2.0

编译，并拷贝到`lua_so`文件夹：

    make
    cp bit.so /path/to/lua_so



### 把lua_so的路径告诉nginx

在`nginx.conf`的http模块下添加以下语句：

    # 查找 *.lua
    lua_package_path '/path/to/lua_so/?.lua;;';
    # # 查找 *.so
    lua_package_cpath '/path/to/lua_so/?.so;;';

