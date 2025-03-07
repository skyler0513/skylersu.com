---
title: 生成器、迭代器、装饰器
date: 2022-08-15T14:11:51+08:00
lastmod: 2022-08-15T14:11:51+08:00
cover: https://img.sysummery.top/python-property.png
avatar: https://img.sysummery.top/coffee.jpg
authorlink: https:www.skylersu.com
categories:
  - 后端
tags:
  - python
---

因为这3个条目是我觉得是python中用起来很方便的“语法糖”，因此想写个文章总结一下，顺便归拢一下自己的思路。

<!--more-->

## 生成器
我们会经常使用类似这种方便的写法生成一个list
```python
l = [2 * x for x in range(10)]
print(l)
```
```
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```
但是把`[]`换成`()`
```python
l = (2 * x for x in range(10))
print(l)
```
```
<generator object <genexpr> at 0x10e5ce0c0>
```
就变成了生成器，我们可以使用next函数一步一步的获取这个生成器的值，下面调用这个函数11次
```python
print(next(g))
print(next(g))
print(next(g))
print(next(g))
print(next(g))
print(next(g))
print(next(g))
print(next(g))
print(next(g))
print(next(g))
print(next(g))
```
```
0
2
4
6
8
10
12
14
16
18
Traceback (most recent call last):
  File "/Users/skyler.s/py/practise/yield.py", line 17, in <module>
    print(next(g))
          ^^^^^^^
StopIteration

Process finished with exit code 1
```
因为只有10个元素，因此第11次调用的时候报错了。因此保险的办法还是使用`for`循环
```python
for v in g:
    print(v)
```
除了上面的定义方式外，还有一种使用`yield`定义，就拿最常见的`斐波那契数列`举例子
```python
def fib():
    n, a, b = 0, 0, 1
    while n < 1000:
        print(b)
        a, b = b, a + b
        n = n + 1
f = fib
print(f)
```
```
<function fib at 0x10c2d23e0>
```
如果稍微修改一下代码
```python
def fib():
    n, a, b = 0, 0, 1
    while n < 1000:
        yield b
        a, b = b, a + b
        n = n + 1
f = fib()
print(f)
```
修改的地方就2点

* `print b`改成了`yield b`
* `f = fib`改成了`f = fib()`

此时`f`就不再是个函数了而是一个生成器。获取生成器值的方式就是使用next或者for循环获取。每次执行的时候遇到`yield`关键字就返回，下次从`yield`的下一行开始执行。

### 生成器 vs list
通过上面的例子，感觉生成器与列表很相似，那么它存在的意义是什么呢？还是拿`斐波那契数列`为例子，如果我们想获取前10000项，那么就得开辟一段内存保存这10000数字，但是使用生成器的话就只需要保存函数的代码，然后依次生成就可以了，无论是100个、100000个还是10000000个成本都是固定的。

## 迭代器
python中自带的可以使用for循环的数据类型有`list、tuple、dict、set、str`。除此以外，我们上面讲了生成器也是可以使用for循环迭代数据的。**这些能够使用for循环迭代数据的类型或者生成器叫做可迭代对象。**

python中专门有个对象类型叫`Iterable`，因此我们可以使用`isinstance()`判断一个变量是不是`Iterable`
```python
from collections.abc import Iterable
l = [1,2,3]
print(isinstance(l, Iterable))
```
```
True
```

### 迭代器与可迭代对象
迭代器一定是可迭代对象，但是可迭代对象不一定是迭代器。只有可以使用`next()`函数获取下一个元素的类型才是迭代器。因此生成器肯定是迭代器，`list、tuple、dict、set、str`这些类型是可迭代对象并不是迭代器。

```python
from collections.abc import Iterator
l = [1,2,3]

def func():
    for i in l:
        yield i
f = func()

print(isinstance(l, Iterator))
print(isinstance(f, Iterator))
```
```
False
True
```
但是python内置的iter函数可以直接把可迭代对象转化成迭代器

```python
from collections.abc import Iterator
l = [1,2,3]
print(isinstance(iter(l), Iterator))
```
```
True
```

## 装饰器
python中一切皆对象，因此一个函数也可以像一个int值一样：
1. 作为另外一个函数的参数被传递
2. 作为另外一个函数的返回值
3. 赋值给一个变量

### 不带参数的装饰器
这些就是装饰器的基础。一个简单的不带参数的装饰器示例
```python
def my_decorator(func):
    print("in decorator")
    func()
    
@my_decorator
def main():
    print("in main")
```
执行脚本发现虽然没有调用main函数，但是输出了
```
in decorator
in main
```
也就是说main函数及其装饰器函数都执行了。在python中，装饰器函数是会被自动执行的，不需要显示的调用。但是我们的诉求一般是调用某个函数的时候，才会触发其装饰器函数的调用，因此需要对上面的代码稍作调整，需要把逻辑放到一个固定的函数`wrapper`里面
```python
def my_decorator(func):
    def wrapper():
        print("in decorator")
        func()
    return wrapper

@my_decorator
def main():
    print("in main")
```
再次执行脚本，是没有任何输出的。如果我们调用`main`函数就会输出
```
in decorator
in main
```
很符合预期。但是如果我们执行
```python
print(main.__name__)
```
会输出
```
wrapper
```
不符合预期啊。其实装饰器就是把**原函数的逻辑**与**装饰器函数的逻辑**包裹在了一起，然后以wrapper函数返回，main函数其实就变成了wrapper函数内部的一个对象，它的metadata也被wrapper给吞噬掉了。如果main函数还想保持“独立自主”那么可以变成下面这样
```python
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper():
        print("in decorator")
        func()
    return wrapper

@my_decorator
def main():
    print("in main")

print(main.__name__)
```
1. from functools import wraps
2. 在wrapper函数出使用`@wraps(func)`，代表这个被包裹后的函数使用原函数的metadata

### 带参数的装饰器
上面的例子中，装饰器函数my_decorator只有一个参数，用于表示原函数。如果我想要增加其他参数怎么办呢？
```python

from functools import wraps

def my_decorator(extra_param):
    def between_decorator_and_wrapper(func):
        @wraps(func)
        def wrapper():
            print(extra_param)
            print("in decorator")
            func()
        return wrapper
    return between_decorator_and_wrapper

@my_decorator("123")
def main():
    print("in main")

main()
```
其实也很好理解，这次`my_decorator`返回了一个“入参是函数“的函数，`extra_param`这个变量与`between_decorator_and_wrapper`这个函数都是`my_decorator`内部的一个对象，`between_decorator_and_wrapper` 就理所当然的作为闭包可以引用`my_decorator`中的任意数据. 我们再做一次修改，让装饰器可以接收任意的参数
```python

from functools import wraps

def my_decorator(*args, **kwargs):
    def between_decorator_and_wrapper(func):
        @wraps(func)
        def wrapper():
            print(args[0])
            print(kwargs["key"])
            print("in decorator")
            func()
        return wrapper
    return between_decorator_and_wrapper

@my_decorator("123", key="456")
def main():
    print("in main")

main()
```
```
123
456
in decorator
in main
```
这里有个小tips需要注意，不要把逻辑写到`wrapper`函数之外，因为这部分逻辑会在脚本被引用的时候自动执行。所以带参数的装饰器虽然外面多包裹了一层，但是所有的逻辑还是要写到最里面的`wrapper`里面。

### 系统自带的装饰器
#### @functools.wraps
这个在上面的例子中出现好几次了，应该是最常用的装饰器了，它的作用就是保留原函数的metadata

#### @staticmethod
用于将类中的一个方法定义为一个静态方法。

在类中定义的函数，如果没有`@classmethod`和`@staticmethod`装饰，那么这个方法是属于具体的对象的，这个方法会有一个固定的参数`self`，代表一个具体的对象。

如果加有`@classmethod`装饰器，那么这个方法会有一个固定的参数`cls`，代表类。

如果加有`@staticmethod`装饰器就是静态方法。这个方法只是定义在类空间中，使用上与一般函数无异。

看个具体例子
```python
class MyClass(object):
    title = 'i belong to the class'

    def __init__(self):
        self.instance_title = 'i belong to the instance'

    @staticmethod
    def static_method():
        return "i am static method"

    @classmethod
    def class_method(cls):
        return cls.title

    def instance_method(self):
        return self.instance_title
```
`static_method`可以有参数，但是需要外面传。

#### @classmethod
用于定义类方法。上面例子中`class_method`就是类方法，通过`cls`参数访问只属于类的方法和属性

#### @property
使用`property`装饰的函数可以使用`对象名.方法名`的方式调用方法。有一个可选的装饰器`方法名.setter`用于调用另一个同名函数
```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @property
    def area(self):
        return 3.14159 * (self._radius ** 2)

circle = Circle(5)
# 调用@property装饰的radius方法
print(circle.radius)
# 调用@property装饰的area方法
print(circle.area)

# 会调用@radius.setter装饰的radius方法
circle.radius = 10
# 10
print(circle.radius)  
# 10 * 10 * 3.14159
print(circle.area)    
```

#### @overload
用于定义一系列名称相同但是签名不同的函数，这些个函数每一个都使用`@overload`装饰，最终会有一个名称相同但是没有`@overload`装饰的函数来执行具体的逻辑
```python
class MyClass(object):
    @overload
    def transform(self, data:int) -> str:
        ...

    @overload
    def transform(self, data:str) -> int:
        ...

    def transform(self, data):
        if isinstance(data, int):
            return str(data)
        elif isinstance(data, str):
            return int(data)
```
