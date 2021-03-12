---
layout: post
title:  Dubbo Zookeeper Spring MacOS 短期快速学习
category: [dubbo]
tags: [dubbo]
---

# Dubbo Zookeeper Spring MacOS 短期快速学习-001

@Author:Heng Gao
@Date:2021-03-10

## 第一步 安装Zookeeper

- 常用注册中心之一，作为使用初次学习需要安装的前提条件

1. brew 直接装

``` bash
brew install zookeeper
```

2. 改一下zookeeper的配置文件

   cd /usr/local/Homebrew/etc/zookeeper



   - 由于我是macos，且我的brew配置默认下载地址在/usr/local/Homebrew/etc/
   - 若是手动安装，配置文件可能出现在 /usr/local/etc/zookeeper/
   - 对应文件夹![image-20210310125126648](https://cdn.jsdelivr.net/gh/henggao98/imgbed/posts/image-20210310125126648.png)

3. 开机启动或手动zookeeper

   ```bash
   ==> zookeeper
   To have launchd start zookeeper now and restart at login:
     brew services start zookeeper
   Or, if you don't want/need a background service you can just run:
     zkServer start
   ```

4. 其他命令

   1. zkCli （连接zooKeeper ---client）
   2. zkServer start
   3. zkServer stop
   4. zkServer status
      - ![Screen Shot 2021-03-10 at 13.05.24](https://cdn.jsdelivr.net/gh/henggao98/imgbed/posts/Screen%20Shot%202021-03-10%20at%2013.05.24.png)





## 第二步 Dubbo Demo

Dubbo官方文档提供quickstart demo的源码，官网地址：https://dubbo.apache.org/zh/docs/v2.7/user/quick-start/

Gihub原网地址：https://github.com/apache/dubbo.git

- 可将github直接替换成gitee加速

## 基本的代码解释

官网快速开始：https://dubbo.apache.org/zh/docs/v2.7/user/quick-start/

### 理解结构 — 服务方和消费者通过注册中心远程调用服务

主要解释了服务提供和消费两个角色，阅读过程中我带的问题是“这俩兄弟聊天需要知道哪些信息”，可以拆分如下

- 消费者和服务方两者的<u>身份表示</u>——>“参与者身份”
- 中介的<u>身份表示</u>—-->“第三方身份”
- 服务的表示-“service”
- 其他
  - 端口
  - ip地址
  - 通讯模式

### 代码加载问题

![image-20210310191000956](https://cdn.jsdelivr.net/gh/henggao98/imgbed/posts/image-20210310191000956.png)

### 简单的问题原因 — 暴露服务时指定了注册中心，配置registry时没有设置对应的id。

### 如何指定registry的id？

```xml
<dubbo:registry id="registry1" address="zookeeper://localhost:2181"/>
```

### 如何设置暴露的服务所对应的registry（注册中心）？

```xml
<dubbo:service interface="org.apache.dubbo.demo.DemoService" timeout="3000" registry="registry1" ref="demoService"/>
```

表格为两者关系（通过简单实验）：

| 指定id与否                                                   | <dubbo:registry <u>id="registry1"</u> address="zookeeper://localhost:2181"/> | <dubbo:registry address="zookeeper://localhost:2181"/> |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------ |
| <dubbo:service interface="org.apache.dubbo.demo.DemoService" timeout="3000” **<u>registry=“registry1</u>** ”ref="demoService"/> | ✅                                                            | ❌                                                      |
| <dubbo:service interface="org.apache.dubbo.demo.DemoService" timeout="3000" ref="demoService"/> | ✅                                                            | ✅                                                      |



![image-20210310191213417](https://cdn.jsdelivr.net/gh/henggao98/imgbed/posts/image-20210310191213417.png)

​																				Figure_0: 某次详情



### 导致问题的原因 : 官方文档中提供的zookeeper registry配置代码（🈚️id） , 通过复制黏贴修改配置时<u>**导致id丢失**</u>。

首先，Github源码是正确的，且下载后，仅需保证zookeeper安装成功及运行，源码即可直接运行。（默认配置为zookeeper）

但是官方文档，provider的实例建议使用zookeeper，并提供了example code，并不包含id=“registry1”。

而我根据文档，复制了其registry配置：

![Screen Shot 2021-03-10 at 19.35.09](https://cdn.jsdelivr.net/gh/henggao98/imgbed/posts/Screen%20Shot%202021-03-10%20at%2019.35.09.png)

- 这种情况，可以回到上一段的表格，对照结果为❌。



### 总结

- 问题的bug报错为registry配置异常，应该围绕配置展开探索。
- “替换zookeeper为multicast也不行 ” != “并不能说明问题和两者无关”
  - 我在尝试替换zookeeper后，将问题引导至端口，并开始排查端口占用等不相关问题，耗时巨大。
  - 偏离了registry的配置问题。
  - 其间，还有一个问题，zookeeper我主观臆断其server内置于spring中，不需要单独运行。
    - 再次记录：zookeeper和mysql一样，需要启动服务器，然后在运行项目
      - 回想过往经验：我在分布式项目中使用过JDK中的registry，也是要预先启动registry。



###关于zookeeper的配置文件

首先，zookeepr会提供一个zoo_sample.cfg，但是其默认加载的配置（*下次可以探索其加载流程*）是zoo.cfg， 所以安装完之后，要给他给个名字。

关于配置文件的内容，暂时不深入，初步观察到的参数如下：

- clientPort：2181 // 将2181端口暴露，接受客户端访问
- dataDir=/usr/local/var/run/zookeeper/data //数据源
- initialLimit,syncLimit等其他限制(初次学习暂不深入)