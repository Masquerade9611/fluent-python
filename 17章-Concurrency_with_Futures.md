# Chapter 17 Concurrcy with Futures
# 第17章 

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

## Example: Web Downloads in Three Styles
## 示例： 三种风格的Web下载

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

## A Sequential Download Script
## 顺序下载的脚本
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

## Downloading with concurrent.futures
## 使用concurrent.futures下载

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
    <!-- 注意示例17-3的download_one函数本质上是 -->

The library is called concurrency.futures yet there are no futures to be seen in Example 17-3, so you may be wondering where they are. The next section explains.


