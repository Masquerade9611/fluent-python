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

The re.finditer function is a lazy version of re.findall which, instead of a list, returns a generator producing re.MatchObject instances on demand. If there are many matches, re.finditer saves a lot of memory. Using it, our third version of Sentence is now lazy: it only produces the next word when it is needed. The code is in Example 14-7.  

Example 14-7. sentence_gen2.py: Sentence implemented using a generator function calling the re.finditer generator function
