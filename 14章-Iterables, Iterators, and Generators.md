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
