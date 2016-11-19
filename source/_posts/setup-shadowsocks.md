---
title: 搭建一个快速稳定的Shadowsocks
date: 2016-10-30 20:27:50
tags: 翻墙
categories: vpn  
---

# 1. Shadowsocks的简单介绍
简单来说`Shadowsocks`（下文简称ss）就是一个vpn，基于`ssl协议`的vpn，ssl协议比较火的vpn还有`Openvpn`，不过Openvpn的协议特征太明显了，很容易就会被封掉了。Openvpn的场景更适合在国内使用，而国外翻墙毫无疑问我们选择ss。                                       
> 对于ss的详细介绍可以看一下这篇文章 http://vc2tea.com/whats-shadowsocks/


# 2. 准备一台境外的VPS主机
如何选择VPS可以参考知乎的讨论：[请问，国外或香港有哪些好用的 VPS 或者独立主机？](https://www.zhihu.com/question/21432244)
几番挣扎最后我选择了[ramnode](http://www.ramnode.com/)这家的OpenVZ主机，可以通过paypal付款，购买的配置：`15美元/年`、`500G流量`、`128M内存/64M SWAP`、`12G SSD固态硬盘`、`西雅图地区`，截了一下价格表：

![ ](http://ofus5xwey.bkt.clouddn.com/ramnode.png)

用了大概差不多一个月了，在周末晚上高峰时段会有点差，其余时间速度还不错，比较稳定的。
> 对了这里要提一点：OpenVZ的主机是不支持ss一些修改系统内核参数等的提速方案的，如果你有这方面的要求，建议购买贵一点的KVM主机


# 3. 安装ss
ramnode默认安装的系统是`CentOS6`发行版本，所以下面的安装都是基于centos系统，如果是其他的系统例如ubuntu，命令会有些许不同，可作为参考。
## 3.1. 安装python
没有特殊变化，下文的安装都基于`root`用户，CentOS6 默认自带了python，不需要安装，保证你的python版本大于`2.4`
```
# python -V
Python 2.6.6

```
## 3.2. 安装pip
我们将通过`pip`这个工具来安装ss，pip默认系统是没有带的，需要手动安装。同时安装pip需要`setuptools`依赖包，所以共需要安装2个包:
[setuptools-28.7.1.tar.gz](https://pypi.python.org/packages/1d/04/97e37cf85972ea19364c22db34ee8192db3876a80ed5bffd6413dcdabe2d/setuptools-28.7.1.tar.gz#md5=263e2e3858187e8ea20b4db3ff8770bb)
[pip-8.1.2.tar.gz](https://pypi.python.org/packages/e7/a8/7556133689add8d1a54c0b14aeff0acb03c64707ce100ecd53934da1aa13/pip-8.1.2.tar.gz#md5=87083c0b9867963b29f7aba3613e8f4a)
上传2个安装包到你的服务器下，开始安装，记得确认自己的路径（下面的教程都是基于相对路径）:

1.安装setuptools

```
# tar -xf setuptools-28.7.1.tar.gz 
# cd setuptools-28.7.1
# python setup.py install

```

2.安装pip

```
# tar -xf pip-8.1.2.tar.gz 
# cd pip-8.1.2
# python setup.py install

```
## 3.3. 安装ss
上文安装pip之后，安装ss就变得非常简单，一个命令搞定：
```
# pip install shadowsocks
```
正确安装成功后，输入`ssserver`有如下提示即可(ssserver就是ss的控制命令)：
```python
# ssserver -h
usage: ssserver [OPTION]...
A fast tunnel proxy that helps you bypass firewalls.

You can supply configurations via either config file or command line arguments.

Proxy options:
  -c CONFIG              path to config file
  -s SERVER_ADDR         server address, default: 0.0.0.0
  -p SERVER_PORT         server port, default: 8388
  -k PASSWORD            password
  -m METHOD              encryption method, default: aes-256-cfb
  -t TIMEOUT             timeout in seconds, default: 300
  --fast-open            use TCP_FASTOPEN, requires Linux 3.7+
  --workers WORKERS      number of workers, available on Unix/Linux
  --forbidden-ip IPLIST  comma seperated IP list forbidden to connect
  --manager-address ADDR optional server manager UDP address, see wiki

General options:
  -h, --help             show this help message and exit
  -d start/stop/restart  daemon mode
  --pid-file PID_FILE    pid file for daemon mode
  --log-file LOG_FILE    log file for daemon mode
  --user USER            username to run as
  -v, -vv                verbose mode
  -q, -qq                quiet mode, only show warnings/errors
  --version              show version information

Online help: <https://github.com/shadowsocks/shadowsocks>
```
## 3.4. 配置和启动ss
首先我们需要写一个ss的配置文件`conf1.json`，让ss启动的时候去读，我的配置文件放在了`/root/lqz/ssConfig`目录下，`conf1.json`内容如下：
```
{
    "server":"0.0.0.0",   # 这是监听的地址，0.0.0.0意味着所有的地址都能来连我
    "server_port":65455,  # 监听的端口
    "password":"123456",  # 这是客户端登陆密码
    "timeout":120,        # 连接超时时间
    "method":"rc4-md5"    # 机密的算法
}
```
写好配置文件后，一句命令启动ss
```
/usr/local/bin/python /usr/local/bin/ssserver -c conf1.json --pid-file /tmp/ssserver1.pid
```
这样接下来就可以通过客户端连接，翻墙上谷歌了。
> 好了到这里，已经搭建完了ss，同时也可以使用了，赶紧动手吧~
如果你想你的ss跑的更稳定更快速，不想有时候突然掉线连不上，可以继续往下看^_^


# 4. 让你的ss变得快速和稳定
让ss变得稳定和快速要做三件事情：
>1. 安装supervisor
>2. 给ss端口分流
>3. 定期重启ss

## 4.1. 安装supervisor
`supervisor`是一个进程集中管理工具，可以管理`ssserver`进程，当ssserver有问题时候，supervisor会自动重启ssserver，说白了就是supervisor变成了ssserver的父亲，保护着ss。
安装supervisor很简单，也是一句命令：
```
# pip install supervisor
```
安装完后配置一下supervisor的配置文件及读取目录：
```
# touch /etc/supervisord.conf && mkdir /etc/supervisord
# echo_supervisord_conf > /etc/supervisord.conf
# vim /etc/supervisord.conf   # 将supervisor配置文件最后一段改为这样子(记得注释[include]前面的;号)
[include]
;files = relative/directory/*.ini
files = /etc/supervisord/*.conf
```
supervisor也是一个进程，为了方便对这个进程的控制，我写了一个控制脚本`superviosrd`，将下面文本复制，命名`superviosrd`，放到/etc/inti.d/下:
```
#!/bin/bash

### BEGIN INIT INFO
# Provides:       supervisord
# Required-Start: 
# Default-Start:  3 5
# Default-Stop:   0 1 2 6
# Short-Description: supervisord server daemon (supervisord)
# Description:    Start supervisord server daemon (supervisord).
### END INIT INFO


$BIN=$(which python | sed 's#/py.*$##g')
PROC_NAME="supervisord"	
PYTHON="$BIN/python"
SUPERVISORD="$BIN/supervisord"
SUPERVISORCTL="$BIN/supervisorctl"
CONF="/etc/supervisord.conf"

# 获取进程id模块
get_pid() {
	PID=$(ps -ef | grep "$SUPERVISORD" | grep  -v grep | awk '{print $2}')
	PID=${PID:-0}
}

# 停止模块
stop(){
	get_pid
    if [ $PID != 0 ];then
		echo "$PROC_NAME stopping..."
        $SUPERVISORCTL stop all
		kill $PID
        sleep 2s
        # 判断是否关闭成功，不成功强制kill
		get_pid
        if [ $PID != 0 ];then
                kill -9 $PID
                sleep 1s
        fi
        echo "$PROC_NAME stopped!"
    else
		echo "$PROC_NAME already stopped,nothing to do."
    fi
}

# 启动模块
start(){
	get_pid
    if [ $PID == 0 ];then
		echo "$PROC_NAME starting..."
		if [ $UID == 0 ];then
			$PYTHON $SUPERVISORD -c $CONF
		elif [ $(id | awk '{print $1}' | grep -c $EXEC_USER) == 1 ];then
			$PYTHON $SUPERVISORD -c $CONF
		else
			echo "please run as user $EXEC_USER!"
			exit 1
		fi
		sleep 1s
		echo "$PROC_NAME started!"
    else
		echo "$PROC_NAME already started,nothing to do."
    fi
}

# 状态模块
status(){
	get_pid
    if [ $PID == 0 ];then
        echo "$PROC_NAME is not running."
	else
        echo "$PROC_NAME is running."
    fi
}

# main
case $1 in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status
        ;;
    restart)
        stop 
        start
        ;;
    *|usage)
        echo "Usage: [$0 {start|stop|status|restart}]"
        ;;
esac

```
　　添加脚本`supervisord`开机自启动：
```
# chmod a+x /etc/init.d/supervisord
# chkconfig --add supervisord
# chkconfig supervisord on
```
## 4.2. 给ss端口分流
我们记得配置ss可以通过`conf1.json`这个配置文件来做，里面写了一个端口对吧？如果所有的流量包括`手机`，`电脑`，`平板`都走这个端口，是不是压力就会很大呢？有没有什么解决办法？
> 写多一个配置文件，让ssserver启动多一个端口。

是的，就用上面那个方法，所以我们写多一个配置文件`conf2.json`:
```
{
    "server":"0.0.0.0",
    "server_port":65456, # 端口要和conf1.json不一样
    "password":"123456", # 密码可以一样，如果你给别人用，最好改一下啦
    "timeout":120,
    "method":"rc4-md5"
}
```
启动命令也要两个：
```shell
/usr/local/bin/python /usr/local/bin/ssserver -c conf1.json --pid-file /tmp/ssserver1.pid
/usr/local/bin/python /usr/local/bin/ssserver -c conf2.json --pid-file /tmp/ssserver2.pid
```

## 4.3. 定期重启ss
我们都知道程序运行久了很容易出问题，定期重启是一个很好的解决方案，所以定期重启ss能让你的ss跑的更稳定，同时减少维护成本。
> 启动了2个端口的ssserver，如果重启则要分别启动，一共打2次命令。
> 如果通过supervisor管理，只用启动supervisor，一共打1次命令

所以发现用了supervisor的好处了吧，同时supervisor还能保护子进程呢。下面开始设置ssssever作为子进程方式运行。

supervisor通过配置文件添加子进程，添加`ssserver1.conf`,`ssserver2.conf`两个配置文件：
```
# vim /etc/supervisord/ssserver1.conf
[program:ssserver1]
directory = /root/lqz/ssConfig
command = /usr/local/bin/ssserver -c conf1.json --pid-file /tmp/ssserver1.pid
autostart = true
autorestart = true
startsecs = 3
startretries = 3
stdout_logfile = /dev/null
stderr_logfile = /dev/null

# vim /etc/supervisord/ssserver2.conf
[program:ssserver2]
directory = /root/lqz/ssConfig
command = /usr/local/bin/ssserver -c conf2.json --pid-file /tmp/ssserver2.pid
autostart = true
autorestart = true
startsecs = 3
startretries = 3
stdout_logfile = /dev/null
stderr_logfile = /dev/null
```
启动supervisor：
```
# service supervisord start
```
用`supervisorctl`查看子进程的运行情况，我们两个ssserver进程都在跑咯
```
# supervisorctl
ssserver1                        RUNNING   pid 6002, uptime 5:20:25
ssserver2                        RUNNING   pid 1410, uptime 5:20:25
supervisor> 
```
最后一步，设置每天早上5点ssserve自动重启一次：
```
# crontab -e
00 05 * * * /etc/init.d/supervisord restart
```


# 5. 下载SS客户端
ss客户端的配置这里就不再阐述了，可以去搜索一下，文档非常的多^_^，提供一下所有版本ss客户端的下载地址：
> https://github.com/breakwa11/shadowsocks-rss/wiki/Server-Setup
