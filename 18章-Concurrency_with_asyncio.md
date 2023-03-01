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

