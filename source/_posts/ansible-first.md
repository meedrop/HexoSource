---
title: 用Ansible来做自动化(一)：开始第一个Ansible例子
date: 2016-11-20 15:13:01
tags: ansible
categories: ansible
---


系统管理工作经常有很多重复性工作，通过脚本可以提升效率。但换一个环境脚本可能就不适用了，需要进行修改，久而久之，各种脚本各种环境，就会`乱！`。`ansible`很轻松的解决了上述的问题，同时在使用ansible之后，我发现ansible还能实现其他很棒的功能，有一种感觉“遇到ansible真是太棒了！”，所以，费劲我的洪荒之力也要把ansible带给你。
<!--more-->

## 1. 为什么选择Ansible
`ansible`定位为一个`配置管理工具`，和`puppet`,`saltstack`是同类型的工具，关于这三个工具的对比，已经有很多文章描述了，自行谷歌一下，至于我选择ansible的原因，只有以下这一点：
> 简单

如果你的服务器规模没有特别大（小于500台），那么ansible绝对是你最好的选择！

## 2. 幂等性
开始往下写ansible之前，要讲一个很重要的东西：`幂等性`。贴一下百科对幂等性的解释：
> 在编程中，一个幂等操作的特点是其任意多次执行所产生的影响均与一次执行的影响相同。

如果用到shell中解释的话，就是执行一条命令和执行多次这条命令的影响是相同的。下面有个例子：
```python
# 往/etc/hosts文件里增加一行解析
echo "10.0.0.1 test.ansible.com " >> /etc/hosts
# 执行后/etc/hosts内容如下：
10.0.0.1 test.ansible.com

# 再执行一次命令：
echo "10.0.0.1 test.ansible.com " >> /etc/hosts
# 执行后/etc/hosts内容如下：
10.0.0.1 test.ansible.com
10.0.0.1 test.ansible.com
```
可以看到上边的例子第二次执行命令不是我们想要的结果，`/etc/hosts`里多余了一条解析，执行命令的结果不满足幂等性。为了满足幂等性，我们可以把命令加工一下：
```python
# 在写入/etc/hosts文件前进行一次判断
RETVAL=$(cat /etc/hosts | grep -c '10.0.0.1 test.ansible.com')
if [ $RETVAL == 0 ]；then
    echo "10.0.0.1 test.ansible.com " >> /etc/hosts
fi
```
虽然上述的方案解决了问题，但是如果任务需要执行的命令多达百条，你估计要写100个if语句了，将会非常麻烦。而ansbile所有模块都是幂等性的，使用ansible来完成你的任务将会变得非常简单，将上面例子用ansible来实现：
```python
ansible host -m lineinfile -a "dest=/etc/hosts line='10.0.0.1 test.ansible.com'"
```
> 简单解析一下上面命令的意思，如果看不懂没关系^_^，下文会继续深入讨论ansible的使用
`host`: 这是指定你要执行的主机
`-m`: module模块的意思，指定要使用什么模块
`lineinfile`: 一个ansible内置模块，看字面上的意思就是一个实现文本操作模块
`-a`: 就是args，指定模块要使用的参数
`dest`： lineinfile模块的参数，指定要远程文件是什么
`line`:  lineinfile模块的参数，行操作，我的例子里是添加行

这个命令无论执行多少次，/etc/hosts/里面永远只有一条解析。是不是很方便呢^_^

## 3. 选择Ansible的版本
现在`ansible`的主流版本主要有`1.x`和`2.x`版本，选择哪个版本都ok，不过我的建议是直接上2.x的版本吧，我用1.x版本遇到了一些莫名其妙的坑，无论如何都解决不了，最终升级2.x才解决了问题。
另外有一点要注意的是：ansible 2.x 版本命令模式执行速度慢，比起1.x要慢很多，不过这是正常的，一开始我一直以为是自己安装出了问题，排查了好久才找到答案。
相对的，虽然命令执行速度慢了，但2.x跑playbook要比1.x快很多。

## 4. 开始第一个Ansible例子
### 4.1. Ansible原理及实验拓扑
ansible的原理非常简单：`通过ssh登录到(1台或多台)远程主机执行命令`
接下来会基于下图拓扑，开始讲解ansible的例子
![ ](http://ofus5xwey.bkt.clouddn.com/ansible1.png)

主机列表：`下面我只列出了被管理主机列表，ansbile管理主机的信息暂时使用不到，不列出混淆`

| 主机名        |  IP   |  SSH端口  |  远程用户  |
| :--------:   | :-----:  | :----:  | :----:   |
|foo1       | 192.168.125.11  | 12222  |  bar |
|foo2       | 192.168.125.21     | 12222  |  bar  |

### 4.2. 安装Ansible
安装基于centos6系统，如果你不是和我一样的系统，可以参考一下官方的安装文档：
[ansible官方文档](http://docs.ansible.com/ansible/intro_installation.html)
[ansible中文翻译文档](http://ansible-tran.readthedocs.io/en/latest/docs/intro_installation.html)

* 添加阿里云的镜像源
```python
wget http://mirrors.aliyun.com/epel/epel-release-latest-6.noarch.rpm
rpm -ivh epel-release-latest-6.noarch.rpm
```
* 安装ansible
```python
# 目前稳定的最新版本是2.1.2
yum install ansible

Dependencies Resolved
===========================================================================================================================================
 Package                         Arch                           Version                                 Repository                    Size
===========================================================================================================================================
Installing:
 ansible                         noarch                         2.1.2.0-1.el6                           epel                         3.5 M

Transaction Summary
===========================================================================================================================================
```
* 检查一下是否安装成功
```python
ansible --version
# 像这样的返回就是安装成功了
ansible 2.1.2.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = Default w/o overrides
```
### 4.3. 执行第一条ansible命令吧
* 首先什么都不要管，创建一个实验目录`/tmp/ansible`，在里面开始我们第一个实验：
```python
mkdir /tmp/ansible
```
* 创建一个host文件，叫什么名字都行，我的取名叫`MyRemoteHosts`
```python
cd /tmp/ansible
vim MyRemoteHosts
# 写入我要控制的两台主机信息
[foo]
foo1 ansible_ssh_port=12222 ansible_ssh_host=192.168.125.11
foo2 ansible_ssh_port=12222 ansible_ssh_host=192.168.125.21
```
> `[foo]`: 指的是foo这个组，下面的foo1,foo2都是组里的主机
`ansible_ssh_port`: 远程主机的SSH端口，不指定默认使用22端口
`ansible_ssh_host`: 远程主机的IP地址，如果这里没有指定，/etc/hosts里必须要有这条解析
>
在生产环境中，一般来说是不会使用默认22端口的，所以这个例子更加贴近生产环境，让你少走弯路。
ansible只能通过主机名的方式来控制远程主机，不能直接通过IP地址来访问，所以要有`host-->ip`的解析

* 把远程主机的时间都打印出来看看，在ansible管理主机上执行：
`ansible -i MyRemoteHosts foo -u bar -m shell -a "date" -k `
输出结果：
![ ](http://ofus5xwey.bkt.clouddn.com/ansible-first-command.png)

* 在命令后面增加-vvv来观察一下ansible的详细执行流程
![ ](http://ofus5xwey.bkt.clouddn.com/ansible-first-vvv.png)
*可以看到其实ansible先把要执行的命令写到本地的一个文件里，然后把文件发送到远程主机的一个目录下，然后执行，再删掉临时文件*

> 解析一下上面命令行的含义
`-i`: 指定hosts文件的路径，如果没有指定默认会使用/etc/ansible/hosts。在指定的host文件后面，写的是主机组，或者主机名
`-u`: 指定远程主机运行命令的用户，如果没有指定，ansible主机当前用户运行。
`-m`: module模块的意思，指定要使用什么模块
`shell`: 使用shell这个内置模块，就像在远程主机的shell下执行命令一样
`-a`: 就是args，指定模块要使用的参数
`-k`: 指定输入远程主机的ssh密码。ansible默认会认为你和远端主机已建立了互信，如果已经建立了ssh互信可以不使用这个参数。有的情况我们需要执行一次性操作，并不要建立ssh互信，这个参数就发挥了用途。
`-vvv`: 这个参数显示ansible执行的详细过程

* 除了上面的命令，你还可以这么做，例如对单台主机操作，试试下面的命令会有什么结果吧^_^
```python
ansible -i MyRemoteHosts foo1 -u bar -m shell -a "date" -k
ansible -i MyRemoteHosts foo2 -u bar -m shell -a "date" -k
ansible -i MyRemoteHosts foo1,foo2 -u bar -m shell -a "date" -k
```

## 5. 小结
上文中我没有提及`inventory`，因为我觉得这个叫法会让人觉得难以理解，其实inventory非常简单，它就等价于上文中的host文件`MyRemoteHosts`。

还有模块的使用方法，这篇文章里不会写，我觉得那并不是ansible的重点，可以参考这篇文档：http://http://breezey.blog.51cto.com/2400275/1555530/
或者通过`ansible-doc 模块名`，英文能力好我建议第二种方法。

会不会觉得ansible的命令会有点长^_^？其实很多的变量可以事先定义好，不过我想让你从底层更清晰的了解ansible的运作方式，稍微弄的复杂了点。简化后的ansible命令可能像这样子：
```python
ansible foo -u bar -m shell -a "date"
```
* `host`文件默认会读`/etc/ansible/hosts`，解析预先在`/etc/hosts`里做好
* 与远程主机事先添加好互信
* ssh端口可以在配置文件`/etc/ansible/ansible.cfg`里定义好，就不用在`inventory`文件在写一次

好了这篇文章先到这里，现在已经可以使用ansible做一点小任务了^_^，ansible更厉害的东西，请等待下一篇文章：*用Ansible来做自动化(二): 简单的不得了的Playbook*
