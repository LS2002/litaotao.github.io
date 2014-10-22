---
category: erlang
published: false
layout: post
title: 深度分析Erlang Application行为
description: 通过仔细分析《Erlang and OTP in Action》中第六章的练习来分析Erlang中Application行为
---

## 从我在Erlang and OTP in Action中第六章中的错误说起
　　前两天看EOIA这本书，觉得终于可以用Erlang来搞点东西玩了，于是决定按照书中流程来实践一下所谓的缓存系统。
谨慎起见，我还是半抄半写把simple_cache的源码写好了，当前目录结构如下：

```
root@kali:~/Desktop/erl/6-EOIA# ls -R
.:
doc  ebin  include  priv  src

./doc:

./ebin:
prim_consult.beam  sc_element.beam  sc_sup.beam       simple_cache.beam
sc_app.beam        sc_store.beam    simple_cache.app

./include:

./priv:

./src:
prim_consult.beam  prim_consult.erl  sc_app.erl  sc_element.erl  sc_store.erl  sc_sup.erl  simple_cache.erl
root@kali:~/Desktop/erl/6-EOIA#
```

**小提示：**   
1、要把src目录下的erl源文件编译，并把编译后的beam文件放到ebin下有一个快捷的方法，在当前目录下执行：erlc -o ebin/ src/*.erl；   
2、上面出现的prim_consult.erl和prim_consult.beam不用管，后面会提到的，这是我阅读application源码时提取出来的； 


## 看看application行为在启动一个otp应用的简单流程


## load应用的过程分析


## start应用的过程分析



















理论上来讲，这是个很简单的事情，甚至可以说不能再简单了。

1. 在VirtualBox里面建一个OpenSolaris 64bit的虚拟机，配置足够大的内存和硬盘
2. 把[SmartOS live ISO](https://us-east.manta.joyent.com/Joyent_Dev/public/SmartOS/smartos-latest.iso)挂载在光驱上
3. 启动，回答屏幕上几个问题，搞定

没错，SmartOS就是这么简单。甚至可以用个[简单的脚本](http://www.perkin.org.uk/posts/automated-virtualbox-smartos-installs.html)自动化完成。

## 创建SmartOS zone

SmartOS里面的zone就是OS container，轻量级，隔离性好，所以一般搞搞都先弄一个，简单的应该是这么搞搞就行了

先导入镜像

```
imgadm import fdea06b0-3f24-11e2-ac50-0b645575ce9d
```

然后创建一个json描述文件

```
cd /opt
vi myvm.json
```



```json
{
 "brand": "joyent",
 "image_uuid": "fdea06b0-3f24-11e2-ac50-0b645575ce9d",
 "alias": "web01",
 "hostname": "web01",
 "max_physical_memory": 512,
 "quota": 20,
 "nics": [
  {
    "nic_tag": "admin",
    "ip": "10.88.88.52",
    "netmask": "255.255.255.0",
    "gateway": "10.88.88.2"
  }
 ]
}
```

最后

```
vmadm create -f myvm.json
```

完事了`zlogin`到你的机器就行了。

好了我就不班门弄斧了，我都是从[这里](http://wiki.smartos.org/display/DOC/How+to+create+a+zone+%28+OS+virtualized+machine+%29+in+SmartOS)抄的。

## 老湿，我的zone上不了外网啊！

**更新：这件事情在最新版的SmartOS中应该已经修复了，见[这里](http://www.listbox.com/member/archive/184463/2014/05/sort/time_rev/page/1/entry/14:199/20140525031110:BCCD6764-E3DB-11E3-AC74-888FDBEDD5D3/)。**

_以下内容应该已经过时，但是还是留下做个参考吧_

> VirtualBox我设置成NAT了？啥啥我也设对了？问题解决不了，joyent的源码我都快看完了，我的zone还是上不了外网啊？？？？？？

我也上不了 :)

在joyent的SmartOS wiki看到这么点[信息](http://wiki.smartos.org/display/DOC/How+to+create+a+Virtual+Machine+in+SmartOS)(页内搜索Zone Networking Issues)。老实说，给出了方向，连接都失效了，细节全留给你自己想了。一句话，领导风范。

具体的这事情是这样：

1. 默认SmartOS ISO只针对vmware做了优化，所以默认建了个vmware的bridge，还把默认网卡绑上了。
用`dladm show-link`看看，有个`vmwarebr`吧。

2. 按照joyent指出的方向，问题就在要建一个叫做`vboxbr`而不是`vmwarebr`。看来是硬编码问题，典型的扯淡问题。

那么来做

先删了那个vmwarebr

```
dladm remove-bridge -l e1000g0 vmwarebr
dladm delete-bridege vmwarebr
```

再建一个

```
dladm create-bridge -l e1000g0 vboxbr
```

理论上讲这样就行了。不用重启zone，就行了。joyent是这么保证的，但是实际上有点问题，可能要把某zone删了重来（或者重新配置网络）。注意zone的网卡的mask和gateway要和e1000g0的一致。

举个栗子吧

```
[root@08-00-27-1e-28-f6 ~]# netstat -rn

Routing Table: IPv4
  Destination           Gateway           Flags  Ref     Use     Interface
-------------------- -------------------- ----- ----- ---------- ---------
default              10.0.2.2             UG        4     122171 e1000g0
10.0.2.0             10.0.2.15            U         3          0 e1000g0
127.0.0.1            127.0.0.1            UH        2         84 lo0
192.168.56.0         192.168.56.13        U         4     223275 e1000g1
```

可见主网卡`e1000g0`的gateway是`10.0.2.2`，网段是`10.0.2.x`。那么zone的nic就应该如下配置

```json
{
  "nics": [
    {
      "nic_tag": "admin",
      "ip": "10.0.2.52",
      "netmask": "255.255.255.0",
      "gateway": "10.0.2.2"
    }
  ]
}
```

其中`ip`那里我随便选了个`10.0.2.52`，反正不和别的机器冲突就行。