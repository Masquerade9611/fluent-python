# Chapter 17 Concurrcy with Futures
# 第17章 使用future处理并发

*The people bashing threads are typically system programmers which have in mind use cases that the typical application programmer will never encounter in her life. […] In 99% of the use cases an application programmer is likely to run into, the simple pattern of spawning a bunch of independent threads and collecting the results in a queue is everything one needs to know.   — Michele Simionato Python deep thinker[1]*  
*抨击线程的人群一般来讲是系统程序员，他们思考的用例是通常的应用程序员一生都无法遇到的。... 在应用程序员很可能碰到的99%的用例中，“生成一堆独立线程并在队列中收集结果”这种简单的模式是他们需要知道的一切。    ——Michele Simonato Python思想家[1]*  

[1] From Michele Simionato’s post Threads, processes and concurrency in Python: some thoughts, subtitled “Removing the hype around the multicore (non) revolution and some (hopefully) sensible comment about threads and other forms of concurrency.”  
    [1] 摘自Michele Simionato的博文《Python的线程，进程与并发：一些想法》，副标题为：“去除关于多核(非)革命的炒作，以及一些(希望)关于线程和其他并发形式的合理评论。”  

This chapter focuses on the concurrent.futures library introduced in Python 3.2, but also available for Python 2.5 and newer as the futures package on PyPI. This library encapsulates the pattern described by Michele Simionato in the preceding quote, making it almost trivial to use.  
    本章重点介绍Python3.2引入的current.futures库，但在作为PyPI的futures包也支持Python 2.5及更新的版本。该库封装了Michele Simionato在前言中描述的模式，使其使用起来非常简单。

Here I also introduce the concept of “futures”—objects representing the asynchronous execution of an operation. This powerful idea is the foundation not only of concurrent.futures but also of the asyncio package, which we’ll cover in Chapter 18.  
    这里我也会介绍“futures”的概念——表示一个动作异步执行的对象。这个强大的想法是不仅是concurrent.futures，也是18章将会介绍的asyncio包的基础。

We’ll start with a motivating example.  
    我们来用一个激动人心的例子开始本章内容。

## 17.1 Example: Web Downloads in Three Styles
## 17.1 示例： 三种风格的Web下载

To handle network I/O efficiently, you need concurrency, as it involves high latency—so instead of wasting CPU cycles waiting, it’s better to do something else until a response comes back from the network.  
    为了高效地处理网络I/O，你需要并发，因为它涉及高延迟——所以与其将CPU循环等待的时间浪费掉，不如在网络的响应回来之前来做一些其他的事。

To make this last point with code, I wrote three simple programs to download images of 20 country flags from the Web. The first one, flags.py, runs sequentially: it only requests the next image when the previous one is downloaded and saved to disk. The other two scripts make concurrent downloads: they request all images practically at the same time, and save the files as they arrive. The flags_threadpool.py script uses the concurrent.futures package, while flags_asyncio.py uses asyncio.  
    为了在代码中实现上述的功能，我写了三个简单的程序，用来从网页下载20个国家国旗的图片。第一个flags.py，有序运行：仅仅是当上一张图下载完成并存入硬盘后，开始请求下一张图片。另外两种脚本做到了并发下载：他们几乎同时请求所有图片，接收后保存。脚本flags_threadpool.py使用了concurrent.futures包，而flags_asyncio.py使用了asyncio。

Example 17-1 shows the result of running the three scripts, three times each. I also posted a 73s video on YouTube so you can watch them running while an OS X Finder window displays the flags as they are saved. The scripts are downloading images from flupy.org, which is behind a CDN, so you may see slower results in the first runs. The results in Example 17-1 were obtained after several runs, so the CDN cache was warm.  
    示例17-1展示这三种脚本的运行结果，每一种运行三次。我也在YouTube上传了一个73秒的视频，所以你可以在OS X Finder窗口观看他们的运行情况。脚本是从flupy.org中下载图片，他基于CDN，所以在首次运行时你可能要晚一些才能看到结果。示例17-1的结果是经过多次运行得到的，所以CDN缓存是热乎的。（？）

Example 17-1. Three typical runs of the scripts flags.py, flags_threadpool.py, and flags_asyncio.py  
    示例17-1、三种典型的运行：flags.py, flags_threadpool.py，flags_asyncio.py

```python
$ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN  # 1
20 flags downloaded in 7.26s  # 2
$ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN
20 flags downloaded in 7.20s
$ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN
20 flags downloaded in 7.09s
$ python3 flags_threadpool.py
DE BD CN JP ID EG NG BR RU CD IR MX US PH FR PK VN IN ET TR
20 flags downloaded in 1.37s  # 3
$ python3 flags_threadpool.py
EG BR FR IN BD JP DE RU PK PH CD MX ID US NG TR CN VN ET IR
20 flags downloaded in 1.60s
$ python3 flags_threadpool.py
BD DE EG CN ID RU IN VN ET MX FR CD NG US JP TR PK BR IR PH
20 flags downloaded in 1.22s 
$ python3 flags_asyncio.py  # 4
BD BR IN ID TR DE CN US IR PK PH FR RU NG VN ET MX EG JP CD
20 flags downloaded in 1.36s
$ python3 flags_asyncio.py
RU CN BR IN FR BD TR EG VN IR PH CD ET ID NG DE JP PK MX US
20 flags downloaded in 1.27s
$ python3 flags_asyncio.py
RU IN ID DE BR VN PK MX US IR ET EG NG BD FR CN JP PH CD TR  # 5
20 flags downloaded in 1.42s

```

1. The output for each run starts with the country codes of the flags as they are downloaded, and ends with a message stating the elapsed time.  
    每次运行的输出都以下载的国旗的国家代码为开始，并以为经历的时间为结束。
2. It took flags.py an average 7.18s to download 20 images.  
    flags.py下载20张图片平均花费7.18秒。
3. The average for flags_threadpool.py was 1.40s.  
    flags_threadpool.py平均时间为1.40秒。
4. For flags_asyncio.py, 1.35 was the average time.  
    对flags_asyncio.py来说，平均时间为1.35。
5. Note the order of the country codes: the downloads happened in a different order every time with the concurrent scripts.  
    注意国家代码的顺序：在并发脚本中的每次下载为不同顺序。

The difference in performance between the concurrent scripts is not significant, but they are both more than five times faster than the sequential script—and this is just for a fairly small task. If you scale the task to hundreds of downloads, the concurrent scripts can outpace the sequential one by a factor or 20 or more.  
    并发脚本间的性能差异并不明显，但他们都要比顺序脚本要快5秒——并且这只是一个相当小的任务。如果你将该任务分配至数百次的下载，那么并发脚本的速度可以超过顺序脚本的1/20甚至更多。

    While testing concurrent HTTP clients on the public Web you may inadvertently launch a denial-of-service (DoS) attack, or be suspected of doing so. In the case of Example 17-1, it’s OK to do it because those scripts are hardcoded to make only 20 requests. For testing nontrivial HTTP clients, you should set up your own test server. The 17-futures/countries/README.rst file in the Fluent Python code GitHub repository has instructions for setting a local Nginx server.  
    当在公共Web上测试并发HTTP客户端时，你可能会无意间发起一个拒绝服务（denial-of-service, DoS）攻击，或者被怀疑做了这种事。示例17-1可行是因为那些脚本时硬编码，仅仅发出20次请求。为了测试不重要的HTTP客户端，你应该配置你自己的服务。在Fluent Python代码GitHub库中的17-futures/countries/README.rst文件介绍了有关配置本地Nginx服务的内容。

Now let’s study the implementations of two of the scripts tested in Example 17-1: flags.py and flags_threadpool.py. I will leave the third script, flags_asyncio.py, for Chapter 18, but I wanted to demonstrate all three together to make a point: regardless of the concurrency strategy you use—threads or asyncio—you’ll see vastly improved throughput over sequential code in I/O-bound applications, if you code it properly.  
    现在我们开始学习17-1的两个脚本（flags.py和flags_threadpool.py）的实现。为了18章我先不介绍第三个脚本flags_asyncio.py，但我想要一起演示所有三种脚本 是为了说明：不管你是用的并发策略——线程或asyncio——只要你编写得合理，你将会看到在绑定IO的应用上，吞吐量相比于顺序代码有着非常大的提高。

On to the code.  
    接下来是代码。

### 17.1.1 A Sequential Download Script
## 17.1.1 顺序下载的脚本
Example 17-2 is not very interesting, but we’ll reuse most of its code and settings to implement the concurrent scripts, so it deserves some attention.  
   示例17-2并没有很有趣，但是我们会将这版代码中的大部分进行复用，并设置为来执行并发脚本，所以这值得关注
。

    For clarity, there is no error handling in Example 17-2. We will deal with exceptions later, but here we want to focus on the basic structure of the code, to make it easier to contrast this script with the concurrent ones.
    为了清楚起见，示例17-2中没有处理错误。我们将在之后处理异常，单在这里我们想要聚焦于代码的基础结构，来更加方便对比此脚本与并发脚本。


Example 17-2. flags.py: sequential download script; some functions will be reused by the other scripts  
    示例17-2 flags.py：顺序下载脚本；其中一些方法会被其他脚本复用。
```python
import os
import time
import sys

import requests  # 1

POP20_CC = ('CN IN US ID BR PK NG BD RU JP '
            'MX PH VN ET EG DE IR TR CD FR').split()  # 2

BASE_URL = 'http://flupy.org/data/flags'  # 3

DEST_DIR = 'downloads/'  # 4


def save_flag(img, filename):  # 5
    path = os.path.join(DEST_DIR, filename)
    with open(path, 'wb') as fp:
        fp.write(img)


def get_flag(cc):  # 6
    url = '{}/{cc}/{cc}.gif'.format(BASE_URL, cc=cc.lower())
    resp = requests.get(url)
    return resp.content


def show(text):  # 7
    print(text, end=' ')
    sys.stdout.flush()


def download_many(cc_list):  # 8
    for cc in sorted(cc_list):  # 9
        image = get_flag(cc)
        show(cc)
        save_flag(image, cc.lower() + '.gif')

    return len(cc_list)


def main(download_many):  # 10
    t0 = time.time()
    count = download_many(POP20_CC)
    elapsed = time.time() - t0
    msg = '\n{} flags downloaded in {:.2f}s'
    print(msg.format(count, elapsed))


if __name__ == '__main__':
    main(download_many)  # 11

```

1. Import the requests library; it’s not part of the standard library, so by convention we import it after the standard library modules os, time, and sys, and separate it from them with a blank line.  
    引入request库；这不是标准库的一部分，所以按照惯例我们在标准库模块os，time和sys之后引入它，并且用一个空行来分隔他们。
2. List of the ISO 3166 country codes for the 20 most populous countries in order of decreasing population.  
    20个人口最多的国家的ISO 3166国家代码，以人口数递减排序来列出的清单。
3. The website with the flag images.[2]  
    国旗图像的网站。[2]
4. Local directory where the images are saved.  
    保存这些图片的本地目录。
5. Simply save the img (a byte sequence) to filename in the DEST_DIR.  
    只需将img（一个字节序列）保存到DEST_DIR中的文件名中。
6. Given a country code, build the URL and download the image, returning the binary contents of the response.  
    传入国家代码，建立URL并下载此图片，返回此请求响应的二进制内容。
7. Display a string and flush sys.stdout so we can see progress in a one-line display; this is needed because Python normally waits for a line break to flush the stdout buffer.   
    显示该字符串并刷新sys.stdout，所以我们可以在一行显示中看到进度；这是必要的，因为Python通常会等待一行返回来刷新stdout缓存。
8. download_many is the key function to compare with the concurrent implementations.  
    download_many是用来对比并发执行情况的关键函数。
9. Loop over the list of country codes in alphabetical order, to make it clear that the ordering is preserved in the output; return the number of country codes downloaded.  
    遍历以字母顺序排序的国家代码列表，来使输出以顺序保存，便于观察；返回已下载的国家代码的数量。
10. main records and reports the elapsed time after running download_many.  
    main函数记录并报告了在运行download_many之后经历的时间。
11. main must be called with the function that will make the downloads; we pass the download_many function as an argument so that main can be used as a library function with other implementations of download_many in the next examples.  
    main函数必须被那些进行下载的函数调用；我们将download_many函数作为一个参数传入，以便main函数可以被用于一个库方法，可以被之后的示例中其他download_many的实现进行调用。

[2]The images are originally from the CIA World Factbook, a public-domain, U.S. government publication. I copied them to my site to avoid the risk of launching a DOS attack on CIA.gov.  
    [2]这些图片根源出处是CIA World Facbook，一个公共域，美国政府出版物。我把它们拷贝至我的网站来避免对CIA.gov进行DOS攻击。
*
    The requests library by Kenneth Reitz is available on PyPI and is more powerful and easier to use than the urllib.request module from the Python 3 standard library. In fact, requests is considered a model Pythonic API. It is also compatible with Python 2.6 and up, while the urllib2 from Python 2 was moved and renamed in Python 3, so it’s more convenient to use requests regardless of the Python version you’re targeting.*
    Kenneth Reitz的requests库可以在PyPI上使用，同时相比于Python3标准库的urllib.request，它会更强大与简单。事实上，requests被认为是一个Pythonic的API。它也兼容Python2.6或更高，而Python2的urllib2在Python3中被修改并重命名，所以无论你的Python版本是什么，使用requests都会更方便。

There’s really nothing new to flags.py. It serves as a baseline for comparing the other scripts and I used it as a library to avoid redundant code when implementing them. Now let’s see a reimplementation using concurrent.futures.  
    flags.py其实没有什么新鲜的。它作为一个用来和其他脚本对比的基准，我将它作为一个库来避免完成他们时的重复代码。现在我们来看看使用concurrent.futures的重新实现。

### 17.1.2 Downloading with concurrent.futures
### 17.1.2 使用concurrent.futures下载

The main features of the concurrent.futures package are the ThreadPoolExecutor and ProcessPoolExecutor classes, which implement an interface that allows you to submit callables for execution in different threads or processes, respectively. The classes manage an internal pool of worker threads or processes, and a queue of tasks to be executed. But the interface is very high level and we don’t need to know about any of those details for a simple use case like our flag downloads.  
    concurrent.futures包的主要功能特点是ThreadPoolExecutor和ProcessPoolExecutor类，这些类完成了一种接口，来支持你分别在不同线程或进程中提交可执行对象的执行。这些类管理工作线程或进程的内部池，以及要执行的一个队列的任务。但这个接口很高级，对于像我们的flag下载这种简单用例不需要知道它的所有细节。

Example 17-3 shows the easiest way to implement the downloads concurrently, using the ThreadPoolExecutor.map method.  
    示例17-3展示了并发下载实现的最简单方式，通过使用ThreadPoolExecutor.map方法。

Example 17-3. flags_threadpool.py: threaded download script using futures.ThreadPoolExecutor  
    示例17-3. flags_threadpool.py：使用futures.ThreadPoolExecutor的线程下载脚本
```python
from concurrent import futures

from flags import save_flag, get_flag, show, main  # 1

MAX_WORKERS = 20  # 2


def download_one(cc):  # 3
    image = get_flag(cc)
    show(cc)
    save_flag(image, cc.lower() + '.gif')
    return cc


def download_many(cc_list):  # 4
    workers = min(MAX_WORKERS, len(cc_list)) 
    with futures.ThreadPoolExecutor(workers) as executor:  # 5
        res = executor.map(download_one, sorted(cc_list))  # 6
    
    return len(list(res))  # 7


if __name__ == '__main__':
    main(download_many)  # 8

```
1. Reuse some functions from the flags module (Example 17-2).  
    对示例17-2 flags模块一些方法的复用。
2. Maximum number of threads to be used in the ThreadPoolExecutor.  
    在ThreadPoolExecutor用到的最大线程数。
3. Function to download a single image; this is what each thread will execute.  
    下载一张单独图像的函数；这是每个线程将执行的内容。
4. Set the number of worker threads: use the smaller number between the maximum we want to allow (MAX_WORKERS) and the actual items to be processed, so no unnecessary threads are created.  
    设置工作线程的数量：在我们想要允许的最大值（MAX_WORKERS）和实际需要处理的项目数这二者间选择较小的一方，来避免创建不必要的线程。
5. Instantiate the ThreadPoolExecutor with that number of worker threads; the executor.__exit__ method will call executor.shutdown(wait=True), which will block until all threads are done.  
    使用工作线程的数量实例化ThreadPoolExecutor；executor.__exit__方法将会调用executor.shutdown(wait=True)，效果是在所有线程都执行结束前会阻塞住。
6. The map method is similar to the map built-in, except that the download_one function will be called concurrently from multiple threads; it returns a generator that can be iterated over to retrieve the value returned by each function.  
    map方法类似于内建的map，不同的是download_one函数将会被多个线程并发地调用；它会返回一个生成器，遍历后来检测每个方法的返回值。
7. Return the number of results obtained; if any of the threaded calls raised an exception, that exception would be raised here as the implicit next() call tried to retrieve the corresponding return value from the iterator.  
    返回获取到的结果数量；如果任意线程的调用抛出了异常，那么当隐式的next()调用尝试从迭代器中检索相应的返回值时，在这里会引发异常。
8. Call the main function from the flags module, passing the enhanced version of download_many.  
    调用flags模块的main函数，传递加强版的download_many。

Note that the download_one function from Example 17-3 is essentially the body of the for loop in the download_many function from Example 17-2. This is a common refactoring when writing concurrent code: turning the body of a sequential for loop into a function to be called concurrently.  
    注意，示例17-3的download_one函数本质上是17-2中download_many函数的for循环体。写并发代码时经常这样重构：把依序执行的for循环体改成一个函数，以便进行并发调用。

The library is called concurrency.futures yet there are no futures to be seen in Example 17-3, so you may be wondering where they are. The next section explains.  
    库名叫做concurrency.futures，然而示例17-3中没有交到futures，所以你可能想要知道他们在哪。下一节会进行解释。

### 17.1.3 Where Are the Futures?
### 17.1.3 Futures在哪里？

Futures are essential components in the internals of concurrent.futures and of asyncio, but as users of these libraries we sometimes don’t see them. Example 17-3 leverages futures behind the scenes, but the code I wrote does not touch them directly. This section is an overview of futures, with an example that shows them in action.  
    Futures是concurrent.futures和asyncio内部的重要组成部分，但作为库的使用者来讲，我们有些时候不能直观地看到他们。示例17-3在背后用到的futures，但是编写的代码并没有直接接触到他们。这一节是futures的概述，会用一个例子来介绍。

As of Python 3.4, there are two classes named Future in the standard library: concurrent.futures.Future and asyncio.Future. They serve the same purpose: an instance of either Future class represents a deferred computation that may or may not have completed. This is similar to the Deferred class in Twisted, the Future class in Tornado, and Promise objects in various JavaScript libraries.  
    从Python 3.4起，标准库中被命名为Future的类有两个：concurrent.futures.Future和asyncio.Future。他们的作用相同：任一个Future类的实例代表着延迟的计算——计算可能已经完成了，也可能没有完成。这有许多类似的实现，如Twisted中的Deferred类，Tornado中的Future类和各种JavaScript库中的Promise对象。

Futures encapsulate pending operations so that they can be put in queues, their state of completion can be queried, and their results (or exceptions) can be retrieved when available.  
    Futrues封装了待完成的操作，这些操作被放置于队列中，其完成状态可以被查询，并且他们的结果（或异常）可以被获取到。

An important thing to know about futures in general is that you and I should not create them: they are meant to be instantiated exclusively by the concurrency framework, be it concurrent.futures or asyncio. It’s easy to understand why: a Future represents something that will eventually happen, and the only way to be sure that something will happen is to schedule its execution. Therefore, concurrent.futures.Future instances are created only as the result of scheduling something for execution with a concurrent.futures.Executor subclass. For example, the Executor.submit() method takes a callable, schedules it to run, and returns a future.  
    关于futures通常我们需要知道的是 你我不应该去创建他们：他们应该仅仅由并发框架来进行实例化，像concurrent.futures或asyncio。这很容易理解：Furure代表着一些终究要发生的事情，而且确定它会发生的唯一方式就是安排它的执行。因此，concurrent.futures.Future的实例仅被规划执行事件（通过concurrent.futures.Executor子类）的结果所创建。举个例子，Executor.submit()方法需要一个调用对象，规划它去运行时期，最后返回一个futures。

Client code is not supposed to change the state of a future: the concurrency framework changes the state of a future when the computation it represents is done, and we can’t control when that happens.  
    客户端代码不应该改变future的状态：当它表示的计算完成时并发框架会改变future的状态，而我们无法控制计算何时结束。

Both types of Future have a .done() method that is nonblocking and returns a Boolean that tells you whether the callable linked to that future has executed or not. Instead of asking whether a future is done, client code usually asks to be notified. That’s why both Future classes have an .add_done_callback() method: you give it a callable, and the callable will be invoked with the future as the single argument when the future is done.  
    Future的所有类型都有.done()方法，该方法不阻塞且返回一个布尔值来通知你future连接的回调对象是否已执行完成。客户端代码通常不会询问future是否运行结束，而是会等待通知。因此，这两个Future类都有.add_done_callback()方法：你给他一个可调用对象，future运行结束后会调用这个可调用对象。

There is also a .result() method, which works the same in both classes when the future is done: it returns the result of the callable, or re-raises whatever exception might have been thrown when the callable was executed. However, when the future is not done, the behavior of the result method is very different between the two flavors of Future. In a concurrency.futures.Future instance, invoking f.result() will block the caller’s thread until the result is ready. An optional timeout argument can be passed, and if the future is not done in the specified time, a TimeoutError exception is raised. In “asyncio.Future: Nonblocking by Design” on page 545, we’ll see that the asyncio.Future.result method does not support timeout, and the preferred way to get the result of futures in that library is to use yield from—which doesn’t work with concurrency.futures.Future instances.  
    此外还有.result()方法，当future完成时，该方法在这两个类中作用相同：返回调用对象的结果或重新引发调用对象执行时抛出的异常。但是，当future没有结束时，result方法的行为在上述两类中不同。在concurrency.futures.Future实例中，调用f.result()会阻塞调用方的线程，直到有结果返回。可传入一个可选的timeout参数，如果future在指定时间内没有完成，就引发TimoutError异常。在“asyncio.Future：为非阻塞而设计”的545页中，我们将看到asyncio.Future.result方法不支持timeout，在那个库中获取future的结果更好方式是使用yield from——这对concurrency.futures.Future实例不支持。

Several functions in both libraries return futures; others use them in their implementation in a way that is transparent to the user. An example of the latter is the Executor.map we saw in Example 17-3: it returns an iterator in which __next__ calls the result method of each future, so what we get are the results of the futures, and not the futures themselves.  
    两个库中的几个方法都返回future；而其他函数则使用future，以用户易于理解的方式来实现自己。我们在示例17-3看到的Executor.map属于后者：他返回一个迭代器，其__next__调用各个future的result方法，所以我们获取到的是future的结果，而不是future他们自身。

To get a practical look at futures, we can rewrite Example 17-3 to use the concurrent.futures.as_completed function, which takes an iterable of futures and returns an iterator that yields futures as they are done.  
    为了从实用角度理解future，我们可以重写示例17-3来使用concurrent.futures.as_completed方法，传入一个包含future对象的可迭代对象，返回一个迭代器用来产出已经完成的future。

Using futures.as_completed requires changes to the download_many function only. The higher-level executor.map call is replaced by two for loops: one to create and schedule the futures, the other to retrieve their results. While we are at it, we’ll add a few print calls to display each future before and after it’s done. Example 17-4 shows the code for a new download_many function. The code for download_many grew from 5 to 17 lines, but now we get to inspect the mysterious futures. The remaining functions are the same as in Example 17-3.  
    使用futures.as_completed只需要修改download_many方法。更高级的executor.map调用被以下两个循环替代：一个用来创建并安排future，另一个来验证future的结果。当我们这样做时，将在每个future的完成前后加入print进行显示。示例17-4展示了新的download_many函数的代码。download_many代码行数从5增加到17，但是现在我们先去看看神秘的future。其余的函数和17-3中一样。

Example 17-4. flags_threadpool_ac.py: replacing executor.map with executor.submit and futures.as_completed in the download_many function  
    示例17-4. flags_threadpool_ac.py：使用download_many函数中的executor.submit和futures.as_completst替换了executor.map

```python
def download_many(cc_list):
    cc_list = cc_list[:5]  # 1
    with futures.ThreadPoolExecutor(max_workers=3) as executor:  # 2
        to_do = []
        for cc in sorted(cc_list):  # 3
            future = executor.submit(download_one, cc)  # 4
            to_do.append(future)  # 5
            msg = 'Scheduled for {}: {}'
            print(msg.format(cc, future))  # 6

        results = []
        for future in futures.as_completed(to_do):  # 7
            res = future.result()  # 8
            msg = '{} result: {!r}'
            print(msg.format(future, res))  # 9
            results.append(res)

    return len(results)
```

1. For this demonstration, use only the top five most populous countries.
    为了演示，仅使用了前五个人口最多的国家
2. Hardcode max_workers to 3 so we can observe pending futures in the output.  
    将max_worker固定为3，所以我们可以在输出中观察到pending的futures
3. Iterate over country codes alphabetically, to make it clear that results arrive out of order.  
    按字母顺序遍历国家代码，以明确结果为无序
4. executor.submit schedules the callable to be executed, and returns a future representing this pending operation.  
    executor.submit规划了被执行的待调用对象，并返回了表示该pending操作的future。
5. Store each future so we can later retrieve them with as_completed.  
    存储每一个future，所以我们可以在之后通过as_completed检查他们
6. Display a message with the country code and the respective future.  
    显示国家代码与各自future的信息
7. as_completed yields futures as they are completed.  
    as_completed产出已经完成的futures
8. Get the result of this future.  
    获取该future的结果
9. Display the future and its result.  
    显示该future与其结果

Note that the future.result() call will never block in this example because the future is coming out of as_completed. Example 17-5 shows the output of one run of Example 17-4.  
    注意，本示例中的future.result()的调用不会阻塞，因为future来自于as_completed。示例17-5展示了17-4的一次运行输出。

Example 17-5. Output of flags_threadpool_ac.py
    示例17-5. flags_threadpool_ac.py的输出
```python
$ python3 flags_threadpool_ac.py
Scheduled for BR: <Future at 0x100791518 state=running>  # 1
Scheduled for CN: <Future at 0x100791710 state=running>
Scheduled for ID: <Future at 0x100791a90 state=running>
Scheduled for IN: <Future at 0x101807080 state=pending>  # 2
Scheduled for US: <Future at 0x101807128 state=pending>
CN <Future at 0x100791710 state=finished returned str> result: 'CN'  # 3
BR ID <Future at 0x100791518 state=finished returned str> result: 'BR'  # 4
<Future at 0x100791a90 state=finished returned str> result: 'ID'
IN <Future at 0x101807080 state=finished returned str> result: 'IN'
US <Future at 0x101807128 state=finished returned str> result: 'US'

5 flags downloaded in 0.70s
```

1. The futures are scheduled in alphabetical order; the repr() of a future shows its state: the first three are running, because there are three worker threads.  
    这些future以字母顺序被进行规划；future的repr()展示了他的状态：前三个在运行，因为共有三个工作线程。
2. The last two futures are pending, waiting for worker threads.  
    后两个future正在pending，等待工作线程。
3. The first CN here is the output of download_one in a worker thread; the rest of the line is the output of download_many.  
    这里的第一个CN是工作线程中download_on的输出；本行剩余部分位download_many的输出。
4. Here two threads output codes before download_many in the main thread can display the result of the first thread.  
    这里的两个线程输出code
`

    If you run flags_threadpool_ac.py several times, you’ll see the order of the results varying. Increasing the max_workers argument to 5 will increase the variation in the order of the results. Decreasing it to 1 will make this code run sequentially, and the order of the results will always be the order of the submit calls.
    如果你多次运行flags_threadpool_ac.py，你将看到各不同顺序的结果。将max_workers数增加至5，将会让结果的顺序变化更大。而减少至1的话，将让该代码顺序执行，结果的顺序将保持与submin调用的顺序一致。

We saw two variants of the download script using concurrent.futures: Example 17-3 with ThreadPoolExecutor.map and Example 17-4 with futures.as_completed. If you are curious about the code for flags_asyncio.py, you may peek at Example 18-5 in Chapter 18.  
    我们看到使用了concurrent.futures的下载脚本的两个变体：17-3用了ThreadPoolExecutor.map，17-4用了futures.as_completed。如果你对flags_asyncio.py的代码感兴趣，你可以参考18章的18-5。

Strictly speaking, none of the concurrent scripts we tested so far can perform downloads in parallel. The concurrent.futures examples are limited by the GIL, and the flags_asyncio.py is single-threaded.  
    严格来说，我们目前测试过的并发脚本没有一个可以实现并行的下载。concurrent.futures示例被GIL所限制，flags_asyncio.py是单线程。

At this point, you may have questions about the informal benchmarks we just did:
    在这点上，你可能会对我们刚刚做的那些非正式基准有些问题：
- How can flags_threadpool.py perform 5× faster than flags.py if Python threads are limited by a Global Interpreter Lock (GIL) that only lets one thread run at any time?  
    如果Python线程被全局解释器锁限制到任何时间只可以运行一条线程，falgs_threadpool.py是如何做到比flags.py快5倍的？
- How can flags_asyncio.py perform 5× faster than flags.py when both are single threaded?  
    同是一条线程，flags_asyncio.py是如何比flags.py快5倍的？

I will answer the second question in “Running Circling Around Blocking Calls” on page 552.  
    我将在552页的“Running Circling Around Blocking Calls”回答第二个问题。

Read on to understand why the GIL is nearly harmless with I/O-bound processing.  
    继续读下去，去理解为什么GIL对IO限制进程几乎是无影响的。


## 17.2 Blocking I/O and the GIL
## 17.2 I/O阻塞与GIL

The CPython interpreter is not thread-safe internally, so it has a Global Interpreter Lock (GIL), which allows only one thread at a time to execute Python bytecodes. That’s why a single Python process usually cannot use multiple CPU cores at the same time.[3]  
    Cpython解释器在内部不是线程安全的，所以他有个全局解释器锁（GIL），在同一时刻它只允许一条线程去执行Python字节码。这就是一个单独的Python进程通常不能同时使用多个CPU核的原因。

When we write Python code, we have no control over the GIL, but a built-in function or an extension written in C can release the GIL while running time-consuming tasks. In fact, a Python library coded in C can manage the GIL, launch its own OS threads, and take advantage of all available CPU cores. This complicates the code of the library considerably, and most library authors don’t do it.  
    当我们编写Python代码时，我们无法控制GIL，但是当运行耗时任务阶段，一个内建的函数或一个由C写的扩展可以释放GIL。事实上，一个由C写的Python库代码可以管理GIL，启动他自己的OS线程，并利用到所有可用的CPU核。这使得库的代码非常复杂，大部分库作者不会这样做。

However, all standard library functions that perform blocking I/O release the GIL when waiting for a result from the OS. This means Python programs that are I/O bound can benefit from using threads at the Python level: while one Python thread is waiting for a response from the network, the blocked I/O function releases the GIL so another thread can run.  
    但是，当在等待OS返回结果时，所有执行阻塞I/O的标准库函数都释放了GIL。这意味着受I/O限制的Python程序可以通过在Python级别使用线程受益：当一个Python线程等待网络的响应，阻塞I/O函数释放了GIL以致其他线程可以运行。

That’s why David Beazley says: “Python threads are great at doing nothing.”[4]  
    这就是为什么David Beazley说：“Python线程很擅长什么都不做。”
·

    Every blocking I/O function in the Python standard library releases the GIL, allowing other threads to run. The time.sleep() function also releases the GIL. Therefore, Python threads are perfectly usable in I/O-bound applications, despite the GIL.
    标准库中的任何阻塞I/O函数都会释放GIL，以允许其他线程继续运行。time.sleep()函数也会释放GIL。因此，尽管存在GIL，Python线程仍然完美地适用于I/O限制的应用中。

Now let’s take a brief look at a simple way to work around the GIL for CPU-bound jobs using concurrent.futures.  
    现在我们看看一个简单的方法，通过使用concurrent.futures来绕开CPU限制任务。

[3] This is a limitation of the CPython interpreter, not of the Python language itself. Jython and IronPython are not limited in this way; but Pypy, the fastest Python interpreter available, also has a GIL.  
    这是Cpython解释器的限制，不是Python语言本身的。Jython和IronPython并那没有该限制；但是最快的Python解释器Pypy同样有GIL。
[4] Slide 106 of “Generators: The Final Frontier”.  
    《Generators: The Final Frontier》106页


## 17.3 Launching Processes with concurrent.futures
## 17.3 使用concurrent.futures启动进程

The concurrent.futures documentation page is subtitled “Launching parallel tasks”. The package does enable truly parallel computations because it supports distributing work among multiple Python processes using the ProcessPoolExecutor class—thus bypassing the GIL and leveraging all available CPU cores, if you need to do CPU-bound processing.  
    concurrent.futures文档页的小标题为“启动并行任务”。该库能够实现真正的并行计算，因为它支持使用ProcessPoolExecutor类在多个Python进程间分配任务——因此绕过了GIL并充分利用到所有可用的CPU核，如果你需要进行CPU受限的任务。

Both ProcessPoolExecutor and ThreadPoolExecutor implement the generic Executor interface, so it’s very easy to switch from a thread-based to a process-based solution using concurrent.futures.  
    ProcessPoolExecutor和ThreadPoolExecutor都实现了通用的Executor接口，所以利用concurrent.futures从一个线程基础切换至进程基础很容易。

There is no advantage in using a ProcessPoolExecutor for the flags download example or any I/O-bound job. It’s easy to verify this; just change these lines in Example 17-3:  
    对之前的flags下载示例或任何IO限制任务来说，使用ProcessPoolExecutor是毫无优势的。这很容易验证，只需要修改示例17-3中的部分代码：

```python
def download_many(cc_list):
    workers = min(MAX_WORKERS, len(cc_list))
    with futures.ThreadPoolExecutor(workers) as executor:
```

To This:

```python
def download_many(cc_list):
    with futures.ProcessPoolExecutor() as executor:
```

For simple uses, the only notable difference between the two concrete executor classes is that ThreadPoolExecutor.__init__ requires a max_workers argument setting the number of threads in the pool. That is an optional argument in ProcessPoolExecutor, and most of the time we don’t use it—the default is the number of CPUs returned by os.cpu_count(). This makes sense: for CPU-bound processing, it makes no sense to ask for more workers than CPUs. On the other hand, for I/O-bound processing, you may use 10, 100, or 1,000 threads in a ThreadPoolExecutor; the best number depends on what you’re doing and the available memory, and finding the optimal number will require careful testing.  
    对于只是简单的使用来说，两个executor类间唯一需要注意的区别是，ThreadPoolExecutor.__init__需要一个max_workers参数来设定池中的线程数，而这在ProcessPoolExecutor中是个可选参数，大部分情况下我们不会用到它——默认使用os.cpu_count()返回的CPU数。这很容易理解：对CPU密集型任务来说，你可能在一个ThreadPoolExecutor中用到10，100或1000个线程；但最佳数量取决于你的任务类型以及可用的内存，需要通过仔细的测试来找到这个最佳值。

A few tests revealed that the average time to download the 20 flags increased to 1.8s with a ProcessPoolExecutor—compared to 1.4s in the original ThreadPoolExecutor version. The main reason for this is likely to be the limit of four concurrent downloads on my four-core machine, against 20 workers in the thread pool version.  
    一些测试显示，使用ProcessPoolExecutor下载20个国旗的平均时间增长到1.8秒——相比于原始的ThreadPoolExecutor版本的1.4秒来讲。这种情况的主要原因是可能是，与线程池版本的20个worker相比，四核机器的并行下载的极限就到这里了。

The value of ProcessPoolExecutor is in CPU-intensive jobs. I did some performance tests with a couple of CPU-bound scripts:  
    ProcessPoolExecutor的价值体现在CPU集中型任务上。我用一对CPU密集型脚本来做一些性能测试：

*arcfour_futures.py*
    Encrypt and decrypt a dozen byte arrays with sizes from 149 KB to 384 KB using a pure-Python implementation of the RC4 algorithm (listing: Example A-7).  
    使用RC4算法的纯Python实现，对12个大小从149KB到384KB的字节序列进行加密与解密（示例A-7）。

*sha_futures.py*
    Compute the SHA-256 hash of a dozen 1 MB byte arrays with the standard library hashlib package, which uses the OpenSSL library (listing: Example A-9).  
    使用标准库hashlib包对12个1MB字节序列计算SHA-256哈希值，该包使用了OpenSSL库（示例A-9)。

Neither of these scripts do I/O except to display summary results. They build and process all their data in memory, so I/O does not interfere with their execution time.  
    除了显示总和的结果外，这两个脚本都没有做其余IO操作。他们在内存中创建并处理数据，所以IO操作不会影响他们的执行时间。

Table 17-1 shows the average timings I got after 64 runs of the RC4 example and 48 runs of the SHA example. The timings include the time to actually spawn the worker processes.  
    表17-1展示了运行64次RC4示例和48次SHAs示例后得到的平均时间。时段包括实际生成工作进程的时间。

Table 17-1. Time and speedup factor for the RC4 and SHA examples with one to four workers on an Intel Core i7 2.7 GHz quad-core machine, using Python 3.4  
    表17-1 在Intel Core i7 2.7GHz 四核机器上，使用Python3.4实现的RC4和SHA示例分别通过1至4个worker所得到的时间和加速因子。

| Workers | RC4 time | RC4 factor | SHA time | SHA factor |
| --- | --- | --- | --- | --- |
| 1 | 11.48s | 1.00x | 22.66s | 1.00x |
| 2 | 8.65s | 1.33x | 14.90s | 1.52x |
| 3 | 6.04s | 1.90x | 11.91s | 1.90x |
| 4 | 5.58s | 2.06x | 10.89s | 2.08x |

In summary, for cryptographic algorithms, you can expect to double the performance by spawning four worker processes with a ProcessPoolExecutor, if you have four CPU cores.  
    概括来讲，对于加密算法，如果你有四个CPU核，你可以期望用ProcessPoolExecutor四个工作线程来实现双倍的性能。

For the pure-Python RC4 example, you can get results 3.8 times faster if you use PyPy and four workers, compared with CPython and four workers. That’s a speedup of 7.8 times in relation to the baseline of one worker with CPython in Table 17-1.  
    对于纯Python实现的RC4示例，如果你使用PyPy和四个worker，会比Cpython+四worker快3.8倍。与表17-1中CPython的单worker的基准相比，是7.8倍的提速。
`

    If you are doing CPU-intensive work in Python, you should try PyPy. The arcfour_futures.py example ran from 3.8 to 5.1 times faster using PyPy, depending on the number of workers used. I tested with PyPy 2.4.0, which is compatible with Python 3.2.5, so it has concurrent.futures in the standard library.
        如果使用Python做CPU集中型的任务，你应该尝试使用PyPy。arcfour_futures.py示例使用PyPy运行速度快3.8到5.5倍，这取决于使用worker的数据。我使用兼容Python3.2.5的PyPy 2.4.0测试，所以标准库中有concurrent.futures。

Now let’s investigate the behavior of a thread pool with a demonstration program that launches a pool with three workers, running five callables that output timestamped messages.
    现在我们通过一个示例程序来研究下线程池的行为，该程序启动一个3 worker的池，运行5个输出时间戳信息的可调用对象。


## Experimenting with Executor.map
## 尝试使用Executor.map

The simplest way to run several callables concurrently is with the Executor.map function we first saw in Example 17-3. Example 17-6 is a script to demonstrate how Executor.map works in some detail. Its output appears in Example 17-7.  
    并发运行不同调用对象的最简单方式是使用Executor.map，我们最初在示例17-3时提到过。示例17-6的脚本详细演示了Executor.map如何工作。示例17-7为它的输出。

Example 17-6. demo_executor_map.py: Simple demonstration of the map method of ThreadPoolExecutor
    示例17-6. demo_executor_map.py：ThreadPoolExecutor中map方法的简单演示

```python
from time import sleep, strftime
from concurrent import futures

def display(*args):  # 1
    print(strftime('[%H:%M:%S]'), end=' ')
    print(*args)

def loiter(n):  # 2
    msg = '{}loiter({}): doing nothing for {}s...'
    display(msg.format('\t'*n, n, n))
    sleep(n)
    msg = '{}loiter({}): done.'
    display(msg.format('\t'*n, n))
    return n * 10  # 3

def main():
    display('Script starting.')
    executor = futures.ThreadPoolExecutor(max_workers=3)  # 4
    results = executor.map(loiter, range(5))  # 5
    display('results:', results)  # 6
    display('Waiting for individual results:')
    for i, result in enumerate(results):  # 7
        display('result {}: {}'.format(i, result))


main()

```

1. This function simply prints whatever argumentsit gets, preceded by a timestamp in the format [HH:MM:SS].  
    该函数简单地打印它接收到的任何参数，在此之前的是[HH:MM:SS]格式的时间戳。
2. loiter does nothing except display a message when it starts, sleep for n seconds, then display a message when it ends; tabs are used to indent the messages according to the value of n.  
    loliter仅仅用来显示启动/睡眠的时间信息，最后在结束时也会显示；tab被用来以n的值来缩进信息。
3. loiter returns n * 10 so we can see how to collect results.  
    loiter最终返回n * 10，所以我们可以看到收集结果的方式。
4. Create a ThreadPoolExecutor with three threads.  
    创建一个3线程的ThreadPoolExecutor。
5. Submit five tasks to the executor (because there are only three threads, only three of those tasks will start immediately: the calls loiter(0), loiter(1), and loiter(2)); this is a nonblocking call.  
    向executor提交5个任务（因为只有3个线程，所以这些任务中的3个会立刻启动：调用loiter(0)，loiter(1)和loiter(2))；这是非阻塞的调用。
6. Immediately display the results of invoking executor.map: it’s a generator, as the output in Example 17-7 shows.  
    立刻显示executor.map的结果：这是一个生成器，和示例17-7的输出一致。
7. The enumerate call in the for loop will implicitly invoke next(results), which in turn will invoke _f.result() on the (internal) _f future representing the first call, loiter(0). The result method will block until the future is done, therefore each iteration in this loop will have to wait for the next result to be ready.  
    for循环中调用到的enumerate将会隐式地调用next(results)，即转而调用（内置的）_f future上的_f.result()，代表第一次调用，loiter(0)。直到futrue结束前result方法将会阻塞，因此该循环中的任一项必须等待下一个result准备完毕。

I encourage you to run Example 17-6 and see the display being updated incrementally. While you’re at it, play with the max_workers argument for the ThreadPoolExecutor and with the range function that produces the arguments for the executor.map call—or replace it with lists of handpicked values to create different delays.  
    我鼓励你亲自运行下示例17-6，看看他的输出不断更新的过程。当你自己运行时，试着使用ThreadPoolExecutor的max_workers参数和用于为executor.map生成参数的range函数——或将其直接替换为自定义的value列表，来创建不同的延迟时间。

Example 17-7 shows a sample run of Example 17-6.

Example 17-7. Sample run of demo_executor_map.py from Example 17-6

```python
$ python3 demo_executor_map.py
[15:56:50] Script starting.  # 1
[15:56:50] loiter(0): doing nothing for 0s...  # 2
[15:56:50] loiter(0): done.
[15:56:50]      loiter(1): doing nothing for 1s...  # 3
[15:56:50]          loiter(2): doing nothing for 2s...
[15:56:50] results: <generator object result_iterator at 0x106517168>  # 4
[15:56:50]              loiter(3): doing nothing for 3s...  # 5
[15:56:50] Waiting for individual results:
[15:56:50] result 0: 0  # 6
[15:56:51]      loiter(1): done.  # 7
[15:56:51]                  loiter(4): doing nothing for 4s...
[15:56:51] result 1: 10  # 8
[15:56:52]          loiter(2): done.  # 9
[15:56:52] result 2: 20
[15:56:53]              loiter(3): done.
[15:56:53] result 3: 30
[15:56:55]                  loiter(4): done.  # 10
[15:56:55] result 4: 40
```

1. This run started at 15:56:50.  
    运行启动于15:56:50
2. The first thread executes loiter(0), so it will sleep for 0s and return even before the second thread has a chance to start, but YMMV.5  
    第一个线程执行loiter(0)，所以将sleep 0秒，甚至有机会在第二个线程启动前就return。
3. loiter(1) and loiter(2) start immediately (because the thread pool has three workers, it can run three functions concurrently).  
    loitor(1)和loiter(2)同样立即启动（因为线程池有3个worker，可以同时运行三个函数）。
4. This shows that the results returned by executor.map is a generator; nothing so far would block, regardless of the number of tasks and the max_workers setting.  
    通过executor.map实例化的results是一个生成器；到目前为止，不管task数量与max_workers如何设置，都不会发生阻塞。
5. Because loiter(0) is done, the first worker is now available to start the fourth thread for loiter(3).  
    由于loiter(0)已经完成，第一个worker被提供给loiter(3)，启动第四条线程。
6. This is where execution may block, depending on the parameters given to the loiter calls: the __next__ method of the results generator must wait until the first future is complete. In this case, it won’t block because the call to loiter(0) finished before this loop started. Note that everything up to this point happened within the same second: 15:56:50.  
    这里可能是执行出现阻塞的地方，取决于传入loiter的参数：results生成器的__next__方法必须等待第一个future结束。在这组case中，由于loiter(0)的调用结束于该loop启动之前，所以不会产生阻塞。注意，到目前为止所有事件都发生于相同的时刻：15:56:50。
7. loiter(1) is done one second later, at 15:56:51. The thread is freed to start loiter(4).  
    loiter(1)于1秒后的15:56:61结束。该线程被释放，启动loiter(4)。
8. The result of loiter(1) is shown: 10. Now the for loop will block waiting for the result of loiter(2).  
    loiter(1)的result为10。此刻for循环将阻塞等待loiter(2)的result。
9. The pattern repeats: loiter(2) is done, its result is shown;samewith loiter(3).  
    模板重复：loiter(2)结束，展示result；loiter(3)同理。
10. There is a 2s delay until loiter(4) is done, because it started at 15:56:51 and did nothing for 4s.  
    在loiter(4)结束前有2秒延迟，因为他启动于15:56:51且doing nothing for 4秒。

The Executor.map function is easy to use but it has a feature that may or may not be helpful, depending on your needs: it returns the results exactly in the same order as the calls are started: if the first call takes 10s to produce a result, and the others take 1s each, your code will block for 10s as it tries to retrieve the first result of the generator returned by map. After that, you’ll get the remaining results without blocking because they will be done. That’s OK when you must have all the results before proceeding, but often it’s preferable to get the results as they are ready, regardless of the order they were submitted. To do that, you need a combination of the Executor.submit method and the futures.as_completed function, as we saw in Example 17-4. We’ll come back to this technique in “Using futures.as_completed” on page 527.  
    Excutor.map函数很实用，但是他有一个可能不太好用的特性，这取决于你的需求：他会根据你的调用顺序，精确地以相同顺序返回结果：如果第一次的调用花费了10秒产生结果，而其他的都花费1秒，那么当你的代码试图解析map返回的生成器的首个结果时，将会阻塞10秒。在那之后，你获取剩余的结果时将不会阻塞，因为他们都将被完成。当你需要在继续之前获取到所有结果的情况下，这种方式是ok的，但通常情况，更好的办法是一旦ready就获取到结果，不管他们是以何种顺序被提交进来的。为了达到这样的效果，你需要将Executor.submit方法和futures.as_completed函数结合使用，如我们在示例17-4中所视。我们将在527页的“futures._ascomplted函数使用”回到这部分技巧。

    The combination of executor.submit and futures.as_completed is more flexible than executor.map because you can submit different callables and arguments, while executor.map is designed to run the same callable on the different arguments. In addition, the set of futures you pass to futures.as_completed may come from more than one executor—perhaps some were created by a ThreadPoolExecutor instance while others are from a ProcessPoolExecutor.  
    相比于executor.map，executor.submin和futures.as_completed的组合更加灵活，因为你可以提交不同的调用对象及参数，而executor.map被定义为执行相同调用对象+不同参数。此外，传递给futures.as_completed的future集合可能来自于多个executor——可能是一个由ThreadPoolExecutor创建，而其余来自于ProcessPoolExecutor。

In the next section, we will resume the flag download examples with new requirements that will force us to iterate over the results of futures.as_completed instead of using executor.map.
    在下一节中，我们将带着新需求继续下载国旗的例子，这会迫使我们使用futures.as_completed代替executor.map来遍历结果。


## Downloads with Progress Display and Error Handling
## 下载进度显示与Error控制

As mentioned, the scripts in “Example: Web Downloads in Three Styles” on page 505 have no error handling to make them easier to read and to contrast the structure of the three approaches: sequential, threaded, and asynchronous.
    如上所述，505页中“示例：三种风格的Web下载”脚本都没有进行错误控制，这样做让脚本更容易阅读，且更加方便对比这三种方法：顺序的，利用线程的以及异步的。

In order to test the handling of a variety of error conditions, I created the flags2 examples:
    为了测试对各种错误情况的处理，我创建了flags示例：

flags2_common.py  
    This module contains common functions and settings used by all flags2 examples, including a main function, which takes care of command-line parsing, timing, and reporting results. This is really support code, not directly relevant to the subject of this chapter, so the source code is in Appendix A, Example A-10.  
    该模块包含了用于所有flags2示例的common方法与设置，包含一个main方法，负责命令行的解析，计时以及报告结果。这只是辅助代码，与本章主题没有直接关系，所以源码放置于附录A，示例A-10。  
flags2_sequential.py  
    A sequential HTTP client with proper error handling and progress bar display. Its download_one function is also used by flags2_threadpool.py.  
    顺序HTTP客户端，包含合适的错误处理与进度条显示。其中download_one方法也用于flags2_threadpool.py。
flags2_threadpool.py  
    Concurrent HTTP client based on futures.ThreadPoolExecutor to demonstrate error handling and integration of the progress bar.  
    以futures.ThreadPoolExecutor为基础的并发HTTP客户端，用于演示错误处理与进度条。
flags2_asyncio.py  
    Same functionality as previous example but implemented with asyncio and aiohttp. This will be covered in “Enhancing the asyncio downloader Script” on page 554, in Chapter 18.  
    功能与上面的例子一样，但基于asyncio与aiohttp。将在554页的18章“升级asyncio下载器及脚本”介绍。

    Be Careful When Testing Concurrent Clients  
    When testing concurrent HTTP clients on public HTTP servers, you may generate many requests persecond, and that’s how denial of-service (DoS) attacks are made. We don’t want to attack anyone, just learn how to build high-performance clients. Carefully throttle your clients when hitting public servers. For high concurrency experiments, set up a local HTTP server for testing. Instructions for doing it are in the README.rst file in the 17-futures/countries/ directory of the Fluent Python code repository.  
    测试并发客户端需注意：  
    当在公用HTTP服务上测试并发HTTP客户端时，你可能每秒生成许多请求，这会形成拒绝服务攻击（Dos）。我们只是想要了解高性能客户端的构造原理，并不想造成任何攻击。所以当访问公用服务时请小心限制你的客户端。对于并发的实验来说，建立一个本地HTTP服务用于测试。关于这部分的介绍在Fluent Python代码库17-futures/countries/目录下的README.rst文件中。


The most visible feature of the flags2 examples is that they have an animated, text mode progress bar implemented with the TQDM package. I posted a 108s video on YouTube to show the progress bar and contrast the speed of the three flags2 scripts. In the video, I start with the sequential download, but I interrupt it after 32s because it was going to take more than 5 minutes to hit on 676 URLs and get 194 flags; I then run the threaded and asyncio scripts three times each, and every time they complete the job in 6s or less (i.e., more than 60 times faster). Figure 17-1 shows two screenshots: during and after running flags2_threadpool.py.  
    flags2示例最直观的特点是他通过TQDM库建立了一个动画的，文本模式进度条。我在YouTube上传了一段108秒的视频，来展示这个进度条，并对比了flags2的三个脚本的运行速度。在该视频中，以顺序下载开头，但32秒后我就将他中断了，因为他会运行超过5分钟并访问676次URL获取194张flag；之后运行了线程式与asyncio的脚本各三次，他们每次的完成用时都在6秒或更少（也就是说快了60多倍）。图示17-1的两张截图：运行flags2_threadpool.py的期间与结束。

[Figure 17-1]  
Figure 17-1. Top-left: flags2_threadpool.py running with live progress bar generated by tqdm; bottom-right: same terminal window after the script is finished.  
    图示17-1。左上角：flags2_threadpool.py运行中由tqdm生成的动态进度条；右下方：脚本运行结束后，相同的终端窗口。
TQDM is very easy to use, the simplest example appears in an animated .gif in the project’s README.md. If you type the following code in the Python console after installing the tqdm package, you’ll see an animated progress bar were the comment is:  
    TQDM很简单，最简单的实例在README.md中的动画.gif中。在安装tqdm库后，如果你在Python控制台输入下列代码，你会在注释位置看到一个动态的进度条：
```python

>>> import time
>>> from tqdm import tqdm
>>> for i in tqdm(range(1000)):
... time.sleep(.01)
...
>>> # -> progress bar will appear here <-
```
Besides the neat effect, the tqdm function is also interesting conceptually: it consumes any iterable and produces an iterator which, while it’s consumed, displays the progress bar and estimates the remaining time to complete all iterations. To compute that estimate, tqdm needs to get an iterable that has a len, or receive as a second argument the expected number of items. Integrating TQDM with our flags2 examples provide an opportunity to look deeper into how the concurrent scripts actually work, by forcing us to use the futures.as_completed and the asyncio.as_completed functions so that tqdm can display progress as each future is completed.  
    除了整洁的效果以外，tqdm函数在概念上也很有趣：他消费任意可迭代对象并生成一个带带器，当他被消费时，显示出进度条并预估出完成所有迭代需要的剩余时间。为了计算出预估时间，tqdm需要获取到具有len的可迭代对象，或是作为第二参数接收的预期项数。将TQDM与我们的flags2整合，可以让我们更深入看到并发脚本实际是如何工作的，通过强迫使用futures.as_completed和asyncio.as_completed方法，以便tqdm可以在每个future结束时显示进度。

The other feature of the flags2 example is a command-line interface. All three scripts accept the same options, and you can see them by running any of the scripts with the -h option. Example 17-8 shows the help text.  
    flags2示例另外的特点在于他是一个命令行接口。所有三个脚本接收相同的选项，你可以通过运行带有-h选项的脚本来查看他们。示例17-8是help内容。

Example 17-8. Help screen for the scripts in the flags2 series  
```python
$ python3 flags2_threadpool.py -h
usage: flags2_threadpool.py [-h] [-a] [-e] [-l N] [-m CONCURRENT] [-s LABEL]
                            [-v]
                            [CC [CC ...]]

Download flags for country codes. Default: top 20 countries by population.
positional arguments:
    CC country code or 1st letter (eg. B for BA...BZ)
optional arguments:
    -h, --help show this help message and exit
    -a, --all get all available flags (AD to ZW)
    -e, --every get flags for every possible code (AA...ZZ)
    -l N, --limit N limit to N first codes
    -m CONCURRENT, --max_req CONCURRENT
                        maximum concurrent requests (default=30)
    -s LABEL, --server LABEL
                        Server to hit; one of DELAY, ERROR, LOCAL, REMOTE(default=LOCAL)
    -v, --verbose output detailed progress info
```

All arguments are optional. The most important arguments are discussed next.  
    所有参数都是可选的。下面讨论最重要的参数。

One option you can’t ignore is -s/--server: it lets you choose which HTTP server and base URL will be used in the test. You can pass one of four strings to determine where the script will look for the flags (the strings are case insensitive):  
    你不能忽视一个参数是-s/--server：这让你选择测试用到的HTTP服务以及基础URL。你可以传入四个字符串中的一个来决定脚本去哪里寻找flags（字符串不区分大小写）：

LOCAL  
Use http://localhost:8001/flags; this is the default. You should configure a local HTTP server to answer at port 8001. I used Nginx for my tests. The README.rst file for this chapter’s example code explains how to install and configure it.  
    使用http://localhost:8001/flags；默认方式。你应该配置一个本地HTTP服务来响应8001端口。我是用Nginx来测试。本章示例代码的README.rst文件展示了安装和配置的方式。
REMOTE  
Use http://flupy.org/data/flags; that is a public website owned by me, hosted on a shared server. Please do not pound it with too many concurrent requests. The flupy.org domain is handled by a free account on the Cloudflare CDN so you may notice that the first downloads are slower, but they get faster when the CDN cache warms up.[6]  
    使用http://flupy.org/data/flags，只是我建立的公共网站，主机位于一个共享服务器。请不要发太多并发请求。flupy.org域名是由Cloudflare CDN的免费账户处理的，所以你可能需要注意第一次下载会较慢，但CDN缓存预热后会快起来。
DELAY  
Use http://localhost:8002/flags; a proxy delaying HTTP responses should be listening at port 8002. I used a Mozilla Vaurien in front of my local Nginx to introduce delays. The previously mentioned README.rst file has instructions for running a Vaurien proxy.  
    使用 http://localhost:8002/flags；一个需要监听8002端口的代理延迟HTTP。我在本地Nginx之前使用了Mozilla Vaurien来引入延迟。前面提到的README.rst文件有运行Vaurien代理的说明。
ERROR  
Use http://localhost:8003/flags; a proxy introducing HTTP errors and delaying responses should be installed at port 8003. I used a different Vaurien configuration for this.  
    使用http://localhost:8003/flags；应该在8003端口安装一个引入HTTP错误与延迟响应的代理。

    The LOCAL option only works if you configure and start a local HTTP server on port 8001. The DELAY and ERROR options require proxies listening on ports 8002 and 8003. Configuring Nginx and Mozilla Vaurien to enable these options is explained in the 17-futures/countries/README.rst file in the Fluent Python code repository on GitHub.  
    只有如果你在8001端口配置并启动了一个本地HTTP服务，才能使用LOCAL选项。DELAY和ERROR选项扥别需要有监听8002和8003端口的代理。为了实现这些选项需要配置Nginx和Mozilla Vaurien，在GitHub库的17-futures/countries/README.rst中有解释。

[6] Before configuring Cloudflare, I got HTTP 503 errors—Service Temporarily Unavailable—when testing the scripts with a few dozen concurrent requests on my inexpensive shared host account. Now those errors are gone.  
    在配置Cloudflare之前，当在我的脸颊共享主机账户上调用几十个并发请求测试脚本，收到了HTTP 503报错——服务暂不可用。现在没有这些报错了。

By default, each flags2 script will fetch the flags of the 20 most populous countries from the LOCAL server (http://localhost:8001/flags) using a default number of concurrent connections, which varies from script to script. Example 17-9 shows a sample run of the flags2_sequential.py script using all defaults.  
    默认情况下，每个flag2脚本将使用默认并发连接数，从LOCAL服务上获取20个人口最多的国家国旗，连接数由脚本决定。示例17-9展示了flags2_sequential.py脚本使用所有默认值的一次简单运行。

Example 17-9. Running flags2_sequential.py with all defaults: LOCAL site, top-20 flags, 1 concurrent connection  
    示例17-9. 使用默认参数运行flags2_squential.py：LOCAL site，前20个国旗，一个并发连接
```python
$ python3 flags2_sequential.py
LOCAL site: http://localhost:8001/flags
Searching for 20 flags: from BD to VN
1 concurrent connection will be used.
--------------------
20 flags downloaded.
Elapsed time: 0.10s
```

You can select which flags will be downloaded in several ways. Example 17-10 shows how to download all flags with country codes starting with the letters A, B, or C.  
    你可以用多种方式选择要下载的国旗。示例17-10介绍了如何下载以字母A，B，C开头的所有国家的国旗。

Example 17-10. Run flags2_threadpool.py to fetch all flags with country codes prefixes A, B, or C from DELAY server  
    示例17-10. 通过运行flags2_threadpool.py从DELAY服务中获取国家名前缀为A/B/C的所有国旗
```python
$ python3 flags2_threadpool.py -s DELAY a b c
DELAY site: http://localhost:8002/flags
Searching for 78 flags: from AA to CZ
30 concurrent connections will be used.
--------------------
43 flags downloaded.
35 not found.
Elapsed time: 1.72s
```

Regardless of how the country codes are selected, the number of flags to fetch can be limited with the -l/--limit option. Example 17-11 demonstrates how to run exactly 100 requests, combining the -a option to get all flags with -l 100.  
    不管国家是以何种方式选择的，获取的国旗数量将被参数选项-l/--limit所限制。示例17-11演示了通过结合参数-a（获取所有flag）和-l 100，准确发送100次请求的方法。

Example 17-11. Run flags2_asyncio.py to get 100 flags (-al 100) from the ERROR server, using 100 concurrent requests (-m 100)  
    示例17-11. 运行flags2_asyncio.py从ERROR服务通过100个并发请求（-m 100）获取100个国旗（-al 100）
```python
$ python3 flags2_asyncio.py -s ERROR -al 100 -m 100
ERROR site: http://localhost:8003/flags
Searching for 100 flags: from AD to LK
100 concurrent connections will be used.
--------------------
73 flags downloaded.
27 errors.
Elapsed time: 0.64s
```
That’s the user interface of the flags2 examples. Let’s see how they are implemented.  
    这是flags2示例的使用接口，我们来看下实现方法。


## Error Handling in the flags2 Examples
## flags2示例中的错误处理

The common strategy adopted in all three examples to deal with HTTP errors is that 404 errors (Not Found) are handled by the function in charge of downloading a single file (download_one). Any other exception propagates to be handled by the download_many function.  
    所有三种示例中为了处理HTTP error共同采用的策略是，404 errors(Not Found)被负责下载单独文件的函数（download_one）所处理。任何其余异常都将由download_many函数处理。

Again, we’ll start by studying the sequential code, which is easier to follow—and mostly reused by the thread pool script. Example 17-12 shows the functions that perform the actual downloads in the flags2_sequential.py and flags2_threadpool.py scripts.  
    我们从sequential的代码开始研究，这部分相对容易理解，同时大部分线程池脚本重用。示例17-12展示了在flags2_sequential.py和flags2_threadpool.py脚本中执行实际的下载。

Example 17-12. flags2_sequential.py: basic functions in charge of downloading; both are reused in flags2_threadpool.py  
    示例17-12. flags2_sequential.py：负责下载的基本方法；都被复用于flags2_threadpool.py
```python

def get_flag(base_url, cc):
    url = '{}/{cc}/{cc}.gif'.format(base_url, cc=cc.lower())
    resp = requests.get(url)
    if resp.status_code != 200:  # 1
        resp.raise_for_status()
    return resp.content


def download_one(cc, base_url, verbose=False):
    try:
        image = get_flag(base_url, cc)
    except requests.exceptions.HTTPError as exc:  # 2
        res = exc.response
        if res.status_code == 404:
            status = HTTPStatus.not_found  # 3
            msg = 'not found'
        else:
            raise
    else:  # 4
        save_flag(image, cc.lower() + '.gif')
        status = HTTPStatus.ok
        msg = 'OK'

    if verbose:  # 5 
        print(cc, msg)

    return Result(status, cc)  # 6

```

1. get_flag does no error handling, it uses requests.Response.raise_for_status to raise an exception for any HTTP code other than 200.  
    get_flag没有处理error，在HTTP code不为200时，用requests.Response.raise_for_status来抛出一个异常。
2. download_one catches requests.exceptions.HTTPError to handle HTTP code 404 specifically…  
    download_one捕捉了requests.exceptions.HTTPError来特殊处理404
3. …by setting its local status to HTTPStatus.not_found; HTTPStatus is an Enum imported from flags2_common (Example A-10).  
    通过将本地状态设置为HTTPStatus.not_found；HTTPStatus是从flags2_common导入的一个枚举。
4. Any other HTTPError exception is re-raised; other exceptions will just propagate to the caller.  
    其余的HTTPError异常会被重新抛出；其他异常只会传播给调用者。
5. If the -v/--verbose command-line option is set, the country code and status message will be displayed; this how you’ll see progress in the verbose mode.  
    如果设置了-v/--verbose命令行选项，将显示出国家代码和状态信息；这就是在verbose模式下可以看到进度的原因。
6. The Result namedtuple returned by download_one will have a status field with a value of HTTPStatus.not_found or HTTPStatus.ok.  
    download_one返回的Result命名元组包含一个status字段，值可能为HTTPStatus.not_found或HTTPStatus.ok。

Example 17-13 lists the sequential version of the download_many function. This code is straightforward, but its worth studying to contrast with the concurrent versions coming up. Focus on how it reports progress, handles errors, and tallies downloads.  
    例17-13列出了download_many函数的顺序版本。代码很简单，但是为了与即将介绍的并发版本对比，值得研究一下。主要留意他是如何报告进度，处理error与统计下载量的。

Example 17-13. flags2_sequential.py: the sequential implementation of download_many  
    例17-13. flags2_sequential.py：download_many的顺序实现

```python
def download_many(cc_list, base_url, verbose, max_req):
    counter = collections.Counter()  # 1
    cc_iter = sorted(cc_list)  # 2
    if not verbose:
        cc_iter = tqdm.tqdm(cc_iter)  # 3
    for cc in cc_iter:  # 4
        try:
            res = download_one(cc, base_url, verbose)  # 5
        except requests.exceptions.HTTPError as exc:  # 6
            error_msg = 'HTTP error {res.status_code} - {res.reason}'
            error_msg = error_msg.format(res=exc.response)
        except requests.exceptions.ConnectionError as exc:  # 7
            error_msg = 'Connection error'
        else:  # 8
            error_msg = ''
            status = res.status
    if error_msg:
        status = HTTPStatus.error  # 9
    counter[status] += 1  # 10
    if verbose and error_msg:  # 11
        print('*** Error for {}: {}'.format(cc, error_msg))
    
    return counter  # 12
```

1. This Counter will tally the different download outcomes: HTTPStatus.ok, HTTPStatus.not_found, or HTTPStatus.error.
    Counter将记录不同的下载结果：HTTPStatus.ok，HTTPStatus.not_found或HTTPStatus.error。
2. cc_iter holds the list of the country codes received as arguments, ordered alphabetically.
    cc_iter保存了参数传入的国家码列表，以字母顺序排序。
3. If not running in verbose mode, cc_iter is passed to the tqdm function, which will return an iterator that yields the items in cc_iter while also displaying the animated progress bar.
    如果你以verbose模式运行，会将cc_iter传入tqdm方法，tqdm会返回出一个迭代器，产出cc_iter各项的同时也显示动态的进度条。
4. This for loop iterates over cc_iter and…
    该for循环遍历cc_iter，并且…
5. …performs the download by successive calls to download_one.
    通过调用download_one来执行下载。
6. HTTP-related exceptions raised by get_flag and not handled by download_one are handled here.
    HTTP相关的异常由get_flag抛出，同时在download_one未处理的部分将在这里处理。
7. Other network-related exceptions are handled here. Any other exception will abort the script, because the flags2_common.main function that calls download_many has no try/except.
    其它的网络相关异常在这里处理。而其余的异常将终止脚本，因为flags2_commom.main方法调用download_many时未使用try/except。
8. If no exception escaped download_one, then the status is retrieved from the HTTPStatus namedtuple returned by download_one.  
    如果没有异常从download_one逃离，那么status就是从download_one中返回的HTTPStatus命名元组接收。
9. If there was an error, set the local status accordingly.
    如果此处是error，设置相应的本地status。
10. Increment the counter by using the value of the HTTPStatus Enum as key.
    将HTTPStatus枚举作为key的value，填入至counter。
11. If running in verbose mode, display the error message for the current country code, if any.
    运行verbose模式时，如果有任何error，则显示当前国家码的错误信息。
12. Return the counter so that the main function can display the numbers in its final report.
    为了让main方法在其最后报告中显示数量，return出counter。

We’ll now study the refactored thread pool example, flags2_threadpool.py.  
    现在我们来学习重构后的线程池示例，flags2_threadpool.py。

## Using futures.as_completed
## 使用futures.as_completed

In order to integrate the TQDM progress bar and handle errors on each request, the flags2_threadpool.py script uses futures.ThreadPoolExecutor with the futures.as_completed function we’ve already seen. Example 17-14 is the full listing of flags2_threadpool.py. Only the download_many function is implemented; the other functions are reused from the flags2_common and flags2_sequential modules.  
    为了在每一次请求上整合TQDM进度条与处理error，脚本flags2_threadpool.py使用了futures.ThreadPoolExecutor和我们已经见过的futures.as_completed。例17-14是flags_threadpool.py的完整列表。其中只实现download_many函数；其余函数都是复用于flags2_common和flags2_sequential模块。

Example 17-14. flags2_threadpool.py: full listing
```python
import collections
from concurrent import futures
import requests
import tqdm  # 1

from flags2_common import main, HTTPStatus  # 2
from flags2_sequential import download_one  # 3

DEFAULT_CONCUR_REQ = 30  # 4
MAX_CONCUR_REQ = 1000  # 5


def download_many(cc_list, base_url, verbose, concur_req):
    counter = collections.Counter()
    with futures.ThreadPoolExecutor(max_workers=concur_req) as executor:  # 6
        to_do_map = {}  # 7
        for cc in sorted(cc_list):  # 8
            future = executor.submit(download_one,
                            cc, base_url, verbose)  # 9
            to_do_map[future] = cc  # 10
        done_iter = futures.as_completed(to_do_map)  # 11
        if not verbose:
            done_iter = tqdm.tqdm(done_iter, total=len(cc_list))  # 12
        for future in done_iter:  # 13
            try:
                res = future.result()  # 14
            except requests.exceptions.HTTPError as exc:  # 15
                error_msg = 'HTTP {res.status_code} - {res.reason}'
                error_msg = error_msg.format(res=exc.response)
            except requests.exceptions.ConnectionError as exc:
                error_msg = 'Connection error'
            else:
                error_msg = ''
                status = res.status

            if error_msg:
                status = HTTPStatus.error
            counter[status] += 1
            if verbose and error_msg:
                cc = to_do_map[future]  # 16
                print('*** Error for {}: {}'.format(cc, error_msg))

    return counter


if __name__ == '__main__':
    main(download_many, DEFAULT_CONCUR_REQ, MAX_CONCUR_REQ)

```

1. Import the progress-bar display library.
    导入进度条显示库。
2. Import one function and one Enum from the flags2_common module.
    从flags2_common导入一个函数与Enum。
3. Reuse the donwload_one from flags2_sequential (Example 17-12).
    复用flags2_sequential中的download_one函数。
4. If the -m/--max_req command-line option is not given, this will be the maximum number of concurrent requests, implemented as the size of the thread pool; the actual number may be smaller, if the number of flags to download is smaller.
    如果命令行中未获取到-m/--max_req，该值即并发请求的最大数量，作为线程池的size使用；如果要下载的flag更少，该值的实际值会更小。
5. MAX_CONCUR_REQ caps the maximum number of concurrent requests regardless of the number of flags to download or the -m/--max_req command-line option; it’s a safety precaution.
    不管要下载多少flag或是-m/--max_req是多少，并发请求的最大数由MAX_CONCUR_REQ所限制；这是安全措施。
6. Create the executor with max_workers set to concur_req, computed by the main function as the smaller of: MAX_CONCUR_REQ, the length of cc_list, and the value of the -m/--max_req command-line option. This avoids creating more threads than necessary.
    创建executor，max_worker由concur_req传入值设置，由main函数计算出MAX_CONCUR_REQ，cc_list长度和-m/--max_req这三个中最小的值。这样避免创建多余的线程。
7. This dict will map each Future instance—representing one download—with the respective country code for error reporting.
    该字典将为每个Future实例（表示一次下载）匹配各自的国家码，用于报告error。
8. Iterate over the list of country codes in alphabetical order. The order of the results will depend on the timing of the HTTP responses more than anything, but if the size of the thread pool (given by concur_req) is much smaller than len(cc_list), you may notice the downloads batched alphabetically.
    以字母顺序遍历国家码列表。结果的顺序主要由HTTP响应时间决定，但是如果线程池的size（来自于concur_req）比len(cc_list)小，你可能会注意到下载顺序是按照字母顺序分批的。
9. Each call to executor.submit schedules the execution of one callable and returns a Future instance. The first argument is the callable, the rest are the arguments it will receive.
    executor.submit的每次调用会预定一个可调用对象的执行，并返回一个Future实例。第一个参数是可调用对象，其余为他接受的参数。
10. Store the future and the country code in the dict.
    将fucure和国家码的对应保存进字典。
11. futures.as_completed returns an iterator that yields futures as they are done.
    futures.as_completed返回一个迭代器，在他们结束时产出future。
12. If not in verbose mode, wrap the result of as_completed with the tqdm function to display the progress bar; because done_iter has no len, we must tell tqdm what is the expected number of items as the total= argument, so tqdm can estimate the work remaining.
    如果不是verbose模式，会用tqdm函数对as_completed的结果进行装饰，以显示进度条；由于done_iter没有长度，我们需要通知tqdm期待的项数作为total = argument，以帮助tqdm可以评估出剩余的工作量。
13. Iterate over the futures as they are completed.
    当futures完成时对他们进行遍历。
14. Calling the result method on a future either returns the value returned by the callable, or raises whatever exception was caught when the callable was executed. This method may block waiting for a resolution, but not in this example because as_completed only returns futures that are done.
    调用future对象的result方法，要么返回调用对象的返回结果，要么抛出在其执行时捕捉到的任何异常。本方法可能会阻塞以等待解析，但是由于as_completed只返回完成的future，所以该示例中没有。
15. Handle the potential exceptions; the rest of this function is identical to the sequential version of download_many (Example 17-13), except for the next callout. 
    处理潜在的异常；此方法其余部分为与顺序版本的download_many完全一致，除下一次注释部分以外。
16. To provide context for the error message, retrieve the country code from the to_do_map using the current future as key. This was not necessary in the sequential version because we were iterating over the list of country codes, so we had the current cc; here we are iterating over the futures.
    为了提供error message的文本，使用当前future作为key从to_do_map接获取国家码。在顺序版本里这不是必需的，因为我们遍历国家码列表后可以获取当前cc；在这里我们遍历futures。

Example 17-14 uses an idiom that’s very useful with futures.as_completed: building a dict to map each future to other data that may be useful when the future is completed. Here the to_do_map maps each future to the country code assigned to it. This makes it easy to do follow-up processing with the result of the futures, despite the fact that they are produced out of order.  
    例17-14使用了一个对futures.as_completed非常有用的方法：构建一个字典，当future完成时，匹配每个future和其他可能有用的数据。这里的to_do_map将每个future匹配了其对应的国家码。这样做对后续future结果的操作很方便，尽管这是无序产生的。

Python threads are well suited for I/O-intensive applications, and the concurrent.futures package makes them trivially simple to use for certain use cases. This concludes our basic introduction to concurrent.futures. Let’s now discuss alternatives for when ThreadPoolExecutor or ProcessPoolExecutor are not suitable.  
    Python线程很适合I/O密集型的应用，而concurrent.fugures包让这些在某些用例上更为容易使用。这就是对concurrent.future的基本介绍。接下来讨论当ThreadPoolExecutor或ProcessPoolExecutor都不适合时的替代方案。


## Threading and Multiprocessing Alternatives
## 线程与多进程的替代方案

Python has supported threads since its release 0.9.8 (1993); concurrent.futures is just the latest way of using them. In Python 3, the original thread module was deprecated in favor of the higher-level threading module.[7] If futures.ThreadPoolExecutor is not flexible enough for a certain job, you may need to build your own solution out of basic threading components such as Thread, Lock, Semaphore, etc.—possibly using the thread-safe queues of the queuemodule for passing data between threads. Those moving parts are encapsulated by futures.ThreadPoolExecutor.  
    Python自发布0.9.8(1993)以来支持了线程操作；concurrent.futures仅仅是一种最新的使用他们的方式。在Python3中，原始的线程模块被弃用，取而代之的是更高级的threading模块。[7] 如果对某个任务来说，futures.ThreadPoolExecutor没有足够灵活，你可以需要使用基本的threading元素（如Thread，Lock，Semaphore等），构建你自己的解决方案。

For CPU-bound work, you need to sidestep the GIL by launching multiple processes. The futures.ProcessPoolExecutor is the easiest way to do it. But again, if your use case is complex, you’ll need more advanced tools. The multiprocessing package emulates the threading API but delegates jobs to multiple processes. For simple programs, multiprocessing can replace threading with few changes. But multiprocessing also offers facilities to solve the biggest challenge faced by collaborating processes: how to pass around data.  
    对于CPU绑定的工作，你需要通过启动多进程来回避GIL。futures.ProcessPoolExecutor是最简单的方式。不过，如果你的用例很复杂，你需要更多先进工具。multiprocessing包模拟了threading的API，但把工作委托给了多进程。对于简单的程序来说，多进程只需要很小的改动就可以替换多线程。但multiprocessing也提供工具去解决面对协作进程时的最大挑战：如何传递数据。

[7]. The threading module has been available since Python 1.5.1 (1998), yet some insist on using the old thread module. In Python 3, it was renamed to _thread to highlight the fact that it’s just a low-level implementation detail, and shouldn’t be used in application code.
    [7]. threading模块自Python 1.5.1(1998)时可用，但有些人仍然坚持使用旧版的thread模块。在Python3中，他被重命名为_thread以强调“这只是一个低级的实现细节，不应该被用在应用程序的代码中”。


# Chapter Summary
# 章节总结

We started the chapter by comparing two concurrent HTTP clients with a sequential one, demonstrating significant performance gains over the sequential script.  
    本章一开头，我们对比两种并发HTTP客户端和一种顺序HTTP客户端，演示了相对于顺序脚本的显著性能提升。

After studying the first example based on concurrent.futures, we took a closer look at future objects, either instances of concurrent.futures.Future, or asyncio.Future, emphasizing what these classes have in common (their differences will be emphasized in Chapter 18). We saw how to create futures by calling Executor.submit(…), and iterate over completed futures with concurrent.futures.as_completed(…).   
    在学习首个基于concurrent.futures的示例后，我们更仔细地关注了future对象，要么是concurrent.futures.Future的实例，要么是asyncio.Future的，强调了这些类的共同点（差异将在18章介绍）。然后我们看到如何通过调用Executor.submit(...)创建futures，并利用concurrent.futures.as_completed(…)遍历那些已完成的future。

Next, we saw why Python threads are well suited for I/O-bound applications, despite the GIL: every standard library I/O function written in C releases the GIL, so while a given thread is waiting for I/O, the Python scheduler can switch to another thread. We then discussed the use of multiple processes with the concurrent.futures.ProcessPoolExecutor class, to go around the GIL and use multiple CPU cores to run cryptographic algorithms, achieving speedups of more than 100% when using four workers.  
    接下来，我们看到了Python线程更适合IO绑定型应用的原因，尽管存在GIL：每个用C编写的标准库I/O方法都会释放GIL，搜易当一条线程为I/O而等待时，Python调度程序会切换到另一条线程。然后我们谈论了使用concurrent.futures.ProcessPoolExecutor类的多进程的使用，为了绕开GIL并利用多CPU核来运行密码算法，当使用4个worker时可以达到超过100%的速度提升。

In the following section, we took a close look at how the concurrent.futures.ThreadPoolExecutor works, with a didactic example launching tasks that did nothing for a few seconds, except displaying their status with a timestamp.  
    在接下来的小节中，我们更仔细观察了concurrent.futures.ThreadPoolExecutor的工作原理，用一个仅作示范的例子启动那些几秒内什么都不做的任务，只是显示他们的时间戳。

Next we went back to the flag downloading examples. Enhancing them with a progress bar and proper error handling prompted further exploration of the future.as_completed generator function showing a common pattern: storing futures in a dict to link further information to them when submitting, so that we can use that information when the future comes out of the as_completed iterator.  
    之后我们继续会的flag下载的示例。用一个进度条和正确的error处理逻辑增强功能，促使对future.as_completed生成器函数的更深层次的探索展示出的共同特点：在一个字典中保存futures，以便在提交时链接其对应的信息，所以当future从as_completed迭代器中出来时，我们一个可以使用这些信息。

We concluded the coverage of concurrency with threads and processes with a brief reminder of the lower-level, but more flexible threading and multiprocessing modules, which represent the traditional way of leveraging threads and processes in Python.  
    我们总结了线程和进程在并发的覆盖性，并简单介绍了低级但更灵活的threading和multiprocessing模块，他们代表着Python中充分利用线程和进程的传统方式。


## Further Reading
## 延伸阅读

The concurrent.futures package was contributed by Brian Quinlan, who presented it in a great talk titled “The Future Is Soon!” at PyCon Australia 2010. Quinlan’s talk has no slides; he shows what the library does by typing code directly in the Python console.  

As a motivating example, the presentation features a short video with XKCD cartoonist/programmer Randall Munroe making an unintended DOS attack on Google Maps to build a colored map of driving times around his city. The formal introduction to the library is PEP 3148 - futures - execute computations asynchronously. In the PEP, Quinlan wrote that the concurrent.futures library was “heavily influenced by the Java java.util.concurrent package.”  

Parallel Programming with Python (Packt), by Jan Palach, covers several tools for concurrent programming, including the concurrent.futures, threading, and multiprocessing modules. It goes beyond the standard library to discuss Celery, a task queue used to distribute work across threads and processes, even on different machines. In the Django community, Celery is probably the most widely used system to offload heavy tasks such as PDF generation to other processes, thus avoiding delays in producing an HTTP response.  

In the Beazley and Jones Python Cookbook, 3E (O’Reilly) there are recipes using concurrent.futures starting with “Recipe 11.12. Understanding Event-Driven I/O.” “Recipe 12.7. Creating a Thread Pool” shows a simple TCP echo server, and “Recipe 12.8. Performing Simple Parallel Programming” offers a very practical example: analyzing a whole directory of gzip compressed Apache logfiles with the help of a ProcessPoolExecutor. For more about threads, the entire Chapter 12 of Beazley and Jones is great, with special mention to “Recipe 12.10. Defining an Actor Task,” which demonstrates the Actor model: a proven way of coordinating threads through message passing.  

Brett Slatkin’s Effective Python (Addison-Wesley) has a multitopic chapter about concurrency, including coverage of coroutines, concurrent.futures with threads and processes, and the use of locks and queues for thread programming without the ThreadPoolExecutor.  

High Performance Python (O’Reilly) by Micha Gorelick and Ian Ozsvald and The Python Standard Library by Example (Addison-Wesley), by Doug Hellmann, also cover threads and processes.  

For a modern take on concurrency without threads or callbacks, Seven Concurrency Models in Seven Weeks, by Paul Butcher (Pragmatic Bookshelf) is an excellent read. I love its subtitle: “When Threads Unravel.” In that book, threads and locks are covered in Chapter 1, and the remaining six chapters are devoted to modern alternatives to concurrent programming, as supported by different languages. Python, Ruby, and JavaScript are not among them.  

If you are intrigued about the GIL, start with the Python Library and Extension FAQ(“Can’t we get rid of the Global Interpreter Lock?”). Also worth reading are posts by Guido van Rossum and Jesse Noller (contributor of the multiprocessing package): “It isn’t Easy to Remove the GIL” and “Python Threads and the Global Interpreter Lock.” Finally, David Beazley has a detailed exploration on the inner workings of the GIL: “Understanding the Python GIL.”[8]  In slide #54 of the presentation, Beazley reports some alarming results, including a 20× increase in processing time for a particular benchmark with the new GIL algorithm introduced in Python 3.2. However, Beazley apparently used an empty while True: pass to simulate CPU-bound work, and that is not realistic. The issue is not significant with real workloads, according to a comment by Antoine Pitrou—who implemented the new GIL algorithm—in the bug report submitted by Beazley.  

While the GIL is real problem and is not likely to go away soon, Jesse Noller and Richard Oudkerk contributed a library to make it easier to work around it in CPU-bound applications: the multiprocessing package, which emulates the threading API across processes, along with supporting infrastructure of locks, queues, pipes, shared memory,etc. The package was introduced in PEP 371 — Addition of the multiprocessing package to the standard library. The official documentation for the package is a 93 KB .rst file—that’s about 63 pages—making it one of the longest chapters in the Python standard library. Multiprocessing is the basis for the concurrent.futures.ProcessPoolExecutor.  

For CPU- and data-intensive parallel processing, a new option with a lot of momentum in the big data community is the Apache Spark distributed computing engine, offering a friendly Python API and support for Python objects as data, as shown in their examples page.  

Two elegant and super easy libraries for parallelizing tasks over processes are lelo by João S. O. Bueno and python-parallelize by Nat Pryce. The lelo package defines a @parallel decorator that you can apply to any function to magically make it unblocking: when you call the decorated function, its execution is started in another process. Nat Pryce’s python-parallelize package provides a parallelize generator that you can use to distribute the execution of a for loop over multiple CPUs. Both packages use the multiprocessing module under the covers.









