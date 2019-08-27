---
layout: post
title: Python中的协程、线程和进程
category: 编程
description: 通过简单的生产-消费的Python实现来体会协程、线程及进程的不同
tags: ["Hadoop"]
---

通过一个简单的生产-消费模型，来体会一下Python中协程、线程及进程的使用。

首先是协程。分为Python2.5中yield/send实现的生成器和Python3.5中async/await进一步完善的协程。看代码和输出:

```py
class YieldTest(object):
    @staticmethod
    def consumer(name):
        n = ''
        while True:
            n = yield n
            print(f"消费者{name}: 处理消息--{n}"
            sleep(1)
            n = f"消费者{name}处理完毕"

    @staticmethod
    def producer(cs, n):
        for i in range(n):
            print(f"生产者: 产生消息--{i}")
            r = cs[i % len(cs)].send(i)
            print(f"生产者: 收到回复--{r}")

    @classmethod
    def run(cls):
        start = time()
        cs = [cls.consumer(c) for c in range(2)]
        for c in cs: c.send(None)
        cls.producer(cs, 6)
        for c in cs: c.close()
        end = time()
        print(f"YieldTest共耗时:{end-start:.2f}s")
```

```text
生产者: 产生消息--0
消费者0: 处理消息--0
生产者: 收到回复--消费者0处理完毕
生产者: 产生消息--1
消费者1: 处理消息--1
生产者: 收到回复--消费者1处理完毕
生产者: 产生消息--2
消费者0: 处理消息--2
生产者: 收到回复--消费者0处理完毕
生产者: 产生消息--3
消费者1: 处理消息--3
生产者: 收到回复--消费者1处理完毕
生产者: 产生消息--4
消费者0: 处理消息--4
生产者: 收到回复--消费者0处理完毕
生产者: 产生消息--5
消费者1: 处理消息--5
生产者: 收到回复--消费者1处理完毕
YieldTest共耗时:6.02s
```

所以说yield/send是个伪协程，它只是提供了在yield处保存现场并切换控制权，仍然需要其他函数调用send/next来触发，另外它并没有提供并发的能力，主线程需要等待生成器调用，包括IO等待，从结果上看耗时没有任何降低。生成器只能降低内存开销，无法提供并发能力，上面的生产-消费代码也是强耦合。如果想要通过yield/send提高并发，需要自己实现事件驱动的异步IO，比较麻烦。因此Python后续进一步完善，提供了asyncio和其他异步库。async/await代码及结果如下:

```py
class AsyncTest(object):

    @staticmethod
    async def consumer(name, queue):
        while True:
            n = await queue.get()
            print(f"消费者{name}: 处理消息--{n}")
            await asyncio.sleep(1)
            queue.task_done()

    @staticmethod
    async def producer(queue, n):
        for i in range(n):
            print(f"生产者: 产生消息--{i}")
            await queue.put(i)

    @staticmethod
    async def main(n):
        q = asyncio.Queue()
        cs = [asyncio.ensure_future(AsyncTest.consumer(c, q)) for c in range(2)]
        await AsyncTest.producer(q, n)
        await q.join()
        for c in cs: c.cancel()

    @classmethod
    def run(cls):
        start = time()
        loop = asyncio.get_event_loop()
        loop.run_until_complete(cls.main(6))
        loop.close()
        end = time()
        print(f"AsyncTest共耗时:{end-start:.2f}s")
```

```text
生产者: 产生消息--0
生产者: 产生消息--1
生产者: 产生消息--2
生产者: 产生消息--3
生产者: 产生消息--4
生产者: 产生消息--5
消费者0: 处理消息--0
消费者1: 处理消息--1
消费者0: 处理消息--2
消费者1: 处理消息--3
消费者0: 处理消息--4
消费者1: 处理消息--5
AsyncTest共耗时:3.01s
```

在这里通过队列解耦了生产者和消费者，通过异步IO提供并发，两个消费者使得处理时间减半。下面再看看多线程:

```py
class ThreadTest(object):

    @staticmethod
    def consumer(name, queue):
        while True:
            n = queue.get()
            if n is None:
                break
            print(f"消费者{name}: 处理消息--{n}")
            sleep(1)

    @staticmethod
    def producer(queue, n, size):
        for i in range(n):
            print(f"生产者: 产生消息--{i}")
            queue.put(i)
        # 发送None结束consumer线程
        for i in range(size):
            queue.put(None)

    @classmethod
    def run(cls):
        start = time()
        q = queue.Queue()
        cs = [threading.Thread(target=ThreadTest.consumer, args=(c, q)) for c in range(2)]
        for c in cs: c.start()
        p = threading.Thread(target=ThreadTest.producer, args=(q, 6, len(cs)))
        p.start()
        for c in cs: c.join()
        end = time()
        print(f"ThreadTest共耗时:{end-start:.2f}s")
```

```text
生产者: 产生消息--0
生产者: 产生消息--1
消费者0: 处理消息--0
生产者: 产生消息--2
生产者: 产生消息--3
生产者: 产生消息--4
生产者: 产生消息--5
消费者1: 处理消息--1
消费者0: 处理消息--2
消费者1: 处理消息--3
消费者0: 处理消息--4
消费者1: 处理消息--5
ThreadTest共耗时:3.01s
```

由于GIL的存在，Python多线程也是阉割版的，跑IO密集程序还可以，跑CPU密集就是浪费切换时间了。在此处也和协程一样，有效降低了处理时间。

最后是多进程，结构和多线程类似，最终处理时间增加也说明了新建进程开销还是很大滴:

```py
class ProcessTest(object):

    @classmethod
    def run(cls):
        start = time()
        q = multiprocessing.Queue()
        cs = [multiprocessing.Process(target=ThreadTest.consumer, args=(c, q)) for c in range(2)]
        for c in cs: c.start()
        p = multiprocessing.Process(target=ThreadTest.producer, args=(q, 6, len(cs)))
        p.start()
        for c in cs: c.join()
        end = time()
        print(f"ProcessTest共耗时:{end-start:.2f}s")
```

```text
生产者: 产生消息--0
生产者: 产生消息--1
生产者: 产生消息--2
生产者: 产生消息--3
生产者: 产生消息--4
消费者0: 处理消息--0
消费者1: 处理消息--1
生产者: 产生消息--5
消费者1: 处理消息--2
消费者0: 处理消息--3
消费者0: 处理消息--4
消费者1: 处理消息--5
ProcessTest共耗时:3.04s
```
