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
    例18-2. spinner_asyncio.py：使用协程让文本旋转起来

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
    slow_function在这里是协程，使用yield from来让事件循环开始运行，而这个协程通过sleeping假装做一些I/O操作。
6. The yield from asyncio.sleep(3) expression handles the control flow to the main loop, which will resume this coroutine after the sleep delay.  
    表达式yield from asyncio.sleep(3)让控制流进入主循环，在sleep延迟后将恢复该协程。
7. supervisor is now a coroutine as well,so it can drive slow_function with yield from.  
    supervisot在这里也是协程，所以他可以通过yield from来驱动slow_function。
8. asyncio.async(…) schedules the spin coroutine to run, wrapping it in a Task object, which is returned immediately.  
    asyncio.async()安排spin协程的运行，将其包装进Task对象，该对象立刻返回。
9. Display the Task object. The output looks like <Task pending coro=<spin() running at spinner_asyncio.py:12>>.  
    显示Task对象。输出内容如下<Task pending coro=<spin() running at spinner_asyncio.py:12>>。
10. Drive the slow_function(). When that is done, get the returned value. Meanwhile, the event loop will continue running because slow_function ultimately uses yield from asyncio.sleep(3) to hand control back to the main loop.  
    驱动slow_function()。当其执行完成时获取返回值。同时，由于slow_function最终使用yield from asynico.sleep(3)让控制流交还至主循环，事件循环将继续运行。
11. A Task object can be cancelled; this raises asyncio.CancelledError at the yield line where the coroutine is currently suspended. The coroutine may catch the exception and delay or even refuse to cancel.  
    Task对象可以被取消；这会在当前暂停的协程位置的yield行会抛出asyncio.CancelledError。协程可能捕捉该异常并延迟甚至拒绝取消。
12. Get a reference to the event loop.  
    获取事件循环的引用。
13. Drive the supervisor coroutine to completion; the return value of the coroutine is the return value of this call.  
    驱动supervisor协程执行完成；协程的返回值是本次调用的返回值。

    Never use time.sleep(…) in asyncio coroutines unless you want to block the main thread, therefore freezing the event loop and probably the whole application as well. If a coroutine needs to spend some time doing nothing, it should yield from asyncio.sleep(DELAY).  
    永远不要在asyncio协程中使用time.sleep()，除非你想要阻塞主线程，从而冻结事件循环甚至整个应用。如果一个协程需要在一段时间什么都不做，应该使用yield from asyncio.sleep(DELAY)。

The use of the @asyncio.coroutine decorator is not mandatory, but highly recommended: it makes the coroutines stand out among regular functions, and helps with debugging by issuing a warning when a coroutine is garbage collected without being yielded from—which means some operation was left unfinished and is likely a bug. This is not a priming decorator.  
    @asyncio.coroutine装饰器不是强制使用的，但非常建议使用：他让协程在那些规则的函数中脱颖而出，且当一条协程被垃圾回收而没有yield from，通过一条warning信息很容易调试——这意味着某些操作是未完成的，这很可能是个bug。

Note that the line count of spinner_thread.py and spinner_asyncio.py is nearly the same. The supervisor functions are the heart of these examples. Let’s compare them in detail.  
    注意spinner_thread.py与spinner_asyncio.py的行数很接近。supervisor函数都是各自的核心。接下来我们详细地对比他们。

Example 18-3 lists only the supervisor from the Threading example.  
    示例18-3列出了只有线程示例的supervisor。

Example 18-3. spinner_thread.py: the threaded supervisor function  
    例18-3. spinner_thread.py：线程式的supervisor函数

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
    为了对比，例18-4为协程式的supervisor

Example 18-4. spinner_asyncio.py: the asynchronous supervisor coroutine  
    例18-4 spinner_asyncio.py：异步的supervisor协程

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
    如下是这两种supervisor实现的主要区别的概要：

- An asyncio.Task is roughly the equivalent of a threading.Thread. Victor Stinner, special technical reviewer for this chapter, points out that “a Task is like a green thread in libraries that implement cooperative multitasking, such as gevent.”  
    一个asyncio.Task大约等价于一个threading.Thread。本章的特别技术审查员Victor Stinner指出，“一个Task就像是完成协作多任务的库中一条绿色的线程，诸如gevent”。
- A Task drives a coroutine, and a Thread invokes a callable.  
    一个Task驱动一条协程，一条线程调用一个可调用对象。
- You don’t instantiate Task objects yourself, you get them by passing a coroutine to asyncio.async(…) or loop.create_task(…).  
    你不能自行实例化Task对象，你需要通过将协程传入asyncio.async()或loop.create_task()这种方式来获取他。
- When you get a Task object, it is already scheduled to run (e.g., by asyncio.async); a Thread instance must be explicitly told to run by calling its start method.  
    当你获取到一个Task对象，他已经被计划好了运行时机（如 通过asyncio.async）；一个线程实例必须通过调用它的start方法，明确通知其运行。
- In the threaded supervisor, the slow_function is a plain function and is directly invoked by the thread. In the asyncio supervisor, slow_function is a coroutine driven by yield from.  
    在线程式的supervisor中，slow_function是一个单纯的函数，他直接被线程所调用。在asyncio supervisor中，slow_function是一个由yield from所驱动的协程。
- There’s no API to terminate a thread from the outside, because a thread could be interrupted at any point, leaving the system in an invalid state. For tasks, there is the Task.cancel() instance method, which raises CancelledError inside the coroutine. The coroutine can deal with this by catching the exception in the yield where it’s suspended.  
    外部没有终止线程的API，因为一条线程可以在任意节点被中断，使系统处于无效状态。对tasks来说，对应的是Task.cancel()实例方法，他会在协程内部抛出CancelledError。协程可以通过捕捉暂停位置yield内的异常，来处理这个问题。
- The supervisor coroutine must be executed with loop.run_until_complete in the main function.  
    supervisor协程必须被main函数中的loop.run_until_complete执行。

This comparison should help you understand how concurrent jobs are orchestrated with asyncio, in contrast to how it’s done with the more familiar Threading module.  
    这些对比应该会帮你理解如何通过asyncio来协调并发工作，与使用我们更熟悉的Threading模块的方式形成对比。

One final point related to threads versus coroutines: if you’ve done any nontrivial programming with threads, you know how challenging it is to reason about the program because the scheduler can interrupt a thread at any time. You must remember to hold locks to protect the critical sections of your program, to avoid getting interrupted in the middle of a multistep operation—which could leave data in an invalid state.  
    关于线程vs协程的最后一点：如果你用线程做过任何重要的编程工作，你会知道对这个程序进行推理多具有挑战性，因为调度器可以在任意时间中断线程。你必须记住为你程序中关键的部分上锁来进行保护，这样避免在多步骤操作中间被中断——这可能让数据处于无效状态。

With coroutines, everything is protected against interruption by default. You must explicitly yield to let the rest of the program run. Instead of holding locks to synchronize the operations of multiple threads, you have coroutines that are “synchronized” by definition: only one of them is running at any time. And when you want to give up control, you use yield or yield from to give control back to the scheduler. That’s why it is possible to safely cancel a coroutine: by definition, a coroutine can only be cancelled when it’s suspended at a yield point, so you can perform cleanup by handling the CancelledError exception.  
    对协程来说，默认情况下所有内容都被保护不被中断。你必须明确地让步，让程序的其余部分运行。与多线程中用锁来同步操作不同，协程在定义之初即为“同步的”：任意时间点时只有其中一个在运行。当你想要放弃控制，你可以用yield或yield from让控制权交还至调度器。这就是取消一个协程可能是安全的原因：由于定义，一个协程只能在yield阶段暂停时被取消，所以你可以通过控制CancelledError异常来执行清理。

We’ll now see how the asyncio.Future class differs from the concurrent.futures.Future class we saw in Chapter 17.  
    我们现在将看到asyncio.Future类与17章的concurrent.futures.Future类有何不同。

### asyncio.Future: Nonblocking by Design
### asyncio.Future: 设计无阻塞

The asyncio.Future and the concurrent.futures.Future classes have mostly the same interface, but are implemented differently and are not interchangeable. PEP-3156 — Asynchronous IO Support Rebooted: the “asyncio” Module has this to say about this unfortunate situation:
    asyncio.Future与concurrent.futures.Future类有着几乎相同的接口，但是各自实现方式不同且不能交换使用。PEP-3156 —— 异步IO支持：关于这个不幸的情况，“asyncio”模块作了如下说明：

    In the future (pun intended) we may unify asyncio.Future and concurrent.futures.Future (e.g., by adding an __iter__ method to the latter that works with yield from).
    在future（一语双关）中，我们可能将asyncio.Future和concurrent.futures.Future（例如，通过向后者添加__iter__方法，与yield from共同作用）。

As mentioned in “Where Are the Futures?” on page 511, futures are created only as the result of scheduling something for execution. In asyncio, BaseEventLoop.create_task(…) takes a coroutine, schedules it to run, and returns an asyncio.Task instance—which is also an instance of asyncio.Future because Task is a subclass of Future designed to wrap a coroutine. This is analogous to how we create concurrent.futures.Future instances by invoking Executor.submit(…).  
    511页的“Where Are the Futures?”中提到，futures仅作为规划执行的执行结果所创建。在asyncio中，BaseEventLoop.create_task(…)创建一个协程，然后规划他执行，最终返回一个asyncio.Task实例——这也是asyncio.Future的一个实例，因为Task是Future的子类，用来包装协程。这与我们通过调用Executor.submit(…)来创建concurrent.futures.Future实例的方式是类似的。

Like its concurrent.futures.Future counterpart, the asyncio.Future class provides .done(), .add_done_callback(…), and .results() methods, among others. The first two methods work as described in “Where Are the Futures?” on page 511, but .result() is very different. In asyncio.Future, the .result() method takes no arguments, so you can’t specify a timeout. Also, if you call .result() and the future is not done, it does not block waiting for the result. Instead, an asyncio.InvalidStateError is raised.
However, the usual way to get the result of an asyncio.Future is to yield from it, as we’ll see in Example 18-8.  
    比如与concurrent.futures.Future对应的asyncio.Future类提供了.done()，.add_done_callback(…)与.result()方法等。前两个方法在511页的“Where Are the Futures?”有描述，但.result()不同。在asyncio.Future中，.result()方法不需要参数，所以你不能指定timeout。作为代替，会抛出InvalidStateError。但是，获取asyncio.Future结果的通常方式是yield from他，正如我们将在示例18-8中看到的。

Using yield from with a future automatically takes care of waiting for it to finish, without blocking the event loop—because in asyncio, yield from is used to give control back to the event loop. Note that using yield from with a future is the coroutine equivalent of the functionality offered by add_done_callback: instead of triggering a callback, when the delayed operation is done, the event loop sets the result of the future, and the yield from expression produces a return value inside our suspended coroutine, allowing it to resume.  
    对future自动使用yield from需要注意等他结束不阻塞事件循环——因为在asyncio中，yield from被用于将控制流交还给事件循环。注意，对一个future使用yield from相当于由add_done_callback提供功能的协程：为了取代触发回调，当延迟操作完成，事件循环设置future的结果，然后yield from表达式在我们暂停的协程中产出一个返回值，允许他恢复。

In summary, because asyncio.Future is designed to work with yield from, these methods are often not needed:  
    总而言之，因为asyncio.Future被定义为使用yield from运作，通常不需要这些方法：

- You don’t need my_future.add_done_callback(…) because you can simply put whatever processing you would do after the future is done in the lines that follow yield from my_future in your coroutine. That’s the big advantage of having coroutines: functions that can be suspended and resumed.  
    不需要my_future.add_done_callback(…)，因为你可以在自己的协程中yield from my_future行的后面，简单地指定future结束后要做的任何事情。这是一个巨大优势：函数可以被暂停和恢复。

- You don’t need my_future.result() because the value of a yield from expression on a future is the result (e.g., result = yield from my_future).
    不需要my_future.result()，因为在future上使用yield from表达式的值就是结果。（如，result = yield from my_future）。

Of course, there are situations in which .done(), .add_done_callback(…), and .results() are useful. But in normal usage, asyncio futures are driven by yield from, not by calling those methods.  
    当然，在某些情况下.done()，.add_done_callback(…)与.result()还是有用的。但通常使用下，asyncio futures是被yield from所驱动，而不是通过上面这些方法。

We’ll now consider how yield from and the asyncio API brings together futures, tasks, and coroutines.  
    我们现在思考一下yield from与asyncio API如何将futures，tasks与coroutines结合起来。

### Yielding from Futures, Tasks, and Coroutines

In asyncio, there is a close relationship between futures and coroutines because you can get the result of an asyncio.Future by yielding from it. This means that res = yield from foo() works if foo is a coroutine function (therefore it returns a coroutine object when called) or if foo is a plain function that returns a Future or Task instance.This is one of the reasons why coroutines and futures are interchangeable in many parts of the asyncio API.  
    在asyncio中，futures与协程有着密切的关系，因为你可以通过yield from asyncio.Future来获取他的结果。这意味着res = yield from foo()工作方式：如果foo是一个协程函数（因此当被调用时返回一个协程对象），或foo是一个单纯的函数，他会返回一个Future或Task实例。这是协程与future在许多asyncio API中可以交互使用的原因之一。

In order to execute, a coroutine must be scheduled, and then it’s wrapped in an asyncio.Task. Given a coroutine, there are two main ways of obtaining a Task:  
    为了执行，一个协程必须被规划好，然后被包装进asyncio.Task中。给定一个协程，有如下两种主要方式来获取Task：

asyncio.async(coro_or_future, *, loop=None)
    This function unifies coroutines and futures: the first argument can be either one. If it’s a Future or Task, it’s returned unchanged. If it’s a coroutine, async calls loop.create_task(…) on it to create a Task. An optional event loop may be passed as the loop= keyword argument; if omitted, async gets the loop object by calling asyncio.get_event_loop().  
    该函数将协程与future统一起来：首个参数可以是二者间任意一个。如果是Future或Task，原样返回。如果是协程，async对其调用loop.create_task(…)来创建Task。可以传入一个可选的事件循环，作为loop = keyword参数；如果省略，async通过调用asyncio.get_event_loop()来获取循环对象。

BaseEventLoop.create_task(coro)
    This method schedules the coroutine for execution and returns an asyncio.Task object. If called on a custom subclass of BaseEventLoop, the object returned may be an instance of some other Task-compatible class provided by an external library (e.g., Tornado).  
    该函数规划了协程的执行并返回一个asyncio.Task对象。如果在BaseEventLoop的自定义子类上调用，返回的对象可能是一些其他Task的实例——有一个额外库提供的兼容类（如Tornado）。

    BaseEventLoop.create_task(…) is only available in Python 3.4.2 or later. If you’re using an older version of Python 3.3 or 3.4, you need to use asyncio.async(…), or install a more recent version of asyncio from PyPI.  
    BaseEventLoop.create_task(…)仅在Python3.4.2及之后的版本支持。如果你使用Python3.3或3.4之前的版本，你需要用asyncio.async(…)，或从PyPI安装更近版本的asyncio。

Several asyncio functions accept coroutines and wrap them in asyncio.Task objects automatically, using asyncio.async internally. One example is BaseEventLoop.run_until_complete(…).  
    各种asyncio方法接受协程，并使用内置的asyncio.async自动将他们包装进asyncio.Task对象中。BaseEventLoop.run_until_complete(…)就是一个例子。

If you want to experiment with futures and coroutines on the Python console or in small tests, you can use the following snippet:[3]  
    如果你想要在Python控制台或小型测试中实验future与协程，你可以使用如下的片段：[3]

```

>>> import asyncio
>>> def run_sync(coro_or_future):
...     loop = asyncio.get_event_loop()
...     return loop.run_until_complete(coro_or_future)
...
>>> a = run_sync(some_coroutine())
```

The relationship between coroutines, futures, and tasks is documented in section 18.5.3. Tasks and coroutines of the asyncio documentation, where you’ll find this note:  
    18.5.3小节记录了协程，future与task的关系。asycnio文档的Task与coroutine，在那里你会发现如下的注释：
`
    In this documentation, some methods are documented as coroutines, even if they are plain Python functions returning a Future. This is intentional to have a freedom of tweaking the implementation of these functions in the future.  
    在此文档中，一些方法被当做协程记录，甚至他们只是返回Future的普通Python函数。这是有意为之的，为了将来可以自由调整这些函数的实现。

[3]. Suggested by Petr Viktorin in a September 11, 2014, message to the Python-ideas list.  
    这是由Petr Viktorin在2014年9月11日给Python-indeas提出的。

Having covered these fundamentals, we’ll now study the code for the asynchronous flag download script flags_asyncio.py demonstrated along with the sequential and thread pool scripts in Example 17-1 (Chapter 17).  
    在介绍这些基础后，我们现在学习示例17-1演示的的异步flag下载脚本flags_asyncio.py，以及顺序和线程池的脚本。

## Downloading with asyncio and aiohttp

As of Python 3.4, asyncio only supports TCP and UDP directly. For HTTP or any other protocol, we need third-party packages; aiohttp is the one everyone seems to be using for asyncio HTTP clients and servers at this time.  
    自Python3.4起，asyncio只直接支持TCP与UDP。对HTTP及其他协议，我们需要用到三方库：aiohttp是每个人目前被用作asyncio HTTP客户端与服务端的那个库。

Example 18-5 is the full listing for the flag downloading script flags_asyncio.py. Here is a high-level view of how it works:  
    例18-5是flag下载脚本flags_asyncio.py的完整列表。这里是他工作方式的高级视图：

1. We start the process in download_many by feeding the event loop with several coroutine objects produced by calling download_one.  
    在download_many中，通过调用download_one并将其产出的各种协程传入事件循环，以启动流程。
2. The asyncio event loop activates each coroutine in turn.  
    asyncio事件循环依次激活每个协程。
3. When a client coroutine such as get_flag uses yield from to delegate to a library coroutine—such as aiohttp.request—control goes back to the event loop, which can execute another previously scheduled coroutine.  
    当诸如get_flag的一个客户端协程使用yield from来委托给一个库协程（如aiohttp.request）时，控制权交还给事件循环，事件循环可以执行任意之前规划好的协程。
4. The event loop uses low-level APIs based on callbacks to get notified when a blocking operation is completed.  
    事件循环使用基于回调的低级API，来在阻塞操作完成时获取到通知。
5. When that happens, the main loop sends a result to the suspended coroutine.  
    当发生这种情况时，主循环向暂停的协程send结果。
6. The coroutine then advances to the next yield, for example, yield from resp.read() in get_flag. The event loop takes charge again. Steps 4, 5, and 6 repeat until the event loop is terminated.  
    然后协程推进到下一个yield，举个例子，get_flag中的yield from resp.read()。事件循环再次获取控制权，重复4，5，6知道事件循环结束。

This is similar to the example we looked at in “The Taxi Fleet Simulation” on page 490, where a main loop started several taxi processes in turn. As each taxi process yielded, the main loop scheduled the next event for that taxi (to happen in the future), and proceeded to activate the next taxi in the queue. The taxi simulation is much simpler, and you can easily understand its main loop. But the general flow is the same as in asyncio: a single-threaded program where a main loop activates queued coroutines one by one. Each coroutine advances a few steps, then yields control back to the main loop, which then activates the next coroutine in the queue.  
    这和我们在490页看到的“Taxi车队模拟”很类似，有一个主循环不断循环启动各种taxi进程。当任意taxi进程被产出，主循环会为该taxi（在将来发生）规划下一个事件，然后继续激活队列中的下一个taxi。这个taxi模型很简单，你可以很轻易理解他的主循环。但是通常的流程与asyncio类似：一个单独的线程程序，其中的主循环逐个激活队列中的协程。每个协程推进一小段，然后yield控制交还给主循环，继而激活队列的下一个协程。

Now let’s review Example 18-5 play by play.  
    现在我们详细解析下例18-5

Example 18-5. flags_asyncio.py: asynchronous download script with asyncio and aiohttp  
    例18-5. flags_asyncio.py：使用asyncio与aiohttp的异步下载脚本

```python
import asyncio
import aiohttp  # 1

from flags import BASE_URL, save_flag, show, main  # 2

@asyncio.coroutine  # 3
def get_flag(cc):
    url = '{}/{cc}/{cc}.gif'.format(BASE_URL, cc=cc.lower())
    resp = yield from aiohttp.request('GET', url)  # 4
    image = yield from resp.read()  # 5
    return image


@asyncio.coroutine
def download_one(cc):  # 6
    image = yield from get_flag(cc)  # 7
    show(cc)
    save_flag(image, cc.lower() + '.gif')
    return cc


def download_many(cc_list):
    loop = asyncio.get_event_loop()  # 8
    to_do = [download_one(cc) for cc in sorted(cc_list)]  # 9
    wait_coro = asyncio.wait(to_do)  # 10
    res, _ = loop.run_until_complete(wait_coro)  # 11
    loop.close()  # 12
    return len(res)


if __name__ == '__main__':
    main(download_many)
```

1. aiohttp must be installed—it’s not in the standard library.  
    aiohttp需要安装，不是标准库。
2. Reuse some functions from the flags module (Example 17-2).  
    复用flags模块中部分函数。
3. Coroutines should be decorated with @asyncio.coroutine.  
    协程需要使用@asyncio.coroutine装饰器。
4. Blocking operations are implemented as coroutines, and your code delegates to them via yield from so they run asynchronously.  
    阻塞操作被实现为协程，你的代码必须通过yield from委托给他们，所以他们是异步运行的。
5. Reading the response contents is a separate asynchronous operation.  
    读取响应内容是一个单独的异步操作。
6. download_one must also be a coroutine, because it uses yield from.  
    download_one也必须是协程，因为他使用了yield from。
7. The only difference from the sequential implementation of download_one are the words yield from in this line; the rest of the function body is exactly as before.  
    与顺序下载的download_one唯一的不同是本行的yield from；函数中其他部分与之前完全一致。
8. Get a reference to the underlying event-loop implementation.  
    获取对底层事件循环的引用。
9. Build a list of generator objects by calling the download_one function once for each flag to be retrieved.  
    构建一个生成器对象的列表，通过为每个要检索的flag调用download_one函数。
10. Despite its name, wait is not a blocking function. It’s a coroutine that completes when all the coroutines passed to it are done (that’s the default behavior of wait; see explanation after this example).  
    尽管他的名字有wait，但这不是一个阻塞函数。他是一个协程，当所有传给他的协程都已完成时，该协程完成（这是wait的默认行为；看看下面的说明）。
11. Execute the event loop until wait_coro is done; this is where the script will block while the event loop runs. We ignore the second item returned by run_until_complete. The reason is explained next.  
    执行事件循环直到wait_coro完成；在事件循环运行的阶段，脚本将阻塞在这里。我们由run_until_complete返回的第二项。下面解释原因。
12. Shut down the event loop.  
    关闭事件循环。

`
    It would be nice if event loop instances were context managers, so we could use a with block to make sure the loop is closed. However, the situation is complicated by the fact that client code never creates the event loop directly, but gets a reference to it by calling asyncio.get_event_loop(). Sometimes our code does not “own” the event loop, so it would be wrong to close it. For example, when using an external GUI event loop with a package like Quamash, the Qt library is responsible for shutting down the loop when the application quits.  
    如果事件循环实例是上下文控制器就太好了，我们可以用一个with模块来保证循环的关闭。但是这种情况在客户代码未直接创建事件循环时会变得复杂，不过可以通过asyncio.get_event_loop()来获取他的引用。然而，由于客户端代码从不直接创建事件循环，而是通过调用asyncio.get_event_loop()获取对它的引用，因此情况变得复杂了。有时候我们的代码并不“拥有”事件循环，所以关闭他的时候可能会出问题。举个例子，当通过某个像Quamash的库使用了一个外部GUI事件循环，Qt库负责在应用退出时关闭这个循环。

The asyncio.wait(…) coroutine accepts an iterable of futures or coroutines; wait wraps each coroutine in a Task. The end result is that all objects managed by wait become instances of Future, one way or another. Because it is a coroutine function, calling wait(…) returns a coroutine/generator object; this is what the wait_coro variable holds.To drive the coroutine, we pass it to loop.run_until_complete(…).  
    asyncio.wait(…)协程接受一个由future或协程组成的可迭代对象；wait将每个协程包装进Task。最终的结果是所有由wait管理的对象都以某种方式成为了Future的实例。因为这是一个协程函数，调用wait(…)会返回一个协程/生成器对象；这就是wait_coro变量包含的内容。为了驱动协程，我们将他传入loop.run_until_complete(…)。

The loop.run_until_complete function accepts a future or a coroutine. If it gets a coroutine, run_until_complete wraps it into a Task, similar to what wait does. Coroutines, futures, and tasks can all be driven by yield from, and this is what run_until_complete does with the wait_coro object returned by the wait call. When wait_coro runs to completion, it returns a 2-tuple where the first item is the set of completed futures, and the second is the set of those not completed. In Example 18-5, the second set will always be empty—that’s why we explicitly ignore it by assigning to _. But wait accepts two keyword-only arguments that may cause it to return even if some of the futures are not complete: timeout and return_when. See the asyncio.wait documentation for details.  
    loop.run_until_complete函数接收一个future或一个协程。如果获取到协程，run_until_complete会将他包装进Task，这与wait所做的类似。协程，future与task这三者都可以被yield from所驱动，这就是run_until_complete与wait_coro对象（有wait调用所返回）所做的工作。当wait_coro运行完成，他会返回一个二元的元组，第一个值是已完成future的集合，第二个值是哪些没有完成的集合。在示例18-5中，第二个几个总是为空——这是因为我们通过将其赋值给_来显式地忽略这一项。但是 wait 接受两个仅包含关键字的参数，即使某些futures未完成也可能导致它返回：timeout和return_when。

Note that in Example 18-5 I could not reuse the get_flag function from flags.py (Example 17-2) because that uses the requests library, which performs blocking I/O. To leverage asyncio, we must replace every function that hits the network with an asynchronous version that is invoked with yield from, so that control is given back to the event loop. Using yield from in get_flag means that it must be driven as a coroutine.  
    注意，在18-5中我不能复用flags.py的get_flag函数（例17-2），因为它使用了requests库，这阻塞了I/O。要利用asyncio，我们必须将所有访问网络的函数都替换为一个异步版本，该异步版本通过yield from调用，以便控制权可以被交还给事件循环。在get_flag中使用yield from意味着他必须作为一个协程被驱动。

That’s why I could not reuse the download_one function from flags_threadpool.py (Example 17-3) either. The code in Example 18-5 drives get_flag with yield_from, so download_one is itself also a coroutine. For each request, a download_one coroutine object is created in download_many, and they are all driven by the loop.run_until_complete function, after being wrapped by the asyncio.wait coroutine.  
    这也是我没有复用例17-3 falgs_threadpool.py中download_one函数的原因。18-5的代码通过yield from驱动了get_flag，所以download_one本身也是一个协程。为了每次请求，一个download_one协程对象在download_many中创建，他们都在被 asyncio.wait 协程包装之后，由loop.run_until_complete所驱动。

There are a lot of new concepts to grasp in asyncio but the overall logic of Example 18-5 is easy to follow if you employ a trick suggested by Guido van Rossum himself: squint and pretend the yield from keywords are not there. If you do that, you’ll notice that the code is as easy to read as plain old sequential code.  
    asyncio中有许多新概念需要掌握，但如果你使用Guido van Rossum他自己的小技巧：眯起眼睛，然后假装这里没有yield from，这样示例18-5的整体逻辑就很容易理解了。如果按这样做，你会发现代码像是简单的旧顺序代码一样容易阅读。

For example, imagine that the body of this coroutine…  
    例如，想想这个协程的结构体…

```python
@asyncio.coroutine
def get_flag(cc):
    url = '{}/{cc}/{cc}.gif'.format(BASE_URL, cc=cc.lower())
    resp = yield from aiohttp.request('GET', url)
    image = yield from resp.read()
    return image
```

…works like the following function, except that it never blocks:  
    像是下面的函数一样运作，除了他不会阻塞外：

```python
def get_flag(cc):
    url = '{}/{cc}/{cc}.gif'.format(BASE_URL, cc=cc.lower())
    resp = aiohttp.request('GET', url)
    image = resp.read()
    return image
```

Using the yield from foo syntax avoids blocking because the current coroutine is suspended (i.e., the delegating generator where the yield from code is), but the control flow goes back to the event loop, which can drive other coroutines. When the foo future or coroutine is done, it returns a result to the suspended coroutine, resuming it.  
    使用yield from foo语法可以避免阻塞，因为当前协程被暂停了（也就是说，yield from代码所在的委托生成器），但控制流回归至事件循环，用来驱动其他协程。当foo future或协程执行结束，他会返回结果至挂起的协程来重启他。

At the end of the section “Using yield from” on page 477, I stated two facts about every usage of yield from. Here they are, summarized:  
    在477页“Using yield form”节的结尾，关于yield from的每一种用法我都陈述了两个事实。这里总结一下：

- Every arrangement of coroutines chained with yield from must be ultimately driven by a caller that is not a coroutine, which invokes next(…) or .send(…) on the outermost delegating generator, explicitly or implicitly (e.g., in a for loop).  
    链接yield from与协程的每一次编排，最终必须由一个非协程的调用者所驱动，其在最外层的委托生成器上显示或隐式地调用next(…)或.send(…)（如 在一个for循环中）。
- The innermost subgenerator in the chain must be a simple generator that uses just yield—or an iterable object.  
    该链接中最内层的子生成器，必须是仅使用yield或一个可迭代对象的一个简单生成器。

When using yield from with the asyncio API, both facts remain true, with the following specifics:  
    当与asyncio API使用yield from是，这两个事实仍然成立，具体如下：

- The coroutine chains we write are always driven by passing our outermost delegating generator to an asyncio API call, such as loop.run_until_complete(…). In other words, when using asyncio our code doesn’t drive a coroutine chain by calling next(…) or .send(…) on it—the asyncio event loop does that.  
    我们编写的协程链总是通过将最外层的委托生成器至asyncio API调用来驱动，如loop.run_until_complete(…)。换句话说，当使用asuycio时，我们的代码不能同调用next()或send()来驱动协程——事件循环会做这件事。
- The coroutine chains we write always end by delegating with yield from to some asyncio coroutine function or coroutine method (e.g., yield from asyncio.sleep(…) in Example 18-2) or coroutines from libraries that implement higher-level protocols (e.g., resp = yield from aiohttp.request('GET', url) in the get_flag coroutine of Example 18-5).  In other words, the innermost subgenerator will be a library function that does the actual I/O, not something we write.  
    我们写的协程链总是以如下情况结束，通过分配yield from至一些asyncio协程函数或协程方法（如18-2的yield from asyncio.sleep(…)），或实现高级协议的库中的协程（如18-5 get_flag协程中的resp = yield from aiohttp.request('GET', url)）。换句话说，最内层的子生成器将是一个执行实际I/O的库函数，而不是我们编写的东西。

To summarize: as we use asyncio, our asynchronous code consists of coroutines that are delegating generators driven by asyncio itself and that ultimately delegate to asyncio library coroutines—possibly by way of some third-party library such as aiohttp. This arrangement creates pipelines where the asyncio event loop drives—through our coroutines—the library functions that perform the low-level asynchronous I/O.  
    总而言之：当我们使用asyncio，我们的异步代码由协程构成，这些协程是由asyncio自己驱动的委托生成器，且直接委托给asyncio库协程——可能通过一些像是aiohttp的第三方库。这种分配创建了asyncio事件循环驱动的管道——通过我们的协程——执行低级异步I/O的库函数。
We are now ready to answer one question raised in Chapter 17:  
    现在我们准备回答17章提出的一个问题：

- How can flags_asyncio.py perform 5× faster than flags.py when both are single threaded?  
    当两者都是单线程时，flags_asyncio.py是怎样比flags.py快5倍的？

### Running Circling Around Blocking Calls
### 围绕阻塞调用的运行

Ryan Dahl, the inventor of Node.js, introduces the philosophy of his project by saying "We’re doing I/O completely wrong.[4]" He defines a blocking function as one that does disk or network I/O, and argues that we can’t treat them as we treat nonblocking functions. To explain why, he presents the numbers in the first two columns of Table 18-1.  
    Node.js的开发者Ryan Dahl，通过说明“我们做的I/O是完全错误的[4]”以介绍他的项目哲学。他定义了一个阻塞函数用作处理磁盘或网络IO，并且主张我们不能像对待非阻塞函数一样对待他。为了解释原因，他展示了表18-1前两列的数字。

Table 18-1. Modern computer latency for reading data from different devices; third column shows proportional times in a scale easier to understand for us slow humans  
    表18-1. 当代计算机不同设备读取数据的延迟；第三列展示了修改比例的时间，便于我们理解

|Device| CPU cycles | Proportional “human” scale |
|---|---|---|
|L1 cache| 3 |3 seconds|
|L2 cache | 14 | 14 seconds|
|RAM | 250 | 250 seconds|
|disk | 41,000,000 | 1.3 years|
|network | 240,000,000 | 7.6 years|

[4] Video: Introduction to Node.js at 4:55.  

To make sense of Table 18-1, bear in mind that modern CPUs with GHz clocks run billions of cycles per second. Let’s say that a CPU runs exactly 1 billion cycles per second. That CPU can make 333,333,333 L1 cache reads in one second, or 4 (four!) network reads in the same time. The third column of Table 18-1 puts those numbers in perspective by multiplying the second column by a constant factor. So, in an alternate universe, if one read from L1 cache took 3 seconds, then a network read would take 7.6 years!  
    为了理解表18-1，请记住拥有GHz时钟的当代CPU每秒可以运行数十亿个周期。假设一个CPU每秒实际运行十亿个周期，那该CPU可以在1秒内执行333,333,333个L1 cache的读取，或是执行4次网络读取。表的第三列是第二列乘以一个常数因子获取的数字。所以在另一个宇宙中，如果读取L1缓存花费3秒，那么读取网络将花费7.6年！

There are two ways to prevent blocking calls to halt the progress of the entire application:  
    有两种方法可以防止阻塞调用以停止整个应用程序的进程：

- Run each blocking operation in a separate thread.  
    在单独的线程中运行每个阻塞操作。
- Turn every blocking operation into a nonblocking asynchronous call.  
    将每个阻塞操作转换为非阻塞异步调用。

Threads work fine, but the memory overhead for each OS thread—the kind that Python uses—is on the order of megabytes, depending on the OS. We can’t afford one thread per connection if we are handling thousands of connections.  
    线程很棒，但是每个OS线程（Python使用的那种）的内存开销是百万兆字节的，这取决于OS。如果我们要处理数千个连接，不能为每次连接都提供一个线程。

Callbacks are the traditional way to implement asynchronous calls with low memory overhead. They are a low-level concept, similar to the oldest and most primitive concurrency mechanism of all: hardware interrupts. Instead of waiting for a response, we register a function to be called when something happens. In this way, every call we make can be nonblocking. Ryan Dahl advocates callbacks for their simplicity and low overhead.  
    回调是实现异步调用传统方式，拥有着低内存开销。这是一个低级的概念，类似于最古老最原始的并发机制：硬件中断。为了等待一个响应，我们注册一个函数，当发生了某些事时被调用。这种方式下，我们的每次调用都是非阻塞的。Ryan Dahl提倡了回调的简易与低开销。

Of course, we can only make callbacks work because the event loop underlying our asynchronous applications can rely on infrastructure that uses interrupts, threads, polling, background processes, etc. to ensure that multiple concurrent requests make progress and they eventually get done.[5] When the event loop gets a response, it calls back our code. But the single main thread shared by the event loop and our application code is never blocked—if we don’t make mistakes.  
    当然，我们只能让回调工作，因为我们异步应用底层的事件循环可以依赖于中断，线程，轮询后台进程等基础设施，来确保多个并发请求都在进行并最终完成。当事件循环获取一个响应时，他会回调我们的代码。但单独的主线程（由事件循环与我们的应用代码共享）用不阻塞——如果没犯错的话。

When used as coroutines, generators provide an alternative way to do asynchronous programming. From the perspective of the event loop, invoking a callback or calling .send() on a suspended coroutine is pretty much the same. There is a memory overhead for each suspended coroutine, but it’s orders of magnitude smaller than the overhead for each thread. And they avoid the dreaded “callback hell,” which we’ll discuss in “From Callbacks to Futures and Coroutines” on page 562.  
    生成器在被用作协程时，会提供一种异步编程的替代方式。从事件循环的角度看，调用回调或者在挂起协程上调用.send()基本是一样的。每个挂起的协程都有内存开销，但这比线程的开销要小几个数量级。并且这避免了可怕的“回调地狱”，我们会在562页“从回调到Future与协程”讨论到。

Now the five-fold performance advantage of flags_asyncio.py over flags.py should make sense: flags.py spends billions of CPU cycles waiting for each download, one after the other. The CPU is actually doing a lot meanwhile, just not running your program. In contrast, when loop_until_complete is called in the download_many function of flags_asyncio.py, the event loop drives each download_one coroutine to the first yield from, and this in turn drives each get_flag coroutine to the first yield from, calling aiohttp.request(…). None of these calls are blocking, so all requests are started in a fraction of a second.  
    现在，flags_asycnio.py对flags.py的5倍性能优势是可以讲得通了：flags.py花费了数十亿次CPU周期等待每次下载，一次又一次地。CPU实际上这时做了很多事，只是没有运行你的程序。对比之下，当flags_asyncio.py中download_mandy函数的loop_until_complete被调用时，事件循环会驱动每个download_one协程至首个yield from，这反过来驱动了每个get_flag协程至首个yield from，来调用aiohttp.request(…)。这些调用都没有阻塞，所以所有请求在不到一秒内都启动了。

As the asyncio infrastructure gets the first response back, the event loop sends it to the waiting get_flag coroutine. As get_flag gets a response, it advances to the next yield from, which calls resp.read() and yields control back to the main loop. Other responses arrive in close succession (because they were made almost at the same time). As each get_flag returns, the delegating generator download_flag resumes and saves the image file.  
    当asyncio基础结构获取到首个返回的响应，事件循环会将他send至等待的get_flag协程。当get_flag获取到一个响应，他会推动下一个yield from，调用resp.read()并将控制权交还给主循环。其他响应接踵而至（因为他们几乎是同时发送的）。当每个get_flag返回时，委托生成器download_flag恢复并保存图片文件。
`
    For maximum performance, the save_flag operation should be asynchronous, but asyncio does not provide an asynchronous filesystem API at this time—as Node does. If that becomes a bottleneck in your application, you can use the loop.run_in_executor function to run save_flag in a thread pool. Example 18-9 will show how.  
    为了最大化性能，save_flag操作应该是异步的，但asyncio此时不能提供一个异步文件系统API——Node提供。如果这成为你的应用中的瓶颈，你可以在一个线程池中用loop.run_in_executor函数来运行save_flag。示例18-9会告诉你怎么做。

Because the asynchronous operations are interleaved, the total time needed to download many images concurrently is much less than doing it sequentially. When making 600 HTTP requests with asyncio I got all results back more than 70 times faster than with a sequential script.  
    因为异步操作是交错进行的，所以并发下载多个图片的总时间要比顺序版本小得多。当用asyncio发送600次HTTP请求时，我得到的所有结果的速度要比顺序脚本快70多倍。

Now let’s go back to the HTTP client example to see how we can display an animated progress bar and perform proper error handling.  
    现在我们返回到HTTP clien示例，来看看怎样显示一个动态的进度条，以及执行正确的Error处理。

## Enhancing the asyncio downloader Script

Recall from “Downloads with Progress Display and Error Handling” on page 520 that the flags2 set of examples share the same command-line interface. This includes the flags2_asyncio.py we will analyze in this section. For instance, Example 18-6 shows how to get 100 flags (-al 100) from the ERROR server, using 100 concurrent requests (-m 100).  
    回想一下520页的“带有进度条与错误处理的下载”，flags2示例共享的相同命令行界面。这包含了我们将在本节分析的flags2_asyncio.py。例如，例18-6展示了如何通过100条并发请求（-m 100）从ERROR服务中获取100个flag（-al 100）。

Example 18-6. Running flags2_asyncio.py

```
$ python3 flags2_asyncio.py -s ERROR -al 100 -m 100
ERROR site: http://localhost:8003/flags
Searching for 100 flags: from AD to LK
100 concurrent connections will be used.
--------------------
73 flags downloaded.
27 errors.
Elapsed time: 0.64s
```

    Act Responsibly When Testing Concurrent Clients  
    在测试并发客户端时采取负责任的行动
    Even if the overall download time is not different between the threaded and asyncio HTTP clients, asyncio can send requests faster, so it’s even more likely that the server will suspect a DOS attack. To really exercise these concurrent clients at full speed, set up a local HTTP server for testing, as explained in the README.rst inside the 17-futures/countries/ directory of the Fluent Python code repository.  
    即使线程和asyncio HTTP客户端总的下载时间所差无几，但asyncio可以更快的发送请求，所以更有可能发生的是服务端会怀疑收到DOS攻击。为了真的全速运行这些并发客户端，配置一个本地HTTP服务用于测试，如Fluent Python代码存储库的 17-futures/countries/ 目录中的README.rst中所述。

Now let’s see how flags2_asyncio.py is implemented.  

### Using asyncio.as_completed

In Example 18-5, I passed a list of coroutines to asyncio.wait, which—when driven by loop.run_until _complete—would return the results of the downloads when all were done. But to update a progress bar we need to get results as they are done. Fortunately, there is an asyncio equivalent of the as_completed generator function we used in the thread pool example with the progress bar (Example 17-14).  
    在示例18-5中，我将一组协程列表传入asyncio.wait（在被loop.run_until_complete驱动时），当他们全部结束后会返回下载结果。但为了更新进度条，他们一旦完成我们就需要获取到结果。幸运的是，有一个asyncio函数，等效于我们之前带进度条的线程池示例中的as_completed生成器函数。（示例17-14）

Writing a flags2 example to leverage asyncio entails rewriting several functions that the concurrent future version could reuse. That’s because there’s only one main thread in an asyncio program and we can’t afford to have blocking calls in that thread, as it’s the same thread that runs the event loop. So I had to rewrite get_flag to use yield from for all network access. Now get_flag is a coroutine, so download_one must drive it with yield from, therefore download_one itself becomes a coroutine。Previously, in Example 18-5, download_one was driven by download_many: the calls to download_one were wrapped in an asyncio.wait call and passed to loop.run_until_complete. Now we need finer control for progress reporting and error handling, so I moved most of the logic from download_many into a new downloader_coro coroutine, and use download_many just to set up the event loop and schedule downloader_coro.  
    编写一个flags示例来利用asyncio，需要重写顺序版的函数，并发future版本可以复用。那是因为asyncio程序里只有一个主线程，我们不能在这个线程中进行阻塞调用，因为这同样是运行事件循环的线程。所以我重写了get_flag，为所有网络访问使用yield from。现在get_flag就是一个协程，所以download_one必须用yiled from来驱动他，因此download_one自己也变成了一个协程。在之前的示例18-5中，download_one由download_many所驱动：对download_one的调用被asuncio.wait调用所包装并传入至loop.run_until_complete。现在为了进度的报告与error处理，我们需要更细致的控制，所以我将download_many的大部分逻辑移至了一个新协程downloader_coro，并且使用download_many仅去配置事件循环并调度downloader_coro。

Example 18-7 shows the top of the flags2_asyncio.py script where the get_flag and download_one coroutines are defined. Example 18-8 lists the rest of the source, with downloader_coro and download_many.  
    例18-7展示了flags2_asyncio.py脚本的顶部部分，定义了get_flga与download_one协程。例18-8列出了源代码的其余部分downloader_coro与download_many。

Example 18-7. flags2_asyncio.py: Top portion of the script; remaining code is in Example 18-8
```python
import asyncio
import collections

import aiohttp
from aiohttp import web
import tqdm
from flags2_common import main, HTTPStatus, Result, save_flag

# default set low to avoid errors from remote site, such as
# 503 - Service Temporarily Unavailable
DEFAULT_CONCUR_REQ = 5
MAX_CONCUR_REQ = 1000

class FetchError(Exception):  # 1
    def __init__(self, country_code):
        self.country_code = country_code


@asyncio.coroutine
def get_flag(base_url, cc)  # 2
    url = '{}/{cc}/{cc}.gif'.format(base_url, cc=cc.lower())
    resp = yield from aiohttp.request('GET', url)
    if resp.status == 200:
        image = yield from resp.read()
        return image
    elif resp.status == 404:
        raise web.HTTPNotFound()
    else:
        raise aiohttp.HttpProcessingError(
            code=resp.status, message=resp.reason,
            headers=resp.headers)


@asyncio.coroutine
def download_one(cc, base_url, semaphore, verbose):  # 3
    try:
        with (yield from semaphore):  # 4
            image = yield from get_flag(base_url, cc)  # 5
    except web.HTTPNotFound:  # 6
        status = HTTPStatus.not_found
        msg = 'not found'
    except Exception as exc:
        raise FetchError(cc) from exc  # 7
    else:
        save_flag(image, cc.lower() + '.gif')  # 8
        status = HTTPStatus.ok
        msg = 'OK'

    if verbose and msg:
        print(cc, msg)

    return Result(status, cc)
```

1. This custom exception will be used to wrap other HTTP or network exceptions and carry the country_code for error reporting.  
    此自定义异常将被用于包装其他HTTP或网络异常，且携带country_code用于报告错误。
2. get_flag will either return the bytes of the image downloaded, raise web.HTTPNotFound if the HTTP response status is 404, or raise an aiohttp.HttpProcessingError for other HTTP status codes.  
    get_flag将会返回下载的图片的字节，如果HTTP相应状态为404会抛出web.HTTPNotFound，其余HTTP状态码则抛出aiohttp.HttpProcessingError。
3. The semaphore argument is an instance of asyncio.Semaphore, a synchronization device that limits the number of concurrent requests.  
    参数semaphore是asyncio.Semaphore的实例，是一种限制并发请求数的同步设备。
4. A semaphore is used as a context manager in a yield from expression so that the system as whole is not blocked: only this coroutine is blocked while the semaphore counter is at the maximum allowed number.  
    yield from表达式中的semaphore被用作文本管理器，这样整个系统就不会阻塞：当信号量计数器处于允许的最大数量时，只有这条协程被阻塞
5. When this with statement exits, the semaphore counter is decremented, unblocking some other coroutine instance that may be waiting for the same semaphore object.  
6. If the flag was not found, just set the status for the Result accordingly.
7. Any other exception will be reported as a FetchError with the country code and the original exception chained using the raise X from Y syntax introduced in PEP 3134 — Exception Chaining and Embedded Tracebacks.
8. This function call actually saves the flag image to disk.

In Example 18-7, you can see that the code for get_flag and download_one changed significantly from the sequential version because these functions are now coroutines using yield from to make asynchronous calls.  
Network client code of the sort we are studying should always use some throttling mechanism to avoid pounding the server with too many concurrent requests—the overall performance of the system may degrade if the server is overloaded. In flags2_threadpool.py (Example 17-14), the throttling was done by instantiating the ThreadPoolExecutor with the required max_workers argument set to concur_req in the download_many function, so only concur_req threads are started in the pool. In flags2_asyncio.py, I used an asyncio.Semaphore, which is created by the downloader_coro function (shown next, in Example 18-8) and is passed as the semaphore argument to download_one in Example 18-7.[6]  

A Semaphore is an object that holds an internal counter that is decremented whenever we call the .acquire() coroutine method on it, and incremented when we call the .release() coroutine method. The initial value of the counter is set when the Semaphore is instantiated, as in this line of downloader_coro:  

    semaphore = asyncio.Semaphore(concur_req)

Calling .acquire() does not block when the counter is greater than zero, but if the counter is zero, .acquire() will block the calling coroutine until some other coroutine calls .release() on the same Semaphore, thus incrementing the counter. In Example 18-7, I don’t call .acquire() or .release(), but use the semaphore as a context manager in this block of code inside download_one:

    with (yield from semaphore):
        image = yield from get_flag(base_url, cc)

That snippet guarantees that no more than concur_req instances of get_flags coroutines will be started at any time.  
Now let’s take a look at the rest of the script in Example 18-8. Note that most functionality of the old download_many function is now in a coroutine, downloader_coro. This was necessary because we must use yield from to retrieve the results of the futures yielded by asyncio.as_completed, therefore as_completed must be invoked in a coroutine. However, I couldn’t simply turn download_many into a coroutine, because I must pass it to the main function from flags2_common in the last line of the script, and that main function is not expecting a coroutine, just a plain function. Therefore I created downloader_coro to run the as_completed loop, and now download_many simply sets up the event loop and schedules downloader_coro by passing it to loop.run_until_complete.  

Example 18-8. flags2_asyncio.py: Script continued from Example 18-7

```python
@asyncio.coroutine
def downloader_coro(cc_list, base_url, verbose, concur_req):  # 1
    counter = collections.Counter()
    semaphore = asyncio.Semaphore(concur_req)  # 2
    to_do = [download_one(cc, base_url, semaphore, verbose)
             for cc in sorted(cc_list)]  # 3

    to_do_iter = asyncio.as_completed(to_do)  # 4 
    if not verbose:
        to_do_iter = tqdm.tqdm(to_do_iter, total=len(cc_list))  # 5
    for future in to_do_iter:  # 6
        try:
            res = yield from future  # 7
        except FetchError as exc:  # 8
            country_code = exc.country_code  # 9
            try:
                error_msg = exc.__cause__.args[0]  # 10
            except IndexError:
                error_msg = exc.__cause__.__class__.__name__  # 11
            if verbose and error_msg:
                msg = '*** Error for {}: {}'
                print(msg.format(country_code, error_msg))
            status = HTTPStatus.error
        else:
            status = res.status
    
        counter[status] += 1  # 12

    return counter  # 13


def download_many(cc_list, base_url, verbose, concur_req):
    loop = asyncio.get_event_loop()
    coro = downloader_coro(cc_list, base_url, verbose, concur_req)
    counts = loop.run_until_complete(coro)  # 14
    loop.close()  # 15

    return counts


if __name__ == '__main__':
    main(download_many, DEFAULT_CONCUR_REQ, MAX_CONCUR_REQ)
```

1. The coroutine receives the same arguments as download_many, but it cannot be invoked directly from main precisely because it’s a coroutine function and not a plain function like download_many.
2. Create an asyncio.Semaphore thatwill allowup to concur_req active coroutines among those using this semaphore.
3. Create a list of coroutine objects, one per call to the download_one coroutine.
4. Get an iterator that will return futures as they are done.
5. Wrap the iterator in the tqdm function to display progress.
6. Iterate over the completed futures; this loop is very similar to the one in download_many in Example 17-14; most changes have to do with exception handling because of differences in the HTTP libraries (requests versus aiohttp).
7. The easiest way to retrieve the result of an asyncio.Future is using yield from instead of calling future.result().
8. Every exception in download_one is wrapped in a FetchError with the original exception chained.
9. Get the country code where the error occurred from the FetchError exception.
10. Try to retrieve the error message from the original exception (__cause__).
11. If the error message cannot be found in the original exception, use the name of the chained exception class as the error message.
12. Tally outcomes.
13. Return the counter, as done in the other scripts.
14. download_many simply instantiates the coroutine and passes it to the event loop with run_until_complete.
15. When all work is done, shut down the event loop and return counts.

In Example 18-8, we could not use the mapping of futures to country codes we saw in Example 17-14 because the futures returned by asyncio.as_completed are not necessarily the same futures we pass into the as_completed call. Internally, the asyncio machinery replaces the future objects we provide with others that will, in the end, produce the same results.[7]  

Because I could not use the futures as keys to retrieve the country code from a dict in case of failure, I implemented the custom FetchError exception (shown in Example 18-7). FetchError wraps a network exception and holds the country code associated with it, so the country code can be reported with the error in verbose mode.If there is no error, the country code is available as the result of the yield from future expression at the top of the for loop.  

This wraps up the discussion of an asyncio example functionally equivalent to the flags2_threadpool.py we saw earlier. Next, we’ll implement enhancements to flags2_asyncio.py that will let us explore asyncio further.  

While discussing Example 18-7, I noted that save_flag performs disk I/O and should be executed asynchronously. The following section shows how.
