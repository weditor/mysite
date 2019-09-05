---
title: Python多进程(二)
date: 2019-08-31 00:35:23
tags:
categories:
---


# Python 多进程的实现

主要通过 `Process` 单进程，展开到 `Pool` 进程池的实现，以及序列化带来的一些坑。

## Process 父子进程通信

一个父子进程通过 Pipe 通信的例子

<!-- more -->

```python
from multiprocessing import Process, Pipe


def calculator(pipe):
    while True:
        try:
            a, b = pipe.recv()
        except EOFError:
            break
        ret = a+b
        pipe.send(ret)


# duplex=True, 申请双向管道
father_pipe, child_pipe = Pipe(duplex=True)
p = Process(target=calculator, args=(child_pipe, ))
p.start()
father_pipe.send((1, 2))
father_pipe.send((1, 3))
father_pipe.send((1, 4))

print(father_pipe.recv())
print(father_pipe.recv())
print(father_pipe.recv())
father_pipe.close()
print('closed')
p.join()
```
输出
```
3
4
5
closed
```

两个进程通过 Pipe 的两端互相发送消息。 

根据官方文档，当pipe中已经没有数据并且对方已经关闭时，`Pipe.recv` 方法会抛出 `EOFError` 异常，while循环就能退出，完美!

### 父子进程内存复制

上面程序第一个坑就是程序并不会如愿退出的问题。
虽然 `father_pipe` 明明 close 了。

通过 `Process` 或者 `fork` 从当前进程分离出一个子进程，子进程会从父进程那里复制一份相同的内存内容（linux 是延迟复制，但是这些细节没有关系）。（windows好像不行，需要确认下。）
所以当Process执行那一刻，父进程所有已经产生的对象都可以被子进程使用，这次复制**几乎可以复制任何对象**（某些特殊的对象复制后会出现bug，如数据库连接），复制之后，父子进程的相同对象互相独立。

问题就出在这里，执行 Process 前， `father_pipe` 就已经产生，也就被复制了一份到子进程，只有当 father_pipe 不被任何进程使用时，才算真正关闭。

所以应当在子进程里面先关闭`father_pipe`, 程序修改如下：

```python
def calculator(pipe, pipe2):
    pipe2.close()
    while True:
        ... ...
p = Process(target=calculator, args=(child_pipe, father_pipe))
... ...
```

这样，进程就会如愿退出了。

不过如果不想这样显式传递，可以在确认子进程处理完后 `p.terminate` ,其所持有的所有文件句柄都会被回收，不用显式关闭。


### 父子进程通信


过了上面那一阶段后，如果父子进程还想通信，一般都使用 Queue/Pipe。

修改一下，接收两个可迭代的对象，把他们转成数组拼接成一个新数组。

```python
def calculator(pipe):
    while True:
        iter1, iter2 = pipe.recv()
        ret = list(iter1)+list(iter2)
        pipe.send(ret)

father_pipe.send(([1, 2], {3, 4}))
print(father_pipe.recv())
father_pipe.send((range(3), range(4)))
print(father_pipe.recv())

def myrange():
    for n in range(3):
        yield n

father_pipe.send((myrange(), myrange()))
print(father_pipe.recv())
```

输出
```
[1, 2, 3, 4]
[0, 1, 2, 0, 1, 2, 3]
Traceback (most recent call last):
  File "/home/weditor/work/201909/test_charm/share.py", line 26, in <module>
    father_pipe.send((myrange(), myrange()))
  File "/usr/local/anaconda37/lib/python3.7/multiprocessing/connection.py", line 206, in send
    self._send_bytes(_ForkingPickler.dumps(obj))
  File "/usr/local/anaconda37/lib/python3.7/multiprocessing/reduction.py", line 51, in dumps
    cls(buf, protocol).dump(obj)
TypeError: can't pickle generator objects
```

前两个正常，第三个就报错了。上面显示 yield generator 不能使用 pickle 进行序列化。

Pipe/Queue作为文件类型句柄类型的传输介质，并没有什么特殊限制，一般都用来传递二进制流。问题在于 Python 对象序列化为二进制数据这一步上，用的是pickle，而很多对象无法被 pickle 序列化，如 lambda/generator等，以及包含这些对象的类。

如果多进程环境下，需要传递这类无法pickle的对象，最好在产生进程前生成，借助第一个步骤**内存复制**传递过去。

大部分情况下， pickle error 对程序利用multiprocessing都是致命的，所以要把握好适当的时候利用**内存复制**共享变量。
不过大部分Python基础对象都是可以 pickle 的。

### 一个例子

内存复制传递lambda/generator的例子

```python
def calculator(out_pipe, func, datas):
    for n in datas:
        out_pipe.send(func(n))


def myrange():
    for n in range(3):
        yield n


# duplex=True, 申请双向管道
father_pipe, child_pipe = Pipe(duplex=True)

# 直接将 lambda/generator 传过去。
p = Process(target=calculator, args=(child_pipe, lambda x: x*2, myrange()))
p.start()
print(father_pipe.recv())
print(father_pipe.recv())
print(father_pipe.recv())
```

输出
```
0
2
4
```


## Pool

Python中 Pool的使用很灵活。

对于Process，在创建的那一刻，它所执行的函数以及函数参数就已经确定，但是Pool在创建时只是创建了一个进程池，这个进程池要执行什么函数，执行时需要什么参数，都是不确定的，所以Pool在使用时更加灵活，甚至在执行过程中可以在不同的函数之间来回切换。

### Pool的任务

见 `multiprocessing.pool.worker`
```python
def worker(inqueue, outqueue, initializer=None, initargs=(), maxtasks=None,
           wrap_exception=False):
    # 下面是伪代码
    job, i, func, args, kwds = inqueue.get()
    outqueue.put((job, i, func(*args, **kwds)))
```
为了实现上述的灵活性， Pool 在创建的时候，虽然没有制定执行的目标函数，但是内部有隐式指定worker为目标函数。
可以看出，worker通过 `inqueue`/`outqueue` 与外界通信，外界将需要执行的 `func` 以及参数 `args`/`kwds` 通过 queue传进来，
将返回结果传出去。

所以 Pool 执行的所有东西实际上都是外界动态传进来的，这种传递依赖于 pickle。

### Pool.imap

1. imap 会顺序处理序列中的数据并按顺序返回。
2. imap是多线程安全的。也就是说下面这种在Pool中同时执行两个函数是合法的:

    ```python
    pool = Pool(4)
        for ret1, ret2 in zip(pool.imap(func1, iter1), pool.imap(func2, iter2)):
        ...
    ```
为了保证结果有序，序列中每个元素都会有一个编号: `i`。
为了实现多线程安全，每次调用 `imap`, 会给本次调用分配一个任务编号: `job` 。两个编号可以组成一个唯一标识 `(job, i)` 。

虽然所有task都在 inqueue/outqueue 中传递，为了保证job之间的返回结果不混淆，每个job的返回结果有一个单独的缓冲区存放在字典 `Pool._cache` 中。
为了保证返回结果有序，使用 IMapIterator 维护返回结果的序列。

几个队列:
* _inqueue: 数据入口，压入 task 任务
* _outqueue： 结果出口，返回结果。
* _taskqueue： 每次调用一次生成的临时任务队列。

执行流程:
```text
imap()[job1]{1,2,3...} -> _taskqueue --                         -- cache[job1] -- IMapIterator排序
                                        \                      /
                                         _inqueue -> _outqueue
                                        /                      \
imap()[job2]{1,2,3...} -> _taskqueue --                         -- cache[job2] -- IMapIterator排序
```

后台线程:
* `_worker_handler` : 维护多进程。当有进程退出后，负责维护启动新的进程。
* `_task_handler` : 将 _taskqueue 任务传递到 inqueue.

## 坑

### 序列化
虽然 Pool 设计的很灵活，但是任务在传递的时候使用了 pickle ，
有很多东西都无法使用pickle序列化，现实生活中这个条件还是很苛刻的，这极大的影响了 Pool 的用途。

另外，每次处理序列中的一个元素，需要将函数以及绑定在函数上的依赖都序列化一遍，一起送到 `worker` 执行，效率比较低下。

### 函数状态

这个问题还是由于序列化导致的，由于函数每次都需要序列化，这要求我们的函数必须是无状态的。

比如下面的例子。

```python
from multiprocessing import Pool


class MyCls:
    def __init__(self):
        # 计数当前api调用了多少次
        self.count = 0

    def add(self, x):
        self.count += 1
        print('count add to', self.count)
        return sum(x)


my_obj = MyCls()
pool = Pool(1)
for n in pool.imap(my_obj.add, [(1, 1), (2, 2), (3, 3)]):
    print('return: ', n)

print('total call times:', my_obj.count)
```

会惊讶地发现。
1. 最后打印的 `self.count` 是0, 根本就没有增加，因为每次都只是改变了子进程中的值，主进程不受影响。
2. 虽说子进程中 `self.count` 增加了，但是值始终都是 1。因为子进程中每次得到的函数都是全新的。

输出:
```
count add to 1
count add to 1
count add to 1
return:  2
return:  4
return:  6
total call times: 0
```
