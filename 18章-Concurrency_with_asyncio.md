# Chapter 18 Concurrency with asyncio
# 第18章 使用asyncio处理并发

*Concurrency is about dealing with lots of things at once.  
Parallelism is about doing lots of things at once.  
Not the same, but related.  
One is about structure, one is about execution.  
Concurrency provides a way to structure a solution to solve a problem that may (but not necessarily) be parallelizable.[1]*  
*并发是指同时处理许多件事，*  
*并行是指同时做许多件事，*  
*二者不同，但有所关联。*  
*前者关乎结构，后者关乎执行。*  
*并发提供了一种构建解决方案的方式，用于解决可能（但不一定）的并行问题。*

*<p align="right">— Rob Pike*
*<p align="right">Co-inventor of the Go language*

Professor Imre Simon[2] liked to say there are two major sins in science: using different words to mean the same thing and using one word to mean different things. If you do any research on concurrent or parallel programming you will find different definitions for “concurrency” and “parallelism.” I will adopt the informal definitions by Rob Pike, quoted above.  
    Imer Simon教授喜欢说科学中有两大罪：用不同的词表示相同的事物，用一个词表示不同事物。如果你对并发或并行编程做过任何研究，你都会对“并发”与“并行”的不同定义。我将采用上面引用的Rob Pike的非正式定义。

For real parallelism, you must have multiple cores. A modern laptop has four CPU cores but is routinely running more than 100 processes at any given time under normal, casual use. So, in practice, most processing happens concurrently and not in parallel. The computer is constantly dealing with 100+ processes, making sure each has an opportunity to make progress, even if the CPU itself can’t do more than four things at once. Ten years ago we used machines that were also able to handle 100 processes concurrently, but on a single core. That’s why Rob Pike titled that talk “Concurrency Is Not Parallelism (It’s Better).”  
    对于真正的并行来说，你必须拥有多核。当代的笔记本有4核CPU，但日常使用中的一个·普通时间点上都通常会运行着超过100个进程。所以实际上，大多数进程都是并发而不是在并行。计算机不断地处理100+个进程，以确保有机会来使其都取得进展，即使他的CPU不能同时做超过四件事。10年前我们用的机器同样支持同时处理100个进程，即使他只有单核。这就是Rob Pike为什么将演讲命名为“并发不是并行（他更好）”的原因。

[1] Slide 5 of the talk “Concurrency Is Not Parallelism (It’s Better)”.  
    演讲“Concurrency Is Not Parallelism (It’s Better)”的第五页幻灯片。
[2] Imre Simon (1943-2009) was a pioneer of computer science in Brazil who made seminal contributions to Automata Theory and started the field of Tropical Mathematics. He was also an advocate of free software and free culture. I was fortunate to study, work, and hang out with him.  
    Imre Simon(1942-2009)是一个巴西的计算机科学先驱，他对自动机理论做了巨大贡献并创建了Tropical Mathematics领域。他还是一个自由软件与自由文化的拥护者。我很幸运能与他一起学习，工作，玩耍。

This chapter introduces asyncio, a package that implements concurrency with coroutines driven by an event loop. It’s one of the largest and most ambitious libraries ever added to Python. Guido van Rossum developed asyncio outside of the Python repository and gave the project a code name of “Tulip”—so you’ll see references to that flower when researching this topic online. For example, the main discussion group is still called python-tulip.  
    本章介绍asyncio，这是一个依靠由事件循环驱动的协程实现并发的包。它是Python中添加的最大、最有野心的库之一。Guido van Rossum在Python存储库之外开发了asyncio，并给这个项目起了一个代号“郁金香”——所以当你在网上搜索这个主题时，你将会看到关于这种花的文献。举个例子，最主要的讨论组仍然被称作python-tulip。

Tulip was renamed to asyncio when it was added to the standard library in Python 3.4. It’s also compatible with Python 3.3—you can find it on PyPI under the new official name. Because it uses yield from expressions extensively, asyncio is incompatible with older versions of Python.
    当Tulip在Python3.4被追加至标准库时，他被重命名为asyncio。这在Python3.3中也兼容——你可以在PyPI上找到他的新官方名称。因为该库广泛使用了yield from语法，asyncio不兼容更旧版本的Python。

    The Trollius project—also named after a flower—is a backport of asyncio to Python 2.6 and newer, replacing yield from with yield and clever callables named From and Return. A yield from … expression becomes yield From(…); and when a coroutine needs to return a result, you write raise Return(result) instead of return result. Trollius is led by Victor Stinner, who is also an asyncio core developer, and who kindly agreed to review this chapter as this book was going into production.  
    同样以一种花命名的Trollius项目，是asyncio对Python2.6及其新版的向后移植，使用yield和被称为From和Return的聪明的调用对象。yield from … 表达式变为 yield From(…)；当协程需要返回一个结果时，你可以用raise Return(result)来代替return result。Trollius由Victor Stinner领衔开发，他也是asyncio的核心开发人员，在本书即将推行阶段他很友好地同意对本章的审核。


In this chapter we’ll see:  
    在本章，我们将看到：

- A comparison between a simple threaded program and the asyncio equivalent, showing the relationship between threads and asynchronous tasks  
    一个简单线程程序与对应asyncio程序的对比，展示线程任务与异步任务直接的关系
- How the asyncio.Future class differs from concurrent.futures.Future  
    asyncio.Future类与concurrent.futures.Future的区别
- Asynchronous versions of the flag download examples from Chapter 17  
    17章下载flag示例的异步版本
- How asynchronous programming manages high concurrency in network applications, without using threads or processes  
    网络应用中，不使用线程或进程的异步程序对高并发的管理
- How coroutines are a major improvement over callbacks for asynchronous programming  
    对异步编程来说，协程是如何对回调进行重大改进的
- How to avoid blocking the event loop by offloading blocking operations to a thread pool  
    如何通过将阻塞操作卸给线程池，避免对事件循环的阻塞
- Writing asyncio servers, and how to rethink web applications for high concurrency  
    编写asyncio服务，及如何重新思考web应用的高并发
- Why asyncio is poised to have a big impact in the Python ecosystem  
    为什么asyncio会对Python生态系统有着重大影响

Let’s get started with the simple example contrasting threading and asyncio.  
    让我们从一个对比threading与asyncio的简单例子开始。

## Thread Versus Coroutine: A Comparison
## 线程 vs 协程

During a discussion about threads and the GIL, Michele Simionato posted a simple but fun example using multiprocessing to display an animated spinner made with the ASCII characters "|/-\" on the console while some long computation is running.  
    在讨论线程与GIL的期间，Michele Simionato贴出了一份简单但有趣的例子，使用了multiprocessing在控制台展示一个由ASCII字符"|/-\"之制作的动画转轮，同时运行一些长时间的运算。

I adapted Simionato’s example to use a thread with the Threading module and then a coroutine with asyncio, so you can see the two examples side by side and understand how to code concurrent behavior without threads.  
    我调整了Simionato的例子，改用Threading模块调用线程，以及后续用asyncio调用协程，所以你会一起看到两个例子并理解不使用线程的并发行为是如何编写的。

The output shown in Examples 18-1 and 18-2 is animated, so you really should run the scripts to see what happens. If you’re in the subway (or somewhere else without a WiFi connection), take a look at Figure 18-1 and imagine the \ bar before the word “thinking” is spinning. 
    例18-1与18-2的输出是动态的，所以你真的应该亲自运行下脚本看看会发生什么。如果你在地铁上（或是一些没有WiFi的地方），可以看看图示18-1，想象一下"thinking"前的"\"正在旋转。

Figure 18-1. The scripts spinner_thread.py and spinner_asyncio.py produce similar output: the repr of a spinner object and the text Answer: 42. In the screenshot, spinner_asyncio.py is still running, and the spinner message \ thinking! is shown; when the script ends, that line will be replaced by the Answer: 42.
    图18-1. spinner_thread.py脚本与spinner_asyncio.py产出类似的输出：spinner对象的repr与text Answer: 42。在截图中，spinner_asyncio.py仍在运行，展示旋转信息"\ thinking!"；当脚本结束后，该行被替换为"Answer: 42"。

Let’s review the spinner_thread.py script first (Example 18-1).  
    首先研究下脚本spinner_thread.py

Example 18-1. spinner_thread.py: animating a text spinner with a thread  
    例18-1. spinner_thread.py：使用线程，让一段文本旋转器运作起来
```python
import threading
import itertools
import time
import sys


class Signal:  # 1
    go = True


def spin(msg, signal):  # 2
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):  # 3
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))  # 4
        time.sleep(.1)
        if not signal.go:  # 5
            break
    write(' ' * len(status) + '\x08' * len(status))  # 6


def slow_function():  # 7
    # pretend waiting a long time for I/O
    time.sleep(3)  # 8
    return 42


def supervisor():  # 9
    signal = Signal()
    spinner = threading.Thread(target=spin,
                               args=('thinking!', signal))
    print('spinner object:', spinner)  # 10
    spinner.start()  # 11
    result = slow_function()  # 12
    signal.go = False  # 13
    spinner.join()  # 14
    return result


def main():
    result = supervisor()  # 15
    print('Answer:', result)


if __name__ == '__main__':
    main()
```

1. This class defines a simple mutable object with a go attribute we’ll use to control the thread from outside.  
    此类定义了一个带有go属性的简单的可变对象，我们可以用来从外部控制线程。
2. This function will run in a separate thread. The signal argument is an instance of the Signal class just defined.  
    该函数将会在一个独立线程中运行。信号参数是刚刚定义的Signal类的实例。
3. This is actually an infinite loop because itertools.cycle produces items cycling from the given sequence forever.  
    这实际是个无限循环，因为itertools.cycle将一直循环传入的序列并产出项。
4. The trick to do text-mode animation: move the cursor back with backspace characters (\x08).  
    文本动画的技巧：用退格字符（\x08）将光标向后移动。
5. If the go attribute is no longer True, exit the loop.  
    如果go属性不再是True，终止该循环。
6. Clear the status line by overwriting with spaces and moving the cursor back to the beginning.  
    通过用空格覆盖并将光标移回开始位置来清理status行。
7. Imagine this is some costly computation.  
    想象这里是一些费时的计算操作。
8. Calling sleep will block the main thread, but crucially, the GIL will be released so the secondary thread will proceed.  
    调用sleep将阻塞主线程，但另外更关键的是，GIL将被释放，第二条线程将启动。
9. This function sets up the secondary thread, displays the thread object, runs the slow computation, and kills the thread.  
    该函数设置了第二条线程，显示出线程对象，运行缓慢的计算操作并kill掉线程。
10. Display the secondary thread object. The output looks like <Thread(Thread-1, initial)>.  
    显示第二条线程的对象。输出内容诸如<Thread(Thread-1, initial)>。
11. Start the secondary thread.  
    启动第二条线程。
12. Run slow_function; this blocks the main thread. Meanwhile, the spinner is animated by the secondary thread.  
    运行slow_function；这将阻塞主线程。同时旋转器通过第二条线程开始运作。
13. Change the state of the signal; this will terminate the for loop inside the spin function.  
    修改信号状态；这将终止spin函数内部的for循环。
14. Wait until the spinner thread finishes.  
    等待spinner线程结束。
15. Run the supervisor function.  
    运行supervisor函数。

Note that, by design, there is no API for terminating a thread in Python. You must send it a message to shut down. Here I used the signal.go attribute: when the main thread sets it to false, the spinner thread will eventually notice and exit cleanly.  
    注意，通过设计，在Python中没有终止线程的API。你必须向其发送一条信息来关闭。这里我用了signal.go属性：当主线程将其设置为false，spinner线程最终会注意到并干净地退出。

Now let’s see how the same behavior can be achieved with an @asyncio.coroutine instead of a thread.
    现在我们看看用@asyncio.coroutine代替线程如何实现相同的效果。

    As noted in the “Chapter Summary” on page 498 (Chapter 16), asyncio uses a stricter definition of “coroutine.” A coroutine suitable for use with the asyncio API must use yield from and not yield in its body. Also, an asyncio coroutine should be driven by a caller invoking it through yield from or by passing the coroutine to one of the asyncio functions such as asyncio.async(…) and others covered in this chapter. Finally, the @asyncio.coroutine decorator should be applied to coroutines, as shown in the examples.  
    正如498页（16章）章节总结提到的，asyncio使用了“协程”的更严格定义。适合与asyncio API一起使用的协程必须使用yield from且不能在他的结构中产出。此外，一个asyncio协程应该由调用者来驱动，通过经由yield from调用或通过将协程传给其中一个asyncio函数（像是asyncio.async(…)）和本章介绍的其他函数。最终，@asyncio.coroutine装饰器应该应用于协程，如示例所示。

Take a look at Example 18-2.
Example 18-2. spinner_asyncio.py: animating a text spinner with a coroutine

```python
import asyncio
import itertools
import sys


@asyncio.coroutine  # 1
def spin(msg):  # 2
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        try:
            yield from asyncio.sleep(.1)  # 3
        except asyncio.CancelledError:  # 4
            break
    write(' ' * len(status) + '\x08' * len(status))


@asyncio.coroutine
def slow_function():  # 5
    # pretend waiting a long time for I/O
    yield from asyncio.sleep(3)  # 6
    return 42


@asyncio.coroutine
def supervisor():  # 7
    spinner = asyncio.async(spin('thinking!'))  # 8
    print('spinner object:', spinner)  # 9
    result = yield from slow_function()  # 10
    spinner.cancel()  # 11
    return result


def main():
    loop = asyncio.get_event_loop()  # 12
    result = loop.run_until_complete(supervisor())  # 13
    loop.close()
    print('Answer:', result)


if __name__ == '__main__':
    main()

```

1. Coroutines intended for use with asyncio should be decorated with @asyncio.coroutine. This not mandatory, but is highly advisable. See explanation following this listing.  
    用于asyncio的协程应该被@asyncio.coroutine装饰。这不是强制的，但非常可取。请参考下面的解释说明。
2. Here we don’t need the signal argument that was used to shut down the thread in the spin function of Example 18-1.  
    这里我们不再需要信号参数，该参数在示例18-1中国被用于关闭spin函数的线程。
3. Use yield from asyncio.sleep(.1) instead of just time.sleep(.1), to sleep without blocking the event loop.  
    使用yield from asyncio.sleep(.1)来代替time.sleep(.1)，来sleep而不阻塞事件循环。
4. If asyncio.CancelledError is raised after spin wakes up, it’s because cancellation was requested, so exit the loop.  
    如果唤醒spin后抛出了asyncio.CancelledError，这是因为请求了cancellation，所以退出循环。
5. slow_function is now a coroutine, and uses yield from to let the event loop proceed while this coroutine pretends to do I/O by sleeping.  
    slow_function目前是协程，
6. The yield from asyncio.sleep(3) expression handles the control flow to the main loop, which will resume this coroutine after the sleep delay.
7. supervisor is now a coroutine as well,so it can drive slow_function with yield from.
8. asyncio.async(…) schedules the spin coroutine to run, wrapping it in a Task object, which is returned immediately.
9. Display the Task object. The output looks like <Task pending coro=<spin() running at spinner_asyncio.py:12>>.
10. Drive the slow_function(). When that is done, get the returned value. Meanwhile, the event loop will continue running because slow_function ultimately uses yield from asyncio.sleep(3) to hand control back to the main loop.
11. ATask object can be cancelled; thisraises asyncio.CancelledError at the yield line where the coroutine is currently suspended. The coroutine may catch the exception and delay or even refuse to cancel.
12. Get a reference to the event loop.
13. Drive the supervisor coroutine to completion; the return value of the coroutine is the return value of this call.  


    Never use time.sleep(…) in asyncio coroutines unless you want to block the main thread, therefore freezing the event loop and probably the whole application as well. If a coroutine needs to spend some time doing nothing, it should yield from asyncio.sleep(DELAY).

The use of the @asyncio.coroutine decorator is not mandatory, but highly recommended: it makes the coroutines stand out among regular functions, and helps with debugging by issuing a warning when a coroutine is garbage collected without being yielded from—which means some operation was left unfinished and is likely a bug. This is not a priming decorator.

Note that the line count of spinner_thread.py and spinner_asyncio.py is nearly the same. The supervisor functions are the heart of these examples. Let’s compare them in detail. 
Example 18-3 lists only the supervisor from the Threading example.

Example 18-3. spinner_thread.py: the threaded supervisor function

```python
def supervisor():
    signal = Signal()
    spinner = threading.Thread(target=spin,
                               args=('thinking!', signal))
    print('spinner object:', spinner)
    spinner.start()
    result = slow_function()
    signal.go = False
    spinner.join()
    return result

```

For comparison, Example 18-4 shows the supervisor coroutine.

Example 18-4. spinner_asyncio.py: the asynchronous supervisor coroutine

```python

@asyncio.coroutine
def supervisor():
    spinner = asyncio.async(spin('thinking!'))
    print('spinner object:', spinner)
    result = yield from slow_function()
    spinner.cancel()
    return result

```

Here is a summary of the main differences to note between the two supervisor implementations:
- An asyncio.Task is roughly the equivalent of a threading.Thread. Victor Stinner, special technical reviewer for this chapter, points out that “a Task is like a green thread in libraries that implement cooperative multitasking, such as gevent.”
- A Task drives a coroutine, and a Thread invokes a callable.
- You don’t instantiate Task objects yourself, you get them by passing a coroutine to asyncio.async(…) or loop.create_task(…).
- When you get a Task object, it is already scheduled to run (e.g., by asyncio.async); a Thread instance must be explicitly told to run by calling its start method.
- In the threaded supervisor, the slow_function is a plain function and is directly invoked by the thread. In the asyncio supervisor, slow_function is a coroutine driven by yield from.
- There’s no API to terminate a thread from the outside, because a thread could be interrupted at any point, leaving the system in an invalid state. For tasks, there is the Task.cancel() instance method, which raises CancelledError inside the coroutine. The coroutine can deal with this by catching the exception in the yield where it’s suspended.
- The supervisor coroutine must be executed with loop.run_until_complete in the main function.

This comparison should help you understand how concurrent jobs are orchestrated with asyncio, in contrast to how it’s done with the more familiar Threading module.

One final point related to threads versus coroutines: if you’ve done any nontrivial programming with threads, you know how challenging it is to reason about the program because the scheduler can interrupt a thread at any time. You must remember to hold locks to protect the critical sections of your program, to avoid getting interrupted in the middle of a multistep operation—which could leave data in an invalid state.

With coroutines, everything is protected against interruption by default. You must explicitly yield to let the rest of the program run. Instead of holding locks to synchronize the operations of multiple threads, you have coroutines that are “synchronized” by definition: only one of them is running at any time. And when you want to give up control, you use yield or yield from to give control back to the scheduler. That’s why it is possible to safely cancel a coroutine: by definition, a coroutine can only be cancelled when it’s suspended at a yield point, so you can perform cleanup by handling the CancelledError exception.

We’ll now see how the asyncio.Future class differs from the concurrent.futures.Future class we saw in Chapter 17.
