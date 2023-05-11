---
title: 容易被忽略误解的Python细节列表
date: 2020-10-21 21:14:34 +0800
categories: [Python, curated-list]
tags: [curated-list, python] 
comments: true
pin: true

---



## 1.被装饰的函数的关键字参数使用默认值时，装饰器在不定长关键字参数（**kwargs）中取不到这个参数

这是在开发过程中遇到的问题，探究后发现的。

示例：

先构造一个打印入参的装饰器和一个被装饰函数

```python
def decorator(f):
    def wrapper(*arg,**kwargs):
        print("args:",args)
        print("kwargs",kwargs)
        return f(*args,**kwargs)
    return wrapper

@decorator
def test(a,b,c=1,d=2,f=3):
    print(a,b,c,d,e,f)
```

测试：

```
In [3]: test(1,2)
   ...:
---->args: (1, 2)
---->kwargs {}
in test: 1 2 1 2 3

In [4]: test(1,2,c=3)
   ...:
---->args: (1, 2)
---->kwargs {'c': 3}
in test: 1 2 3 2 3

In [5]: test(1,2,c=3,d=4,e=5)
   ...:
---->args: (1, 2)
---->kwargs {'c': 3, 'd': 4, 'e': 5}
in test: 1 2 3 4 5

```


在元类中使用`__call__`方法时也是如此：

```python

class M(type):
    def __init__(self, *args, **kwargs):
        self.a = None
        super().__init__(*args, **kwargs)

    def __call__(self, *args, **kwds):
        print("-------", kwds)
        return super().__call__(*args, **kwds)


class A(metaclass=M):
    def __init__(self, a, b=1):
        self.a = a
        self.b = b

```
测试：
```
In [2]: a = A(1)
------- {}

In [3]: b = A(1, b=2)
   ...:
------- {'b': 2}
```

结论：装饰器获取入参或者元类预处理时，不定长关键字参数（keyword Variable Arguments）的字典里，如果没有显式地传递，默认值是获取不到。所有中途拦截到的参数都是实际传递的，不包括默认值，因为默认值本质上是方法的内在属性。如果要获取该值，只能去被装饰函数的元数据里取。在利用装饰器/元类开发时，要注意判断，否则会取到错误的None值。

## 2. `-O` 和 `__debug__`搭配使用，有优化效果


> You can use the -O or -OO switches on the Python command to reduce the size of a compiled module. The -O switch removes assert statements, the -OO switch removes both assert statements and __doc__ strings. Since some programs may rely on having these available, you should only use this option if you know what you’re doing. “Optimized” modules have an opt- tag and are usually smaller. Future releases may change the effects of optimization.


## 3. Iterator Types

有 ```__iter__``` 方法只是iterable，而不是iterator，要升级必须有 ```__next__``` 方法。即 ```__iter__```方法必须返回一个iterator，可以是self，也可以是其他类


## 4. import 的细节

1. >all packages are modules, but not all modules are packages.any module that contains a __path__ attribute is considered a package.

2. >During import, the module name is looked up in sys.modules and if present, the associated value is the module satisfying the import, and the process completes. However, if the value is None, then a ModuleNotFoundError is raised. If the module name is missing, Python will continue searching for the module.

3. >Beware though, as if you keep a reference to the module object, invalidate its cache entry in sys.modules, and then re-import the named module, the two module objects will not be the same. By contrast, importlib.reload() will reuse the same module object, and simply reinitialise the module contents by rerunning the module’s code.

4. module中没有__all__时，
     - ```from module import *``` 不导单下划线开头的变量，但手动指明（```import _a```）可以导入
     - 有```__all__```时，```from module import ×``` 只导```__all__```中的变量，手动依然可以导所有。
     - 如果```__all__```中有不存在的变量，会在导包时报 ```AttributeError: module '<module>' has no attribute '<attr>'```，```__all_```_所在文件本身不会报错，即使-O也会报

5. ```from module import *```只能在module层级用，在方法或类用会直接报 ```SyntaxError: import * only allowed at module level```


## 5. Data model相关


* int 类型范围无限，取决于内存空间
* float 机器级别的双精度浮点数，溢出由底层实现处理，python不支持单精度浮点数
* set类型中，1 和1.0 是一样的。数字类型都是这样
* dict 
  > Dictionaries preserve insertion order, meaning that keys will be produced in the same order they were added sequentially over the dictionary. Replacing an existing key does not change the order, however removing a key and re-inserting it will add it to the end instead of keeping its old place.
  > ...
  > Changed in version 3.7: Dictionaries did not preserve insertion order in versions of Python before 3.6. In CPython 3.6, insertion order was preserved, but it was considered an implementation detail at that time rather than a language guarantee.

* ```__new__```
  > If ```__new__()``` does not return an instance of cls, then the new instance’s __init__() method will not be invoked.

* ```__del__```
  > ```del x``` doesn’t directly call ```x.__del__()``` — the former decrements the reference count for x by one, and the latter is only called when x’s reference count reaches zero.

* [```__hash__```](https://docs.python.org/3.9/reference/datamodel.html#object.__hash__)
    hash值一样，hash值一定一样
    > The only required property is that objects which compare equal have the same hash value

    > If a class does not define an ```__eq__()``` method it should not define a ```__hash__()``` operation either; if it defines ```__eq__()``` but not ```__hash__()```, its instances will not be usable as items in hashable collections. If a class defines mutable objects and implements an ```__eq__()``` method, it should not implement ```__hash__()```, since the implementation of hashable collections requires that a key’s hash value is immutable (if the object’s hash value changes, it will be in the wrong hash bucket).
    > User-defined classes have ```__eq__()``` and ```__hash__()``` methods by default; with them, all objects compare unequal (except with themselves) and ```x.__hash__()``` returns an appropriate value such that ```x == y``` implies both that ```x is y``` and ```hash(x) == hash(y)```.
    > A class that overrides ```__eq__()``` and does not define ```__hash__()``` will have its ```__hash__()``` implicitly set to None. When the ```__hash__()``` method of a class is None, instances of the class will raise an appropriate TypeError when a program attempts to retrieve their hash value, and will also be correctly identified as unhashable when checking ```isinstance(obj, collections.abc.Hashable)```.
    > If a class that overrides ```__eq__()``` needs to retain the implementation of ```__hash__()``` from a parent class, the interpreter must be told this explicitly by setting ```__hash__ = <ParentClass>.__hash__```.
    > If a class that does not override ```__eq__()``` wishes to suppress hash support, it should include ```__hash__ = None``` in the class definition. A class which defines its own ```__hash__()``` that explicitly raises a ```TypeError``` would be incorrectly identified as hashable by an ```isinstance(obj, collections.abc.Hashable)``` call.

* ```__bool__```
    未定义时调用```__len__()```，都没定义则该类所有实例判为true
    

* ```__missing__```
  用于dict缺项时



## 6.with as语句的逻辑



根据PEP 343的解释，with…as…会被翻译成以下语句：
```python
    mgr = (EXPR)
    exit = type(mgr).__exit__  # Not calling it yet
    value = type(mgr).__enter__(mgr)
    exc = True
    try:
        try:
            VAR = value  # Only if "as VAR" is present
            BLOCK
        except:
            # The exceptional case is handled here
            exc = False
            if not exit(mgr, *sys.exc_info()):
                raise
            # The exception is swallowed if exit() returns true
    finally:
        # The normal and non-local-goto cases are handled here
        if exc:
            exit(mgr, None, None, None)
```