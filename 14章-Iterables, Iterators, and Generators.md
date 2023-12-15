# CHAPTER 14 Iterables, Iterators, and Generators 第十四章 可迭代对象，迭代器与生成器

*When I see patterns in my programs, I consider it a sign of trouble. The shape of a program should reflect only the problem it needs to solve. Any other regularity in the code is a sign, to me at least, that I’m using abstractions that aren’t powerful enough—often that I’m generating by hand the expansions of some macro that I need to write.[1] — Paul Graham  Lisp hacker and venture capitalist*
*当我在我的程序中看到模式时，我认为这是麻烦的迹象。程序的形状应该仅反映出他需要解决的问题。代码中任何其他规律性都是一个标志，至少对我来说是这样，我使用的抽象不够强大——通常是我手动生成一些我需要编写的宏的扩展。[1]  —— Paul Graham Lisp黑客与风险投资家*

Iteration is fundamental to data processing. And when scanning datasets that don’t fit in memory, we need a way to fetch the items lazily, that is, one at a time and on demand. This is what the Iterator pattern is about. This chapter shows how the Iterator pattern is built into the Python language so you never need to implement it by hand.  
    迭代是基础的数据操作。当扫描那些不适合内存的数据集时，我们需要一种方式来延迟获取项目，即一次一个按需获取。这就是Iterator模式。本章展示Iterator模式是如何内建在Python语言中的，所以你不需要手动完成他。

Python does not have macros like Lisp (Paul Graham’s favorite language), so abstracting away the Iterator pattern required changing the language: the yield keyword was added in Python 2.2 (2001).[2] The yield keyword allows the construction of generators, which work as iterators.  
    Python没有像Lisp（Paul Graham最爱的语言）一样的宏，所以将Iterator模式抽象出来需要改变语言：关键字yield被追加至Python 2.2 (2001)。[2] 关键字yield允许构建用作迭代器的生成器。

    Every generator is an iterator: generators fully implement the iterator interface. But an iterator—as defined in the GoF book—retrieves items from a collection, while a generator can produce items “out of thin air.” That’s why the Fibonacci sequence generator is a common example: an infinite series of numbers cannot be stored in a collection. However, be aware that the Python community treats iterator and generator as synonyms most of the time.  
    每种生成器都是迭代器：生成器完全实现了迭代器接口。但迭代器（如GoF书中定义的一样）从集合中检索项，而生成器可以“凭空”生成项。这就是为什么斐波那契序列生成器是通用的例子：一个无穷数列不能存储在集合中。但是请注意，大部分时间中Python社区都将迭代器与生成器视作同义词。

[1]. From “Revenge of the Nerds”, a blog post.  
    [1]. 源自“Revenge of the Nerds”，一篇博文。
[2]. Python 2.2 users could use yield with the directive from __future__ import generators; yield became available by default in Python 2.3.  
    [2]. Python 2.2用户可以通过指令from __future__ import generators来使用yield；Python 2.3中yield即可默认使用。

Python 3 uses generators in many places. Even the range() built-in now returns a generator-like object instead of full-blown lists like before. If you must build a list from range, you have to be explicit (e.g., list(range(100))).  
    Python3在许多地方都使用了生成器。甚至内建的range()返回一个生成器——类对象，而不是像之前的完整的列表。如果你必须从range建立一个list，你必须明确注明（如 list(range(100))）。

Every collection in Python is iterable, and iterators are used internally to support:  
    Python中的每个集合都是可迭代的，迭代器在内部用于支持：

- for loops  
    for循环
- Collection types construction and extension  
    集合类型的构建与扩展
- Looping over text files line by line  
    逐行遍历文本文件  
- List, dict, and set comprehensions  
    列表，字典与集合理解
- Tuple unpacking  
    元组解包
- Unpacking actual parameters with \* in function calls  
    在函数调用中通过 \* 解包实际参数

This chapter covers the following topics:  
    本章涵盖如下主题：

- How the iter(…) built-in function is used internally to handle iterable objects  
    内建函数iter(…)是如何在内部被用于处理可迭代对象的
- How to implement the classic Iterator pattern in Python  
    如何在Python中实现经典Iterator模式  
- How a generator function works in detail, with line-by-line descriptions  
    生成器函数如何工作，逐行描述  
- How the classic Iterator can be replaced by a generator function or generator expression  
    经典Iterator如何被生成器函数或表达式替代  
- Leveraging the general-purpose generator functions in the standard library  
    利用标准库中的通用生成器函数  
- Using the new yield from statement to combine generators  
    使用新的yield from声明结合生成器
- A case study: using generator functions in a database conversion utility designed to work with large datasets  
    案例研究：在设计用于处理大型数据集的数据库转换实用程序中使用生成器函数  
- Why generators and coroutines look alike but are actually very different and should not be mixed  
    生成器与协程看起来一样，但实际非常不同且不能结合的原因

We’ll get started studying how the iter(…) function makes sequences iterable.  
    我们将从iter()函数如何使序列可迭代开始学习。

## Sentence Take #1: A Sequence of Words  句型#1：单词序列

We’ll start our exploration of iterables by implementing a Sentence class: you give its constructor a string with some text, and then you can iterate word by word. The first version will implement the sequence protocol, and it’s iterable because all sequences are iterable, as we’ve seen before, but now we’ll see exactly why.  
    我们通过实现一个Sentence类来开始我们对可迭代对象的探索：

Example 14-1 shows a Sentence class that extracts words from a text by index.  
    例14-1展示了Sentence类通过索引从文本中提取单词。

Example 14-1. sentence.py: A Sentence as a sequence of words  
    例14-1. sentence.py：由单词序列

```python
import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)  # 1

    def __getitem__(self, index):
        return self.words[index]  # 2
    
    def __len__(self): 
        return len(self.words)  # 3
 
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)  # 4
```

1. re.findall returns a list with all nonoverlapping matches of the regular expression, as a list of strings.  
    re.findall返回一个列表，包含该正则的所有非重叠匹配，作为字符串列表。
2. self.words holds the result of .findall, so we simply return the word at the given index.  
    self.words保存了.findall的结果，所以我们简单地用给定索引返回单词。
3. To complete the sequence protocol, we implement __len__—but it is not needed to make an iterable object.  
    为了完成序列的协议，我们实现了__len__，但这不是创建可迭代对象所必需的。
4. reprlib.repr is a utility function to generate abbreviated string representations of data structures that can be very large.[3]  
    reprlib.repr是一个实用的函数，用于生成可以特别大的数据结构的缩略字符串表示。[3]

By default, reprlib.repr limits the generated string to 30 characters. See the console session in Example 14-2 to see how Sentence is used.  
    默认情况下，reprlib.repr将生成的字符串限制为30个字符。观察例14-2的控制台会话中Sentence是如何使用的。

Example 14-2. Testing iteration on a Sentence instance  
    例14-2. Sentence实例的迭代测试

```
>>> s = Sentence('"The time has come," the Walrus said,') # 1
>>> s
Sentence('"The time ha... Walrus said,') # 2
>>> for word in s: # 3
... print(word)
The
time
has
come
the
Walrus
said
>>> list(s) # 4
['The', 'time', 'has', 'come', 'the', 'Walrus', 'said']
```

1. A sentence is created from a string.  
    通过一段字符串创建sentence。
2. Note the output of __repr__ using ... generated by reprlib.repr.  
    注意使用__repr__的输出...由reprlib.repr生成。
3. Sentence instances are iterable; we’ll see why in a moment.  
    Sentence实例是可迭代的；我们一会儿会看到原因。
4. Being iterable, Sentence objects can be used as input to build lists and other iterable types  
    为了可迭代，Sentence对象可以被用作输入至创建list或其他迭代类型。

In the following pages, we’ll develop other Sentence classes that pass the tests in Example 14-2. However, the implementation in Example 14-1 is different from all the others because it’s also a sequence, so you can get words by index:  
    在接下来的页面中，我将开发其他通过例14-2测试的Sentence类。但是例14-1中的实现与其他所有都不同，因为他也是一个序列，你可以通过索引来获取单词：


```
>>> s[0]
'The'
>>> s[5]
'Walrus'
>>> s[-1]
'said'
```

Every Python programmer knows that sequences are iterable. Now we’ll see precisely why.  
    每个Python开发者都知道徐磊是可迭代的。现在我们来看看为什么。

### Why Sequences Are Iterable: The iter Function  为什么序列是可迭代的：iter方法

Whenever the interpreter needs to iterate over an object x, it automatically calls iter(x).
The iter built-in function:  
    不论解释器何时需要遍历一个对象x，他都会自动调用iter(x)。

1. Checks whether the object implements __iter__, and calls that to obtain an iterator.  
    检查对象是否实现了__iter__，然后调用来获取一个生成器。
2. If __iter__ is not implemented, but __getitem__ is implemented, Python creates an iterator that attempts to fetch items in order, starting from index 0 (zero).  
    如果没有实现__iter__，而实现了__getitem__，Python会创建一个试图顺序获取项目的生成器，从index 0开始。
3. If that fails, Python raises TypeError, usually saying “C object is not iterable,” where C is the class of the target object.  
    如果这失败了，Python会抛出TypeError，通过显示“C object is not iterable”，这里C是目标对象的类。

That is why any Python sequence is iterable: they all implement __getitem__. In fact, the standard sequences also implement __iter__, and yours should too, because the special handling of __getitem__ exists for backward compatibility reasons and may be gone in the future (although it is not deprecated as I write this).  
    这就是为什么Python序列是可迭代的：他们全部实现了__getitem__。事实上，标准的序列通用实现__iter__，你的序列也同样，因为__getitem__的特殊处理的存在是出于向后兼容的原因，在未来可能会消失（虽然当我写到这的时候还没有被弃用）。
As mentioned in “Python Digs Sequences” on page 310, this is an extreme form of duck typing: an object is considered iterable not only when it implements the special method __iter__, but also when it implements __getitem__, as long as __getitem__ accepts int keys starting from 0.  
    如310页“Python Digs Sequences”中提醒的一样，这是鸭子类型的一种极端形式：一个对象被认作可迭代，不仅在于他实现了特殊方法__iter__，而且当他实现了__getitem__时，只要__getitem__接收从0开始的int键。

In the goose-typing approach, the definition for an iterable is simpler but not as flexible: an object is considered iterable if it implements the __iter__ method. No subclassing or registration is required, because abc.Iterable implements the __subclasshook__, as seen in “Geese Can Behave as Ducks” on page 338. Here is a demonstration:  
    在鹅型方法中，一个可迭代对象的定义更简单，但不那么灵活：如果一个对象实现了__iter__方法，他就被认作可迭代对象。不需要子类或登记，因为abc.Iterable实现了__subclasshoot__，如338页的“Gesse Can Behave as Ducks”中所见。下面是演示：

```
>>> class Foo:
... def __iter__(self):
... pass
...
>>> from collections import abc
>>> issubclass(Foo, abc.Iterable)
True
>>> f = Foo()
>>> isinstance(f, abc.Iterable)
True
```

However, note that our initial Sentence class does not pass the issubclass(Sentence, abc.Iterable) test, even though it is iterable in practice.  
    但是，请注意我们的初始Sentence类不能通过issubclass(Sentence, abc.Iterable)测试，即使它在实践中是可迭代的。

    As of Python 3.4, the most accurate way to check whether an object x is iterable is to call iter(x) and handle a TypeError exception if it isn’t. This is more accurate than using isinstance(x, abc.Iterable), because iter(x) also considers the legacy __getitem__ method, while the Iterable ABC does not.  
    截至Python3.4，检测一个对象x是否可迭代的最准确方式是调用iter(x)，如果不是的话处理一个TypeError异常。这相比使用isinstance(x, abc.Iterable)更准确一点，因为iter(x)也考虑了遗留__getitem__反复，而Iterable ABC没有。

Explicitly checking whether an object is iterable may not be worthwhile if right after the check you are going to iterate over the object. After all, when the iteration is attempted on a noniterable, the exception Python raises is clear enough: TypeError: 'C' object is not iterable . If you can do better than just raising TypeError, then do so in a try/except block instead of doing an explicit check. The explicit check may make sense if you are holding on to the object to iterate over it later; in this case, catching the error early may be useful.  
    如果检查准确后立刻迭代对象，那么准确检查他是否可迭代可能并不重要。毕竟，当迭代过程是在非迭代对象上进行的话，Python抛出的异常也足够清晰：TypeError: 'C' object is not iterable。如果你想要比仅抛出TypeError更进一步，那么就在try/except块中做，而不是显式地检测。如果您持有该对象以便稍后对其进行迭代，则显式检查可能有意义； 在这种情况下，尽早发现错误可能会有用。

The next section makes explicit the relationship between iterables and iterators.  
    下一节来明确可迭代对象与迭代器间的关系。

## Iterables Versus Iterators

From the explanation in “Why Sequences Are Iterable: The iter Function” on page 404 we can extrapolate a definition:  
    从404页“Why Sequences Are Iterable: The iter Function”的解释来看，我们可以推断出定义：
iterable
    Any object from which the iter built-in function can obtain an iterator. Objects implementing an __iter__ method returning an iterator are iterable. Sequences are always iterable; as are objects implementing a __getitem__ method that takes 0-based indexes.  
    iter内建函数可以从中获取迭代器的任何对象。实现了返回迭代器的__iter__函数的对象是可迭代对象。序列总是可迭代的；实现__getitem__函数的对象也是如此，该方法使用了基于0的索引。

It’s important to be clear about the relationship between iterables and iterators: Python obtains iterators from iterables.  
    搞清楚iterable与iterator的关系很重要：Python从可迭代对象中获取迭代器。

Here is a simple for loop iterating over a str. The str 'ABC' is the iterable here. You don’t see it, but there is an iterator behind the curtain:  
    这里是一个遍历字符串的简单for循环。字符串'ABC'是一个可迭代对象。你看不到他，但这里隐含了一个迭代器：

```
>>> s = 'ABC'
>>> for char in s:
...     print(char)
...
A
B
C
```

If there was no for statement and we had to emulate the for machinery by hand with a while loop, this is what we’d have to write:  
    如果没有for语句，我们则不得不用while循环手动模拟for的机制，这就是我们必须写的：

```
>>> s = 'ABC'
>>> it = iter(s) # 1
>>> while True:
...     try:
...         print(next(it)) # 2
...     except StopIteration: # 3
...         del it # 4
...         break # 5
...
A
B
C
```

1. Build an iterator it from the iterable.  
    通过可迭代对象构建一个迭代器。
2. Repeatedly call next on the iterator to obtain the next item.  
    重复调用迭代器的next来获取下一项。
3. The iterator raises StopIteration when there are no further items.  
    当没有其他项时，迭代器抛出StopIteration。
4. Release reference to it—the iterator object is discarded.  
    释放对他的引用——迭代器对象被丢弃。
5. Exit the loop.  
    退出循环。

StopIteration signals that the iterator is exhausted. This exception is handled internally in for loops and other iteration contexts like list comprehensions, tuple unpacking, etc.  
    StopIteration标志着迭代器已枯竭。该异常被for循环及其他迭代上下文（如list推导，元组解包）中进行了内部处理。

The standard interface for an iterator has two methods:  
    迭代器的标准接口有两个函数：
__next__
    Returns the next available item, raising StopIteration when there are no more items.  
    返回下一个可用项，没有项时抛出StopIteration。

__iter__
    Returns self; this allows iterators to be used where an iterable is expected, for example, in a for loop.  
    返回本身；这使得迭代器被用在需要可迭代对象的地方，例如在for循环中。

This is formalized in the collections.abc.Iterator ABC, which defines the __next__ abstract method, and subclasses Iterable—where the abstract __iter__ method is defined. See Figure 14-1.  
    这在collections.abc.Iterator ABC中被形式化了，其定义了__next__抽象方法和Iterable的子类——其中定义了抽象__iter__方法。参加图示14-1。

Figure 14-1. The Iterable and Iterator ABCs. Methods in italic are abstract. A concrete Iterable.iter should return a new Iterator instance. A concrete Iterator must implement next. The Iterator.iter method just returns the instance itself.  

The Iterator ABC implements __iter__ by doing return self. This allows an iterator to be used wherever an iterable is required. The source code for abc.Iterator is in Example 14-3.  
    Iterator ABC通过返回他本身实现了__iter__。这使得一个迭代器被用于需要可迭代对象的任何地方。例14-3是abc.Iterator的源码。

Example 14-3. abc.Iterator class; extracted from Lib/_collections_abc.py  
    例14-3. abc.Iterator类；摘自Lib/_collections_abc.py

```python
class Iterator(Iterable):
    __slots__ = ()
    @abstractmethod
    def __next__(self):
        'Return the next item from the iterator. When exhausted, raise StopIteration'
        raise StopIteration

    def __iter__(self):
        return self
    @classmethod
    def __subclasshook__(cls, C):
        if cls is Iterator:
            if (any("__next__" in B.__dict__ for B in C.__mro__) and
                any("__iter__" in B.__dict__ for B in C.__mro__)):
                return True
        return NotImplemented
```

    The Iterator ABC abstract method is it.__next__() in Python 3 and it.next() in Python 2. As usual, you should avoid calling special methods directly. Just use the next(it): this built-in function does the right thing in Python 2 and 3.  
    Iterator ABC 抽象方法在Python3中是it.__next__()，在Python2中是it.next()。通常你应该避免直接调用魔术方法。只要使用next(it)：这种内建方法在Python2和3中都是正确的。

The Lib/types.py module source code in Python 3.4 has a comment that says:
    # Iterators in Python aren't a matter of type but of protocol. A large
    # and changing number of builtin types implement *some* flavor of
    # iterator. Don't check the type! Use hasattr to check for both
    # "__iter__" and "__next__" attributes instead.
    Python3.4中Lib/types.py模块源码有着这样的注释：
        Python中的Iterator不是类型问题，而是协议问题。大量且不断变化的内置类型实现了某些风格的迭代器。不要检测类型，使用hasattr来检查“__iter__”和“__next__”属性。

In fact, that’s exactly what the __subclasshook__ method of the abc.Iterator ABC does (see Example 14-3).  
    事实上，这就是abc.Iterator ABC的__subclasshook__方法实际做的事情（见例14-3）。

    Taking into account the advice from Lib/types.py and the logic implemented in Lib/_collections_abc.py, the best way to check if an object x is an iterator is to call isinstance(x, abc.Iterator). Thanks to Iterator.__subclasshook__, this test works even if the class of x is not a real or virtual subclass of Iterator.  
    考虑到Lib/types.py中的建议与Lib/_collections_abc.py中实现的逻辑，检查对象x是否为迭代器的最好方式是调用isinstance(x, acd.Iterator)。感谢Iterator.__subclasshoot__，即使x的类不是Iterator的真实或虚拟子类，该测试也有效。

Back to our Sentence class from Example 14-1, you can clearly see how the iterator is built by iter(…) and consumed by next(…) using the Python console:  
    回到我们例14-1的Sentence类，你可以使用Python控制台清楚看到iterator是如何通过iter(…)所构建并由next(…)所调用的。

```
>>> s3 = Sentence('Pig and Pepper') # 1
>>> it = iter(s3) # 2
>>> it # doctest: +ELLIPSIS
<iterator object at 0x...>
>>> next(it) # 3
'Pig'
>>> next(it)
'and'
>>> next(it)
'Pepper'
>>> next(it) # 4
Traceback (most recent call last):
    ...
StopIteration
>>> list(it) # 5
[]
>>> list(iter(s3)) # 6
['Pig', 'and', 'Pepper']
```

1. Create a sentence s3 with three words.  
    创建一个三个单词的sentence s3。
2. Obtain an iterator from s3.  
    通过s3获取一个迭代器。
3. next(it) fetches the next word.  
    next(it)获取到下一个单词。
4. There are no more words, so the iterator raises a StopIteration exception.  
    没有更多的单词，所以迭代器抛出StopIteration异常。
5. Once exhausted, an iterator becomes useless.  
    一旦耗尽，迭代器就变为不可用。
6. To go over the sentence again, a new iterator must be built.  
    为了再过一遍句子，则必须创建新的迭代器。

Because the only methods required of an iterator are __next__ and __iter__, there is no way to check whether there are remaining items, other than to call next() and catch StopInteration. Also, it’s not possible to “reset” an iterator. If you need to start over, you need to call iter(…) on the iterable that built the iterator in the first place. Calling iter(…) on the iterator itself won’t help, because—as mentioned—Iterator.__iter__ is implemented by returning self, so this will not reset a depleted iterator.  
    因为迭代器必需的方法只有__next__和__iter__，没有办法检查是否有剩余的项，除非调用next()，然后捕获StopIteration。此外，“重置”一个迭代器也是不可能实现的。如果你想从头开始，则需要首先对构建迭代器的可迭代对象调用iter()。对迭代器本身调用iter()不起作用，因为——如上所述——Iterator.__iter__通过返回他本身来实现，所以这不会重置一个耗尽了的迭代器。

To wrap up this section, here is a definition for iterator:  
    总结本节，这里是迭代器的定义：
iterator
    Any object that implements the __next__ no-argument method that returns the next item in a series or raises StopIteration when there are no more items. Python iterators also implement the __iter__ method so they are iterable as well.  
    实现了无参数的__next__方法的任何对象，该方法会返回一个序列中的下一项，或是没有更多项时抛出StopIteration。此外Python迭代器还实现了__iter__方法，所以他也是可迭代对象。

This first version of Sentence was iterable thanks to the special treatment the iter(…) built-in gives to sequences. Now we’ll implement the standard iterable protocol.  
    Sentence的首个版本是可迭代的，这归功于内置iter()对序列的特殊处理。现在我们将实现标准的可迭代对象协议。

## Sentence Take #2: A Classic Iterator 经典迭代器

The next Sentence class is built according to the classic Iterator design pattern following the blueprint in the GoF book. Note that this is not idiomatic Python, as the next refactorings will make very clear. But it serves to make explicit the relationship between the iterable collection and the iterator object.  
    下一个Sentence类根据经典Iterator定义模式按照GoF书中的蓝图构建的。注意这不是地道的Python，下面的重构将非常清晰地说明。但他用于明确可迭代对象集合与迭代器对象之间的关系。

Example 14-4 shows an implementation of a Sentence that is iterable because it implements the __iter__ special method, which builds and returns a SentenceIterator. This is how the Iterator design pattern is described in the original Design Patterns book.  
    例14-4展示了一种可迭代的Sentence的实现，因为他实现了__iter__方法，该方法构建并返回一个SentenceIterator。这就是原始的设计模式书中描述Iterator设计模式的方式。

We are doing it this way here just to make clear the crucial distinction between an iterable and an iterator and how they are connected.  
    我们这样做只是为了理清可迭代对象和迭代器直接最关键的差异，以及他们是如何连接的。

Example 14-4. sentence_iter.py: Sentence implemented using the Iterator pattern  
    例14-4. sentence_iter.py: 使用Iterator模式实现的Sentence

```python
import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)
    
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)
 
    def __iter__(self):  # 1
        return SentenceIterator(self.words)  # 2


class SentenceIterator:
    def __init__(self, words):
        self.words = words  # 3
        self.index = 0  # 4
    
    def __next__(self):
    try:
        word = self.words[self.index]  # 5
    except IndexError:
        raise StopIteration()  # 6
    self.index += 1  # 7

    return word  # 8

    def __iter__(self):  # 9
        return self
```

1. The __iter__ method is the only addition to the previous Sentence implementation. This version has no __getitem__, to make it clear that the class is iterable because it implements __iter__.  
    __iter__是相比之前实现的Sentence唯一增加的方法。该版本没有__getitem__，以明确表明该类是可迭代对象，以为他实现了__iter__。

2. __iter__ fulfills the iterable protocol by instantiating and returning an iterator.  
    __iter__通过实例化并返回一个迭代器，满足了可迭代对象协议。

3. SentenceIterator holds a reference to the list of words.  
    SentenceIterator保存了words列表的引用。

4. self.index is used to determine the next word to fetch.  
    self.index用于决定下一个获取的word。

5. Get the word at self.index.  
    获取self.index位置的word。

6. If there is no word at self.index, raise StopIteration.  
    如果self.index位置没有word，抛出StopIeration。

7. Increment self.index.
    增加self.index。

8. Return the word.  
    返回word。

9. Implement self.__iter__.  
    实现self.__iter__。

The code in Example 14-4 passes the tests in Example 14-2.  
    例14-4的代码通过了14-2的测试。

Note that implementing __iter__ in SentenceIterator is not actually needed for this example to work, but the it’s the right thing to do: iterators are supposed to implement both __next__ and __iter__, and doing so makes our iterator pass the issubclass(SentenceIterator, abc.Iterator) test. If we had subclassed SentenceIterator from abc.Iterator, we’d inherit the concrete abc.Iterator.__iter__ method.  
    注意，SentenceIterator中实现的__iter__实际上并不是该示例工作所需要的，而是因为正确而去做的：迭代器都应该实现__next__和__iter__，这样做可以使我们的迭代器可以通过issubclass(SentenceIterator, abc.Iterator)的测试。如果我们从abc.Iterator继承了SentenceIterator，我们将继承具体的abc.Iterator.__iter__方法。

That is a lot of work (for us lazy Python programmers, anyway). Note how most code in SentenceIterator deals with managing the internal state of the iterator. Soon we’ll see how to make it shorter. But first, a brief detour to address an implementation shortcut that may be tempting, but is just wrong.  
    这是很繁重的工作（不论怎样，对我们的懒惰的Python程序员来说）。注意SentenceIterator中大部分的代码如何处理管理迭代器的内部状态。很快我们将看到怎样让他变得更短。但首先，绕个弯路来解决可能很诱人但却是错误的实施捷径。

### Making Sentence an Iterator: Bad Idea 将Sentence改做迭代器：是个馊主意

A common cause of errors in building iterables and iterators is to confuse the two. To be clear: iterables have an __iter__ method that instantiates a new iterator every time. Iterators implement a __next__ method that returns individual items, and an __iter__ method that returns self.  
    在构建可迭代对象与迭代器时有误的通常原因是混淆了二者。要清楚：可迭代对象拥有__iter__方法，这可以随时实例化出一个新的迭代器。而迭代器实现了返回单个项的__next__方法，与返回他本身的__iter__方法。

Therefore, iterators are also iterable, but iterables are not iterators. It may be tempting to implement __next__ in addition to __iter__ in the Sentence class, making each Sentence instance at the same time an iterable and iterator over itself. But this is a terrible idea. It’s also a common anti-pattern, according to Alex Martelli who has a lot of experience with Python code reviews.  
    因此，迭代器也是可迭代对象，而可迭代对象不是迭代器。在Sentence类中除了__iter__还实现__next__这可能很诱人，这让每个Sentence实例同时成为可迭代对象和自身的迭代器。

The “Applicability” section[4] of the Iterator design pattern in the GoF book says:  
    GoF book中Iterator设计模式中“适用性”小节这样写到：

    Use the Iterator pattern  
        - to access an aggregate object’s contents without exposing its internal representation.  
            访问聚合对象的内容而不暴露其内部表示。
        - to support multiple traversals of aggregate objects.  
            支持对局核对下的多次遍历。
        - to provide a uniform interface for traversing different aggregate structures (that is, to support polymorphic iteration).  
            为遍历不同聚合结构提供统一的接口（也就是说，支持多态迭代）。

To “support multiple traversals” it must be possible to obtain multiple independent iterators from the same iterable instance, and each iterator must keep its own internal state, so a proper implementation of the pattern requires each call to iter(my_iterable) to create a new, independent, iterator. That is why we need the SentenceIterator class in this example.  
    为了“支持多重遍历”，必须能从同一个可迭代实例获取多个独立的迭代器，每个迭代器必须保持它自己内部的状态，所以该模式的正确实现需要每个都去调用iter(my_iterable)来创建一个新的，独立的迭代器。这就是我们在该示例中需要SentenceIterator类的原因。

    An iterable should never act as an iterator over itself. In other words, iterables must implement __iter__, but not __next__. On the other hand, for convenience, iterators should be iterable. An iterator’s __iter__ should just return self.  
        可迭代对象永远不应该充当其自身的迭代器。换句话说，可迭代对象必须实现__iter__，但不能实现__next__。另一方面，为了方便，迭代器应该是可迭代的。迭代器的__iter__应该只返回他本身。

Now that the classic Iterator pattern is properly demonstrated, we can get let it go. The next section presents a more idiomatic implementation of Sentence.  
    既然经典的迭代器模式已经得到了正确的演示，我们就可以开始下一段了。下一节会展示Sentence的更常用实现。

## Sentence Take #3: A Generator Function 生成器函数

A Pythonic implementation of the same functionality uses a generator function to replace the SequenceIterator class. A proper explanation of the generator function comes right after Example 14-5.  
    相同功能的Python风格实现使用了生成器函数来替换SequenceIterator类。例14-5之后是对生成器函数的正确解释。

Example 14-5. sentence_gen.py: Sentence implemented using a generator function  
    例14-5 sentence_gen.py：使用生成器函数实现Sentence

```python

import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        for word in self.words:  # 1
            yield word  # 2
        return  # 3

# done!  # 4
```

1. Iterate over self.word.  
    遍历self.word。
2. Yield the current word.  
    产出当前的word。
3. This return is not needed; the function can just “fall-through” and return automatically. Either way, a generator function doesn’t raise StopIteration: it simply exits when it’s done producing values.[5]  
    该retuen不是必需的；这个函数可以“失败”并自动返回。无论哪种方法，生成器函数不能抛出StopIteration：当他完成生产value时就会退出。
4. No need for a separate iterator class!  
    不需要单独的迭代器类！

Here again we have a different implementation of Sentence that passes the tests in Example 14-2.  
    这里我们再次拥有了一个Sentence的不同实现，他通过了例14-2的测试。

Back in the Sentence code in Example 14-4, __iter__ called the SentenceIterator constructor to build an iterator and return it. Now the iterator in Example 14-5 is in fact a generator object, built automatically when the __iter__ method is called, because __iter__ here is a generator function.  
    回到14-4的Sentece代码，__iter__调用SentenceIterator构造函数来创建一个迭代器并返回。现在14-5的迭代器事实上是个生成器对象，当调用__iter__方法时自动构建，因为这里的__iter__是一个生成器方法。

A full explanation of generator functions follows.  
    下面是生成器函数的完整解释。

### How a Generator Function Works 生成器函数是如何工作的

Any Python function that has the yield keyword in its body is a generator function: a function which, when called, returns a generator object. In other words, a generator function is a generator factory.  
    任何函数体内存在yield关键词的Python函数都是生成器函数：一个函数，在被调用时返回一个生成器对象。换句话说，生成器函数是一个生成器工厂。

    The only syntax distinguishing a plain function from a generator function is the fact that the latter has a yield keyword somewhere in its body. Some argued that a new keyword like gen should be used for generator functions instead of def, but Guido did not agree. His arguments are in PEP 255 — Simple Generators.[6]
    区别普通函数与生成器函数唯一的语法就是后者函数体内的某处存在yield关键词。一些人认为对生成器函数应该使用像是gen的新关键词来取代def，但Guido并没有同意。他的论点在PEP 255中——Simple Generators。[6]

Here is the simplest function useful to demonstrate the behavior of a generator:[7]  
    这里是最简单的函数，可用于演示生成器的行为：[7]

```
>>> def gen_123(): # 1
...     yield 1 # 2
...     yield 2
...     yield 3
...
>>> gen_123 # doctest: +ELLIPSIS
<function gen_123 at 0x...> # 3
>>> gen_123() # doctest: +ELLIPSIS
<generator object gen_123 at 0x...> # 4
>>> for i in gen_123(): # 5
... print(i)
1
2
3
>>> g = gen_123() # 6
>>> next(g) # 7
1
>>> next(g)
2
>>> next(g)
3
>>> next(g) # 8
Traceback (most recent call last):
 ...
StopIteration

```

1. Any Python function that contains the yield keyword is a generator function.  
    任何包含yield关键词的Python函数就是生成器函数。
2. Usually the body of a generator function has loop, but not necessarily; here I just repeat yield three times.  
    通常生成器函数中会包含循环，但不是必需的；这里我只是重复yield了三次。
3. Looking closely, we see gen_123 is a function object.  
    仔细观察，我们看到gen_123是一个函数对象。
4. But when invoked, gen_123() returns a generator object.  
    但当调用后，gen_123()返回了生成器对象。
5. Generators are iterators that produce the values of the expressions passed to yield.  
    生成器是迭代器，产出传递给yield的表达式的值。
6. For closer inspection, we assign the generator object to g.  
    为了更细致地检查，我能将生成器对象赋值给g。
7. Because g is an iterator, calling next(g) fetches the next item produced by yield.  
    因为g是迭代器，调用next(g)获取到由yield产出的下一项。
8. When the body of the function completes, the generator object raises a StopIteration.  
    当函数体完成，生成器对象抛出StopIteration。

A generator function builds a generator object that wraps the body of the function. When we invoke next(…) on the generator object, execution advances to the next yield in the function body, and the next(…) call evaluates to the value yielded when the function body is suspended. Finally, when the function body returns, the enclosing generator object raises StopIteration, in accordance with the Iterator protocol.  
    生成器函数构建一个包装函数体的生成器对象。当我们对生成器对象调用next()，执行前进至函数体内的下一个yield，并且next()调用的计算结果为函数体suspended时生成的值。最终当函数体返回，按照迭代器协议关闭的生成器对象抛出StopIteration。

    I find it helpful to be strict when talking about the results obtained from a generator: I say that a generator yields or produces values. But it’s confusing to say a generator “returns” values. Functions return values. Calling a generator function returns a generator. A generator yields or produces values. A generator doesn’t “return” values in the usual way: the return statement in the body of a generator function causes StopIteration to be raised by the generator object.[8]  
    我发现在当谈论获取自生成器的结果时保持严格是有帮助的：我指的是生成器yield或产出的值。但说生成器“return”值是容易让人困惑的。
    函数返回值。调用生成器函数返回生成器。一个生成器yield或生产值。生成器不会以通常的方式“return”值：返回生成器函数体中的状态，因为生成器对象抛出StopIteration。[8]

Example 14-6 makes the interaction between a for loop and the body of the function more explicit.  
    例14-6使得for循环与函数体之间的交互更加明确。

Example 14-6. A generator function that prints messages when it runs
    例14-6. 运行时打印信息的生成器函数

```
>>> def gen_AB(): # 1
...     print('start')
...     yield 'A' # 2
...     print('continue')
...     yield 'B' # 3
...     print('end.') # 4
...
>>> for c in gen_AB(): # 5
...     print('-->', c) # 6
...
start # 7
--> A # 8
continue # 9
--> B # 10
end. # 11
>>> # 12
```

1. The generator function is defined like any function, but uses yield.  
    生成器函数像是任何函数一样定义，但使用yilde

2. The first implicit call to next() in the for loop at will print 'start' and stop at the first yield, producing the value 'A'.  
    for循环中对next()的第一次隐式调用将打印'start'然后停在第一个yield位置，产出值'A'。

3. The second implicit call to next() in the for loop will print 'continue' and stop at the second yield, producing the value 'B'.  
    第二次隐式调用将打印'continue'并停在第二个yield，产出值'B'。

4. The third call to next()will print 'end.' and fall through the end of the function body, causing the generator object to raise StopIteration.  
    第三次调用将打印'end.'并跌落至函数体的结尾，导致生成器对象抛出StopIteration。

5. To iterate, the for machinery does the equivalent of g = iter(gen_AB()) to get a generator object, and then next(g) at each iteration.  
    为了迭代，该for机制做着等价于g = iter(gen_AB())的操作来获取生成器对象，然后在每次迭代时执行next(g)。

6. The loop block prints --> and the value returned by next(g). But this output will be seen only after the output of the print calls inside the generator function.  
    该循环块打印: --> 以及next(g)返回的值。但该输出仅在生成器内部print调用输出后才能看到。

7. The string 'start' appears as a result of print('start') in the generator function body.  
    'start'作为生成器函数体中print('start')的结果出现。

8. yield 'A' in the generator function body produces the value A consumed by the for loop, which gets assigned to the c variable and results in the output --> A.  
    函数体内的yield 'a'产出了for循环消耗的值A，该值被赋值给变量c并产生输出 --> A。

9. Iteration continues with a second call next(g), advancing the generator function body from yield 'A' to yield 'B'. The text continue is output because of the second print in the generator function body.  
    迭代继续第二次的调用next(g)，将生成器函数体从yield 'A'前进至yield 'B'。由于函数体中的第二个print，文本继续输出。

10. yield 'B' produces the value B consumed by the for loop, which gets assigned to the c loop variable, so the loop prints --> B.  
    yield 'B'产出了被for循环消耗的值B，该值被赋值给c循环遍历，所以loop打印--> B。

11. Iteration continues with a third call next(it), advancing to the end of the body of the function. The text end. appears in the output because of the third print in the generator function body.  
    迭代继续第三次调用next(it)，推进至函数体的结尾。由于函数体中的第三个pritn，文本输出出现end.。

12. When the generator function body runs to the end, the generator object raises StopIteration. The for loop machinery catches that exception, and the loop terminates cleanly.  
    当生成器函数体运行至结尾，生成器对象抛出StopIteration。for循环机制捕捉该异常并干净的终止。

Now hopefully it’s clear how Sentence.__iter__ in Example 14-5 works: __iter__ is a generator function which, when called, builds a generator object that implements the iterator interface, so the SentenceIterator class is no longer needed.  
    希望现在大家都清楚了例14-5中Sentence.__iter__的工作逻辑：__iter__是一个生成器函数，不论在何时哪里调用都会构建一个实现了迭代器接口的生成器对象，所以不再需要SentenceIterator类。

This second version of Sentence is much shorter than the first, but it’s not as lazy as it could be. Nowadays, laziness is considered a good trait, at least in programming languages and APIs. A lazy implementation postpones producing values to the last possible moment. This saves memory and may avoid useless processing as well.  
    Sentence的第二版比第一版短得多，但它并没有那么懒。如今，懒惰被认为是个优秀的特点，至少在编程语言与API中是这样。懒惰的实现将生成value的操作推迟到最后的时刻，这节省了内存同时也可以避免无用的操作。

We’ll build a lazy Sentence class next.  
    下一节我们将构建一个懒惰的Sentence。

## Sentence Take #4: A Lazy Implementation 惰性实现

The Iterator interface is designed to be lazy: next(my_iterator) produces one item at a time. The opposite of lazy is eager: lazy evaluation and eager evaluation are actual technical terms in programming language theory.  
    Iterator接口被定义为lazy：next(my_iterator)一次只生成一项。lazy的对立面是eager：lazy evaluation与eager evaluation是编程语言理论中的实际技术用语。

Our Sentence implementations so far have not been lazy because the __init__ eagerly builds a list of all words in the text, binding it to the self.words attribute. This will entail processing the entire text, and the list may use as much memory as the text itself(probably more; it depends on how many nonword characters are in the text). Most of this work will be in vain if the user only iterates over the first couple words.  
    到目前为止我们的Sentence实现并不lazy，因为__iter__很“eager”地构建了文本中所有单词的列表，并将他绑定至self.words属性。这意味着将处理整个文本，并且列表可能用了和文本本身一样多的内存（有可能更多；这取决于文本中有多少非单词字符）。如果用户仅迭代前两个单词，那么大部分工作将是徒劳的。

Whenever you are using Python 3 and start wondering “Is there a lazy way of doing this?”, often the answer is “Yes.”  
    不论你何时使用Python3然后开始思考“这是一个lazy方式吗？”，通常回答都是“是的。”

The re.finditer function is a lazy version of re.findall which, instead of a list, returns a generator producing re.MatchObject instances on demand. If there are many matches, re.finditer saves a lot of memory. Using it, our third version of Sentence is now lazy: it only produces the next word when it is needed. The code is in Example 14-7.  
    re.finditer方法是re.findall的lazy版本，取代list而返回了生成器，该生成器会按需生成re.MatchObject实例。如果有许多匹配，re.finditer会节省许多内存。通过使用这个，我们的第三版Sentence现在就是lazy的：仅在被需要时才生成下一个单词。具体代码见例14-7。

Example 14-7. sentence_gen2.py: Sentence implemented using a generator function calling the re.finditer generator function  
    例14-7. sentence_gen2.py：使用了通过调用re.finditer生成器方法的生成器函数的Sentence。

```python
import re
import reprlib
RE_WORD = re.compile('\w+')

class Sentence:
    def __init__(self, text):
        self.text = text  # 1
 
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)
 
    def __iter__(self):
        for match in RE_WORD.finditer(self.text):  # 2
            yield match.group()  # 3
```

1. No need to have a words list.  
    不再需要单词列表。
2. finditer builds an iterator over the matches of RE_WORD on self.text, yielding MatchObject instances.  
    finditer在self.text的RE_WORD匹配基础上构建了一个迭代器，产出MatchObject实例。
3. match.group() extracts the actual matched text from the MatchObject instance.  
    match.group()从MatchObject实例中提取实际的匹配文本。

Generator functions are an awesome shortcut, but the code can be made even shorter with a generator expression.  
    生成器函数是一个很棒的快捷方式，但使用生成器表达式可以使代码变得更短。

## Sentence Take #5: A Generator Expression  生成器表达式

Simple generator functions like the one in the previous Sentence class (Example 14-7) can be replaced by a generator expression.  
    像上一个Sentence类（示例 14-7）中的简单生成器函数可以用生成器表达式替换。

A generator expression can be understood as a lazy version of a list comprehension: it does not eagerly build a list, but returns a generator that will lazily produce the items on demand. In other words, if a list comprehension is a factory of lists, a generator expression is a factory of generators.  
    生成器表达式可以被理解为列表推导式的lazy版本：他不会急忙构建一个列表，而是返回一个生成器，该生成器将在有需求的时候lazy地生成项。换句话说，如果一个列表表达式是列表的工厂，生成器表达式则是生成器的工厂。

Example 14-8 is a quick demo of a generator expression, comparing it to a list comprehension.  
    例14-8是生成器表达式的快速演示，与列表表达式进行了对比。

Example 14-8. The gen_AB generator function is used by a list comprehension, then by a generator expression  
    例14.8 gen_AB生成器函数被一条列表表达式使用，然后被生成器表达式使用。

```
>>> def gen_AB(): # 1
...     print('start')
...     yield 'A'
...     print('continue')
...     yield 'B'
...     print('end.')
...
>>> res1 = [x*3 for x in gen_AB()] # 2
start
continue
end.
>>> for i in res1: # 3
...     print('-->', i)
...
--> AAA
--> BBB
>>> res2 = (x*3 for x in gen_AB()) # 4
>>> res2 # 5
<generator object <genexpr> at 0x10063c240>
>>> for i in res2: # 6
...     print('-->', i)
...
start
--> AAA
continue
--> BBB
end.

```

1. This is the same gen_AB function from Example 14-6.  
    这与例14-6中gen_AB函数相同。
2. The list comprehension eagerly iterates over the items yielded by the generator object produced by calling gen_AB(): 'A' and 'B'. Note the output in the next lines: start, continue, end.  
    列表表达式急切地遍历了通过调用gen_AB()生成器对象产出的项："A"与"B"。注意后面几行的输出：start, continue, end。
3. This for loop is iterating over the res1 list produced by the list comprehension.  
    for循环在遍历由列表表达式生成的res1列表。
4. The generator expression returns res2. The call to gen_AB() is made, but that call returns a generator, which is not consumed here.  
    生成器表达式返回res2。调用gen_AB()，但该调用返回生成器，在这里没有消耗。
5. res2 is a generator object.  
    res2是生成器对象。
6. Only when the for loop iterates over res2, the body of gen_AB actually executes. Each iteration of the for loop implicitly calls next(res2), advancing gen_AB to the next yield. Note the output of gen_AB with the output of the print in the for loop.  
    只有当for循环遍历res2的时候，gen_AB函数体才实际执行。for循环的每次迭代都在隐式地调用next(res2)，推动gen_AB至下一个yield。注意gen_AB输出与for循环中print的输出。

So, a generator expression produces a generator, and we can use it to further reduce the code in the Sentence class. See Example 14-9.  
    所以，生成器表达式生产生成器，然后我们可以使用他来进一步减少Sentenc类的代码量。见例14-9。

Example 14-9. sentence_genexp.py: Sentence implemented using a generator expression  
    例14-9. sentence_genxp.py：通过生成器表达式实现的Sentence

```python
import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:
    def __init__(self, text):
        self.text = text

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        return (match.group() for match in RE_WORD.finditer(self.text))
```

The only difference from Example 14-7 is the __iter__ method, which here is not a generator function (it has no yield) but uses a generator expression to build a generator and then returns it. The end result is the same: the caller of __iter__ gets a generator object.  
    与例14-7唯一的不同是__iter__方法，这里不是生成器函数（他没有yield），而是用生成器表达式来构建一个生成器并在之后返回他。最终结果是相同的：__iter__的调用者会获得一个生成器对象。

Generator expressions are syntactic sugar: they can always be replaced by generator functions, but sometimes are more convenient. The next section is about generator expression usage.  
    生成器表达式是一个语法糖：他们总是能被生成器函数来替代，但有时更方便。下一节有关于生成器表达式的用法。

## Generator Expressions: When to Use Them 生成器表达式：使用时机

I used several generator expressions when implementing the Vector class in Example 10-16. Each of the methods __eq__, __hash__, __abs__, angle, angles, format, __add__, and __mul__ has a generator expression. In all those methods, a list comprehension would also work, at the cost of using more memory to store the intermediate list values.  
    当实现例10-16中的Vector类的时候，我使用了几个生成器表达式。每个方法__eq__, __hash__, __abs__, angle, angles, format, __add__与__mul__都有生成器表达式。所有这些方法，用列表推导也可以，但代价就是用更多内存来存储中间的列表值。

In Example 14-9, we saw that a generator expression is a syntactic shortcut to create a generator without defining and calling a function. On the other hand, generator functions are much more flexible: you can code complex logic with multiple statements, and can even use them as coroutines (see Chapter 16).  
    例14-9中，我们看到生成器表达式是脱离定义与调用函数来创建生成器的语法捷径。另一方面，生成器函数更加灵活：你可以用多个语句编写复杂的逻辑；甚至可以将他们用作协程（见16章）。

For the simpler cases, a generator expression will do, and it’s easier to read at a glance, as the Vector example shows.  
    对于更简单的情况，生成器表达式就可以了，而且更容易一目了然，如Vector示例所示。

My rule of thumb in choosing the syntax to use is simple: if the generator expression spans more than a couple of lines, I prefer to code a generator function for the sake of readability. Also, because generator functions have a name, they can be reused. You can always name a generator expression and use it later by assigning it to a variable, of course, but that is stretching its intended usage as a one-off generator.  
    在选择使用哪种语法上我的经验法则很简单：如果生成器表达式跨越了几行，我更倾向于写一个生成器函数，这样更易读。同样，因为生成器函数有名称，可以复用。你也可以为生成器表达式命名，通过将其赋值给一个变量以在后续使用，当然，这超出了他作为一个一次性生成器的预期用途。

    Syntax Tip 语法技巧
    When a generator expression is passed as the single argument to a function or constructor, you don’t need to write a set of parentheses for the function call and another to enclose the generator expression. A single pair will do, like in the Vector call from the __mul__ method in Example 10-16, reproduced here. However, if there are more function arguments after the generator expression, you need to enclose it in parentheses to avoid a SyntaxError:  
    当生成器表达式作为一个单参数被传给函数或构造函数，你不需要为函数调用写括号，也不需要为生成器表达式写另一组括号。一对就可以，就像示例 10-16 中 __mul__ 方法的 Vector 调用一样，在此处复制。 但是，如果生成器表达式后面有更多函数参数，则需要将其括在括号中以避免语法错误：
        def __mul__(self, scalar):
            if isinstance(scalar, numbers.Real):
                return Vector(n * scalar for n in self)
            else:
                return NotImplemented

The Sentence examples we’ve seen exemplify the use of generators playing the role of classic iterators: retrieving items from a collection. But generators can also be used to produce values independent of a data source. The next section shows an example of that.  
    我们看到的Sentence示例说明了生成器在传统迭代器中扮演的角色：从集合中检索项目。但生成器也可以被用于不依赖数据源产出value。下一节会对此展示一个示例。

## Another Example: Arithmetic Progression Generator 等差数列生成器

The classic Iterator pattern is all about traversal: navigating some data structure. But a standard interface based on a method to fetch the next item in a series is also useful when the items are produced on the fly, instead of retrieved from a collection. For example, the range built-in generates a bounded arithmetic progression (AP) of integers, and the itertools.count function generates a boundless AP.  
    经典Iterator模式都是关于遍历的：导航一些数据结构。但是，当item是动态生成而不是从集合中检索时，基于用于在序列中获取下一项的方法的标准接口也很有用。举个例子，内建的range生成整数的有界等差级数（AP），itertools.count函数生成无界AP。

We’ll cover itertools.count in the next section, but what if you need to generate a bounded AP of numbers of any type?  
    我们将在下一节介绍itertools.count，但是如果你需要生成一个任意类型数字的有界AP该怎么办？

Example 14-10 shows a few console tests of an ArithmeticProgression class we will see in a moment. The signature of the constructor in Example 14-10 is ArithmeticProgression(begin, step[, end]). The range() function is similar to the ArithmeticProgression here, but its full signature is range(start, stop[, step]). I chose to implement a different signature because for an arithmetic progression the step is mandatory but end is optional. I also changed the argument names from start/stop to begin/end to make it very clear that I opted for a different signature. In each test in Example 14-10 I call list() on the result to inspect the generated values.  
    例14-10展示了ArithmeticProgression类的一点控制台测试，我们马上就能看到。例14-10中构造函数的用法是ArithmeticProgression(begin, step[, end])。这里range()函数近似于ArithmeticProgression，但完整的用法是range(start, stop[, step])。我选择实现不同的用法，因为对等差数列来说，step是强制的而end是可选的。我也改变了参数名，从start/stop到begin/end，这样会非常清晰表明出我选择了不同的用法。在例14-10中的任何测试中，我对结果都调用了list()来检查生成的值。

Example 14-10. Demonstration of an ArithmeticProgression class  
    例14-10. ArithmeticProgression类的示范

```
 >>> ap = ArithmeticProgression(0, 1, 3)
 >>> list(ap)
 [0, 1, 2]
 >>> ap = ArithmeticProgression(1, .5, 3)
 >>> list(ap)
 [1.0, 1.5, 2.0, 2.5]
 >>> ap = ArithmeticProgression(0, 1/3, 1)
 >>> list(ap)
 [0.0, 0.3333333333333333, 0.6666666666666666]
 >>> from fractions import Fraction
 >>> ap = ArithmeticProgression(0, Fraction(1, 3), 1)
 >>> list(ap)
 [Fraction(0, 1), Fraction(1, 3), Fraction(2, 3)]
 >>> from decimal import Decimal
 >>> ap = ArithmeticProgression(0, Decimal('.1'), .3)
 >>> list(ap)
 [Decimal('0.0'), Decimal('0.1'), Decimal('0.2')]
```

Note that type of the numbers in the resulting arithmetic progression follows the type of begin or step, according to the numeric coercion rules of Python arithmetic. In Example 14-10, you see lists of int, float, Fraction, and Decimal numbers.  
    注意，根据Python算术的数字强制规则，算术级数中的数字类型遵循begin或step的类型。示例14-10中，你看到了int，float，Fraction与Decimal数字的列表。

Example 14-11 lists the implementation of the ArithmeticProgression class.  
    例14-11列出了ArithmeticProgression类的实现。

Example 14-11. The ArithmeticProgression class  
    例14-111 ArithmeticProgression类

```python
class ArithmeticProgression:
    def __init__(self, begin, step, end=None):  # 1
        self.begin = begin
        self.step = step
        self.end = end # None -> "infinite" series
 
    def __iter__(self):
        result = type(self.begin + self.step)(self.begin)  # 2
        forever = self.end is None  # 3
        index = 0
        while forever or result < self.end:  # 4
            yield result  # 5
            index += 1
            result = self.begin + self.step * index  # 6
```

1. __init__ requires two arguments: begin and step. end is optional, if it’s None, the series will be unbounded.  
    __init__需要两个参数：begin与step。end为可选，如果为None，级数就是无限。
2. This line produces a result value equal to self.begin, but coerced to the type of the subsequent additions.[9]  
    该行生成一个等于self.begin的值，但强制转换为后续添加的类型。
3. For readability, the forever flag will be True if the self.end attribute is None, resulting in an unbounded series.  
    为了便于阅读，forever标记将在self.end属性为None时置为True，从而产生无界级数。
4. This loop runs forever or until the result matches or exceeds self.end. When this loop exits, so does the function.  
    这个循环将无限循环或是直到结果大于等于self.end。当循环退出时，函数也退出。
5. The current result is produced.  
    产出当前结果。
6. The next potential result is calculated. It may never be yielded, because the while loop may terminate.  
    计算下一个潜在的结果。他可能永远不会被产出，因为while循环有可能终止。

In the last line of Example 14-11, instead of simply incrementing the result with self.step iteratively, I opted to use an index variable and calculate each result by adding self.begin to self.step multiplied by index to reduce the cumulative effect of errors when working with with floats.  
    在例14-11的最后一行，我没有简单地使用self.step迭代地递增结果，而是选择用index变量然后计算各自的结果（通过self.begin + self.step * index）以减少使用浮点数时错误的累计影响。

The ArithmeticProgression class from Example 14-11 works as intended, and is a clear example of the use of a generator function to implement the __iter__ special method. However, if the whole point of a class is to build a generator by implementing __iter__, the class can be reduced to a generator function. A generator function is, after all, a generator factory.  
    例14-11的ArithmeticProgression类按预期运行，这是一个使用生成器函数完成__iter__特殊方法的清晰示例。但如果一个类的重点是为了通过完成__iter__来创建生成器，那这个类就可以简化成一个生成器函数。毕竟，生成器函数就是一个生成器工厂。

Example 14-12 shows a generator function called aritprog_gen that does the same job as ArithmeticProgression but with less code. The tests in Example 14-10 all pass if you just call aritprog_gen instead of ArithmeticProgression.[10]  
    例14-12展示了一个叫做aritprog_gen的生成器函数，和ArithmeticProgression功能一样但用了更少的代码。在例14-10中，如果将ArithmeticProgression都替换为ariprog_gen也都可以pass。

Example 14-12. The aritprog_gen generator function
    例14-12. aritprog_gen生成器函数
```python
def aritprog_gen(begin, step, end=None):
    result = type(begin + step)(begin)
    forever = end is None
    index = 0
    while forever or result < end:
        yield result
        index += 1
        result = begin + step * index
```

Example 14-12 is pretty cool, but always remember: there are plenty of ready-to-use generators in the standard library, and the next section will show an even cooler implementation using the itertools module.  
    例14-12非常酷，但要记住：标准库中有大量供使用的生成器，下一节将展示通过使用itertools模块的更酷的实现。

[9]. In Python 2, there was a coerce() built-in function but it’s gone in Python 3, deemed unnecessary because the numeric coercion rules are implicit in the arithmetic operator methods. So the best way I could think of to coerce the initial value to be of the same type as the rest of the series was to perform the addition and use its type to convert the result. I asked about this in the Python-list and got an excellent response from Steven D’Aprano.  
    在Python2中，这里是内建函数coerce()但他在Python3中被删掉了，在算术运算符方法中数字强制规则隐含其中，因此这被视为非必需的。所以我认为将初始值强制归属为该系列其余部分相同类型的最好方式是执行加法然后使用它的类型来转换结果。我在Python-list中询问了这个问题，然后从Steven D'Aprano那里获取了很好的回答。
[10]. The 14-it-generator/ directory in the Fluent Python code repository includes doctests and a script, aritprog_runner.py, which runs the tests against all variations of the aritprog*.py scripts.  

### Arithmetic Progression with itertools 使用itertools的等差数列

The itertools module in Python 3.4 has 19 generator functions that can be combined in a variety of interesting ways.  
    Python3.4中的itertools模块拥有19个生成器函数，他们可以用各种有趣的方式结合使用。

For example, the itertools.count function returns a generator that produces numbers. Without arguments, it produces a series of integers starting with 0. But you can provide optional start and step values to achieve a result very similar to our aritprog_gen functions:  
    举个例子，itertools.count方法返回了一个产出数字的生成器。不传参数，他会产出一系列从0开始的整数。但你可以传入可选的start与step值，以取得和我们的aritprog_gen方法近似的效果。
```
>>> import itertools
>>> gen = itertools.count(1, .5)
>>> next(gen)
1
>>> next(gen)
1.5
>>> next(gen)
2.0
>>> next(gen)
2.5
```

However, itertools.count never stops, so if you call list(count()), Python will try to build a list larger than available memory and your machine will be very grumpy long before the call fails.  
    然而，itertools.count永不停止，所以如果你调用了list(count())，Python将构建出一个比可用内存更大的list，在调用失败的很长一段时间之前你的电脑将会非常暴躁。

On the other hand, there is the itertools.takewhile function: it produces a generator that consumes another generator and stops when a given predicate evaluates to False. So we can combine the two and write this:  
    换句话说，itertools.takewhile函数：他生产了一个生成器，该生成器消费了其他的生成器，并且当给定断言计算为False时结束。所以我们可以结合这两个，写成这样：
```
>>> gen = itertools.takewhile(lambda n: n < 3, itertools.count(1, .5))
>>> list(gen)
[1, 1.5, 2.0, 2.5]
```

Leveraging takewhile and count, Example 14-13 is sweet and short.  
    利用takewhile与count，例14-13会很简洁。

Example 14-13. aritprog_v3.py: this works like the previous aritprog_gen functions  
    例14-13. aritprog_v3.py：机制类似之前的aritprog_gen函数

```python
import itertools

def aritprog_gen(begin, step, end=None):
    first = type(begin + step)(begin)
    ap_gen = itertools.count(first, step)
    if end is not None:
        ap_gen = itertools.takewhile(lambda n: n < end, ap_gen)
    return ap_gen
```

Note that aritprog_gen is not a generator function in Example 14-13: it has no yield in its body. But it returns a generator, so it operates as a generator factory, just as a generator function does.  
    注意，14-13中的aritprog_gen不是生成器函数：函数体内没有yield。但是他返回一个生成器，所以他使用起来像生成器工厂，就像生成器函数一样。

The point of Example 14-13 is: when implementing generators, know what is available in the standard library, otherwise there’s a good chance you’ll reinvent the wheel. That’s why the next section covers several ready-to-use generator functions.  
    例14-13的重点是：当实现生成器时，请了解标准库中那些是可用的，否则你很可能重新造轮子。这就是下一节将介绍几个即用的生成器函数的原因。

## Generator Functions in the Standard Library 标准库中的生成器函数

The standard library provides many generators, from plain-text file objects providing line-by-line iteration, to the awesome os.walk function, which yields filenames while traversing a directory tree, making recursive filesystem searches as simple as a for loop.  
    标准库提供了许多生成器，从提供逐行迭代的纯文本文件对象，到惊人的os.walk函数，当遍历目录树时他会产出文件名，这让递归的文件系统搜索起来和for循环一样简单。

The os.walk generator function is impressive, but in this section I want to focus on general-purpose functions that take arbitrary iterables as arguments and return generators that produce selected, computed, or rearranged items. In the following tables, I summarize two dozen of them, from the built-in, itertools, and functools modules. For convenience, I grouped them by high-level functionality, regardless of where they are defined.  
    os.walk生成器函数令人钦佩，但本节我更想聚焦于一般用途的函数，他们将随意的可迭代对象作为参数，然后返回生成选定、计算或重新排列的项的生成器。在下面的表格中，我，从内建的itertools与functools模块中选择出这种生成器，并汇总出两打。为了方便，我将他们按照高级功能进行了分组，无论他们是在哪里定义的。
·
    Perhaps you know all the functions mentioned in this section, but some of them are underused, so a quick overview may be good to recall what’s already available.  
    也许你了解所有本节提到的函数，但他们中有一些是没有充分利用的，所以快速概览可能会有助于回忆已有的内容。

The first group are filtering generator functions: they yield a subset of items produced by the input iterable, without changing the items themselves. We used itertools.take while previously in this chapter, in “Arithmetic Progression with itertools” on page 423. Like takewhile, most functions listed in Table 14-1 take a predicate, which is a one-argument Boolean function that will be applied to each item in the input to determine whether the item is included in the output.  
    第一组是筛选的生成器函数：他们产出一个由输出的可迭代对象产出的项的子集，而不改变这些项本身。在本章前面内容中我们使用了itertools.take，在423页的“Arithmetic Progression with itertools”中。像takewhile一样，表14-1中列的大部分函数都要获取一个断言，这是一个单参数布尔型函数，将应用于输入中的每一项，以决定该项是否被包含在输出中。

Table 14-1. Filtering generator functions  
    表14-1 过滤型生成器函数
| Module | Function | Description |
| --- | --- | --- |
| itertools | compress(it, selector_it) | Consumes two iterables in parallel; yields items from it whenever the corresponding item in selector_it is truthy |
| itertools | dropwhile(predicate, it) | Consumes it skipping items while predicate computes truthy, then yields every remaining item (no further checks are made) |
| (built-in) | filter(predicate,it) | Applies predicate to each item of iterable, yielding the item if predicate(item) is truthy; if predicate is None, only truthy items are yielded |
| itertools | filterfalse(predicate, it) | Same as filter, with the predicate logic negated: yields items whenever predicate computes falsy |
| itertools | islice(it, stop) or islice(it, start, stop, step=1) | Yields items from a slice of it, similar to s[:stop] or s[start:stop:step] except it can be any iterable, and the operation is lazy |
| itertools | takewhile(predicate, it) | Yields items while predicate computes truthy, then stops and no further checks are made |

| 模块 | 方法 | 描述 |
| --- | --- | --- |
| itertools | compress(it, selector_it) | 并行使用两个可迭代对象; 每当selector_it中的响应项为真的时候，就从it中产出项 |
| itertools | dropwhile(predicate, it) | 当predicate计算为真时消耗it跳过项，然后产出所有剩余的项（不进行进一步的检查） |
| (built-in) | filter(predicate,it) | 将predicate应用于iterable的每一项，如果predicate(item)为真就产出这个item；如果predicate为None，只有真的items会被产出 |
| itertools | filterfalse(predicate, it) | 类似于filter，predicate逻辑相反：每当predicate计算为假时产出项 |
| itertools | islice(it, stop) or islice(it, start, stop, step=1) | 从it的切片产出项，类似于s[:stop]或s[start:stop:step]，除了他可以是任何可迭代对象，并且操作是惰性的 |
| itertools | takewhile(predicate, it) | 当predicate计算为真时产出项，然后终止，不进行进一步检查 |

The console listing in Example 14-14 shows the use of all functions in Table 14-1.  
    例14-14的控制台列表展示了表14-1中所有函数的用法。

Example 14-14. Filtering generator functions examples  
    例14-14. 筛选生成器函数示例
```
>>> def vowel(c):
... return c.lower() in 'aeiou'
...
>>> list(filter(vowel, 'Aardvark'))
['A', 'a', 'a']
>>> import itertools
>>> list(itertools.filterfalse(vowel, 'Aardvark'))
['r', 'd', 'v', 'r', 'k']
>>> list(itertools.dropwhile(vowel, 'Aardvark'))
['r', 'd', 'v', 'a', 'r', 'k']
>>> list(itertools.takewhile(vowel, 'Aardvark'))
['A', 'a']
>>> list(itertools.compress('Aardvark', (1,0,1,1,0,1)))
['A', 'r', 'd', 'a']
>>> list(itertools.islice('Aardvark', 4))
['A', 'a', 'r', 'd']
>>> list(itertools.islice('Aardvark', 4, 7))
['v', 'a', 'r']
>>> list(itertools.islice('Aardvark', 1, 7, 2))
['a', 'd', 'a']
```

The next group are the mapping generators: they yield items computed from each individual item in the input iterable—or iterables, in the case of map and starmap.[11] The generators in Table 14-2 yield one result per item in the input iterables. If the input comes from more than one iterable, the output stops as soon as the first input iterable is exhausted.  
    下一组是mapping生成器：他们产出从输入的一个或多个iterable中的每个独立item计算得出的项（在map与starmap的情况下）。表14-2的生成器在输入的iterables中每个item就产出一个结果。如果输入来自多个iterables，那一旦首个输入的iterable为空，就结束输出。

[11]. Here the term “mapping” is unrelated to dictionaries, but has to do with the map built-in
    这里的术语“mapping”与字典无关，而是与内建的map有关

Table 14-2. Mapping generator functions
    表14-2. Mapping生成器函数

| Module | Function | Description |
| --- | --- | --- |
| itertools | accumulate(it, [func]) | Yields accumulated sums; if func is provided, yields the result of applying it to the first pair of items, then to the first result and next item, etc. |
| (built-in) | enumerate(iterable, start=0) | Yields 2-tuples of the form (index, item), where index is counted from start, and item is taken from the iterable |
| (built-in) | map(func, it1, [it2, …, itN]) | Applies func to each item of it, yielding the result; if N iterables are given, func must take N arguments and the iterables will be consumed in parallel |
| itertools | starmap(func, it)  | Applies func to each item of it, yielding the result; the input iterable should yield iterable items iit, and func is applied as func(*iit) |

| 模块 | 方法 | 描述 |
| --- | --- | --- |
| itertools | accumulate(it, [func]) | 产出累计的总和；如果提供了func，产出将it应用于第一对item的结果，然后应用于第一个结果与下一个item，以此类推。 |
| (built-in) | enumerate(iterable, start=0) | 产出2元组(index, item)，index从start计算而来，而item取自iterable |
| (built-in) | map(func, it1, [it2, …, itN]) | 将func应用于it的每项，产出结果；如果传入了N个iterable，func必须接收N个参数然后iterable将被并行地消费 |
| itertools | starmap(func, it)  | 将func应用于it的每项，产出结果；输入的iterable应该产出可迭代的项iit，然后func被应用作func(*iit) |

Example 14-15 demonstrates some uses of itertools.accumulate.  
    例14-15演示了itertools.accumulate的部分用法

Example 14-15. itertools.accumulate generator function examples  
    例14-15. itertools.accumulate生成器示例
```
>>> sample = [5, 4, 2, 8, 7, 6, 3, 0, 9, 1]
>>> import itertools
>>> list(itertools.accumulate(sample)) # 1
[5, 9, 11, 19, 26, 32, 35, 35, 44, 45]
>>> list(itertools.accumulate(sample, min)) # 2
[5, 4, 2, 2, 2, 2, 2, 0, 0, 0]
>>> list(itertools.accumulate(sample, max)) # 3
[5, 5, 5, 8, 8, 8, 8, 8, 9, 9]
>>> import operator
>>> list(itertools.accumulate(sample, operator.mul)) # 4
[5, 20, 40, 320, 2240, 13440, 40320, 0, 0, 0]
>>> list(itertools.accumulate(range(1, 11), operator.mul))
[1, 2, 6, 24, 120, 720, 5040, 40320, 362880, 3628800] # 5
```

1. Running sum.  
    运行总和。
2. Running minimum.  
    运行最小值。
3. Running maximum.  
    运行最大值。
4. Running product.  
    运行乘积。
5. Factorials from 1! to 10!.  
    从1!到10!的阶乘。

The remaining functions of Table 14-2 are shown in Example 14-16.  
    例14-16展示了表14-2中剩余的函数。

Example 14-16. Mapping generator function examples  
    例14-16. mapping生成器函数示例
```
>>> list(enumerate('albatroz', 1)) # 1
[(1, 'a'), (2, 'l'), (3, 'b'), (4, 'a'), (5, 't'), (6, 'r'), (7, 'o'), (8, 'z')]
>>> import operator
>>> list(map(operator.mul, range(11), range(11))) # 2
[0, 1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
>>> list(map(operator.mul, range(11), [2, 4, 8])) # 3
[0, 4, 16]
>>> list(map(lambda a, b: (a, b), range(11), [2, 4, 8])) # 4
[(0, 2), (1, 4), (2, 8)]
>>> import itertools
>>> list(itertools.starmap(operator.mul, enumerate('albatroz', 1))) # 5
['a', 'll', 'bbb', 'aaaa', 'ttttt', 'rrrrrr', 'ooooooo', 'zzzzzzzz']
>>> sample = [5, 4, 2, 8, 7, 6, 3, 0, 9, 1]
>>> list(itertools.starmap(lambda a, b: b/a,
... enumerate(itertools.accumulate(sample), 1))) # 6
[5.0, 4.5, 3.6666666666666665, 4.75, 5.2, 5.333333333333333,
5.0, 4.375, 4.888888888888889, 4.5]
```

1. Number the letters in the word, starting from 1.  
    从1开始给单词中的字母编号。
2. Squares of integers from 0 to 10.  
    从0到10整数的平方。
3. Multiplying numbers from two iterables in parallel: results stop when the shortest iterable ends.  
    从两个iterable并行地计算乘数：当最短的iterable结束时结果停止。
4. This is what the zip built-in function does.  
    这就是内建zip函数所做的工作。
5. Repeat each letter in the word according to its place in it, starting from 1.  
    从1开始，根据单词中每个字母的位置进行重复。
6. Running average.  
    取平均数。

Next, we have the group of merging generators—all of these yield items from multiple input iterables. chain and chain.from_iterable consume the input iterables sequentially (one after the other), while product, zip, and zip_longest consume the input iterables in parallel. See Table 14-3.  
    接下来，我们有一组merging生成器——这些都会通过多个输入iterable来产出项。chain与chain.from_iterable顺序地消费输入iterable（一个接一个地），而product，zip与zip_longest并行地消费输入iterable。如表14-3所示。

Table 14-3. Generator functions that merge multiple input iterables  
    表14-3. 合并多个输入iterable的生成器函数
| Module | Function | Description |
| --- | --- | --- |
| itertools | chain(it1, …, itN) | Yield all items from it1, then from it2 etc., seamlessly |
| itertools | chain.from_iterable(it) | Yield all items from each iterable produced by it, one after the other, seamlessly; it should yield iterable items, for example, a list of iterables |
| itertools | product(it1, …, itN, repeat=1) | Cartesian product: yields N-tuples made by combining items from each input iterable like nested for loops could produce; repeat allows the input iterables to be consumed more than once |
| (built-in) | zip(it1, …, itN) | Yields N-tuples built from items taken from the iterables in parallel,silently stopping when the first iterable is exhausted |
| itertools | zip_longest(it1, …, itN, fillvalue=None) | Yields N-tuples built from items taken from the iterables in parallel, stopping only when the last iterable is exhausted, filling the blanks with the fillvalue |

| 模块 | 方法 | 描述 |
| --- | --- | --- |
| itertools | chain(it1, …, itN) | Yield all items from it1, then from it2 etc., seamlessly无缝地从it1产出所有项，接着从it2，…等等 |
| itertools | chain.from_iterable(it) | 从it生成的每个iterable无缝且一个接一个地产出所有项；it应该产出可迭代的项，如一个列表的iterable |
| itertools | product(it1, …, itN, repeat=1) | 笛卡尔积：产出通过组合每个输入iterable的item所构成的N维元组，就像嵌套for循环可以生成的那样；重复允许输入iterable被多次使用 |
| (built-in) | zip(it1, …, itN) | 产出N维元组，通过从iterable并行取出的项所构建，当首个iterable被耗尽后会安静终止 |
| itertools | zip_longest(it1, …, itN, fillvalue=None) | 产出N维元组，通过从iterable并行取出的项所构建，只有当最后的iterable耗尽时才结束，用fillvalue来填充空白部分 |

Example 14-17 shows the use of the itertools.chain and zip generator functions and their siblings. Recall that the zip function is named after the zip fastener or zipper (no relation with compression). Both zip and itertools.zip_longest were introduced in “The Awesome zip” on page 293.  
    例14-17展示了itertool.chain与zip生成器函数及其兄弟函数的使用。回想一下，zip函数以纽扣或拉链命名（与压缩无关）。zip与itertools.zip_longest都在293页的"The Awesome zip"所介绍过。

Example 14-17. Merging generator function examples  
    例14-17. merging生成器函数示例
```
>>> list(itertools.chain('ABC', range(2))) # 1
['A', 'B', 'C', 0, 1]
>>> list(itertools.chain(enumerate('ABC'))) # 2
[(0, 'A'), (1, 'B'), (2, 'C')]
>>> list(itertools.chain.from_iterable(enumerate('ABC'))) # 3
[0, 'A', 1, 'B', 2, 'C']
>>> list(zip('ABC', range(5))) # 4
[('A', 0), ('B', 1), ('C', 2)]
>>> list(zip('ABC', range(5), [10, 20, 30, 40])) # 5
[('A', 0, 10), ('B', 1, 20), ('C', 2, 30)]
>>> list(itertools.zip_longest('ABC', range(5))) # 6
[('A', 0), ('B', 1), ('C', 2), (None, 3), (None, 4)]
>>> list(itertools.zip_longest('ABC', range(5), fillvalue='?')) # 7
[('A', 0), ('B', 1), ('C', 2), ('?', 3), ('?', 4)]
```

1. chain is usually called with two or more iterables.  
    chain通常用两个或以上的iterable来调用。
2. chain does nothing useful when called with a single iterable.  
    当用一个iterable调用时，chain不会起什么作用。
3. But chain.from_iterable takes each item from the iterable, and chains them in sequence, as long as each item is itself iterable.  
    但chain.from_iterable会从iterable从获取每一项，然后顺序地连接他们，只要每项本身都是可迭代的。
4. zip is commonly used to merge two iterables into a series of two-tuples.  
    zip通常被用于将两个iterable合并为一系列二元组。
5. Any number of iterables can be consumed by zip in parallel, but the generator stops as soon as the first iterable ends.  
    iterable的任何数字都可以被并发地消费，但一旦首个iterable结束则生成器立刻终止。
6. itertools.zip_longest works like zip, except it consumes all input iterables to the end, padding output tuples with None as needed.  
    itertools.zip_longest和zip类似，除了他会将所有输入的iterable都消费完才结束，其中根据需要用None填充元组。
7. The fillvalue keyword argument specifies a custom padding value.  
    参数fillvalue指定一个通用的填充值。

The itertools.product generator is a lazy way of computing Cartesian products, which we built using list comprehensions with more than one for clause in “Cartesian Products” on page 23. Generator expressions with multiple for clauses can also be used to produce Cartesian products lazily. Example 14-18 demonstrates itertools.product.  
    itertools.product生成器是一种计算笛卡尔积的懒惰方式，我们使用23页的"Cartesian Products"中的多个for语句的列表推导式来构建他。拥有多for语句的生成器表达式也可以被用于延迟生产笛卡尔积。例14-18演示了itertools.product。

Example 14-18. itertools.product generator function examples  
    例14-18. itertools.product生成器函数示例
```
>>> list(itertools.product('ABC', range(2))) # 1
[('A', 0), ('A', 1), ('B', 0), ('B', 1), ('C', 0), ('C', 1)]
>>> suits = 'spades hearts diamonds clubs'.split()
>>> list(itertools.product('AK', suits)) # 2
[('A', 'spades'), ('A', 'hearts'), ('A', 'diamonds'), ('A', 'clubs'),
('K', 'spades'), ('K', 'hearts'), ('K', 'diamonds'), ('K', 'clubs')]
>>> list(itertools.product('ABC')) # 3
[('A',), ('B',), ('C',)]
>>> list(itertools.product('ABC', repeat=2)) # 4
[('A', 'A'), ('A', 'B'), ('A', 'C'), ('B', 'A'), ('B', 'B'),
('B', 'C'), ('C', 'A'), ('C', 'B'), ('C', 'C')]
>>> list(itertools.product(range(2), repeat=3))
[(0, 0, 0), (0, 0, 1), (0, 1, 0), (0, 1, 1), (1, 0, 0),
(1, 0, 1), (1, 1, 0), (1, 1, 1)]
>>> rows = itertools.product('AB', range(2), repeat=2)
>>> for row in rows: print(row)
...
('A', 0, 'A', 0)
('A', 0, 'A', 1)
('A', 0, 'B', 0)
('A', 0, 'B', 1)
('A', 1, 'A', 0)
('A', 1, 'A', 1)
('A', 1, 'B', 0)
('A', 1, 'B', 1)
('B', 0, 'A', 0)
('B', 0, 'A', 1)
('B', 0, 'B', 0)
('B', 0, 'B', 1)
('B', 1, 'A', 0)
('B', 1, 'A', 1)
('B', 1, 'B', 0)
('B', 1, 'B', 1)
```

1. The Cartesian product of a str with three characters and a range with two integers yields six tuples (because 3 * 2 is 6).  
    由3个字符组成的字符串与2个整数range的笛卡尔积产出6个元组（由于3 × 2为6）
2. The product of two card ranks ('AK'), and four suits is a series of eight tuples.  
    AK两个牌阶及四种花色的积是一系列八个元组
3. Given a single iterable, product yields a series of one-tuples, not very useful.  
    传入一个单独的iterable，乘积产出一系列单元组，这不是很有用。
4. The repeat=N keyword argument tells product to consume each input iterable N times.  
    repeat=N关键词参数使得乘积去对每个输入的iterable消费N次。

Some generator functions expand the input by yielding more than one value per input item. They are listed in Table 14-4.  
    一些生成器函数通过为每个输入项产出大于一个的值，来扩展输入。表14-4列出了这些。

Table 14-4. Generator functions that expand each input item into multiple output items  
    表14-4. 将每个输入项
| Module | Function | Description |
| --- | --- | --- |
| itertools | combinations(it, out_len) | Yield combinations of out_len items from the items yielded by it |
| itertools | combinations_with_replacement(it, out_len) | Yield combinations of out_len items from the items yielded by it, including combinations with repeated items |
| itertools | count(start=0, step=1) | Yields numbers starting at start, incremented by step, indefinitely |
| itertools | cycle(it) | Yields items from it storing a copy of each, then yields the entire
sequence repeatedly, indefinitely |
| itertools | permutations(it, out_len=None) | Yield permutations of out_len items from the items yielded by it; by default, out_len is len(list(it)) |
| itertools | repeat(item, [times]) | Yield the given item repeadedly, indefinetly unless a number of times is given |

| 模块 | 方法 | 描述 |
| --- | --- | --- |
| itertools | combinations(it, out_len) | 从it产出的项中产出out_len项的结合 |
| itertools | combinations_with_replacement(it, out_len) | 从it产出项中产出out_len项的结合，包含重复项的结合 |
| itertools | count(start=0, step=1) | 产出数字，从start开始，以step无限递增 |
| itertools | cycle(it) | Yields items from it storing a copy of each, then yields the entire
sequence repeatedly, indefinitely 从it中产出项并存储每项的副本，然后无限地重复产出整个序列 |
| itertools | permutations(it, out_len=None) | 从it产出项中产出out_len项的排列；默认情况下，out_len是len(list(it)) |
| itertools | repeat(item, [times]) | 无限重复产出给定的item，除非传入了times数字 |

The count and repeat functions from itertools return generators that conjure items out of nothing: neither of them takes an iterable as input. We saw itertools.count in “Arithmetic Progression with itertools” on page 423. The cycle generator makes a backup of the input iterable and yields its items repeatedly. Example 14-19 illustrates the use of count, repeat, and cycle.  
    itertools中的count与repeat方法都返回生成器，这些生成器无中生有地生成项：这两种方法都没有接收iterable作为输入。我们在423页的“Arithmic Progression with itertools”中看到itertools.count。cycle生成器制作了输入iterable的备份并重复产出他的项。例14-19阐明了count, repeat与cycle的使用

Example 14-19. count, cycle, and repeat  
```
>>> ct = itertools.count() # 1
>>> next(ct) # 2
0
>>> next(ct), next(ct), next(ct) # 3
(1, 2, 3)
>>> list(itertools.islice(itertools.count(1, .3), 3)) # 4
[1, 1.3, 1.6]
>>> cy = itertools.cycle('ABC') # 5
>>> next(cy)
'A'
>>> list(itertools.islice(cy, 7)) # 6
['B', 'C', 'A', 'B', 'C', 'A', 'B']
>>> rp = itertools.repeat(7) # 7
>>> next(rp), next(rp)
(7, 7)
>>> list(itertools.repeat(8, 4)) # 8
[8, 8, 8, 8]
>>> list(map(operator.mul, range(11), itertools.repeat(5))) # 9
[0, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50]
```

1. Build a count generator ct.  
    构建一个count生成器ct。
2. Retrieve the first item from ct.  
    检索ct的首项。
3. I can’t build a list from ct, because ct never stops, so I fetch the next three items.  
    我没有从ct构建一个列表，因为ct永不停止，所以我取了接下来的三项。
4. I can build a list from a count generator if it is limited by islice or takewhile.  
    如果它受islice或takewhile的限制，我可以从count生成器构建一个列表。
5. Build a cycle generator from 'ABC' and fetch its first item, 'A'.  
    用'ABC'构建一个cycle生成器，检索首项'A'。
6. A list can only be built if limited by islice; the next seven items are retrieved here.  
    只有在受islice限制时才可以构建列表；这里检索了后7项。
7. Build a repeat generator that will yield the number 7 forever.  
    构建一个repeat生成器，他用U按值会产出数字7。
8. A repeat generator can be limited by passing the times argument: here the number 8 will be produced 4 times.  
    repeat生成器可以被传入的times参数限制：这里的数字8将被生产4次。
9. A common use of repeat: providing a fixed argument in map; here it provides the 5 multiplier.  
    repeat的一种通常用法：在map中提供固定的参数；这里他提供了5的乘数。

The combinations, combinations_with_replacement, and permutations generator functions—together with product—are called the combinatoric generators in the itertools documentation page. There is a close relationship between itertools.product and the remaining combinatoric functions as well, as Example 14-20 shows.  
    combinations，combinations_with_replacement与permutations生成器函数（连同生成的）这些在itertools文档页中都被称作组合生成器。itertools.product与其余的组合生成器也有着亲密的关系，如例14-20所示。

Example 14-20. Combinatoric generator functions yield multiple values per input item  
    例14-20. 每个输入项产出多值的组合生成器函数
```
>>> list(itertools.combinations('ABC', 2)) # 1
[('A', 'B'), ('A', 'C'), ('B', 'C')]
>>> list(itertools.combinations_with_replacement('ABC', 2)) # 2
[('A', 'A'), ('A', 'B'), ('A', 'C'), ('B', 'B'), ('B', 'C'), ('C', 'C')]
>>> list(itertools.permutations('ABC', 2)) # 3
[('A', 'B'), ('A', 'C'), ('B', 'A'), ('B', 'C'), ('C', 'A'), ('C', 'B')]
>>> list(itertools.product('ABC', repeat=2)) # 4
[('A', 'A'), ('A', 'B'), ('A', 'C'), ('B', 'A'), ('B', 'B'), ('B', 'C'),
('C', 'A'), ('C', 'B'), ('C', 'C')]
```

1. All combinations of len()==2 from the items in 'ABC'; item ordering in the generated tuples is irrelevant (they could be sets).  
    项'ABC'中所有长度为2的组合；生成元组中项的顺序无关（这些可以是集合）。
2. All combinations of len()==2 from the items in 'ABC', including combinations with repeated items.  
    项'ABC'中所有长度为2的组合；包括重复项的组合。
3. All permutations of len()==2 from the items in 'ABC'; item ordering in the generated tuples is relevant.  
    项'ABC'中所有长度为2的组合；生成元组中项顺序是相关的。
4. Cartesian product from 'ABC' and 'ABC' (that’s the effect of repeat=2).  
    'ABC'与'ABC'的笛卡尔积（repeat=2所导致）。

The last group of generator functions we’ll cover in this section are designed to yield all items in the input iterables, but rearranged in some way. Here are two functions that return multiple generators: itertools.groupby and itertools.tee. The other generator function in this group, the reversed built-in, is the only one covered in this section that does not accept any iterable as input, but only sequences. This makes sense: because reversed will yield the items from last to first, it only works with a sequence with a known length. But it avoids the cost of making a reversed copy of the sequence by yielding each item as needed. I put the itertools.product function together with the merging generators in Table 14-3 because they all consume more than one iterable, while the generators in Table 14-5 all accept at most one input iterable.  
    我们本节介绍的最后一组生成器函数旨在产出输入iterables的所有项，但以某种方式重新排列。这里是两个返回多生成器的方法：itertools.groupby与itertools.tee。本组另一个生成器函数（内置的reversed）是本节唯一一个不接受任何iterable作为参数，只接受序列的函数。这是合理的：因为反向将从后向前产出项，只能用已知长度的序列。但他通过按需产出每一项来抵消为序列制作反向拷贝的代价。我将itertools.product函数与表14-3中的合并生成器放在一起，是因为他们都消费大于一个的iterable，而表14-5的生成器都最多接收一个iterable作为输入。

Table 14-5. Rearranging generator functions
    表14-5.重新排列生成器函数
| Module | Function | Description |
| --- | --- | --- |
| itertools | groupby(it, key=None) | Yields 2-tuples of the form (key, group), where key is the grouping criterion and group is a generator yielding the items in the group |
| (built-in) | reversed(seq) | Yields items from seq in reverse order, from last to first; seq must be a sequence or implement the __reversed__ special method |
| itertools | tee(it, n=2) | Yields a tuple of ngenerators, each yielding the items of the input iterable independently |

| 模块 | 方法 | 描述 |
| --- | --- | --- |
| itertools | groupby(it, key=None) | 产出(key, group)形式的二元组，key是分组标准，而group是产出组中项目的生成器 |
| (built-in) | reversed(seq) | 以反向顺序从seq产出项，从最后向开头；seq必须是一个序列或实现了__reversed__魔术方法的 |
| itertools | tee(it, n=2) | Yields a tuple of n generators, each yielding the items of the input iterable independently产出一个n生成器的元组，每个生成器独立产出输入iterable的项 |

Example 14-21 demonstrates the use of itertools.groupby and the reversed built-in. Note that itertools.groupby assumes that the input iterable is sorted by the grouping criterion, or at least that the items are clustered by that criterion—even if not sorted.  
    例14-21演示了itertools.groupby与内建reversed的使用。注意，itertools.groupby假定了输入iterable按分组标准进行排了序，或者至少项是通过那个标准聚集在一起（即使没有排序）。

Example 14-21. itertools.groupby  

```
>>> list(itertools.groupby('LLLLAAGGG')) # 1
[('L', <itertools._grouper object at 0x102227cc0>),
('A', <itertools._grouper object at 0x102227b38>),
('G', <itertools._grouper object at 0x102227b70>)]
>>> for char, group in itertools.groupby('LLLLAAAGG'): # 2
... print(char, '->', list(group))
...
L -> ['L', 'L', 'L', 'L']
A -> ['A', 'A',]
G -> ['G', 'G', 'G']
>>> animals = ['duck', 'eagle', 'rat', 'giraffe', 'bear',
... 'bat', 'dolphin', 'shark', 'lion']
>>> animals.sort(key=len) # 3
>>> animals
['rat', 'bat', 'duck', 'bear', 'lion', 'eagle', 'shark',
'giraffe', 'dolphin']
>>> for length, group in itertools.groupby(animals, len): # 4
... print(length, '->', list(group))
...
3 -> ['rat', 'bat']
4 -> ['duck', 'bear', 'lion']
5 -> ['eagle', 'shark']
7 -> ['giraffe', 'dolphin']
>>> for length, group in itertools.groupby(reversed(animals), len): # 5
... print(length, '->', list(group))
...
7 -> ['dolphin', 'giraffe']
5 -> ['shark', 'eagle']
4 -> ['lion', 'bear', 'duck']
3 -> ['bat', 'rat']
>>>
```

1. groupby yields tuples of (key, group_generator).  
    groupby产出(key, group_generator)这样的元组。
2. Handling groupby generators involves nested iteration: in this case, the outer for loop and the inner list constructor.  
    处理groupby生成器涉及嵌套的迭代：本例中，是外层for循环与内层列表构造函数。
3. To use groupby, the input should be sorted; here the words are sorted by length.  
    为了利用groupby，输入应该是排序的；这里的单词按长度排序。
4. Again, loop over the key and group pair, to display the key and expand the group into a list.  
    再一次，遍历key与对应的词组，显示key并将组展开为列表。
5. Here the reverse generator is used to iterate over animals from right to left.  
    这里是反转生成器用于从右向左迭代动物。

The last of the generator functions in this group is iterator.tee, which has a unique behavior: it yields multiple generators from a single input iterable, each yielding every item from the input. Those generators can be consumed independently, as shown in Example 14-22.  
    本组最后的生成器函数是iterator.tee，他有一种独特的行为：从单个输入iterable中产出多个生成器，每个生成器从输入产出每项。这些生成器可以被独立消费，像例14-22中展示的一样。

Example 14-22. itertools.tee yields multiple generators, each yielding every item of the input generator  
    例14-22. itertools.tee产出多生成器，每个生成器产出输入生成器的每一项。

```
>>> list(itertools.tee('ABC'))
[<itertools._tee object at 0x10222abc8>, <itertools._tee object at 0x10222ac08>]
>>> g1, g2 = itertools.tee('ABC')
>>> next(g1)
'A'
>>> next(g2)
'A'
>>> next(g2)
'B'
>>> list(g1)
['B', 'C']
>>> list(g2)
['C']
>>> list(zip(*itertools.tee('ABC')))
[('A', 'A'), ('B', 'B'), ('C', 'C')]
```

Note that several examples in this section used combinations of generator functions. This is a great feature of these functions: because they all take generators as arguments and return generators, they can be combined in many different ways.  
    注意，本节的几个示例使用了生成器函数的结合。这是这些函数中很棒的特点：因为他们都是接收生成器作参数，并返回生成器，他们可以用许多不同的方式结合。

While on the subject of combining generators, the yield from statement, new in Python 3.3, is a tool for doing just that.  
    当讨论结合生成器的主体时，Python3.3新增的yield from语法就是为了实现这一点的工具。
