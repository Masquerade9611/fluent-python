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

- for loops
- Collection types construction and extension
- Looping over text files line by line
- List, dict, and set comprehensions
- Tuple unpacking
- Unpacking actual parameters with * in function calls

This chapter covers the following topics:

- How the iter(…) built-in function is used internally to handle iterable objects
- How to implement the classic Iterator pattern in Python
- How a generator function works in detail, with line-by-line descriptions
- How the classic Iterator can be replaced by a generator function or generator expression
- Leveraging the general-purpose generator functions in the standard library
- Using the new yield from statement to combine generators
- A case study: using generator functions in a database conversion utility designed to work with large datasets
- Why generators and coroutines look alike but are actually very different and should not be mixed

We’ll get started studying how the iter(…) function makes sequences iterable.

## Sentence Take #1: A Sequence of Words

We’ll start our exploration of iterables by implementing a Sentence class: you give its constructor a string with some text, and then you can iterate word by word. The first version will implement the sequence protocol, and it’s iterable because all sequences are iterable, as we’ve seen before, but now we’ll see exactly why.  

Example 14-1 shows a Sentence class that extracts words from a text by index.  

Example 14-1. sentence.py: A Sentence as a sequence of words

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
2. self.words holds the result of .findall, so we simply return the word at the given index.
3. To complete the sequence protocol, we implement __len__—but it is not needed to make an iterable object.
4. reprlib.repr is a utility function to generate abbreviated string representations of data structures that can be very large.[3]
