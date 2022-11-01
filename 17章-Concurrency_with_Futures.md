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

## A Sequential Download Script
Example 17-2 is not very interesting, but we’ll reuse most of its code and settings to implement the concurrent scripts, so it deserves some attention.

    For clarity, there is no error handling in Example 17-2. We will deal with exceptions later, but here we want to focus on the basic structure of the code, to make it easier to contrast this script with the concurrent ones.

Example 17-2. flags.py: sequential download script; some functions will be reused by the other scripts
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
2. List of the ISO 3166 country codes for the 20 most populous countries in order of decreasing population.
3. The website with the flag images.[2]
4. Local directory where the images are saved.
5. Simply save the img (a byte sequence) to filename in the DEST_DIR.
6. Given a country code, build the URL and download the image, returning the binary contents of the response.
7. Display a string and flush sys.stdout so we can see progress in a one-line display; this is needed because Python normally waits for a line break to flush the stdout buffer. 
8. download_many is the key function to compare with the concurrent implementations.
9. Loop over the list of country codes in alphabetical order, to make it clear that the ordering is preserved in the output; return the number of country codes downloaded.
10. main records and reports the elapsed time after running download_many.
11. main must be called with the function that will make the downloads; we pass the download_many function as an argument so that main can be used as a library function with other implementations of download_many in the next examples.  

[2]The images are originally from the CIA World Factbook, a public-domain, U.S. government publication. I copied them to my site to avoid the risk of launching a DOS attack on CIA.gov.


    The requests library by Kenneth Reitz is available on PyPI and is more powerful and easier to use than the urllib.request module from the Python 3 standard library. In fact, requests is considered a model Pythonic API. It is also compatible with Python 2.6 and up, while the urllib2 from Python 2 was moved and renamed in Python 3, so it’s more convenient to use requests regardless of the Python version you’re targeting.*

There’s really nothing new to flags.py. It serves as a baseline for comparing the other scripts and I used it as a library to avoid redundant code when implementing them. Now let’s see a reimplementation using concurrent.futures.

## Downloading with concurrent.futures

The main features of the concurrent.futures package are the ThreadPoolExecutor and ProcessPoolExecutor classes, which implement an interface that allows you to submit callables for execution in different threads or processes, respectively. The classes manage an internal pool of worker threads or processes, and a queue of tasks to be executed. But the interface is very high level and we don’t need to know about any of those details for a simple use case like our flag downloads.

Example 17-3 shows the easiest way to implement the downloads concurrently, using the ThreadPoolExecutor.map method.

Example 17-3. flags_threadpool.py: threaded download script using futures.ThreadPoolExecutor
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
2. Maximum number of threads to be used in the ThreadPoolExecutor.
3. Function to download a single image; this is what each thread will execute.
4. Set the number of worker threads: use the smaller number between the maximum we want to allow (MAX_WORKERS) and the actual items to be processed, so no unnecessary threads are created.
5. Instantiate the ThreadPoolExecutor with that number of worker threads; the executor.__exit__ method will call executor.shutdown(wait=True), which will block until all threads are done.
6. The map method is similar to the map built-in, except that the download_one function will be called concurrently from multiple threads; it returns a generator that can be iterated over to retrieve the value returned by each function.
7. Return the number of results obtained; if any of the threaded calls raised an exception, that exception would be raised here as the implicit next() call tried to retrieve the corresponding return value from the iterator.
8. Call the main function from the flags module, passing the enhanced version of download_many.

Note that the download_one function from Example 17-3 is essentially the body of the for loop in the download_many function from Example 17-2. This is a common refactoring when writing concurrent code: turning the body of a sequential for loop into a function to be called concurrently.

The library is called concurrency.futures yet there are no futures to be seen in Example 17-3, so you may be wondering where they are. The next section explains.


