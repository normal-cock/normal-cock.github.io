---
title: Http-And-Tcp——Http Server漫游(2)
date: 2019-12-29 10:44:49
categories: Http Server漫游
tags: 
     - HTTP
     - TCP
     - Keep-Alive
     - Asynchronous
     - Multiplexing
---
通过手动代码实验，感受从HTTP/1.0到HTTP/2的演进过程中，剖析HTTP与TCP的“爱恨情仇”。

<!-- more -->
## 1. 写在开头
上回我们介绍了[阻塞\非阻塞与同步\异步](https://zhuanlan.zhihu.com/p/88728018)的区别和联系。

本次我们将通过自己动手写代码，做实验，感受从HTTP/1.0，到HTTP/1.1，再到HTTP/2的演进过程中，HTTP与TCP的爱恨情仇（既然是HTTP漫游系列，不说HTTP就有点过分了）。

有同学可能会问，HTTP/3都快出来了，为什么你只说到HTTP/2呢？因为从HTTP/3开始，HTTP已经正式跟TCP分手，跟UDP组了CP，以后会找个机会跟大家分享一下TCP如何被UDP挖了墙角的。

## 2. 先说结论

1. **结论1**：从HTTP/1.0到HTTP/2，都是利用TCP作为底层协议进行通信的。
2. **结论2**：HTTP/1.1，引进了长连接(keep-alive)，减少了建立和关闭连接的消耗和延迟。
3. **结论3**：HTTP/2，引入了多路复用：连接共享，提高了连接的利用率。

**结论1**不用解释，也不是我们今天的重点。
至于**结论2**和**结论3**中分别提到的**长连接**和**多路复用**，如果你跟我一样第一眼看到之后一脸懵逼，隐约明白什么意思，但仔细一想越想越糊涂感觉两者之间并没有区别的话，那么恭喜你，这篇文章就是为你而生。本次分享的目的就是为了帮助有缘人通过自己动手来理解这些晦涩的文字，即讲清楚**长连接**和**多路复用**的区别。

## 3. 长连接和多路复用

还是先用白话文说一下结论：

* 长连接是连接层面的优化，但***同一个HTTP/1.1连接只能串行处理请求***。
* 多路复用是协议层面的优化，能够让***多个请求可以共用同一个HTTP/2连接且互不影响***。

举个例子。HTTP/1.1的时候，假如A请求正在占用连接C（即client发送了request，但还没有收到response），此时B请求只能等待A请求结束之后才能再次利用连接C来处理自己。那有没有更快的方法？有！就是当一个请求占用了一个连接时，建立一个新的连接来处理请求B。

而有了多路复用，即使A请求正在占用着连接C，B请求依然可以利用连接C，同时丝毫不受请求A是否完成的影响。

## 4. 实验思路

好，结论有了，我们就来看看如何通过实验来验证我们的结论。

我们的实验思路是：通过连续向**同一个连接**向服务端发送了两个sleep(5s)的请求，并观察HTTP/1.1和HTTP/2的服务器处理这两个请求分别需要多长时间，也就是client收到response分别需要多久。

按照我们的结论，对于HTTP/1.1的服务器，两个请求需要10s来完成；对于HTTP/2的服务器，两个请求只需要5s来完成。

## 5. 开始试验

### 5.1. 实验需要准备的东西

* python3
* nc(netcat)。用于发起HTTP/1.1请求
* [hyper](https://github.com/python-hyper/hyper)。用于发起HTTP/2请求。
* [gunicorn](https://github.com/benoitc/gunicorn)。用与搭建HTTP/1.1 Server。
* [h2](https://github.com/python-hyper/hyper-h2)。用于搭建HTTP/2 Server。

### 5.2. 注意点

1. gunicorn默认的Sync worker_class没有支持keep-alive（长连接，即我们要测试的HTTP/1.1特性），需要使用其他的worker_class，例如eventlet。
2. 为什么用nc而不用requests库？通过阅读requests和urllib3代码发现，一个connection在使用时会pop，导致无法再访问到该connection。所以无法使用requests和urllib3模拟多个请求争抢同一个连接的情况。
3. linux下使用nc有个坑，http协议使用的\r\n的换行，而linux下按下回车键只输入\n。所以需要使用`nc -C`（大写的C）命令将回车变成\r\n。

### 5.3. 服务端的部署

不管是启动HTTP/1.1 Server还是HTTP/2 Server，都可以使用wsgi协议，所以代码是同一套。如下所示：

```python
# app.py
import time
def app(environ, start_response):
    """Simplest possible application object"""
    data = b'Hello, World!\n'
    status = '200 OK'
    response_headers = [
        ('Content-type', 'text/plain'),
        ('Content-Length', str(len(data)))
    ]
    time.sleep(5)
    start_response(status, response_headers)
    return iter([data])
```

#### 5.3.1. 启动HTTP/1.1 Server

```shell
gunicorn app:app --keep-alive 10 -b 0.0.0.0:8010 -w 2 --log-level DEBUG -k eventlet
```
关键参数解释:
* `app:app`。前一个`app`代表的是文件名，即`app.py`；后一个`app`代表的是函数名。
* `--keep-alive 10`。代表开启HTTP/1.1的Keep Alive功能，且连接的保持时间为10s。
* `-k eventlet`。代表使用`eventlet`作为`work_class`，这个上面已经解释过，默认的`work_class`不支持Keep Alive。
* `-w 2`。代表说明worker数量对HTTP/1.1的表现并没有影响。

#### 5.3.2. 启动HTTP/2 Server

目前没有现成的可以直接启动HTTP/2 Server的python包，`h2`仅仅是提供了解析HTTP/2协议的能力。但是`h2`的文档上有提供启动HTTP/2 Server的[实例代码](https://python-hyper.org/projects/h2/en/stable/wsgi-example.html)，虽然用于生产环境还有很多工作要做，但是用于我们的实验是妥妥的足够了。

启动步骤为：

1. 将实例代码保存到名为`asyncio-server.py`的文件中，并且与`app.py`放在同一个文件夹下。
2. `python asyncio-server.py app:app`。注意，端口默认为`8443`，可以通过修改`asyncio-server.py`中的代码来修改端口。

总的来说还是挺方便的。

### 5.4. 发起请求观察实验结果

#### 5.4.1. HTTP/1.1

##### 5.4.1.1. 发起请求

我们知道，nc会建立一个tcp连接。所以下面的命令是借用一个tcp连接来发送两次http请求

```shell
echo -e "GET / HTTP/1.1\r\n\r\nGET / HTTP/1.1\r\n\r\n"|nc 127.0.0.1 8010
```

##### 5.4.1.2. 实验结果

HTTP/1.1的请求结果如下，可以看到，第一次请求结束5s只有，第二次请求才结束。也就是说两次请求总共花了10s的时间，验证了**同一个HTTP/1.1连接只能串行处理请求**的结论。

```shell
HTTP/1.1 200 OK
Server: gunicorn/19.9.0
Date: Sun, 05 Jan 2020 05:40:34 GMT
Connection: keep-alive
Content-type: text/plain
Content-Length: 14

Hello, World!
HTTP/1.1 200 OK
Server: gunicorn/19.9.0
Date: Sun, 05 Jan 2020 05:40:39 GMT
Connection: keep-alive
Content-type: text/plain
Content-Length: 14

Hello, World!
```

#### 5.4.2. HTTP/2

##### 5.4.2.1. 发起请求

```python
import hyper
import time
def test():
    begin_time = time.time()
    conn = hyper.HTTP20Connection('localhost:8443')
    # 发送两次请求
    first_stream_id = conn.request('GET', '/get')
    second_stream_id = conn.request('GET', '/get')

    first_resp = conn.get_response(first_stream_id)
    second_resp = conn.get_response(second_stream_id)
    print(('two request cost %.2fs' % (time.time()-begin_time)))
    print(first_resp.read())
    print(second_resp.read())

test()

```

##### 5.4.2.2. 实验结果

请求结果如下，可以看到，通过同一个连接处理两次请求，总共花了5s。验证了我们的结论：**多个请求可以共用同一个HTTP/2连接且互不影响**。

```shell
two request cost 5.05s
b'Hello, World!\n'
b'Hello, World!\n'
```

## 6. 参考资料

* https://en.wikipedia.org/wiki/HTTP/2
