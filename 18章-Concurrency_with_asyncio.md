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
    yield from表达式中的semaphore被用作文本管理器，这样整个系统就不会阻塞：当信号量计数器处于允许的最大数量时，只有这条协程被阻塞。
5. When this with statement exits, the semaphore counter is decremented, unblocking some other coroutine instance that may be waiting for the same semaphore object.  
    当这条with语句退出时，信号量计数会递减，这会解除其他等待相同信号量对象的协程实例的阻塞。
6. If the flag was not found, just set the status for the Result accordingly.  
    如果未找到flag，只需要为结果设置相应的状态。
7. Any other exception will be reported as a FetchError with the country code and the original exception chained using the raise X from Y syntax introduced in PEP 3134 — Exception Chaining and Embedded Tracebacks.  
    任何其他的异常将报告为带有国家code的FetchError与使用PEP 3134中引入的raise X from Y语法链接的原始异常——异常链接和嵌入式回溯。
8. This function call actually saves the flag image to disk.  
    本次调用将flag图片实际保存至硬盘。

In Example 18-7, you can see that the code for get_flag and download_one changed significantly from the sequential version because these functions are now coroutines using yield from to make asynchronous calls.  
    在示例18-7中，你可以看到get_flag与download_one的代码与顺序版本相比有了很大变化，因为这函数现在是使用了yield from的协程，来进行异步调用。

Network client code of the sort we are studying should always use some throttling mechanism to avoid pounding the server with too many concurrent requests—the overall performance of the system may degrade if the server is overloaded. In flags2_threadpool.py (Example 17-14), the throttling was done by instantiating the ThreadPoolExecutor with the required max_workers argument set to concur_req in the download_many function, so only concur_req threads are started in the pool. In flags2_asyncio.py, I used an asyncio.Semaphore, which is created by the downloader_coro function (shown next, in Example 18-8) and is passed as the semaphore argument to download_one in Example 18-7.[6]  
    我们现在研究的网络客户代码总是使用了一些节流机制，来避免过多并发请求导致对服务的冲击——如果服务器过载，系统的整体性能呢可能会下降。在flags2_threadpool.py（例17-14），节流是通过实例化ThreadPoolExecutor完成的，将需要的max_workers参数设置仅download_many函数中，这会在downloader_coro函数中创建（接下来的示例18-8会展示），并作为信号量参数传至示例18-7的download_one中。

A Semaphore is an object that holds an internal counter that is decremented whenever we call the .acquire() coroutine method on it, and incremented when we call the .release() coroutine method. The initial value of the counter is set when the Semaphore is instantiated, as in this line of downloader_coro:  
    Semaphore是一个对象，控制内部计数器，不论何时当我们对其调用了.acquire()协程方法就会减少，调用.release()协程方法时就会上升。该计数器的初始值在Semaphore实例化时被设置，如下面downloader_coro该行所示：

    semaphore = asyncio.Semaphore(concur_req)

Calling .acquire() does not block when the counter is greater than zero, but if the counter is zero, .acquire() will block the calling coroutine until some other coroutine calls .release() on the same Semaphore, thus incrementing the counter. In Example 18-7, I don’t call .acquire() or .release(), but use the semaphore as a context manager in this block of code inside download_one:  
    当计数器大于0时，调用.acquire()不会阻塞，但如果小于0，.acquire()将阻塞对协程的调用直到一些其他协程对统一Semaphore调用了.release()从而对计数器进行递增。在例18-7中，我没有调用.acquire()或.release()，而是在download_one代码这部分将信号量作为一个上下文管理器使用：

    with (yield from semaphore):
        image = yield from get_flag(base_url, cc)

That snippet guarantees that no more than concur_req instances of get_flags coroutines will be started at any time.  
    这段代码保证了由任意时间启动的get_flags协程实例数量都不会超过concur_req。
Now let’s take a look at the rest of the script in Example 18-8. Note that most functionality of the old download_many function is now in a coroutine, downloader_coro. This was necessary because we must use yield from to retrieve the results of the futures yielded by asyncio.as_completed, therefore as_completed must be invoked in a coroutine. However, I couldn’t simply turn download_many into a coroutine, because I must pass it to the main function from flags2_common in the last line of the script, and that main function is not expecting a coroutine, just a plain function. Therefore I created downloader_coro to run the as_completed loop, and now download_many simply sets up the event loop and schedules downloader_coro by passing it to loop.run_until_complete.  
    现在我们来看看示例18-8脚本的其余部分。注意，旧版download_many函数的大部分功能性内容现在都在协程downloader_coro中。这是必须的，因为我们必须用yield from来检索由asyncio.as_completed产出的future的结果，因此as_completed必须在协程中调用。但是，我不能简单地让download_many变成一个协程，因为我必须在脚本的最后一行将他从flags2_common传入至主函数，且主函数不需要协程，只是一个普通函数。因此我创建了downloader_coro来运行as_completed循环，现在download_many只是简单设置了事件循环，通过将他传入loop.run_until_complete来规划downloader_coro。

Example 18-8. flags2_asyncio.py: Script continued from Example 18-7  
    例18-8 flags2_asyncio.py：继18-7之后的脚本

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
    该协程接收的参数与download_many一样，但恰恰因为他是一个协程函数而不是像download_many一样的普通函数，不能从主函数中直接调用。
2. Create an asyncio.Semaphore that will allow up to concur_req active coroutines among those using this semaphore.  
    创建一个asyncio.Semaphore，在使用该信号量的下次中最多允许concur_req个协程。
3. Create a list of coroutine objects, one per call to the download_one coroutine.  
    创建一个协程对象的列表，每个都调用download_one协程。
4. Get an iterator that will return futures as they are done.  
    获取一个迭代器，在future完成时返回他们。
5. Wrap the iterator in the tqdm function to display progress.  
    将迭代器包装进tqdm方法，用于显示进度。
6. Iterate over the completed futures; this loop is very similar to the one in download_many in Example 17-14; most changes have to do with exception handling because of differences in the HTTP libraries (requests versus aiohttp).  
    遍历已经完成的协程；该循环与例17-14download_may中的那个很相似；大多数改动与异常处理有关，由于HTTP库的差异（requests和aiohttp）。
7. The easiest way to retrieve the result of an asyncio.Future is using yield from instead of calling future.result().  
    检索asyncio.Future结果的最简单方式是用yield from替代调用future.result()。
8. Every exception in download_one is wrapped in a FetchError with the original exception chained.  
    download_one的每个异常都被包装进一个FetchError，并链接了原始异常。
9. Get the country code where the error occurred from the FetchError exception.  
    从FetchError异常中获取异常发生处的国家码。
10. Try to retrieve the error message from the original exception (__cause__).  
    尝试从原始异常(__cause__)检索error信息。
11. If the error message cannot be found in the original exception, use the name of the chained exception class as the error message.  
    如果其中找不到error信息，就用连接的异常类的名称作为error信息。
12. Tally outcomes.  
    统计结果。
13. Return the counter, as done in the other scripts.  
    返回计数，和其他脚本中一样。
14. download_many simply instantiates the coroutine and passes it to the event loop with run_until_complete.  
    download_many简单的实例化了协程，并通过run_until_complete传入至事件循环。
15. When all work is done, shut down the event loop and return counts.  
    当所有工作完成，关闭事件循环并返回数量。

In Example 18-8, we could not use the mapping of futures to country codes we saw in Example 17-14 because the futures returned by asyncio.as_completed are not necessarily the same futures we pass into the as_completed call. Internally, the asyncio machinery replaces the future objects we provide with others that will, in the end, produce the same results.[7]  
    例18-8中，我们不能使用例17-14中看到的对future和国家码的匹配，因为asyncio.as_completed返回的future不是一定与传入as_completed的顺序是一致的。在内部，asyncio机制将我们提供的future对象替换为最终会产生相同结果的其他对象。

Because I could not use the futures as keys to retrieve the country code from a dict in case of failure, I implemented the custom FetchError exception (shown in Example 18-7). FetchError wraps a network exception and holds the country code associated with it, so the country code can be reported with the error in verbose mode.If there is no error, the country code is available as the result of the yield from future expression at the top of the for loop.  
    因为在失败的情况下，我不能从字典中将future用作key来检索国家码，所以我实现了自定义的FetchError异常。FetchError包装了一个network异常并带有与之相关的国家码，所以国家码可以用详细模式下的error来报告。如果没有error，作为for循环的顶部yield from future表达式的结果，国家码是可用的。

This wraps up the discussion of an asyncio example functionally equivalent to the flags2_threadpool.py we saw earlier. Next, we’ll implement enhancements to flags2_asyncio.py that will let us explore asyncio further.  
    这就终结了对功能等同于我们之前看到的flags2_threadpool.py的asyncio示例的讨论。接下来，我们将完成对flags2_asyncio.py的升级，这将让我们进一步探索asyncio。

While discussing Example 18-7, I noted that save_flag performs disk I/O and should be executed asynchronously. The following section shows how.  
    在讨论示例18-7的时候，我有提示save_flag执行了磁盘I/O，且他应给被异步执行。下面的小节介绍怎样实现。

### Using an Executor to Avoid Blocking the Event Loop 使用一个Executor来避免阻塞事件循环

In the Python community, we tend to overlook the fact that local filesystem access is blocking, rationalizing that it doesn’t suffer from the higher latency of network access (which is also dangerously unpredictable). In contrast, Node.js programmers are constantly reminded that all filesystem functions are blocking because their signatures require a callback. Recall from Table 18-1 that blocking for disk I/O wastes millions of CPU cycles, and this may have a significant impact on the performance of the application.  
    在Python社区中，我们倾向于忽视了一个事实，本地的文件系统访问是阻塞的，合理化它不会受到网络访问的更高延迟（这也是危险的不可预测的）。经过对比，Node.js程序员经常被提醒到，所有文件系统的方法都是阻塞的，因为他们的签名需要回调。回顾表18-1，磁盘I/O的阻塞花费数百万次的CPU周期，这可能会对应用程序的性能产生重大影响。

In Example 18-7, the blocking function is save_flag. In the threaded version of the script (Example 17-14), save_flag blocks the thread that’s running the download_one function, but that’s only one of several worker threads. Behind the scenes, the blocking I/O call releases the GIL, so another thread can proceed. But in flags2_asyncio.py, save_flag blocks the single thread our code shares with the asyncio event loop, therefore the whole application freezes while the file is being saved. The solution to this problem is the run_in_executor method of the event loop object.  
    在例18-7中，阻塞的方法是save_flag。在线程版本的脚本（例17-14）中，save_flag阻塞了运行download_one函数的线程，但那只是许多工作线程中的一条。在幕后，阻塞的I/O调用释放了GIL，所以其他线程才可以执行。但在flags2_asyncio.py中，save_flag阻塞了我们代码与asyncio事件循环共享的唯一线程，因此在保存文件的那段时间中整个应用处于冻结状态。解决这个问题的方式就是事件循环对象中的run_in_executor方法。

Behind the scenes, the asyncio event loop has a thread pool executor, and you can send callables to be executed by it with run_in_executor. To use this feature in our example, only a few lines need to change in the download_one coroutine, as shown in Example 18-9.  
    在幕后，asyncni事件循环有一个线程池执行器，你可以用run_in_executor向其发送可调用对象进行执行。为了在我们的示例使用这一特性，仅有download_one协程的几行需要修改，如示例18-9所示。

[7] A detailed discussion about this can be found in a thread I started in the python-tulip group, titled “Which other futures my come out of asyncio.as_completed?”. Guido responds, and gives insight on the implementation of as_completed as well as the close relationship between futures and coroutines in asyncio.  

Example 18-9. flags2_asyncio_executor.py: Using the default thread pool executor to run save_flag  
    例18-9. flags2_asyncio_executor.py：使用默认线程池执行器来运行save_flag

```python
@asyncio.coroutine
def download_one(cc, base_url, semaphore, verbose):
    try:
        with (yield from semaphore):
            image = yield from get_flag(base_url, cc)
    except web.HTTPNotFound:
        status = HTTPStatus.not_found
        msg = 'not found'
    except Exception as exc:
        raise FetchError(cc) from exc
    else:
        loop = asyncio.get_event_loop()  # 1
        loop.run_in_executor(None,  # 2
                save_flag, image, cc.lower() + '.gif')  # 3
        status = HTTPStatus.ok
        msg = 'OK'

    if verbose and msg:
        print(cc, msg)

    return Result(status, cc)
```

1. Get a reference to the event loop object.  
    获取对事件循环对象的引用。
2. The first argument to run_in_executor is an executor instance; if None, the default thread pool executor of the event loop is used.  
    run_in_executor的第一个参数是一个执行器示例；如果为None，就使用事件循环的默认线程池执行器。
3. The remaining arguments are the callable and its positional arguments.  
    其余的参数是可调用对象和他的位置参数。  
`

    When I tested Example 18-9, there was no noticeable change in performance for using run_in_executor to save the image files because they are not large (13 KB each, on average). But you’ll see an effect if you edit the save_flag function in flags2_common.py to save 10 times as many bytes on each file—just by coding fp.write(img*10) instead of fp.write(img). With an average download size of 130 KB, the advantage of using run_in_executor becomes clear. If you’re downloading megapixel images, the speedup will be significant.  
    当我测试例18-9时，关于使用run_in_executor来保存图片文件没有明显的性能变化，因为他们都不大（平均每个13KB）。但是如果你将flags2_common.py中save_flag函数进行编辑，在每个文件上节省10倍的字节——只是将fp.write(img)替代为fp.write(img*10)。平均下载大小为130KB，使用run_in_executor的优势就变得明显了。如果你下载的是百万像素的图片，速度提升将是显著的。

The advantage of coroutines over callbacks becomes evident when we need to coordinate asynchronous requests, and not just make completely independent requests. The next section explains the problem and the solution.  
    当我们需要协调异步请求，且不仅仅是完成独立的请求是，协程相较于回调的优势显而易见。下一节解释了问题与解决方案。

## From Callbacks to Futures and Coroutines

Event-oriented programming with coroutines requires some effort to master, so it’s good to be clear on how it improves on the classic callback style. This is the theme of this section.  
    使用协程的面向事件编程需要一些努力才能掌握，所以最好搞清楚他是如何改善了传统的回调风格。这就是本节的主题。

Anyone with some experience in callback-style event-oriented programming knows the term “callback hell”: the nesting of callbacks when one operation depends on the result of the previous operation. If you have three asynchronous calls that must happen in succession, you need to code callbacks nested three levels deep. Example 18-10 is an example in JavaScript.  
    使用过回调风格进行面向事件编程的人都知道术语“回调地狱”：当某次操作依赖于之前操作的结果时的回调的嵌套。如果你有三个必须连续发生的异步调用，你需要写嵌套三级的回调。例18-10是JavaScript中的例子。

Example 18-10. Callback hell in JavaScript: nested anonymous functions, a.k.a. Pyramid of Doom  
    例18-10. JavaScript中的回调地狱：嵌套匿名函数，又名金字塔的厄运

```javascript
api_call1(request1, function (response1) {
    // stage 1
    var request2 = step1(response1);
    api_call2(request2, function (response2) {
        // stage 2
        var request3 = step2(response2);
        api_call3(request3, function (response3) {
            // stage 3
            step3(response3);
        });
    });
});
```

In Example 18-10, api_call1, api_call2, and api_call3 are library functions your code uses to retrieve results asynchronously—perhaps api_call1 goes to a database and api_call2 gets data from a web service, for example. Each of these take a callback function, which in JavaScript are often anonymous functions (they are named stage1, stage2, and stage3 in the following Python example). The step1, step2, and step3 here represent regular functions of your application that process the responses received by the callbacks.  
    在示例18-10中，api_call1, api_call2与api_call3都是你的代码用来异步检索结果的库函数——例如，也许api_call1进入了一个数据库，api_call2从web服务获取数据。他们都有一个回调函数，这在JS中是通常是匿名函数（这在下面Python例子中被称为stage1，stage2，stage3）。这里的step1，step2与step3代表了你的应用中的常规函数，用来处理回调函数接收到地响应。

Example 18-11 shows what callback hell looks like in Python.  
    例18-11展示了Python中的回调地狱是什么样子的。

Example 18-11. Callback hell in Python: chained callbacks  
    例18-11. Python中的回调地狱：连锁回调

```python
def stage1(response1):
    request2 = step1(response1)
    api_call2(request2, stage2)


def stage2(response2):
    request3 = step2(response2)
    api_call3(request3, stage3)


def stage3(response3):
    step3(response3)


api_call1(request1, stage1)
```

Although the code in Example 18-11 is arranged very differently from Example 18-10, they do exactly the same thing, and the JavaScript example could be written using the same arrangement (but the Python code can’t be written in the JavaScript style because of the syntactic limitations of lambda).  
    虽然例18-11的代码与18-10的在顺序上完全不同，但他们实际做的是相同的事，JavaScript例子可以用相同的排列来写（但由于lambda的语法限制，Python的代码不能用JS的风格来写）。

Code organized as Example 18-10 or Example 18-11 is hard to read, but it’s even harder to write: each function does part of the job, sets up the next callback, and returns, to let the event loop proceed. At this point, all local context is lost. When the next callback (e.g., stage2) is executed, you don’t have the value of request2 any more. If you need it, you must rely on closures or external data structures to store it between the different stages of the processing.  
    像例18-10或18-11的代码结构阅读起来很困难，但更难的是编写：每个函数做一部分工作，设置下一个回调，然后返回，来让事件循环继续运行。此时，所有本地上下文都丢失了。当下一个回调（如stage2）执行时，你不再拥有request2的值。如果你需要这个值，你必须依赖闭包或额外的数据结构，在不同执行stage之间来存储他。

That’s where coroutines really help. Within a coroutine, to perform three asynchronous actions in succession, you yield three times to let the event loop continue running. When a result is ready, the coroutine is activated with a .send() call. From the perspective of the event loop, that’s similar to invoking a callback. But for the users of a coroutine-style asynchronous API, the situation is vastly improved: the entire sequence of three operations is in one function body, like plain old sequential code with local variables to retain the context of the overall task under way. See Example 18-12.  
    这就是协程真正有用的地方。在协程内部，为了连续执行3个异步动作，你会yield 3次来让事件循环继续运行。当一个结果ready时，协程就会被.send()调用激活。从事件循环的角度来看，这类似于调用回调。但对于协程风格异步API的使用者来说，这种情况大大改善了：三次操作的整体顺序都在一个函数体内，像是带有局部变量的普通旧顺序代码一样，以保留进行中的整个任务的上下文。看看例18-12。

Example 18-12. Coroutines and yield from enable asynchronous programming without callbacks  
    例18-12. 协程与yield from让异步编程脱离回调

```python
@asyncio.coroutine
def three_stages(request1):
    response1 = yield from api_call1(request1)
    # stage 1
    request2 = step1(response1)
    response2 = yield from api_call2(request2)
    # stage 2
    request3 = step2(response2)
    response3 = yield from api_call3(request3)
    # stage 3
    step3(response3)

loop.create_task(three_stages(request1)) # must explicitly schedule execution
```

Example 18-12 is much easier to follow the previous JavaScript and Python examples: the three stages of the operation appear one after the other inside the same function. This makes it trivial to use previous results in follow-up processing. It also provides a context for error reporting through exceptions.  
    例18-12比之前的JS与Python示例更容易理解：操作的三个stage在同一个函数中依次出现。这使得在后续处理中使用前面的结果这种事变的轻而易举。同时也通过异常提供了error报告的上下文。

Suppose in Example 18-11 the processing of the call api_call2(request2, stage2) raises an I/O exception (that’s the last line of the stage1 function). The exception cannot be caught in stage1 because api_call2 is an asynchronous call: it returns immediately, before any I/O is performed. In callback-based APIs, this is solved by registering two callbacks for each asynchronous call: one for handling the result of successful operations, another for handling errors. Work conditions in callback hell quickly deteriorate when error handling is involved.  
    假如例18-11中调用api_call2(request2, stage2)的操作导致了I/O异常（在stage1函数的最后一行）。那这个异常不能在stage1中被捕捉，因为api_call2是一个异步调用：他在任何I/O执行前就立刻返回了。在基于回调的API中，这种情况通过为每次异步调用登记两个回调解决：一个用来处理成功操作的结果，另一个用来处理error。当涉及error处理时，回调地狱的工作条件会迅速恶化。

In contrast, in Example 18-12, all the asynchronous calls for this three-stage operation are inside the same function, three_stages, and if the asynchronous calls api_call1, api_call2, and api_call3 raise exceptions we can handle them by putting the respective yield from lines inside try/except blocks.  
    对比之下，示例18-12中为这三个stage操作进行的所有异步调用都在同一个函数three_stages中，如果这些异步调用api_call1 api_call2 api_call3抛出了异常，我们可以通过将相应的yield from行放在try/except块中来处理他们。

This is a much better place than callback hell, but I wouldn’t call it coroutine heaven because there is a price to pay. Instead of regular functions, you must use coroutines and get used to yield from, so that’s the first obstacle. Once you write yield from in a function, it’s now a coroutine and you can’t simply call it, like we called api_call1(request1, stage1) in Example 18-11 to start the callback chain. You must explicitly schedule the execution of the coroutine with the event loop, or activate it using yield from in another coroutine that is scheduled for execution. Without the call loop.create_task(three_stages(request1)) in the last line, nothing would happen in Example 18-12.  
    这是个比回调地狱好得多的地方，但我不能就将协程称作天堂，因为这要付出代价。你必须用协程替换常规函数，且习惯于yield from，所以这是第一层障碍。一旦你在一个函数中写了yield from，他就变为了协程，你不能简单地像18-11中一样调用api_call1(request1, stage1)一样调用它。而且你必须使用事件循环显式地安排协程的执行逻辑，或者在另一个安排执行的协程中使用yield from来驱动他。删除最后一行的对loop.create_task(three_stages(requests1))的调用的话，示例18-12将不会发生任何事情。

The next example puts this theory into practice.  
    下一个例子将这个理论应用于实践。

### Doing Multiple Requests for Each Download  为每次下载执行多次请求

Suppose you want to save each country flag with the name of the country and the country code, instead of just the country code. Now you need to make two HTTP requests per flag: one to get the flag image itself, the other to get the metadata.json file in the same directory as the image: that’s where the name of the country is recorded.  
    假设你想用国家名与国家码来保存每个国家国旗，而不仅仅是国家码。现在你需要为每个flag做两次HTTP请求：一个用来获取flag图像本身，另一个来获取图像同一目录下的metadata.json文件：这是记录国家名的地方。

Articulating multiple requests in the same task is easy in the threaded script: just make one request then the other, blocking the thread twice, and keeping both pieces of data (country code and name) in local variables, ready to use when saving the files. If you need to do the same in an asynchronous script with callbacks, you start to smell the sulfur of callback hell: the country code and name will need to be passed around in a closure or held somewhere until you can save the file because each callback runs in a different local context. Coroutines and yield from provide relief from that. The solution is not as simple as with threads, but more manageable than chained or nested callbacks.  
    在线程脚本中很容易在同一任务重清楚表达多个请求：仅仅是做一次请求然后再做一次，阻塞线程两次，然后将两组数据（国家码与国家名）保存进本地变量，准备在保存文件时使用。如果你需要在用回调的异步脚本做到相同的事，你会开始嗅到回调地狱的硫磺味道：国家码与国家名将需要在闭包中传递或保存在某个地方，直到你可以保存文件，因为每个回调都运行在不同的本地上下文。协程与yield from可以缓解这种情况。这种解决方法没有比线程式更简单，但相比连锁或嵌套式的回调会更可控。

Example 18-13 shows code from the third variation of the asyncio flag downloading script, using the country name to save each flag. The download_many and downloader_coro are unchanged from flags2_asyncio.py (Examples 18-7 and 18-8). The changes are:  
    例18-13展示了asyncio flag下载脚本的第三个变异版本，使用国家名来保存每个flag。来自flags2_asyncio.py的download_many与downloader_coro没有变化。改动内容如下：

download_one  
    This coroutine now uses yield from to delegate to get_flag and the new get_country coroutine.  
    该协助现在用yield from委托给get_flag与新的get_country协程。
get_flag  
    Most code from this coroutine was moved to a new http_get coroutine so it can also be used by get_country.  
    该协程的大部分代码被移至新协程http_get，所以他也可以被get_country使用。
get_country  
    This coroutine fetches the metadata.json file for the country code, and gets the name of the country from it.  
    该协程获取记录国家码的metadata.json文件，然后从中获取国家名。
http_get  
    Common code for getting a file from the Web.  
    用于从web获取文件的通用代码。

Example 18-13. flags3_asyncio.py: more coroutine delegation to perform two requests per flag  
    例18-13. flag3_asyncio.py：更多的协程委托，为每个flag执行两次请求

```python
@asyncio.coroutine
def http_get(url):
    res = yield from aiohttp.request('GET', url)
    if res.status == 200:
        ctype = res.headers.get('Content-type', '').lower()
        if 'json' in ctype or url.endswith('json'):
            data = yield from res.json()  # 1
        else:
            data = yield from res.read()  # 2
        return data

    elif res.status == 404:
        raise web.HTTPNotFound()
    else:
        raise aiohttp.errors.HttpProcessingError(
            code=res.status, message=res.reason,
            headers=res.headers)


@asyncio.coroutine
def get_country(base_url, cc):
    url = '{}/{cc}/metadata.json'.format(base_url, cc=cc.lower())
    metadata = yield from http_get(url)  # 3
    return metadata['country']


@asyncio.coroutine
def get_flag(base_url, cc):
    url = '{}/{cc}/{cc}.gif'.format(base_url, cc=cc.lower())
    return (yield from http_get(url))  # 4


@asyncio.coroutine
def download_one(cc, base_url, semaphore, verbose):
    try:
        with (yield from semaphore):  # 5
            image = yield from get_flag(base_url, cc)
        with (yield from semaphore):
            country = yield from get_country(base_url, cc)
    except web.HTTPNotFound:
        status = HTTPStatus.not_found
        msg = 'not found'
    except Exception as exc:
        raise FetchError(cc) from exc
    else:
        country = country.replace(' ', '_')
        filename = '{}-{}.gif'.format(country, cc)
        loop = asyncio.get_event_loop()
        loop.run_in_executor(None, save_flag, image, filename)
        status = HTTPStatus.ok
        msg = 'OK'

    if verbose and msg:
        print(cc, msg)

    return Result(status, cc)

```

1. If the content type has 'json' in it or the url ends with .json, use the response .json() method to parse it and return a Python data structure—in this case, a dict.  
    如果content type包含json或url以.json结尾，用response的.json()方法来解析，并返回一个Python数据结构——该case中是字典。
2. Otherwise, use .read() to fetch the bytes as they are.  
    否则用.read()获取原样的字节。
3. metadata will receive a Python dict built from the JSON contents.  
    metadata将接受到一个根据JSON内容构建的Python字典。
4. The outer parentheses here are required because the Python parser gets confused and produces a syntax error when it sees the keywords return yield from lined up like that.  
    这里的小括号是必要的，因为Python解析器在看到像“return yield from”这样的关键字时会感到困惑并产生语法错误。
5. I put the calls to get_flag and get_country in separate with blocks controlled by the semaphore because I want to keep it acquired for the shortest possible time.  
    我将对get_flag与get_country的调用与信号量控制的块分开，因为我想尽可能短时间地获得他。

The yield from syntax appears nine times in Example 18-13. By now you should be getting the hang of how this construct is used to delegate from one coroutine to another without blocking the event loop.  
    yield from语法在例18-13出现了9次。如今你应该知道如何使用这种结构，在不阻塞事件循环的情况下从一个协程委派到另一个协程的窍门了。

The challenge is to know when you have to use yield from and when you can’t use it. The answer in principle is easy, you yield from coroutines and asyncio.Future instances—including tasks. But some APIs are tricky, mixing coroutines and plain functions in seemingly arbitrary ways, like the StreamWriter class we’ll use in one of the servers in the next section.  
    挑战在于知道使用yield from的时机与不能使用的时机。原则上的答案很简单，对协程，asyncio.Future实例（包括task）进行yield from。但一些API很棘手，他们以看似任意的方式将协程与普通函数混合在一起，就像是下一节我们将在一个服务用到的StremWriter类。

Example 18-13 wraps up the flags2 set of examples. I encourage you to play with them to develop an intuition of how concurrent HTTP clients perform. Use the -a, -e, and -l command-line options to control the number of downloads, and the -m option to set the number of concurrent downloads. Run tests against the LOCAL, REMOTE, DELAY, and ERROR servers. Discover the optimum number of concurrent downloads to maximize throughput against each server. Tweak the settings of the vaurien_error_delay.sh script to add or remove errors and delays.  
    例18-13总结了flags2示例集。我鼓励你使用他们直观地了解并发HTTP客户端是如何执行的。使用-a -e -l的命令行参数控制下载数量，-m选项来设置并发下载数。针对LOCAL、REMOTE、DELAY和ERROR服务器运行测试。发现并发下载的最佳数量，以最大限度地提高每台服务器的吞吐量。 调整 vaurien_error_delay.sh 脚本的设置以添加或删除错误和延迟。  

We’ll now go from client scripts to writing servers with asyncio.  
    我们现在从客户端脚本转到用asyncio写服务端。

## Writing asyncio Servers 编写asyncio服务

The classic toy example of a TCP server is an echo server. We’ll build slightly more interesting toys: Unicode character finders, first using plain TCP, then using HTTP. These servers will allow clients to query for Unicode characters based on words in their canonical names, using the unicodedata module we discussed in “The Unicode Database” on page 127. A Telnet session with the TCP character finder server, searching for chess pieces and characters with the word “sun” is shown in Figure 18-2.  
    TCP服务的经典玩具示例是回显服务。我们会略微构建更有趣的玩具：Unicode字符查找器，先使用普通的TCP，然后用HTTP。这些服务允许客户端使用我们在第127页的“Unicode 数据库”中讨论的 unicodedata模块，根据规范名称中的单词查询Unicode字符。图18-2显示了与TCP字符查找服务器的Telnet会话，搜索棋子和带有单词“sun”的字符。

Figure 18-2. A Telnet session with the tcp_charfinder.py server: querying for “chess black” and “sun”.  
    图18-2. tcp_charfinder.py服务的Telnet会话：查询“黑棋”和“太阳”。

Now, on to the implementations.  
    现在，接下来是实现。

### An asyncio TCP Server

Most of the logic in these examples is in the charfinder.py module, which has nothing concurrent about it. You can use charfinder.py as a command-line character finder, but more importantly, it was designed to provide content for our asyncio servers. The code for charfinder.py is in the Fluent Python code repository.  
    这个示例的大部分逻辑在charfinder.py模块，这里没有并发相关内容。你可以用charfinder.py作为一个命令行字符查找器，但更重要的是，他的目的是为我们的asyncio服务提供内容。charfinder.py的代码在Fluent Python代码库中。

The charfinder module indexes each word that appears in character names in the Unicode database bundled with Python, and creates an inverted index stored in a dict. For example, the inverted index entry for the key 'SUN' contains a set with the 10 Unicode characters that have that word in their names. The inverted index is saved in a local charfinder_index.pickle file. If multiple words appear in the query, charfinder computes the intersection of the sets retrieved from the index.  
    charfinder模块为出现在Python捆绑的Unicode数据库中字符名称中的每个单词做索引，并创建一个倒排索引保存在字典中。举个例子，关键词"SUN"的倒排索引包含了一组名称中包含该词的10个Unicode字符。倒排索引被保存在本地的charfinder_index.pickle文件中。如果多个单词出现在搜索中，charfinder会计算从索引中检索到的集合的交集。

We’ll now focus on the tcp_charfinder.py script that is answering the queries in Figure 18-2. Because I have a lot to say about this code, I’ve split it into two parts: Example 18-14 and Example 18-15.  
    我们现在来看tcp_charfinder.py脚本，这回答了图18-2的疑问。因为关于这个代码我有许多要说，我将其区分为示例18-14与18-15。

Example 18-14. tcp_charfinder.py: a simple TCP server using asyncio.start_server; code for this module continues in Example 18-15  
    例18-14. tcp_charfinder.py：一个使用asyncio.start_server的简单TCP服务；此模块的代码将在例18-15中继续

```python
import sys
import asyncio

from charfinder import UnicodeNameIndex  # 1

CRLF = b'\r\n'
PROMPT = b'?> '

index = UnicodeNameIndex()  # 2

@asyncio.coroutine
def handle_queries(reader, writer):  # 3
    while True:  # 4
        writer.write(PROMPT) # can't yield from!  # 5
        yield from writer.drain() # must yield from!  # 6
        data = yield from reader.readline()  # 7
        try:
            query = data.decode().strip()
        except UnicodeDecodeError:  # 8
            query = '\x00'
        client = writer.get_extra_info('peername')  # 9
        print('Received from {}: {!r}'.format(client, query))  # 10
        if query:
            if ord(query[:1]) < 32:  # 11
            break
        lines = list(index.find_description_strs(query))  # 12
        if lines:
            writer.writelines(line.encode() + CRLF for line in lines)  # 13
        writer.write(index.status(query, len(lines)).encode() + CRLF)  # 14

        yield from writer.drain()  # 15
        print('Sent {} results'.format(len(lines)))  # 16
 
    print('Close the client socket')  # 17
    writer.close()  # 18
```

1. UnicodeNameIndex is the class that builds the index of names and provides querying methods.  
    UnicodeNameIndex是构建了名称索引并提供查询方法的类。
2. When instantiated, UnicodeNameIndex uses charfinder_index.pickle, if available, or builds it, so the first run may take a few seconds longer to start.[8]  
    当实例化后，UnicodeNameIndex用到charfinder_index.pickle（如果可用）或构建它，所以首次运行可能会花费几秒才能开始。
3. This is the coroutine we need to pass to asyncio_startserver; the arguments received are an asyncio.StreamReader and an asyncio.StreamWriter.  
    这是我们需要传入asyncio_startserver的协程；收到的参数是一个asyncio.StreamReader与一个asyncio.StreamWriter。
4. This loop handles a session that lasts until any control character is received from the client.  
    这个循环处理一个会话，持续到从客户端收到任何控制符为止。
5. The StreamWriter.write method is not a coroutine, just a plain function; this line sends the ?> prompt.  
    StreamWriter.writer方法不是协程，只是个普通方法；这一行发送?>提示符。
6. StreamWriter.drain flushes the writer buffer; it is a coroutine, so it must be called with yield from.  
    StreamWriter.drain刷新writer的缓存；他是一个协程，所以必须用yield from来调用。
7. StreamWriter.readline is a coroutine; it returns bytes.  
    StreamWriter.readline是协程；他返回字节。
8. A UnicodeDecodeError may happen when the Telnet client sends control characters; if that happens, we pretend a null character was sent, for simplicity.  
    当Telnet客户端发送控制符时可能会发生UnicodeDecodeError；如果发生这种情况，为简单起见，我们假装接收到了一个空字符。
9. This returns the remote address to which the socket is connected.  
    这返回socket连接的远程地址。
10. Log the query to the server console.  
    记录对服务控制台的查询。
11. Exit the loop if a control or null character was received.  
    如果接受到控制符或空字符则退出循环。
12. This returns a generator that yields strings with the Unicode code point, the actual character and its name (e.g., U+0039\t9\tDIGIT NINE); for simplicity, I build a list from it.  
    这行返回一个生成器，生成包含Unicode码位置产出字符，实际的字符和他的名字的字符串（如 U+0039\t9\tDIGIT NINE）；简单起见，我用它构建了一个列表。
13. Send the lines converted to bytes using the default UTF-8 encoding, appending a carriage return and a line feed to each; note that the argument is a generator expression.  
    将用默认的UTF-8编码转为字节的一行发送出去，分别附加一个回车符和一个换行符；注意参数是一个生成器表达式。
14. Write a status line such as 627 matches for 'digit'.  
    写一个状态行，例如'digit'有627个匹配。
15. Flush the output buffer.  
    刷新输出缓存。
16. Log the response to the server console.  
    记录服务控制台的响应。
17. Log the end of the session to the server console.  
    记录本次到服务控制台会话的结尾。
18. Close the StreamWriter.  
    关闭StreamWriter。

[8]. Leonardo Rochael pointed out that building the UnicodeNameIndex could be delegated to another thread using loop.run_with_executor() in the main function of Example 18-15, so the server would be ready to take requests immediately while the index is built. That is true, but querying the index is the only thing this app does, so that would not be a big win. It’s an interesting exercise to do as Leo suggests, though. Go ahead and do it, if you like.  

Note that all I/O in Example 18-14 is in bytes. We need to decode the strings received from the network, and encode strings sent out. In Python 3, the default encoding is UTF-8, and that’s what we are using implicitly.  
    注意，例18-14的所有I/O都用的字节。我们需要将从network接收到的字符串进行解码，并对发出去的字符进行编码。Python3中默认编码为UTF-8，这也是我们隐式使用的编码。

One caveat is that some of the I/O methods are coroutines and must be driven with yield from, while others are simple functions. For example, StreamWriter.write is a plain function, on the assumption that most of the time it does not block because it writes to a buffer. On the other hand, StreamWriter.drain, which flushes the buffer and performs the actual I/O is a coroutine, as is Streamreader.readline. While I was writing this book, a major improvement to the asyncio API docs was the clear labeling of coroutines as such.  
    需要注意的是，一些IO方法是协程，需要用yield from驱动，而其他的是简单的函数。举个例子，StreamWriter.write是一个普通函数，假设大多是时间他都不会因为写入缓存而阻塞。另一方面，StreamWritter.drain，殊勋缓存并执行实际I/O的函数是协程，Streamreader.readline也是。当我在写这本书的时候，asyncio API文档的一个重大改进是协程的清晰标记。

Example 18-15 lists the main function for the module started in Example 18-14.  
    例18-15列出了例18-14中启动的模块的主要功能。

Example 18-15. tcp_charfinder.py (continued from Example 18-14): the main function sets up and tears down the event loop and the socket server  
    例18-15. tcp_charfinder.py（例18-14的延续）：main函数设置与删除事件循环和socket服务。

```python
def main(address='127.0.0.1', port=2323):  # 1
    port = int(port)
    loop = asyncio.get_event_loop()
    server_coro = asyncio.start_server(handle_queries, address, port,
                                loop=loop)  # 2
    server = loop.run_until_complete(server_coro)  # 3

    host = server.sockets[0].getsockname()  # 4
    print('Serving on {}. Hit CTRL-C to stop.'.format(host))  # 5
    try:
        loop.run_forever()  # 6
    except KeyboardInterrupt: # CTRL+C pressed
        pass

    print('Server shutting down.')
    server.close()  # 7
    loop.run_until_complete(server.wait_closed())  # 8
    loop.close()  # 9


if __name__ == '__main__':
    main(*sys.argv[1:])  # 10
```

1. The main function can be called with no arguments.  
    main函数可以不加参数进行调用。
2. When completed, the coroutine object returned by asyncio.start_server returns an instance of asyncio.Server, a TCP socket server.  
    当完成时，asyncio.start_server返回的协程对象会返回一个asyncio.Server（TCP socket服务）的实例。
3. Drive server_coro to bring up the server.  
    驱动server_coro来启动服务。
4. Get address and port of the first socket of the server and…  
    获取server的首个socket的地址与端口
5. …display it on the server console. This is the first output generated by this script on the server console.  
    然后将其显示在server控制台上。这是server控制台上该脚本的首次输出内容。
6. Run the event loop; this is where main will block until killed when CTRL-C is pressed on the server console.  
    运行事件循环；直到当server控制台输入CTRL-C kill掉服务前，main将阻塞在这里。
7. Close the server.  
    关闭server。
8. server.wait_closed() returns a future; use loop.run_until_complete to let the future do its job.  
    server.wait_closed()返回一个future；用loop.run_until_complete让future完成他的工作。
9. Terminate the event loop.  
    终止事件循环。
10. This is a shortcut for handling optional command-line arguments: explode sys.argv[1:] and pass it to a main function with suitable default arguments.  
    这是一个处理可选命令行参数的小技巧：分解sys.argv[1:]然后将其传入具有合适的默认参数的main函数。

Note how run_until_complete accepts either a coroutine (the result of start_server) or a Future (the result of server.wait_closed). If run_until_complete gets a coroutine as argument, it wraps the coroutine in a Task.  
    注意run_until_complete是如何接收了一个协程（start_server的结果）或一个Future（server.wait_closed的结果）的。如果run_until_complete收到协程作为参数，他会将其包装进Task。

You may find it easier to understand how control flows in tcp_charfinder.py if you take a close look at the output it generates on the server console, listed in Example 18-16.  
    如果你仔细看了列在例18-16中的服务控制台生成的输出，你可能会发现更容易理解tcp_charfinder.py的控制流。

Example 18-16. tcp_charfinder.py: this is the server side of the session depicted in Figure 18-2  
    例18-16. tcp_charfinder.py：这是图18-2描绘的会话服务端

```
$ python3 tcp_charfinder.py
Serving on ('127.0.0.1', 2323). Hit CTRL-C to stop.  # 1
Received from ('127.0.0.1', 62910): 'chess black'  # 2
Sent 6 results
Received from ('127.0.0.1', 62910): 'sun'  # 3
Sent 10 results
Received from ('127.0.0.1', 62910): '\x00'  # 4
Close the client socket  # 5
```

1. This is output by main.  
    main的输出。
2. First iteration of the while loop in handle_queries.  
    在handle_queriers中的while循环的第一次迭代。
3. Second iteration of the while loop.  
    while循环的第二次迭代。
4. The user hit CTRL-C; the server receives a control character and closes the session.  
    用户按下了CTRL-C；服务接收到控制符并关闭该会话。
5. The client socket is closed but the server is still running, ready to service another client.  
    客户端socket关闭，但服务仍在运行准备为其他客户端提供服务。

Note how main almost immediately displays the Serving on... message and blocks in the loop.run_forever() call. At that point, control flows into the event loop and stays there, occasionally coming back to the handle_queries coroutine, which yields control back to the event loop whenever it needs to wait for the network as it sends or receives data. While the event loop is alive, a new instance of the handle_queries coroutine will be started for each client that connects to the server. In this way, multiple clients can be handled concurrently by this simple server. This continues until a KeyboardInterrupt occurs or the process is killed by the OS.  
    注意main是如何几乎立刻显示了Serving on...信息和loop.run_forever()调用中的块。此时，控制流进入事件循环并停留在这，偶尔返回至handle_queries协程，当他需要等待网络发送或接受数据时，将控制权交还给事件循环。当事件循环活动时，将为每个连接server的client启动handle_queries协程的新实例。就这样，多个客户端可以被这个简单服务并行处理。该协程持续到KeyboardInterrup发生或OS关闭进程。

The tcp_charfinder.py code leverages the high-level asyncio Streams API that provides a ready-to-use server so you only need to implement a handler function, which can be a plain callback or a coroutine. There is also a lower-level Transports and Protocols API, inspired by the transport and protocols abstractions in the Twisted framework.  
    tcp_charfinder.py代码利用了高级asyncio Stream API，这提供了一个即用型服务，你仅需要实现一个控制器函数，这可以是一个普通的回调或协程。还有一个低级的Transports和Protocols API，灵感来自于Twisted框架中的传输与协议的抽象。

Refer to the asyncio Transports and Protocols documentation for more information, including a TCP echo server implemented with that lower-level API.  
    参考asyncio Transports与Protocols的演示来了解更多，包括使用那个低级API实现的TCP回声服务。

The next section presents an HTTP character finder server.  
    下一节介绍HTTP字符查找服务。

### An aiohttp Web Server

The aiohttp library we used for the asyncio flags examples also supports server-side HTTP, so that’s what I used to implement the http_charfinder.py script. Figure 18-3 shows the simple web interface of the server, displaying the result of a search for a “catface” emoji.  
    我们用于asyncio flags示例的aiohttp库同样支持服务器端HTTP，所以我用他实现http_charfinder.py脚本。图示18-3展示了该服务简单的web接口，显示的是“catface” emoji的搜索结果。

Figure 18-3. Browser window displaying search results for “cat face” on the http_charfinder.py server  
    图示18-3. 浏览器窗口显示http_charfinder.py服务上“cat face”的搜索结果。

    Some browsers are better than others at displaying Unicode. The screenshot in Figure 18-3 was captured with Firefox on OS X, and I got the same result with Safari. But up-to-date Chrome and Opera browsers on the same machine did not display emoji characters like the cat faces. Other search results (e.g., “chess”) looked fine, so it’s likely a font issue on Chrome and Opera on OSX.  
    一些浏览器在显示Unicode方面更好。18-3的截图是抓捕的OS X上的Firefox，我在Safari上获取了相同的结果。但同一机器上的最新版的Chrome与Opera浏览器不能显示像猫脸的emoji字符。其他搜索结果（如chess）看起来很好，所以这像是OSX上的Chrome和Opera的字体问题。

We’ll start by analyzing the most interesting part of http_charfinder.py: the bottom half where the event loop and the HTTP server is set up and torn down. See Example 18-17.  
    我们通过分析http_charfinder.py最有趣的部分来开始：事件循环与HTTP服务设置和拆除的下半部分。来看示例18-17。

Example 18-17. http_charfinder.py: the main and init functions  
    示例18-17. http_charfinder.py：main函数与init函数

```python

@asyncio.coroutine
def init(loop, address, port):  # 1
    app = web.Application(loop=loop)  # 2
    app.router.add_route('GET', '/', home)  # 3
    handler = app.make_handler()  # 4
    server = yield from loop.create_server(handler,
                                           address, port)  # 5
    
    return server.sockets[0].getsockname()  # 6


def main(address="127.0.0.1", port=8888):
    port = int(port)
    loop = asyncio.get_event_loop()
    host = loop.run_until_complete(init(loop, address, port))  # 7
    print('Serving on {}. Hit CTRL-C to stop.'.format(host))
    try:
        loop.run_forever()  # 8
     except KeyboardInterrupt: # CTRL+C pressed
        pass
    print('Server shutting down.')
    loop.close()  # 9


if __name__ == '__main__':
    main(*sys.argv[1:])
```

1. The init coroutine yields a server for the event loop to drive.  
    init协程产生一个服务为事件循环驱动。
2. The aiohttp.web.Application class represents a web application…  
    aiohttp.webApplication类代表一个web应用…
3. …with routes mapping URL patterns to handler functions; here GET / is routed to the home function (see Example 18-18).  
    …其具有将URL模式映射至控制函数的路由；这里的GET / 被路由至home函数（参考例18-18）。
4. The app.make_handler method returns an aiohttp.web.RequestHandler instance to handle HTTP requests according to the routes set up in the app object.  
    app.make_hanler方法返回了一个aiohttp.web.RequestHandler实例，用于根据app object中设置的路由处理HTTP请求。
5. create_server brings up the server, using handler as the protocol handler and binding it to address and port.  
    create_server启动服务，将handler用作协议处理器并绑定地址与端口。
6. Return the address and port of the first server socket.  
    返回首个服务socket的地址与端口。
7. Run init to start the server and get its address and port.  
    运行init以启动服务，并获取其地址和端口。
8. Run the event loop; main will block here while the event loop is in control.  
    运行事件循环；当事件循环处于控制状态时main将在这里阻塞。
9. Close the event loop.  
    关闭事件循环。

As you get acquainted with the asyncio API, it’s interesting to contrast how the servers are set up in Example 18-17 and in the TCP example (Example 18-15) shown earlier.  
    当你开始熟悉asyncio API时，对比示例18-17和前面的TCP示例（18-15）中服务的设置方式是很有趣的。

In the earlier TCP example, the server was created and scheduled to run in the main function with these two lines:  
    在之前的TCP示例汇总，服务被创建并定于在main函数的这个这两行运行：

    server_coro = asyncio.start_server(handle_queries, address, port,
                                       loop=loop)
    server = loop.run_until_complete(server_coro)

In the HTTP example, the init function creates the server like this:  
    在HTTP示例中，init函数创建服务如下：

    server = yield from loop.create_server(handler,
                                           address, port)
But init itself is a coroutine, and what makes it run is the main function, with this line:  
    但init自身是个协程，但让他运行起来的是main函数，用下面这行：

    host = loop.run_until_complete(init(loop, address, port))

Both asyncio.start_server and loop.create_server are coroutines that return asyncio.Server objects. In order to start up a server and return a reference to it, each of these coroutines must be driven to completion. In the TCP example, that was done by calling loop.run_until_complete(server_coro), where server_coro was the result of asyncio.start_server. In the HTTP example, create_server is invoked on a yield_from expression inside the init coroutine, which is in turn driven by the main function when it calls loop.run_until_complete(init(...)).  
    asyncio.start_server与loop.create_server都是协程，他们都返回asyncio.Server对象。为了启动服务并返回对他的引用，这些协程的每一个必须被完成。在TCP示例中，是通过调用loop.run_until_complete(server_coro)完成的，这里的server_coro是asyncio.start的结果。在HTTP示例中，create_server被init协程中的yield from表达式调用，当他调用loop.run_until_complete(init(...))时，他又由main函数驱动。

I mention this to emphasize this essential fact we've discussed before: a coroutine only does anything when driven, and to drive an asyncio.coroutine you either use yield from or pass it to one of several asyncio functions that take coroutine or future arguments, such as run_until_complete.  
    我提到这个是为了强调我们之前讨论的基本事实：协程只能在被驱动的时候做任何事情，为了驱动asyncio.coroutine，要么使用yield from，要么将他传入各种接收协程或future参数的asyncio方法中的某一个，像是run_until_complete。

Example 18-18 shows the home function, which is configured to handle the / (root) URL in our HTTP server.  
    例18-18展示了home方法，他被配置为去处理HTTP服务中的 / (root) URL。

Example 18-18. http_charfinder.py: the home function

```python
def home(request):  # 1
    query = request.GET.get('query', '').strip()  # 2
    print('Query: {!r}'.format(query))  # 3
    if query:  # 4
        descriptions = list(index.find_descriptions(query))
        res = '\n'.join(ROW_TPL.format(**vars(descr))
                        for descr in descriptions)
        msg = index.status(query, len(descriptions))
    else:
        descriptions = []
        res = ''
        msg = 'Enter words describing characters.'
 
    html = template.format(query=query, result=res,  # 5
                           message=msg)
    print('Sending {} results'.format(len(descriptions)))  # 6
    return web.Response(content_type=CONTENT_TYPE, text=html)  # 7
```

1. A route handler receives an aiohttp.web.Request instance.  
    接收aiohttp.web.Request实例的路由处理器。
2. Get the query string stripped of leading and trailing blanks.  
    获取去掉首尾空格的查询字符串。
3. Log query to server console.  
    将查询记录至服务端控制台。
4. If there was a query, bind res to HTML table rows rendered from result of the query to the index, and msg to a status message.  
    如果有查询，将返回值绑定至从查询结果呈现到索引的HTML table行，并将msg绑定至状态消息。
5. Render the HTML page. 
    渲染HTML页面。
6. Log response to server console.  
    将相应记录至服务控制台。
7. Build Response and return it.  
    建立相应并返回出去。

Note that home is not a coroutine, and does not need to be if there are no yield from expressions in it. The aiohttp documentation for the add_route method states that the handler “is converted to coroutine internally when it is a regular function.”  
    注意，home不是协程，如果其中没有yield from表达式就不需要是协程。add_route方法的aiohttp文档将hander陈述为“当他是一个常规函数时，在内部被转化为协程。”

There is a downside to the simplicity of the home function in Example 18-18. The fact that it’s a plain function and not a coroutine is a symptom of a larger issue: the need to rethink how we code web applications to achieve high concurrency. Let’s consider this matter.  
    示例18-18中的home函数的简单性也有不利的一面。事实上他就是一个普通函数，而不是协程，这是一个更大问题的征兆：需要重新考虑我们怎样编写应对高并发的web应用。让我们考虑下这个问题。
