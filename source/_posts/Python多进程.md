---
title: Python多进程(一)
tags: 
    - python
categories: 
    - python
date: 2019-08-30 22:01:27
---

# Python 多进程

Python 多进程常用方法:

* os.fork: 直接调用系统fork接口，Linux下可用，无法在windows中使用。
* subprocess: 启动其他进程，并与之交互。
* multiprocessing: 类似于subprocess，启动一个进程，但是执行的目标是一个当前上下文环境下的函数。

<!-- more -->

os.fork 和 subprocess 相对来说比较简单，这里主要说一下 multiprocessing 。

## multiprocessing 介绍

multiprocessing 的模块：

* Process： 在另外一个进程中启动函数。
* Pool: 在一个进程池中启动函数。提供 apply/imap 等高级封装接口。

## Process

Process 的使用方法也比较简单, 将一个函数及其参数传入 `Process`，
调用start即可启动进程。

```python
import time
from multiprocessing import Process


def do_something(text):
    time.sleep(2)
    print('I like', text)


process = Process(target=do_something, args=("fish",))
process.start()

print('wait process to finish.')
process.join()
```

输出:
```plain
wait process to finish.
I like fish
```

使用 `Process` 执行函数可以开启一个额外的进程执行指定函数，以此绕开GIL限制，
通常用来给CPU密集型任务加速。

这个例子比较局限，子进程并没有返回值，启动后就没有和父进程交互。

## Process 的父子进程通信

multiprocessing 模块提供了 Queue/Pipe 用于进程间通信，Lock锁用于父子进程以及子进程之间的同步。如果需要其他变量共享，可以使用 Manager。

### Queue

Queue是FIFO队列。拥有Queue的任何一方都可以自由地调用 `get/put` 方法获取/传送数据。

```python
from multiprocessing import Process, Queue


def double_number(n: int, out_q):
    out_q.put(n*2)


queue = Queue()
process = Process(target=double_number, args=(3, queue,))
process.start()
process.join()
print(queue.get())
```

输出: `6`

另外还有

* SimpleQueue: 简化版本的 Queue， 只有 get/put/empty
* JoinableQueue: 复杂版的Queue，支持更细粒度的任务控制。每次处理一个数据，需要显示调用 `task_done` 。 

### Pipe

Pipe 是管道，默认是双向管道。每次构建一个Pipe会得到两个对象, 即Pipe的两个输入输出端。

双向Pipe和Queue不同，管道的任意一端调用send后，另一方都可以立即调用recv获得值，即存在两个独立的管道：发送管道，接收管道。
而Queue在任意一方put后，如果队列中已经有其他人放了数据，必须先get完已经存在的数据。

```python
from multiprocessing import Process, Pipe


def double_number(n: int, out):
    out.send(n*2)
    print(out.recv())


father, child = Pipe()
process = Process(target=double_number, args=(3, child,))
process.start()

father.send(-1)
process.join()
print(father.recv())
```
输出:
```text
-1
6
```

### Lock
Lock， 是一种进程安全的同步锁，和一般的互斥锁用法一样。

### Manager

Manager是多进程环境下对共享数据的高级抽象。
除了内存方式实现多进程外，还可以通过网络实现多机器之间的对象共享。

（待补充）

## Pool

Pool 是在 Process 之上实现的进程池。是对多进程的高级抽象，隐藏了多进程通信细节。

`Pool.map` 允许传入一个函数和一个序列，函数会依次执行序列中每个对象并按顺序返回结果。

```python
from multiprocessing import Pool


def double_number(n: int):
    return n*2


# 最大4进程的进程池
pool = Pool(4)
print(pool.map(double_number, range(5)))
```

输出: ```[0, 2, 4, 6, 8]```

注意： Pool 在形式上用起来很简单，实际上进程间通信使用的 pickle 坑的一批，动不动就 PickingError 。

## 其他

另外还有一个与`multiprocessing.Pool` 接口相似的模块 `concurrent.futures`, 
其中提供了对于线程和进程两种并发方式的统一接口封装：`ProcessPoolExecutor`/`ThreadPoolExecutor` 。
接口更加简单和易用, 主要使用了 `Future` 的实现方式， 额外提供了timeout控制，不过Future会耗费一些性能。

## 数据共享

Todo.
* Context.
* Manager
* Value

