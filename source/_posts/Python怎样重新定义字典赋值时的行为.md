---
title: Python怎样重新定义字典赋值时的行为
date: 2020-08-24 09:36:49
categories: 技术
tags: 
- Python
- 技术总结
- 代码风格
---

## 前言

有时我们在处理Python字典时，需要对字典的一些值进行格式的转换或其他形式的数据处理。比如将字符串转换成`int`格式、小数保留两位或是把`False`替换为空字符串（通常是为了配合前端的页面显示）等等。

尽管通常我们可以直接用一个for循环来完成这样的工作；只需要遍历一遍字典的键值并执行操作即可。但是一旦我们遇到复杂的情况（不只是调用一个函数就可以轻松解决的问题），我们的代码将会变得难以维护。

这时，自己创建一个基于字典，却可以实现自己特定要求的类，无疑是首选方案。
<!-- more -->

我们先来看一个简单的例子。在下面的例子中，我们用一个for循环将字典`day_time`内的值都转换为`int`数字格式：

```python
# 假设传入的值均为字符串(string)
day_time = {'hours': '24',
            'minutes': '60',
            'seconds': '60'}

for key, value in day_time.items():
    day_time[key] = int(value)

day_time['seconds_per_day'] = day_time['hours'] * day_time['minutes'] * day_time['seconds']

print(day_time)
{'hours': 24, 'minutes': 60, 'seconds': 60, 'seconds_per_day': 86400}
```

似乎很简单不是吗？但如果我们的例子更加复杂一些，一个for加上一个函数这种方式就不容易搞定了。

比如说我们现在有一个记录了员工日薪和每月工作日的字典，现在想计算出Peter的月薪和时薪。面对同样的需求下，字典内存在一个非数字的值。

```python
staff = {'name': 'Peter',
         'salary_per_day': '200',
         'working_days': '23'}

for key, value in staff.items():
    if value.isdecimal():  # 只将数字转换为int
        staff[key] = int(value)

# 这里计算月薪
staff['salary_per_month']  = staff['salary_per_day'] * staff['working_days']

# Peter一天工作7小时，时薪保留两位小数
staff['salary_per_hour'] = round(staff['salary_per_day'] / 7, 2)

print(staff)
{'name': 'Peter',
 'salary_per_day': 200,
 'working_days': 30,
 'salary_per_month': 6000,
 'salary_per_hour': 28.57}
```

当我们需要面对麻烦的类型转换、小数的处理时，就要一个一个地处理这些问题。例如说还有好几个需要进行转换的小数，那就又要调用好多次`round()`、或者再写一个for循环……有没有优雅的方式来解决这样的麻烦呢？

## 如何重定义字典和取值时的行为

Python 的字典是一个类（class），其拥有[`__setitem__()`](https://docs.python.org/zh-cn/3/reference/datamodel.html#object.__setitem__)、[`__getitem__`](https://docs.python.org/zh-cn/3/reference/datamodel.html#object.__getitem__)两个方法分别处理取值和赋值的行为。

默认的行为类似下面这样：

```python
class dict():
    def __setitem__(self, key, value):
        self[key] = value

    def __getitem__(self, key):
        return self[key]
    
    ...
```

如果我们要改写这两个行为，就要用到**继承**或**组合**的方式。例如继承`UserDict`这个模块，我们马上就会提到它。

## 简单易用的模块：collections.UserDict

很方便的是，Python提供了一个内置模块。[`collections.UserDict`](https://docs.python.org/zh-cn/3/library/collections.html#userdict-objects)可以被直接继承用来创建一个新的*可定制字典* 类。`UserDict`提供了一个名为`self.data`的接口；一个真实的字典，来保存和访问字典的内容。

和上面描述的一样，我们只需要像这样来定义即可：

```python
from collections import UserDict

class IntDict(UserDict):
    def __setitem__(self, key, value):
        self.data[key] = int(value)
```

如上，会在赋值时将传入的值转换为`int`。同时继承`UserDict`也提供了所有的默认类方法：你可以完全将你定制的字典看作是普通的字典来操作。

```python
In [1]: from collections import UserDict
   ...:
   ...: class IntDict(UserDict):
   ...:     def __setitem__(self, key, value):
   ...:         self.data[key] = int(value)
   ...:

In [2]: day_time = {'hours': '24', 'minutes': '60'}

In [3]: int_day_time = IntDict(day_time)

In [4]: int_day_time
Out[4]: {'hours': 24, 'minutes': 60}

In [5]: int_day_time['seconds'] = '60'

In [6]: int_day_time
Out[6]: {'hours': 24, 'minutes': 24, 'seconds': 60}

In [7]: int_day_time['hours'] + 10
Out[7]: 34

In [8]: int_day_time.get('nothing', 0) + 10
Out[8]: 10
```

一切功能开箱即用。然后你就可以**根据需要随意定义字典赋值或取值的行为**。

## 定义一个新的类：包装一个原生字典对象

我们也可以不使用类继承，自己定义一个 dict-like object（近似字典的类）。[Python 文档提供了具体的步骤](https://docs.python.org/zh-cn/3/reference/datamodel.html#emulating-container-types)。

简言之，如果你要定义一个近似类，就应该实现该类原本的基本功能（如`__len__`, `__setitem__` ），同时还**应该**实现应有的附加功能（如`keys()`, `items()`）。

另外因为我们只想定义类方法，而不考虑如何实现相同的数据存储方式。我们可以采用编程思想中“组合”的概念。即在我们自行定制的类内部，创建一个真实的字典来保存数据，然后我们再定义相同的**类方法**继而包装这个字典即可。

{% note success %}
**组合：** 新类中创建已存在类的对象，由此类中会包含已存在的类对象，此种方式为组合。
{% endnote %}

在[Stack Overflow 上的这个回答](https://stackoverflow.com/a/3387975/13954528)里，讲解了具体的操作方式：

```python
from collections.abc import MutableMapping


class TransformedDict(MutableMapping):
    """A dictionary that applies an arbitrary key-altering
       function before accessing the keys"""

    def __init__(self, *args, **kwargs):
        self.store = dict()
        self.update(dict(*args, **kwargs))  # use the free update to set keys

    def __getitem__(self, key):
        return self.store[self.__keytransform__(key)]

    def __setitem__(self, key, value):
        self.store[self.__keytransform__(key)] = value

    def __delitem__(self, key):
        del self.store[self.__keytransform__(key)]

    def __iter__(self):
        return iter(self.store)

    def __len__(self):
        return len(self.store)

    def __keytransform__(self, key):
        return key
```

这个类继承自`collections.abc.MutableMapping`。然后定义了*赋值、取值、删除一个键值、转化为迭代器、返回长度等基本方法* 。其重点是，在`__init__`方法里创建了`self.store`，让该类在初始化对象时，创建一个原生的字典。*并让所有的类方法的操作指向该字典*。 

{% note success %}
**`collections.abc.MutableMapping`在这里有什么用呢？**

`MutableMapping`会自动根据类的基础方法，提供其他的默认附加方法。也就是说，**因为继承了`MutableMapping`这个类，所以我不用再自己编写`get()`, `update()`等方法了**。如果我不需要自己定义它们的功能，可以全部继承自`MutableMapping`。
{% endnote %}

回到刚刚我们写的转换为`int`的例子，我们可以写成这样：

```python
from collections.abc import MutableMapping


class IntDict(MutableMapping):
    def __init__(self, *args, **kwargs):
        self.store = dict()
        self.update(dict(*args, **kwargs))  # self.update() 就是继承自 MutableMapping 得到的

    def __getitem__(self, key):
        return self.store[key]

    def __setitem__(self, key, value):
        self.store[key] = int(value)  # 这里由我们重新定义 __setitem__，赋值时将value转换为int

    def __delitem__(self, key):
        del self.store[key]

    def __iter__(self):
        return iter(self.store)

    def __len__(self):
        return len(self.store)
```

上述代码可以实现一个字典的基本功能，即`IntDict`可以被当做一个普通的字典使用了。

但是！我们其实没有定义`__repr__`和`__str__`。这两个类方法负责处理对字典内容的回显，也就是说，当我们在解释器中调用回显时，会得到下面的结果：

```python
In [2]: day_time = {'hours': '24', 'minutes': '60'}

In [3]: int_day_time = IntDict(day_time)

In [4]: int_day_time['hours'] * int_day_time['minutes']
Out[4]: 1440

In [5]: int_day_time
Out[5]: <__main__.IntDict at 0x25dac559b80>

In [6]: int_day_time.items()
Out[6]: ItemsView(<__main__.IntDict object at 0x0000025DAC559B80>)
```

{% note success %}
上述两个方法的定义方式另见：[这篇Stack Overflow上的回答](https://stackoverflow.com/a/21368848/13954528)
{% endnote %}

该回答还详细说明了这种方式带来的几个其他问题。但是，显而易见的一个结论是：

> **我们有更方便实现这个需求的方式，并不需要自己重写一个新的类。**

## 还有一个办法：继承内置原生字典(dict)类

话说大家有没有想过，我们开始讲了`collections.UserDict`，却没有直接继承一个`dict`类呢？

因为继承一个`dict`，我们如果想改写一个方法的话，依旧需要借助原来的方法才能实现对字典数据的操作。不过，我们可以通过调用[`super()`](https://docs.python.org/3/library/functions.html#super)来实现这个要求。它可以用来调用父类的方法。（要注意的是，我们既然已经继承了一个`dict`了，就可以不用像组合的方式那样再自己在类里面创建一个字典了。而是直接调用父类的方法来操作一个字典）：

```python
class IntDict(dict):
    def __setitem__(self, key, value):
        super().__setitem__(key, int(value))  # 在这里应用我们的操作int(value)
```

来测试一下。

```python
In [42]: d['hours'] = '12'

In [43]: d['hours']
Out[43]: 12

In [44]: d['hours'] + 10
Out[44]: 22
```

似乎对比`UserDict`其实挺方便的？其他的类方法也都可以正常运行（因为我们的类没有提供别的类方法，所以Python默认会寻找并使用父类的方法）。但是，**有一个初始化的坑**：

```python
In [39]: my_int_dict = IntDict(day_time)

In [40]: my_int_dict
Out[40]: {'hours': '24', 'minutes': '60'}

In [41]: my_int_dict['hours'] + 10
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-41-4ef251d97ba3> in <module>
----> 1 my_int_dict['hours'] + 10

TypeError: can only concatenate str (not "int") to str
```

如果我们的类以初始化的形式去创建一个字典，我们定义的方法会失效。原因我在另一篇文章中了解到：

{% centerquote %}
子类化 dict 时的另一个问题是，内置`__init__`不调用`update`，内置`update`不调用`__setitem__`。因此，确保所有的`setitem`操作通过您的`__setitem__`函数。
{% endcenterquote %}

因此我们还需要重新定义自己的`__init__`和`.update()`方法：

```python
class IntDict(dict):
    def __init__(self, *args, **kwargs):
        self.update(*args, **kwargs)

    def update(self, *args, **kwargs):
        for k, v in dict(*args, **kwargs).items():
            self.__setitem__(k, v)

    def __setitem__(self, key, value):
        super().__setitem__(key, int(value))  # 在这里应用我们的操作int(value)
```

也重新定义`__init__`和`.update()`，让这两个方法也通过我们自定义的`__setitem__`方法创建键值，就可以解决初始化的问题。

上述方式虽说不需要导入`UserDict`了，但是完整的代码相比于`UserDict`的方式反而更加繁琐。所以，封装良好的`UserDict`才是最好的实现方式。

## 总结

首先，Python 使用`__setitem__`与`__getitem__`定义字典的赋值和取值行为。

我们可以继承`dict`，`collections.UserDict`，`collections.abc.MutableMapping` 或自己重写一个类（并用组合的方式内置一个dict实例）。这三种方式都可以做到**自己定义一个实现特定行为的字典**。

- [`collections.UserDict`](https://docs.python.org/zh-cn/3/library/collections.html#userdict-objects): 专门设计为用来继承并进行定制化的模块。最直观简洁的方式。提供`self.data`接口来操作字典数据。
- [`collections.abc.MutableMapping`](https://docs.python.org/zh-cn/3/library/collections.abc.html#collections.abc.MutableMapping): 只起到自动提供部分附加函数的功能。其实还是[自己重写一个*dict-like object*](https://docs.python.org/zh-cn/3/reference/datamodel.html#emulating-container-types)。相比起继承原本的 dict class，仅在有更高定制需求时才有意义。
- `dict`: 继承内置类需要借助`super()`函数才能访问字典数据。重点是，如果重定义的是`__setitem__`方法，还需自己重定义`__init__`和`.update()`让这两个内置方法指向重写的`__setitem__`。