---
title: ubuntu下安装mysql + php + php-fpm + nginx
date: 2014-12-26 10:53:00 +0800
tags:
- mysql
- nginx
- php
- fpm
- ubuntu
---

* toc
{:toc}

## 系统环境

ubuntu 14.04 64bit Server LTS, 具备公网访问能力

## 所编译安装的各软件版本

mysql: 5.6.22

php: 5.6.4

nginx: 1.7.9





## 前期环境准备

### 通过`apt-get`安装的组件

    sudo apt-get install libwrap0-dev libncurses5-dev libssl-dev libxml2-dev libsslcommon2-dev libcurl4-openssl-dev pkg-config libbz2-dev libjpeg-dev libpng12-dev libxpm-dev libfreetype6-dev libmcrypt-dev libxslt1-dev libiconv-hook-dev libpcre3-dev

### 通过编译安装的组件

#### iconv

    wget http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.14.tar.gz
    tar zxf libiconv-1.14.tar.gz
    cd libiconv-1.14

在编译安装之前，需要修改一下某处的源代码，不然会遇到错误，编辑`srclib/stdio.in.h`文件，在698行左右，将以下一行

    _GL_WARN_ON_USE (gets, "gets is a security hole - use fgets instead");

注释掉，即加上 // 即可，如下

    //_GL_WARN_ON_USE (gets, "gets is a security hole - use fgets instead");

然后，继续编译安装

    ./configure --prefix=/usr/local
    make
    sudo make install

### 重载下库

重载下库，是为了确保前面安装的那些组件，在下面的步骤中能够被找到，该步骤不一定需要，不过多做一步，安心一些也无妨，命令如下

    sudo /sbin/ldconfig


## mysql

### 下载mysql与解压

    wget http://cdn.mysql.com/Downloads/MySQL-5.6/mysql-5.6.22.tar.gz
    tar zxf mysql-5.6.22.tar.gz
    cd mysql-5.6.22/

### 编译mysql

将其中的`/path/to/mysql`换成想要的安装路径，或不进行指定使用默认位置

    cmake \
    -DCMAKE_INSTALL_PREFIX=/path/to/mysql \
    -DWITH_INNOBASE_STORAGE_ENGINE=1 \
    -DDEFAULT_CHARSET=utf8 \
    -DDEFAULT_COLLATION=utf8_general_ci \
    -DENABLED_LOCAL_INFILE=1 \
    -DENABLED_PROFILING=1 \
    -DWITH_EMBEDDED_SERVER=1 \
    -DWITH_EXTRA_CHARSETS=all \
    -DWITH_LIBWRAP=1 \
    -DWITH_SSL=yes

若在编译过程中遇到问题时，请查看输出进行相应解决，并在`rm CMakeCache.txt`之后再次进行编译，直至成功编译。

### 安装mysql

成功编译之后，进行安装，make这一步骤比较耗时

    make

make成功之后，安装至编译时所指定的位置，需不需要`sudo`取决于该位置，你是否有写权限，若是没有就带上`sudo`吧，有的话就不用了。

    sudo make install   ## 是否需要sudo看安装位置权限而定

### 配置mysql

以下配置内容是我根据自己的机器情况进行参数配置的，***拷贝使用的时候请根据所使用的机器情况进行调整，特别是其中的目录与文件路径***，保存为`my.cnf`文件。

    # For advice on how to change settings please see
    # http://dev.mysql.com/doc/refman/5.6/en/server-configuration-defaults.html
    # http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html
    
    [client]
    port = 3306
    socket = /tmp/mysql.sock
    
    [mysqld]
    # These are commonly set, remove the # and set as required.
    basedir = /path/to/mysql
    datadir = /path/to/data
    port = 3306
    server_id = 1
    socket = /tmp/mysql.sock
    
    # Remove leading # and set to the amount of RAM for the most important data
    # cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
    innodb_buffer_pool_size = 768M
    innodb_additional_mem_pool_size = 64M
    innodb_data_home_dir = /path/to/data
    innodb_data_file_path = ibdata1:256M:autoextend
    
    # Remove leading # to turn on a very important data integrity option: logging
    # changes to the binary log between backups.
    log_bin
    
    
    # 每个线程单独分配缓冲区
    join_buffer_size = 16M
    
    # order by或group by的缓冲区大小
    sort_buffer_size = 8M
    myisam_sort_buffer_size = 8M
    
    # 随机读缓冲区大小。当按任意顺序读取行时(例如，按照排序顺序)，将分配一个随机读缓存区。进行排序查询时，MySQL会首先扫描一遍该缓冲，以避免磁盘搜索，提高查询速
    度，如果需要排序大量数据，可适当调高该值。但MySQL会为每个客户连接发放该缓冲空间，所以应尽量适当设置该值，以避免内存开销过大。
    read_rnd_buffer_size = 8M
    
    read_buffer_size = 2M
    
    sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
    
    # 交互用户不活动连接被自动断开的时间
    interactive_timeout = 7200
    
    # key_buffer_size只对MyISAM表起作用。即使你不使用MyISAM表，但是内部的临时磁盘表是MyISAM表，也要使用该值。可以使用检查状态值created_tmp_disk_tables得知详情。
    key_buffer_size = 256M
    
    # 默认为10秒，这里改为5秒，超过该值即为慢查询
    long_query_time = 5
    
    query_cache_size = 16M
    query_cache_limit = 1M
    
    max_heap_table_size = 128M
    tmp_table_size = 128M

### 初始化mysql

    cd /path/to/mysql
    scripts/mysql_install_db --defaults-file=/path/to/my.cnf

### 启动mysql

    cd /path/to/mysql
    bin/mysqld_safe --defaults-file=/path/to/my.cnf &

### 给root加上密码

    cd /path/to/mysql
    bin/mysqladmin -u root password -S /path/to/mysql.sock

在输出的提示中输入新的密码。

### 关闭数据库

    cd /path/to/mysql
    bin/mysqladmin -u root -p shutdown -S /path/to/mysql.sock


## php

### 下载php与解压

    wget http://cn2.php.net/distributions/php-5.6.4.tar.gz
    tar zxf php-5.6.4.tar.gz
    cd php-5.6.4

### 编译php

编译时开启了php很多常用的特性，这样在线上使用时基本不会缺少相应的库了，***编辑参数中的路径请自行替换***。

    ./configure \
    --prefix=/path/to/php \
    --with-mysql=/path/to/mysql \
    --with-mysqli=/path/to/mysql/bin/mysql_config \
    --with-pdo-mysql=/path/to/msyql \
    --enable-fpm \
    --with-curl \
    --with-pear \
    --with-gd \
    --with-jpeg-dir \
    --with-png-dir \
    --with-zlib \
    --with-xpm-dir \
    --with-freetype-dir \
    --with-mcrypt \
    --with-mhash \
    --with-openssl \
    --with-xmlrpc \
    --with-xsl \
    --with-bz2 \
    --with-gettext \
    --disable-debug \
    --enable-exif \
    --enable-wddx \
    --enable-zip \
    --enable-bcmath \
    --enable-calendar \
    --enable-ftp \
    --enable-mbstring \
    --enable-soap \
    --enable-sockets \
    --enable-shmop \
    --enable-dba \
    --enable-sysvsem \
    --enable-sysvshm \
    --enable-sysvmsg


### 安装php

编译成功后，运行以下命令安装php，sudo看权限需要是否带上。make操作时会比较耗时。

    make ZEND_EXTRA_LIBS='-liconv'
    sudo make install

### 配置php.ini

安装成功后，从php的源代码目录中拷贝`php.ini-production`文件至安装好的php的lib目录下,命令如下：

    cp php.ini-production /path/to/php/lib/php.ini

**对`php.ini`的内容做一些修改，在927行左右，去掉 date.timezone 的注释，并设置为 date.timezone = PRC**，内容大致如下：

    [Date]
    ; Defines the default timezone used by the date functions
    ; http://php.net/date.timezone
    date.timezone = PRC

### 配置php-fpm.conf

    cd /path/to/php/etc
    cp php-fpm.conf.default php-fpm.conf

### php-fpm权限控制与提示

默认php-fpm.conf所指定的user与group均为nobody，ubuntu 14.04 server默认建有nobody用户，但未有nobody组，所以用以下命令建立nobody组，并将nobody用户归到其组下
    
    sudo groupadd nobody
    usermod -g nobody nobody
    sudo usermod -g nobody nobody

#### 权限控制提示

默认使用的nobody用户与组，对于权限来说就已经够用了，下面根据几种情况给出在配合nginx使用时的安全配置的例子

##### 数据缓存目录

数据缓存目录的特点，就是需要777权限，但无须提供给外部用户访问与使用，所以可如下配置

    location ~ "^/cache" {
        return 403;
    }
    
    location ~ "\.php$" {
        fastcgi_pass 127.0.0.0:9000;
        ....................
    }

##### 上传目录

此目录的特点是需要开放访问权限，但所有文件不能由php 引擎解析（包括后缀名改为 gif 的木马文件），可如下配置

    location ~ "^/attachments" {
    
    }
    
    location ~ "\.php$" {
        fastcgi_pass 127.0.0.0:9000;
        ....................
    }
 
注意，上面对attachments 目录的 location 定义中是没有任何语句的。 nginx 对正则表达式的 location 匹配优先级最高，任何一个用正则表达式定义的 location,  只要匹配一次，将不会再匹配其它正则表达式定义的 location 。
 
现在，请在attachments 目录下建立一个 php 脚本文件，再通过浏览器访问安，我们发现浏览器提示下载，这说明 nginx 把 attachments 目录下的文件当成静态文件处理，并没有交给 php fastcgi 处理。这样即使可写目录被植入木马，但因为其无法被执行，网站也就更安全了。

##### 静态文件生成目录

这些目录一般都是php 生成的静态页的保存目录，显然与附件目录有类似之处，按附件目录的权限设置即可。 

可以预见的是，如果我们设置了较严格的权限，即使网站php 程序存在漏洞，木马脚本也只能被写入到权限为 777 的目录中去，如果配合上述严格的目录权限控制，木马也无法被触发运行，整个系统的安全性显然会有显著的提高。

### 启动php-fpm

    sudo /path/to/php/sbin/php-fpm

### 关闭php-fpm

关闭的话，使用以下命令查找出进程号

    sudo ps -x | grep php-fpm

输出结果中应该会有多行，其中有一行有 master process 的字样的 php-fpm 的进程号即为我们要查找的进程号，假设查找到的进程号为 1214 ,则运行以下命令关闭 php-fpm 进行

    sudo kill -QUIT 1214

## nginx

### 下载nginx与解压

    wget http://nginx.org/download/nginx-1.7.9.tar.gz
    tar zxf nginx-1.7.9.tar.gz

### 编译nginx

    ./configure --prefix=/path/to/nginx --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module

其中`--with-http_gzip_static_module`，此模块的作用就是在接到请求后，会到url相同的路径的文件系统去找扩展名为“.gz”的文件，比如：http://www.iteye.com/stylesheets/homepage.css，nginx就会先查找 stylesheets/homepage.css.gz 这个文件，如果存在直接把它发送出去，如果不存在，再将stylesheets/homepage.css文件进行gzip压缩，再发送出去，这样可以避免重复的压缩无谓的消耗资源，这个模块不受gzip_types限制，会对所有请求有效。所以建议不要在全局上使用，因为一般来说大部分都是动态请求，是不会有.gz这个文件的，建议只在局部我们确认有.gz的目录中使用。Nginx不会自动的将压缩结果写入文件系统，这点不同于lighttpd，所以如果想使用gzip_static模块，需要自己写脚本生成.gz文件。如果你的服务器因为某种原因不能使用 Nginx Gzip Static 模块，你也没必要耿耿于怀。事实上，Nginx Gzip Static 模块虽然对于页面访问速度有一定的提升，但是提升幅度是非常有限的，每次压缩后再传输比预读取 gz 文件传输慢不了多少，况且现在的主流浏览器都有缓存机制。
    
### 安装nginx

    make
    sudo make install

### 配置nginx

将 /path/to/nginx/`conf/nginx.conf` 的内容替换为如下的内容，**不过请修改其中某些选项的值，其中的参数是根据我自己机器的情况进行优化的**。  

    user  nobody    nobody;
    
    ## worker_processes的数值设置为cpu的核数
    worker_processes  4;
    ## 绑定每个nginx进程所使用的CPU，使nginx可以利用cpu的多核，根据上面的不同数值进行设置，下面是数值为4时的设置
    worker_cpu_affinity 0001 0010 0100 1000;
    
    #pid        logs/nginx.pid;
    
    events {
        use epoll;
        worker_connections  65536;
    }
    
    http {
        include       mime.types;
        default_type  application/octet-stream;

        ## 启用gzip
        gzip on;
        gzip_disable "msie6";
        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_min_length 1100;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    
        # 缓存被频繁访问的文件相关的信息，默认是off
        # 参数注释如下：
        # max:设置缓存中的最大文件描述符数量，如果缓存被占满，采用LRU算法将描述符关闭。
        # inactive:设置存活时间，默认是10s
        # min_uses:设置在inactive时间段内，日志文件最少使用多少次后，该日志文件描述符记入缓存中，默认是1次
        # valid:设置检查频率，默认60s
        # off：禁用缓存
        open_log_file_cache max=10 inactive=20s valid=60s min_uses=2;
    
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        ## 若考虑性能时，可增加缓冲
        #access_log  logs/access.log main buffer=16k;
        access_log  logs/access.log main;
        error_log   logs/error.log; 
    
        ## 调整客户端超时时间
        client_max_body_size 50M;
        client_body_buffer_size 1m;
        client_body_timeout 15;
        client_header_timeout 15;
        keepalive_timeout 15;
        send_timeout 15;
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
    
        ## 调整输出缓冲区大小
        fastcgi_buffers 256 16k;
        fastcgi_buffer_size 128k;
        fastcgi_connect_timeout 3s;
        fastcgi_send_timeout 120s;
        fastcgi_read_timeout 120s;
        reset_timedout_connection on;
        server_names_hash_bucket_size 100;
    
        server {
            listen       80;
            server_name  localhost;
    
            location / {
                root   html;
                index  index.html index.htm;
            }
    
            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }
        }
    }

### 启动nginx

    sudo /path/to/nginx/sbin/nginx

若nginx正常启动成功，在本机直接访问`http://localhost`会看到nginx的欢迎页面。

### 重载nginx

当对nginx变更了配置之后，可进行重载，对于现网服务不会有任何影响，当配置有错时，重载失败但不影响现网的继续服务，可修改配置后再次进行重载

    sudo /path/to/nginx/sbin/nginx -s reload

### 停止nginx

    sudo /path/to/nginx/sbin/nginx -s stop
