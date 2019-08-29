---
layout: post
title: Python中的单例实现
category: 编程
description: 通过工厂方法、装饰器、类构造方法、元类实现Python单例模式
tags: ["Python"]
---

在Python中实现单例有多种方法，比如通过工厂方法或装饰器控制对象实例的生成、通过类的`__new__`方法控制对象构造过程，通过元类控制类的生成等，以下记录了这几种方法的实现(均不考虑线程安全)。

1. 通过工厂函数

    ```py
    class _Foo():
        pass

    _foo_instance = None

    def Foo():
        global _foo_instance
        if not _foo_instance:
            _foo_instance = _Foo()
        return _foo_instance

    A = Foo()
    B = Foo()
    print(A is B)

    ```
2. 通过装饰器函数，此方法与工厂函数本质是一样的

    ```py
    from functools import wraps

    def singleton(cls):
        _instance = None

        @wraps(cls)
        def wrapper(*args, **kwargs):
            nonlocal _instance
            if not _instance:
                _instance = cls(*args, **kwargs)
            return _instance

        return wrapper

    @singleton
    class Foo():
        pass
    ```
3. 装饰器类

    ```py
    class Singleton():
        def __init__(self, cls):
            self._instance = None
            self._cls = cls

        def __call__(self, *args, **kwargs):
            if not self._instance:
                self._instance = self._cls(*args, **kwargs)
            return self._instance

    @Singleton
    class Foo():
        pass
        ```

4. `__new__`控制构造过程。对象从类中创建的时候，先执行的`__new__`返回对象实例，再通过`__init__`初始化。

    ```py
    class Foo():
        _instance = None

        def __new__(cls, *args, **kwargs):
            if not cls._instance:
                cls._instance = object.__new__(cls, *args, **kwargs)
            return cls._instance
    ```

5. 通过元类控制类的生成。Python是动态语言，可以通过metaclass等手段动态生成类。

    ```py
    class Singleton(type):
        _instance = None

        def __call__(cls, *args, **kwargs):
            if not cls._instance:
                cls._instance = super().__call__(*args, **kwargs)
            return cls._instance

    class Foo(metaclass=Singleton):
        pass
    ```


*[参考文档]*
*[使用元类](https://www.liaoxuefeng.com/wiki/1016959663602400/1017592449371072)*
*[如何正确理解Python函数是第一类对象](https://foofish.net/function-is-first-class-object.html)*
*[一步一步教你认识Python闭包](https://foofish.net/python-closure.html)*
*[python—类对象和实例对象的区别](https://www.cnblogs.com/loleina/p/5409084.html)*
*[Python单例模式(Singleton)的N种实现](https://zhuanlan.zhihu.com/p/37534850)*
