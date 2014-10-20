---
title: nginx + lua实现逻辑处理与日志周期性落地
date: 2014-10-13 17:14:00 +0800
tags:
- nginx
- lua
---

#### 问题由来

对外提供一些API接口时，采用nginx为web服务器，当nginx接收到请求时，将请求转发给真正的处理逻辑系统。这样一来会存在以下不足：

- 服务的稳定性，同时依赖于nginx与后面的业务处理逻辑系统，只有两个都正常时，该服务才是正常的
- 存在请求转发，这里有一定的性能损耗。若转发是在同台服务器或同个网络内，还是可以忽略的；但若是不同网络时，还是需要考虑一下的。
- 业务逻辑处理系统的日志落地方式还是比较可控的，不过，nginx的日志只能落地在一个固定路径下（至少我还没找到其他可配置成周期滚动的方式），另外nginx落地日志的格式可配置的变量只能使用nginx官方所提供的那些配置变量，有时不一定满足想要的字段或格式。

#### 问题思考

针对以上的不足，就想到扩展下nginx，把接收web请求并判断web请求是否成功或满足要求的逻辑在nginx扩展中实现，然后再把相关日志周期性地落地下来，解除nginx与后面的业务处理系统的耦合，后面的业务处理系统通过读取落地好的日志来继续完成后续的事情。这样一来，就可以解决掉上面所说的那些不足了。



#### nginx扩展方式

nginx扩展方式有两种：

- 使用c语言，性能好，但编写代码麻烦，且每次还需重编译布署
- 使用lua语言，性能不会比c语言差多少，且不需要重新编译nginx

所以，我就选用了lua来对nginx进行扩展，在lua语言中实现我需要的判断web请求是否成功并周期性的落地日志的逻辑。

#### 前置条件

- 需要一个安装好了lua扩展环境的nginx，至于如何编译并安装好nginx+lua，这样的文章搜索就有好多，按照教程摸索安装下即可。
- 熟悉lua语法，关键是掌握lua的nginx的相关配置指令及可用的API

#### nginx扩展的逻辑流程

实现以上所说功能，nginx扩展的逻辑流程如下：

- 在 init_by_lua_file 所指向的文件代码中，定义日志文件句柄的全局变量，并定义控制文件句柄的打开、关闭函数（日志滚动时需要调用这两个函数）。
- 在 init_by_lua_file 所指向的文件代码中，定义日志文件当前所属周期（如天级或小时级）的全局变量，用于判断是否周期变动，从来滚动日志文件句柄。另外，定义更新周期变量的函数。
- 在你想要的那个请求路径下配置 content_by_lua_file ，指向使用lua语言实现判断web请求及落地日志的处理逻辑的代码文件，在该代码文件中，使用上面所提到的两个全局变量，进行相关判断即可实现周期性的滚动文件句柄，并进行日志落地。

## 样例

下面以一个例子做为说明。

想在 example.com/test 路径下实现上面所说的处理逻辑，有以下要求：

- example.com/test 是一个api接口路径，请求时需要三个参数，分别为a，b，c。其中，a参数是必须的，b和c参数可传可不传
- 日志按小时级进行落地
- 一次请求为一行日志，一行日志按顺序包括以下字段：time, ip, ua, a, b, c 。一行日志总共有5个字段，每个字段之间用 \x02 字符分割。

#### init_by_lua_file 配置

在 nginx的配置文件中有如下类似配置：

	init_by_lua_file /path/to/init.lua;

	### 定义一个共享字典，在lua代码中可使用，该共享字典用于解决在日志句柄滚动时
	### 的并发问题，用于确保在滚动发生时只有一个请求在更新日志句柄，
	### 而其他的请求在等待那一个请求完成后，使用新文件句柄写数据。
	### 该共享字典在 content_by_lua_file 中会使用到。
	lua_shared_dict example_test_dict 1M;

#### init.lua

因为 init_by_lua_file 只能配置一次，所以只能指向一个文件，该文件是共用的，所以以下内容追加到 init.lua 文件中即可。

	---- example.com init
	
	--- 方便更改日志落地的基本目录
	local EXAMPLE_TEST_LOG_DIR_BASE = "/path/to/log/dir"
	
	--- 日志文件中名称的前面部分，方便识别
	local EXAMPLE_TEST_FILENAME_PRE = "example"
	
	--- 日志所属的当前周期的全局变量，小时级
	example_test_log_hour_level = string.sub(ngx.localtime(), 1, 13)

	
	--- 更新日志所属的当前周期的全局变量的函数
	--- 之所以用函数更新，因为全局变量在跨文件时是无法更新的，下面的函数也是同理
	function example_test_log_hour_level_update(hour_level)
	    example_test_log_hour_level = hour_level
	end
	
	--- 日志落地的文件句柄，全局变量
	example_test_log_fo = nil
	
	--- 更新日志落地文件句柄的全局变量的函数
	function example_test_log_openFile()
	    local curT = ngx.time()     --unix timestamp, in seconds
	    local dir_path = EXAMPLE_TEST_LOG_DIR_BASE .. "/" .. os.date("%Y%m/%d/%H", curT)
	    local exec_code = os.execute("mkdir -m 755 -p " .. dir_path)
	    if 0 ~= exec_code then
	        ngx.log(ngx.ERR, "can't mkdir -m 755 -p " .. dir_path)
	        return nil
	    end
	
	    local file_path = dir_path .. "/" .. EXAMPLE_TEST_FILENAME_PRE .. os.date("_%Y%m%d%H.log", curT)
		
	
	    local err_msg, err_code
	    example_test_log_fo, err_msg, err_code = io.open(file_path, "a")
	    if nil == example_test_log_fo then
	        ngx.log(ngx.ERR, "can't open file: " .. file_path .. ", " .. err_msg .. ", " ..  err_code)
	        return nil
	    else
	        os.execute("chmod 644 " .. file_path)
	        return example_test_log_fo
	    end
	end
	
	--- 关闭日志句柄的函数
	function example_test_log_closeFile()
	    if example_test_log_fo ~= nil then
	        example_test_log_fo:close()
	    end
	end
	
	--- 在init时调用一次，初始化文件句柄
	example_test_log_openFile()


#### example.com的nginx中server对test路径的配置

	server {
		server_name example.com;
	
		...
		
		location = /test {
			content_by_lua_file /path/to/content_test.lua;
		}
		...
	}

#### content_test.lua

	--- 设置返回的html的header
	ngx.header.content_type = 'text/html; charset=utf-8';
	
	local req = ngx.req.get_uri_args()
	
	--- 获取a参数，并判断是否有值，若没有返回701的status，表示请求不成功
	local a  = req["a"]
	if a == nil then
	    ngx.exit(701)
	end
	
	--- 获取time,ip,ua,b,c参数
	local req_headers = ngx.req.get_headers()
	local b = req["b"]
	local c = req["c"]
	local curTime = ngx.localtime()
	local ip = ngx.var.remote_addr
	local ua = req_headers["User-Agent"]
	
	--- 若请求中未带b或c参数，置其为空字符串
	if nil == b then
	    b = ""
	end
	if nil == c then
	    c = ""
	end
	
	--- 拼接一行日志中的内容
	local msg = cur_time .. "\02" .. ip .. "\x02" .. ua .. "\x02" .. a .. "\02" .. b .. "\02" .. c .. "\n"
	
	--- 获取当前时间周期，用于判断是否需要进行文件句柄的滚动	
	local cur_hour_level = string.sub(ngx.localtime(), 1, 13)
	
	--- 当大于时，表示到了需要滚动日志文件句柄的时候
	if cur_hour_level > example_test_log_hour_level then
	    --- 在共享字典中，用add命令实现类似于锁的用途。
	    --- 只有当共享字典中原来没有要add的key时，才能操作成功，否则失败。
	    --- 这样的话，有多个请求时，只能有一个请求add成功，而其他请求失败，休眠0.01秒后重试。
	    --- 唯一add成功的那个请求，则关闭老文件句柄，并滚动新文件句柄，并更新表示文件句柄的时间周期的那个全局变量。
	    local shared_dict = ngx.shared.example_test_dict
	    local rotate_key = "log_rotate" 
	    while true do
	        if cur_hour_level == example_test_log_hour_level then
	            break
	        else
	            -- the exptime, is to prevent dead-locking 
	            local ok, err = shared_dict:add(rotate_key, 1, 10) 
	            if not ok then 
	                ngx.sleep(0.01)
	            else
	                if cur_hour_level > example_test_log_hour_level then
	                    example_test_log_closeFile()
	                    example_test_log_openFile()
	                    if example_test_log_fo == nil then
	                        ngx.log(ngx.ERR, "example_test_log_openFile error")
	                        ngx.exit(911)
	                    end
	                    example_test_log_hour_level_update(cur_hour_level)
	                end
	                shared_dict:delete(rotate_key)
	                break
	            end
	        end
	    end
	end
	
	--- 落地日志内容
	example_test_log_fo:write(msg)
	example_test_log_fo:flush()
	
	--- 正常退出，返回请求，表示成功
	ngx.exit(ngx.HTTP_OK)

#### 注意事项

- 注意确保日志落地目录的权限，确保有足够的权限进行新目录和文件的生成，不然在进行日志文件句柄的初始化或滚动时会出错，无法生成新文件。

#### 性能

本文方法，对于nginx的正常配置下，用apache的ab进行了压测，性能稳定保持在 16000 rps 左右。


