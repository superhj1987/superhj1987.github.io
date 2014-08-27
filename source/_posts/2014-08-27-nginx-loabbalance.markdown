---
layout: post
title: "Nginx负载均衡"
date: 2014-08-27 14:32:50 +0800
comments: true
categories: 
---

##一、特点

###1.1 应用情况

Nginx做为一个强大的Web服务器软件，具有高性能、高并发性和低内存占用的特点。此外，其也能够提供强大的反向代理功能。俄罗斯大约有超过20%的虚拟主机采用Nginx作为反向代理服务器,在国内也有腾讯、新浪、网易等多家网站在使用Nginx作为反向代理服务器。据Netcraft统计，世界上最繁忙的网站中有11.48%使用Nginx作为其服务器或者代理服务器。基于反向代理的功能，Nginx作为负载均衡主要有以下几点理由：

1. 高并发连接

2. 内存消耗少

3. 配置文件非常简单

4. 成本低廉

5. 支持Rewrite重写规则

6. 内置的健康检查功能

7. 节省带宽

8. 稳定性高

<!--more-->

###1.2 架构

![](/images/ngx_arch.jpg "")

nginx在启动后，会以daemon的方式在后台运行，后台进程包含一个master进程和多个worker进程。工作进程以非特权用户运行。

master进程主要用来管理worker进程，包含：接收来自外界的信号，向各worker进程发送信号，监控worker进程的运行状态，当worker进程退出后(异常情况下)，会自动重新启动新的worker进程。

worker进程则是处理基本的网络事件。多个worker进程之间是对等的，他们同等竞争来自客户端的请求，各进程互相之间是独立的。一个请求，只可能在一个worker进程中处理，一个worker进程，不可能处理其它进程的请求。

开发模型：epoll和kqueue。

支持的事件机制：kqueue、epoll、rt signals、/dev/poll 、event ports、select以及poll。

支持的kqueue特性包括EV_CLEAR、EV_DISABLE、NOTE_LOWAT、EV_EOF，可用数据的数量，错误代码.

支持sendfile、sendfile64和sendfilev;文件AIO；DIRECTIO;支持Accept-filters和TCP_DEFER_ACCEP.

###1.3 性能

Nginx的高并发，官方测试支持5万并发连接。实际生产环境能到2-3万并发连接数。10000个非活跃的HTTP keep-alive 连接仅占用约2.5MB内存。三万并发连接下，10个Nginx进程，消耗内存150M。淘宝tengine团队说测试结果是“24G内存机器上，处理并发请求可达200万”。

##二、负载均衡

###2.1 协议支持

Nginx工作在网络的7层，可以针对http应用本身来做分流策略。支持七层HTTP、HTTPS协议的负载均衡。对四层协议的支持需要第三方插件-yaoweibin的ngx_tcp_proxy_module实现了tcp upstream。

[https://github.com/yaoweibin/nginx_tcp_proxy_module](https://github.com/yaoweibin/nginx_tcp_proxy_module)

此外，nginx本身也逐渐在完善对其他协议的支持：

* Nginx 1.3.13 开发版支持WebSocket代理。

* Nginx 1.3.15开发版支持SPDY。

### 2.2 均衡策略

nginx的负载均衡策略可以划分为两大类：内置策略和扩展策略。内置策略包含加权轮询和ip hash，在默认情况下这两种策略会编译进nginx内核，只需在nginx配置中指明参数即可。扩展策略有很多，如fair、通用hash、consistent hash等，默认不编译进nginx内核。

1. 加权轮询（weighted round robin）

	轮询的原理很简单，首先我们介绍一下轮询的基本流程。如下是处理一次请求的流程图：

	![](/images/ngx_wr_process.jpg)

	图中有两点需要注意，第一，如果可以把加权轮询算法分为先深搜索和先广搜索，那么nginx采用的是先深搜索算法，即将首先将请求都分给高权重的机器，直到该机器的权值降到了比其他机器低，才开始将请求分给下一个高权重的机器；第二，当所有后端机器都down掉时，nginx会立即将所有机器的标志位清成初始状态，以避免造成所有的机器都处在timeout的状态，从而导致整个前端被夯住。

2. ip hash

	ip hash是nginx内置的另一个负载均衡的策略，流程和轮询很类似，只是其中的算法和具体的策略有些变化，如下图所示：

	![](/images/ngx_iphash_process.jpg)

	ip hash算法的核心实现如下：
	
	<pre>
	for(i = 0;i < 3;i++){
		hash = (hash * 113 + iphp->addr[i]) % 6271; 
	}
	
	p = hash % iphp->rrp.peers->number;	
	</pre>	


	从代码中可以看出，hash值既与ip有关又与后端机器的数量有关。经过测试，上述算法可以连续产生1045个互异的value，这是该算法的硬限制。对此nginx使用了保护机制，当经过20次hash仍然找不到可用的机器时，算法退化成轮询。因此，从本质上说，ip hash算法是一种变相的轮询算法，如果两个ip的初始hash值恰好相同，那么来自这两个ip的请求将永远落在同一台服务器上，这为均衡性埋下了很深的隐患。

3. fair

	fair策略是扩展策略，默认不被编译进nginx内核。其原理是根据后端服务器的响应时间判断负载情况，从中选出负载最轻的机器进行分流。这种策略具有很强的自适应性，但是实际的网络环境往往不是那么简单，因此要慎用。

4. 通用hash、一致性hash

	这两种也是扩展策略，在具体的实现上有些差别，通用hash比较简单，可以以nginx内置的变量为key进行hash，一致性hash采用了nginx内置的一致性hash环，可以支持memcache。


###2.2 配置示例
1. HTTP

		http {

           	upstream  www.s135.com  {

              server   192.168.1.2:80;

              server   192.168.1.3:80;

        	}

 

   			server{

              listen  80;

              server_name  www.s135.com;

 

              location / {

                       proxy_pass        http://www.s135.com;

                       proxy_set_header   Host             $host;

                       proxy_set_header   X-Real-IP        $remote_addr;

                       proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;

              }

 

              location /nginx_status {

                       stub_status on;

                       access_log off;

                       allow 192.168.1.1;#设置为可访问该状态信息的ip

                       deny all;

              }

			}

		}
	

2. TCP - ngx_tcp_proxy_module

		tcp {

       	 	upstream cluster {

            	# simple round-robin

            	server 192.168.0.1:80;

            	server 192.168.0.2:80;

 

            	check interval=3000 rise=2 fall=5 timeout=1000;

 

            	#check interval=3000 rise=2 fall=5 timeout=1000 type=ssl_hello;

 

            	#check interval=3000 rise=2 fall=5 timeout=1000 type=http;

            	#check_http_send "GET / HTTP/1.0\r\n\r\n";

            	#check_http_expect_alive http_2xx http_3xx;

        	}

 

        	server {

            	listen 8888;

            	proxy_pass cluster;

        	}

    	}

##三、动态负载均衡

###3.1 自身监控
       
内置了对后端服务器的健康检查功能。如果Nginx proxy后端的某台服务器宕机了，会把返回错误的请求重新提交到另一个节点，不会影响前端访问。它没有独立的健康检查模块，而是使用业务请求作为健康检查，这省去了独立健康检查线程，这是好处。坏处是，当业务复杂时，可能出现误判，例如后端响应超时，这可能是后端宕机，也可能是某个业务请求自身出现问题，跟后端无关。

###3.2 可扩展性

Nginx属于典型的微内核设计，其内核非常简洁和优雅，同时具有非常高的可扩展性。如下图所示：

![](/images/ngx_micro.jpg "")    

Nginx是纯C语言的实现，其可扩展性在于其模块化的设计。目前，Nginx已经有很多的第三方模块，大大扩展了自身的功能。nginx_lua_module可以将Lua语言嵌入到Nginx配置中，从而利用Lua极大增强了Nginx本身的编程能力，甚至可以不用配合其它脚本语言（如PHP或Python等），只靠Nginx本身就可以实现复杂业务的处理。

###3.3 配置修改

nginx的配置架构如下图所示：

![](/images/ngx_config.jpg "")

Nginx支持热部署，几乎可以做到7*24不间断运行，即使运行数个月也不需要重新启动。能够在不间断服务的情况下，对软件版本进行进行升级。Nginx的配置文件非常简单，风格跟程序一样通俗易懂，能够支持perl语法。使用nginx –s reload可以在运行时加载配置文件，便于运行时扩容/减容。重新加载配置时，master进程发送命令给当前正在运行的worker进程worker进程接到命令后会在处理完当前任务后退出。同时，master进程会启动新的worker进程来接管工作。

##四、优势和劣势
###4.1 优势

1. 可以很好地进行http 的头处理

2. 对http协议以及https的良好支持

3. 有足够的第三方插件供使用

4. 支持热部署，更改后端是平滑的

###4.2 劣势

1. 缺少对session的支持

2. 对四层tcp的支持不够好

3. post请求写文件系统，造成500 error

4. 缺乏主动的后端服务器健康监测

5. 默认的监控界面统计信息不全

##五、Tengine

###5.1 特性

1. 继承Nginx-1.2.9的所有特性，100%兼容Nginx的配置；

2. 动态模块加载（DSO）支持。加入一个模块不再需要重新编译整个Tengine；

3. 输入过滤器机制支持。通过使用这种机制Web应用防火墙的编写更为方便；

4. 动态脚本语言Lua支持。扩展功能非常高效简单；

5. 支持管道（pipe）和syslog（本地和远端）形式的日志以及日志抽样；

6. 组合多个CSS、JavaScript文件的访问请求变成一个请求；

7. 更加强大的负载均衡能力，包括一致性hash模块、会话保持模块，还可以对后端的服务器进行主动健康检查，根据服务器状态自动上线下线；

8. 自动根据CPU数目设置进程个数和绑定CPU亲缘性；

9. 监控系统的负载和资源占用从而对系统进行保护；

10. 显示对运维人员更友好的出错信息，便于定位出错机器；

11. 更强大的防攻击（访问速度限制）模块；

12. 更方便的命令行参数，如列出编译的模块列表、支持的指令等；

13. 可以根据访问文件类型设置过期时间；

###5.2 负载均衡

负载均衡方面，Tengine主要有以下几个特点，基本上弥补了 nginx在负载均衡方面的欠缺：

1. 支持一致性Hash模块

2. 会话保持模块

3. 对后端服务器的主动健康检查。

4. 增加了请求体不缓存到磁盘的机制
