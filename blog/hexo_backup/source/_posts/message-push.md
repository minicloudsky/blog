---
title: 消息推送简介
date: 2019-10-07 09:49:28
tags:
---

​     在互联网业务中，有很多场景，都需要向用户推送消息，例如，一个论坛系统中，当用户发帖后，其他用户进行回复，需要对发帖者进行消息通知，在电商业务中，当用户下单后，管理后台需要通知用户下单成功；在一些秒杀和购物场景中，当用户提交订单未付款时候，一般会有一个截止时间，当即将到达截止时间时候，需要通知用户及时付款，在这些场景中，如何对用户实现消息通知呢?

### 简单消息推送

  传统的开发模式，一般是前端轮询，例如在论坛系统中，在用户个人页面，前端轮询后端接口，后端返回对应的消息数据，这种方式需要每隔几秒就向后台发请求，对于实时性要求比较高的系统，轮询频率会更高，之前参与维护迭代的pc端消息推送，采用的是这种方式，前端轮询后端接口，后端通过查询数据库，判断是否有新消息，有的话就将数据返回给前端，前端进行弹窗或者其它方式进行展示，数据库设计大概如下：

 <!-- more -->

| user_token | message | add_time   | is_read |
| ---------- | ------- | ---------- | ------- |
| 张三       | hello   | 1551671111 | 0       |
| 李四       | world   | 1561561789 | 1       |
| 王五       | yes     | 1781567177 | 1       |

当需要对某个用户，或者某些业务进行消息推送时候，在消息记录表，插入一条数据，以及消息的阅读状态，已读和未读，然后根据前端需求，返回该用户的全部消息，或者全部未读消息，当用户点击消息后，前端发请求，后端修改消息的阅读状态。

这种方式的实现比较简单无脑，基本上只需要查询修改下基本上就可以完成，适合用户量较少，对于实时性要求不高的业务场景，当用户量较大时候，查询和插入效率很低，需要加缓存或者对数据库进行分库分表，从而提高业务的稳定性和可用性，目前在一些小公司和简单的论坛一类的业务，该方式的消息推送可能还在使用，在未来的开发中，这种方式不推荐使用，因为效率低下，前端频繁轮询查接口，会对后端数据库造成很大压力，而且当有多个场景的消息推送时候，这种方式会让用户客户端的网页异常卡顿，加载速度慢，用户体验很差。

### 多场景消息推送

在电商业务中，很多消息类型，需要向用户发送短信，例如用户下单，退款，下单未付款，快递配送进度等等，这一类消息，通常需要根据不同的业务场景，有些需要给客户发送短信通知，在小程序端，可能需要调用微信开发平台的接口，进行小程序或者公众号消息推送，这一类相对来说比较容易实现，在需要消息推送时候，直接调用发送短信接口，或者微信平台接口进行消息推送，当数据量比较大时候，可以暂时把这些推送服务加入一个异步队列，进行分批次推送，这些适用于对时效性较低的场景，例如微信的每日运动排行。

对于这类消息推送，一般需要在业务执行时候，添加到异步队列中，通过倒计时一类的方式进行触发，常用的有 Redis 的发布与订阅，WebSocket 全双工通信，服务器端可以向浏览器发送消息，或者通过 Celery 执行异步队列，从而实现消息推送，下面简单介绍下 Celery。

Celery 任务队列一般用于线程或计算机之间分配工作的一种机制。

任务队列的输入是一个称为任务的工作单元，有专门的工作进行不断的监视任务队列，进行执行新的任务工作。

Celery 通过消息机制进行通信，通常使用中间人（Broker）作为客户端和职程（Worker）调节。启动一个任务，客户端向消息队列发送一条消息，然后中间人（Broker）将消息传递给一个职程（Worker），最后由职程（Worker）进行执行中间人（Broker）分配的任务。

Celery 可以有多个职程（Worker）和中间人（Broker），用来提高 Celery 的高可用性以及横向扩展能力。

Celery 需要消息中间件来进行发送和接收消息。 RabbitMQ 和 Redis 中间人的功能比较齐全，但也支持其它的实验性的解决方案，其中包括 SQLite 进行本地开发。

Celery 可以在一台机器上运行，也可以在多台机器上运行，甚至可以跨数据中心运行。

首先我们创建一个简单的task。

```python
from celery import Celery

app = Celery('tasks', broker='pyamqp://guest@localhost//')

@app.task
def add(x, y):
    return x + y
```

对于中间人，如果是Rabbitmq的话，可以使用 `amqp://user:123456@localhost/test`，Redis的话可以用 `redis://localhost` ，然后启动下任务队列:

```bash
celery -A tasks worker --loglevel=info
```

```bash
[root@aliyun amqp]# celery -A tasks worker --loglevel=info
/usr/local/lib/python3.6/site-packages/celery/platforms.py:801: RuntimeWarning: You're running the worker with superuser privileges: this is
absolutely not recommended!

Please specify a different user using the --uid option.

User information: uid=0 euid=0 gid=0 egid=0

  uid=uid, euid=euid, gid=gid, egid=egid,
 
 -------------- celery@aliyun v4.4.0 (cliffs)
--- ***** ----- 
-- ******* ---- Linux-4.18.0-147.5.1.el8_1.x86_64-x86_64-with-centos-8.1.1911-Core 2019-10-19 01:14:19
- *** --- * --- 
- ** ---------- [config]
- ** ---------- .> app:         tasks:0x7f12d36d5f28
- ** ---------- .> transport:   amqp://user:**@localhost:5672/test
- ** ---------- .> results:     redis://localhost/
- *** --- * --- .> concurrency: 2 (prefork)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** ----- 
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery
                

[tasks]
  . tasks.add
  . tasks.mul
  . tasks.xsum

[2019-10-19 01:14:20,027: INFO/MainProcess] Connected to amqp://user:**@127.0.0.1:5672/test
[2019-10-19 01:14:20,038: INFO/MainProcess] mingle: searching for neighbors
[2019-10-19 01:14:21,066: INFO/MainProcess] mingle: all alone
[2019-10-19 01:14:21,085: INFO/MainProcess] celery@aliyun ready.
```

可以看到 Celery 已经启动，通过 python shell 可以调用对应的任务，Celery默认的并发数为 CPU 核心数。

调用下试试。

```
In [5]: add.delay(4, 4)                                                                                
Out[5]: <AsyncResult: 8b3818f8-ebd0-443f-bbf8-5dadb536fcaa>

In [6]: result = add.delay(1,1)                                                                        

In [7]: result.ready()                                                                                 
Out[7]: True

In [8]: result.get(timeout=1)                                                                          
Out[8]: 2

In [9]: result.get(propagate=False)                                                                    
Out[9]: 2

In [10]: result.traceback 
```

```bash
Celery可以通过 rate_limit 方向进行频率限制，例如，以下方式限制每分钟不超过10次

celery -A tasks control rate_limit tasks.add 10/m
```

Celery 通过 EventLet 方式实现并发，使用 epoll 或 libevent 实现非阻塞 IO ,这样可以充分提高执行的任务数.

通过  [`celery worker -P`](https://docs.celeryproject.org/en/stable/reference/celery.bin.worker.html#cmdoption-celery-worker-p) 的方式可以指定工作方式

```bash
celery -A proj worker -P eventlet -c 1000 // 1000个线程并发执行
```

Celery 可以 通过配置backend 来保存任务执行结果，我这里使用 Redis 。

Celerys.py

```python
#!/usr/bin/env python
# coding=utf-8
from __future__ import absolute_import, unicode_literals
from celery import Celery
app = Celery(
    broker = 'amqp://user:123456@localhost/test',
    backend='redis://localhost',
    include=[]
)

app.conf.update(
    result_expires=3600,
)

if __name__ == '__main__':
    app.start()
```

tasks.py

```python
#!/usr/bin/env python
# coding=utf-8
from celery import Celery

app = Celery('tasks', broker='amqp://user:123456@localhost/test', backend='redis://localhost')

app.conf.task_serializer = 'json'

@app.task
def add(x, y):
        return x + y

@app.task
def mul(x,y):
    return x*y

@app.task
def xsum(numbers):
    return sum(numbers)




if __name__ == '__main__':
    add(1,2)

```

 查看下 Redis 中是否有结果

![](1.png)

可以查看到容器内的redis中已经有了 task的执行结果，我们可以查看下结果

```bash
127.0.0.1:6379> get celery-task-meta-8b3818f8-ebd0-443f-bbf8-5dadb536fcaa
"{\"status\": \"SUCCESS\", \"result\": 8, \"traceback\": null, \"children\": [], \"date_done\": \"2020-03-16T17:41:33.382702\", \"task_id\": \"8b3818f8-ebd0-443f-bbf8-5dadb536fcaa\"}"
127.0.0.1:6379>
```

可以看到配置 Redis 为backend后，task的执行结果已经保存到 Redis中，以后开发中，可以通过读取 对应的task_id为 key的 数据，就可以得到每个任务的最终执行结果。