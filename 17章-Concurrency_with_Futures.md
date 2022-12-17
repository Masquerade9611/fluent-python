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
    并发脚本间的性能差异并不明显，但他们都要比顺序脚本要快5秒——并且这只是一个相当小的任务。如果你将该任务分配至梳百词的下载，那么并发脚本的速度可以超过顺序脚本的1/20甚至更多。

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
    标准库中的任何阻塞I/O函数都会释放GIL，以运行其他线程继续运行。time.sleep()函数也会释放GIL。因此，尽管存在GIL，Python线程仍然完美地适用于I/O限制的应用中。

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
    穿件一个3线程的ThreadPoolExecutor。
5. Submit five tasks to the executor (because there are only three threads, only three of those tasks will start immediately: the calls loiter(0), loiter(1), and loiter(2)); this is a nonblocking call.  
    向executor提交5个任务（因为只有3个线程，所以这些任务中的3个会立刻启动：调用loiter(0)，loiter(1)和loiter(2))；这是非阻塞的调用。
6. Immediately display the results of invoking executor.map: it’s a generator, as the output in Example 17-7 shows.  
    立刻显示executor.map的结果：这是一个生成器，和示例17-7的输出一致。
7. The enumerate call in the for loop will implicitly invoke next(results), which in turn will invoke _f.result() on the (internal) _f future representing the first call, loiter(0). The result method will block until the future is done, therefore each iteration in this loop will have to wait for the next result to be ready.  
    for循环中调用到的enumerate将会隐式地调用next(results)，即转而调用（内置的）_f future上的_f.result()，代表第一次调用，loiter(0)。直到futrue结束前result方法将会阻塞，因此该循环中的任一项必须等待下一个result准备完毕。

I encourage you to run Example 17-6 and see the display being updated incrementally. While you’re at it, play with the max_workers argument for the ThreadPoolExecutor and with the range function that produces the arguments for the executor.map call—or replace it with lists of handpicked values to create different delays.

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
2. The first thread executes loiter(0), so it will sleep for 0s and return even before the second thread has a chance to start, but YMMV.5
3. loiter(1) and loiter(2) start immediately (because the thread pool has three workers, it can run three functions concurrently).
4. This shows that the results returned by executor.map is a generator; nothing so far would block, regardless of the number of tasks and the max_workers setting.
5. Because loiter(0) is done, the first worker is now available to start the fourth thread for loiter(3).
6. This is where execution may block, depending on the parameters given to the loiter calls: the __next__ method of the results generator must wait until the first future is complete. In this case, it won’t block because the call to loiter(0) finished before this loop started. Note that everything up to this point happened within the same second: 15:56:50.
7. loiter(1) is done one second later, at 15:56:51. The thread is freed to start loiter(4).
8. The result of loiter(1) is shown: 10. Now the for loop will block waiting for the result of loiter(2).
9. The pattern repeats: loiter(2) is done, itsresult isshown;samewith loiter(3).
10. There is a 2s delay until loiter(4) is done, because it started at 15:56:51 and did nothing for 4s.

The Executor.map function is easy to use but it has a feature that may or may not be helpful, depending on your needs: it returns the results exactly in the same order as the calls are started: if the first call takes 10s to produce a result, and the others take 1s each, your code will block for 10s as it tries to retrieve the first result of the generator returned by map. After that, you’ll get the remaining results without blocking because they will be done. That’s OK when you must have all the results before proceeding, but often it’s preferable to get the results as they are ready, regardless of the order they were submitted. To do that, you need a combination of the Executor.submit method and the futures.as_completed function, as we saw in Example 17-4. We’ll come back to this technique in “Using futures.as_completed” on page 527.

    The combination of executor.submit and futures.as_completed is more flexible than executor.map because you can submit different callables and arguments, while executor.map is designed to run the same callable on the different arguments. In addition, the set of futures you passto futures.as_completed may come from more than one executor—perhaps some were created by a ThreadPoolExecutor instance while others are from a ProcessPoolExecutor.

In the next section, we will resume the flag download examples with new requirements that will force us to iterate over the results of futures.as_completed instead of using executor.map.







