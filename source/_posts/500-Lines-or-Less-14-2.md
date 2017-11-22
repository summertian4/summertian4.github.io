---
title: 【500 Lines or Less】-【翻译练习】-【chapter 14】-【简单对象模型】-【第而部分】
date: 2017-10-31 12:14:15
tags:
	- 译文
	- Python
categories:
	- Python
---

> 原文链接：[A Simple Object Model](http://aosabook.org/en/500L/a-simple-object-model.html)
> 作者信息：[Carl Friedrich Bolz](https://twitter.com/cfbolz)

----

### [上一篇：《【500 Lines or Less】-【翻译练习】-【chapter 14】-【简单对象模型】-【第一部分】》](500-Lines-or-Less-14-1)
### 译者注：休息结束，我们继续

----

# 方法调用

现在我们的模型还缺少方法调用的功能，本章我们将会实现一个简单的继承模型。

```python
def test_callmethod_simple():
    # Python code
    class A(object):
        def f(self):
            return self.x + 1
    obj = A()
    obj.x = 1
    assert obj.f() == 2

    class B(A):
        pass
    obj = B()
    obj.x = 1
    assert obj.f() == 2 # works on subclass too

    # Object model code
    def f_A(self):
        return self.read_attr("x") + 1
    A = Class(name="A", base_class=OBJECT, fields={"f": f_A}, metaclass=TYPE)
    obj = Instance(A)
    obj.write_attr("x", 1)
    assert obj.callmethod("f") == 2

    B = Class(name="B", base_class=A, fields={}, metaclass=TYPE)
    obj = Instance(B)
    obj.write_attr("x", 2)
    assert obj.callmethod("f") == 3
```

<!-- More -->

为了正确的实现对象方法的调用，我们开始关注类的方法解析顺序。在方法解析顺序中找到的类的字典中的的第一个方法将被调用：

```python
class Class(Base):
    ...

    def _read_from_class(self, methname):
        for cls in self.method_resolution_order():
            if methname in cls._fields:
                return cls._fields[methname]
        return MISSING
```

完善了 `Base` 类中的 `callmethod` 方法，测试用例就可以通过了。

为了确保方法参数正确传递，以及之前的代码能完成方法重载的功能，我们编写以下代码：


```python
def test_callmethod_subclassing_and_arguments():
    # Python code
    class A(object):
        def g(self, arg):
            return self.x + arg
    obj = A()
    obj.x = 1
    assert obj.g(4) == 5

    class B(A):
        def g(self, arg):
            return self.x + arg * 2
    obj = B()
    obj.x = 4
    assert obj.g(4) == 12

    # Object model code
    def g_A(self, arg):
        return self.read_attr("x") + arg
    A = Class(name="A", base_class=OBJECT, fields={"g": g_A}, metaclass=TYPE)
    obj = Instance(A)
    obj.write_attr("x", 1)
    assert obj.callmethod("g", 4) == 5

    def g_B(self, arg):
        return self.read_attr("x") + arg * 2
    B = Class(name="B", base_class=A, fields={"g": g_B}, metaclass=TYPE)
    obj = Instance(B)
    obj.write_attr("x", 4)
    assert obj.callmethod("g", 4) == 12
```

# 基于属性的模型

现在最简单版本的对象模型已经可以用了，我们可以开会考虑完善它。这一节我们将介绍 `基于方法的模型` 和 `基于属性的模型` 之间的异同点。其实这也是 Smalltalk 、 Ruby 、 JavaScript 、 Python 和 Lua 之间的核心差异。

`基于方法的模型` 将方法调用作为程序执行的基本方式。

```python
result = obj.f(arg1, arg2)
```

`基于属性的模型` 将方法调用分为两步：查找属性和执行返回结果：

```python
method = obj.f
result = method(arg1, arg2)
```

两者的差异可以在下面的测试用例中看出：

```python
def test_bound_method():
    # Python code
    class A(object):
        def f(self, a):
            return self.x + a + 1
    obj = A()
    obj.x = 2
    m = obj.f
    assert m(4) == 7

    class B(A):
        pass
    obj = B()
    obj.x = 1
    m = obj.f
    assert m(10) == 12 # works on subclass too

    # Object model code
    def f_A(self, a):
        return self.read_attr("x") + a + 1
    A = Class(name="A", base_class=OBJECT, fields={"f": f_A}, metaclass=TYPE)
    obj = Instance(A)
    obj.write_attr("x", 2)
    m = obj.read_attr("f")
    assert m(4) == 7

    B = Class(name="B", base_class=A, fields={}, metaclass=TYPE)
    obj = Instance(B)
    obj.write_attr("x", 1)
    m = obj.read_attr("f")
    assert m(10) == 12
```

虽然设置与方法调用的相应测试相同，但调用方法的方式不同。首先，在对象中查找与 `read_attr` 方法传入参数一致的方法名。`read_attr` 返回值是一个对象，该对象封装了对象以及类中找到的对应方法。接下来我们就使用它调用该方法。

为了实现这个功能，我们需要修改 `Base.read_attr` 的实现。如果查找属性不在实例字典中，就应该去类的字典中查找。如果在类的字典中找到了这个属性，那么我们将会返回这个属性，这使用可以闭包来实现。除了更改 `Base.read_attr`，我们也可以修改 `Base.callmethod` 来确保我么你的代码能够通过测试。

```python
class Base(object):
    ...
    def read_attr(self, fieldname):
        """ read field 'fieldname' out of the object """
        result = self._read_dict(fieldname)
        if result is not MISSING:
            return result
        result = self.cls._read_from_class(fieldname)
        if _is_bindable(result):
            return _make_boundmethod(result, self)
        if result is not MISSING:
            return result
        raise AttributeError(fieldname)

    def callmethod(self, methname, *args):
        """ call method 'methname' with arguments 'args' on object """
        meth = self.read_attr(methname)
        return meth(*args)

def _is_bindable(meth):
    return callable(meth)

def _make_boundmethod(meth, self):
    def bound(*args):
        return meth(self, *args)
    return bound
```

其他的代码不需要做更改。

# 元对象协议

除了常规的调用方法，很多动态语言还提供了特殊的方法。这些方法不是直接调用而是通过对象系统调用。在 Python 中这些特殊的方法往往用两个下划线作为开头和结尾，比如 `__init__`。可这些特殊的方法可以覆盖重载普通的操作，并为它们提供自定义功能。因此它们是可以告诉对象模型如何处理不同事物的 hook，关于 Python 中的特殊方法可以参考[这篇文档](https://docs.python.org/2/reference/datamodel.html#special-method-names)。

元对象协议概念由 Smalltalk 引入，但 Common Lisp 这样的对象系统（如CLOS）也广泛的地使用元对象协议。

在本章我们将给我们的对象模型添加三种 meta-hook。它们将可以改变读写属性操作的功能。首先要添加的方法是 `__getattr__` 和 `__setattr__`（看起来和 Python 中类似方法的名字很类似）。

## 自定义读写属性操作

`__getattr__` 在通过常规的属性查找方法无法查找到时被调用（在类和对象方法字典中均未找到）。该方法需要的参数是『需要查找的属性名称』。早期的 `Smalltalk4` 中被称为 `doesNotUnderstand`

`__setattr__` 的情况有点不同。 由于设置属性总是会创建一个属性，所以在设置属性时总是调用`__setattr__`。为了确保 `__setattr__` 存在，我们需要在 `OBEJCT` 中实现 `__setattr__` 方法。保证我们可以向字典中写入属性。也可以让用户可以将自定义的 `__setattr__` 委托给 `OBJECT.__setattr__`。

针对这两个特殊方法的测试用例如下：

```python
def test_getattr():
    # Python code
    class A(object):
        def __getattr__(self, name):
            if name == "fahrenheit":
                return self.celsius * 9. / 5. + 32
            raise AttributeError(name)

        def __setattr__(self, name, value):
            if name == "fahrenheit":
                self.celsius = (value - 32) * 5. / 9.
            else:
                # call the base implementation
                object.__setattr__(self, name, value)
    obj = A()
    obj.celsius = 30
    assert obj.fahrenheit == 86 # test __getattr__
    obj.celsius = 40
    assert obj.fahrenheit == 104

    obj.fahrenheit = 86 # test __setattr__
    assert obj.celsius == 30
    assert obj.fahrenheit == 86

    # Object model code
    def __getattr__(self, name):
        if name == "fahrenheit":
            return self.read_attr("celsius") * 9. / 5. + 32
        raise AttributeError(name)
    def __setattr__(self, name, value):
        if name == "fahrenheit":
            self.write_attr("celsius", (value - 32) * 5. / 9.)
        else:
            # call the base implementation
            OBJECT.read_attr("__setattr__")(self, name, value)

    A = Class(name="A", base_class=OBJECT,
              fields={"__getattr__": __getattr__, "__setattr__": __setattr__},
              metaclass=TYPE)
    obj = Instance(A)
    obj.write_attr("celsius", 30)
    assert obj.read_attr("fahrenheit") == 86 # test __getattr__
    obj.write_attr("celsius", 40)
    assert obj.read_attr("fahrenheit") == 104
    obj.write_attr("fahrenheit", 86) # test __setattr__
    assert obj.read_attr("celsius") == 30
    assert obj.read_attr("fahrenheit") == 86
```

为了通过这个测试，需要完善 `Base.read_attr` 和 `Base.write_attr` 方法：

```python
class Base(object):
    ...

    def read_attr(self, fieldname):
        """ read field 'fieldname' out of the object """
        result = self._read_dict(fieldname)
        if result is not MISSING:
            return result
        result = self.cls._read_from_class(fieldname)
        if _is_bindable(result):
            return _make_boundmethod(result, self)
        if result is not MISSING:
            return result
        meth = self.cls._read_from_class("__getattr__")
        if meth is not MISSING:
            return meth(self, fieldname)
        raise AttributeError(fieldname)

    def write_attr(self, fieldname, value):
        """ write field 'fieldname' into the object """
        meth = self.cls._read_from_class("__setattr__")
        return meth(self, fieldname, value)
```

通过属性名作为参数，如果字段不存在抛出错误。注意 `__getattr__` 只能在类中调用（Python 中的特殊方法也是）以避免递归调用 `self.read_attr("__getattr__")`。

属性的写操作也交给 `__setattr__` 方法。为了完成这个功能，`OBEJCT` 需要实现 `__setattr__` 的基本功能，如下：

```python
def OBJECT__setattr__(self, fieldname, value):
    self._write_dict(fieldname, value)
OBJECT = Class("object", None, {"__setattr__": OBJECT__setattr__}, None)
```

`OBJECT__setattr__` 的行为就像以前的 `write_attr`。 通过这些修改，测试用例可以通过。

## 描述符协议

上述测试用例中反复转换不同温标，十分的烦人。因为属性名要在 `__getattr__` 和 `__setattr__` 中显式的去校验。为了解决这个问题，在 Python 中引入了描述符协议的概念。


我们将从 `__getattr__` 和 `__setattr__` 方法中获取具体的属性，而描述符协议是在属性调用过程结束返回结果时触发一个特殊的方法。描述符协议可以被看作是绑定方法和类的操作。除了绑定方法外，Python中描述符协议最重要的用例是 `staticmethod`、`classmethod` 和 `property`。

在本节中，我们将介绍如何使用描述符协议绑定对象。我们可以通过使用 __get__ 方法来达成这一目标：

```python
def test_get():
    # Python code
    class FahrenheitGetter(object):
        def __get__(self, inst, cls):
            return inst.celsius * 9. / 5. + 32

    class A(object):
        fahrenheit = FahrenheitGetter()
    obj = A()
    obj.celsius = 30
    assert obj.fahrenheit == 86

    # Object model code
    class FahrenheitGetter(object):
        def __get__(self, inst, cls):
            return inst.read_attr("celsius") * 9. / 5. + 32

    A = Class(name="A", base_class=OBJECT,
              fields={"fahrenheit": FahrenheitGetter()},
              metaclass=TYPE)
    obj = Instance(A)
    obj.write_attr("celsius", 30)
    assert obj.read_attr("fahrenheit") == 86
```

现在测试用例可以通过了。之前关于方法绑定的测试用例也依然通过，在 Python 中 `__get__` 方法执行完了将会返回一个已绑定方法对象。

在实践中，描述符协议要更加复杂。它还支持 `__set__` 来设置属性。你现在所看到这里实现的版本是经过一些简化的。注意，前面 `_make_boundmethod` 方法调用 `__get__` 是实现级的操作，而不是使用 `meth.read_attr('__get__')` 。意思是，我们的对象模型是在借用 Python 的函数和方法，而不是展示 Python 的对象模型。一个更完整的对象模型将不得不解决这个问题。



-----

### 译者著：文章有点长，怕大家看着感觉太干，分成了的三篇文章，大家休息一下，继续吧。
### [下一篇：《【500 Lines or Less】-【翻译练习】-【chapter 14】-【简单对象模型】-【第三部分】》](500-Lines-or-Less-14-3)


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼](http://weibo.com/coderfish/)

