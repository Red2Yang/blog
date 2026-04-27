---
title: Apache HTTP Server（Windows）
# author: Red2Yang
date: 2026-04-27 10:00:00 +0800
categories: [项目, 技术]    # 最多两个层级
tags: [写作]          # 小写，多个用空格或逗号
---

## Apache

Apache（音译为阿帕奇）是服务的一种，具体来说，它是一个 **Web 服务** 软件。它把一台普通的计算机变成了一个可以存放网站并让全世界的人访问的服务器。它默默地运行在后台，时刻等待着响应来自网络上的请求。它可以运行在几乎所有广泛使用的计算机平台上。由于其跨平台和安全性被广泛使用，是最流行的Web服务器端软件之一。它快速、可靠并且可通过简单的API扩充，将Perl/Python等解释器编译到服务器中。

不妨在此给出结论：所有的网页都是程序在运行，可能是php、html、python等等各种语言编写的程序。不同的程序占用机器上不同的虚拟端口。只需要用户连接不同的端口，服务器就提供多样的Web服务。

> 可以这样理解。比如要连接到mc服务器，一般要连接到服务器的`25565`端口，而不是任意一个端口。
{: .prompt-tip }

因此，分配好这些端口，同时对Web访问的协议（包括tcp、udp、http、https等）进行管理，对服务器而言是非常重要的事。而类似于Apache、Nginx这类软件就是处理`http`和`https`协议的。

## Apache 的安装

官网windows平台[链接](https://httpd.apache.ac.cn/docs/current/platform/windows.html#down)

Linux可以通过软件包安装，网上教程更是数不胜数。因此本文不再叙述。

1.下载

官方团队不提供windows版本的Apache。但有很多第三方编译的Apache For Windows。

如：https://www.apachelounge.com/download/

（以下内容都以该版本为基础）

2.安装

在安装前，必须要做的是，需要安装`Visual C++ Redistributable`运行库。
其余的配置（如安装目录、文件目录）都可以更改。

```txt
  You must first install the Visual C++ Redistributable for Visual Studio 2017-2026 https://aka.ms/vc14/vc_redist.x64.exe
  Download and Install, if you have not done so already, see:

   https://www.apachelounge.com/download/

  Unzip the Apache24 folder to c:/Apache24 (that is the ServerRoot in the config).
  The default folder for your your webpages is DocumentRoot "c:/Apache24/htdocs"

  When you unzip to an other location: 
  change Define SRVROOT "c:/Apache24"  in httpd.conf, for example to "E:/Apache24"
```

解压到指定位置后，在对应位置的终端运行`.\httpd.exe -k install`，即可安装Apache到系统服务。同理，卸载使用`.\httpd.exe -k uninstall`。

> 如果需要为不同的服务使用特定命名的配置文件（不是网站配置，网站配置见后文），则必须使用以下命令：
>
> ```PowerShell
> .\httpd.exe -k install -n "MyServiceName" -f "c:\files\my.conf"
> ```
{: .prompt-tip }

接下来需要将其注册到环境变量，在设置-系统-系统信息-高级系统设置-环境变量中，找到系统变量里`Path`的值，打开后应该有很多条路径，新建一条`C:\Program Files\Apache24\bin`后，就可以在命令行启用`httpd`命令。

> 常用命令：
>
> ```PowerShell
> httpd -k start   #启动
> httpd -k stop    #停止
> httpd -k reload  #重载配置
> ```
{: .prompt-tip }

## Apache 的使用

首先需要修改`Apache24\conf\httpd.conf`文件，删除注释符就可以启用配置。

1. Apache安装位置：`Define SRVROOT "C:/Program Files/Apache24"`
2. 服务器文件根目录（默认）：`ServerRoot "${SRVROOT}"`

然后需要启动各种模块。

1. 比如常用的`LoadModule ssl_module modules/mod_ssl.so`激活`https`配置，
2. `LoadModule proxy_http_module modules/mod_proxy_http.so`和`LoadModule proxy_module modules/mod_proxy.so`启用反向代理，
3. `LoadModule vhost_alias_module modules/mod_vhost_alias.so`启用虚拟主机模块，这是当前网站配置所必须的，也是接下来内容的基础。

启用虚拟主机模块后，通过`Apache24\conf\extra\httpd-vhosts.conf`就可以指定端口和网站配置。

以`mcsmanager`服务为例，通过反向代理将其相关服务放置到80端口的子路径`/mcsm`下，实现`http://127.0.0.1/mcsm`访问：

```conf
Listen 80
<VirtualHost *:80>
    ErrorLog "logs/80-error_log"
    CustomLog "logs/80-access_log" common

	<Proxy *>
		Order deny,allow
		Allow from all
	</Proxy>
	SetEnv proxy-nokeepalive 1
	SetEnv proxy-sendchunked 1

    ProxyAddHeaders off
    ProxyPreserveHost On	
	ProxyRequests     off
	ProxyVia On
	ProxyTimeout 300
	ProxyIOBufferSize 65536

	RewriteEngine On
	RewriteCond %{HTTP:Upgrade} websocket [NC]
	RewriteCond %{HTTP:Connection} upgrade [NC]

    # mcsm
	ProxyPass /mcsm "http://127.0.0.1:23333/mcsm"
	ProxyPassReverse /mcsm "http://127.0.0.1:23333/mcsm" 
	ProxyPass /mcsm-ws "http://127.0.0.1:24444/mcsm-ws"
	ProxyPassReverse /mcsm-ws "http://127.0.0.1:24444/mcsm-ws" 

</VirtualHost>
```

> 反向代理是更加高阶的内容，主要目的就是实现一个端口多个服务。
>
> 在本例中，`mcsmanager`也需要设置内部路径。
{: .prompt-tip }