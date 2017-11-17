---
title: 【500 Lines or Less】-【翻译练习】-【chapter 14】-【简单对象模型】-【第一部分】
date: 2017-05-08 15:31:46
tags:
	- 译文
	- Python
categories:
	- Python
---

> 原文链接：[A Simple Object Model](http://aosabook.org/en/500L/a-simple-object-model.html)
> 作者信息：[Carl Friedrich Bolz](https://twitter.com/cfbolz)


[Carl Friedrich Bolz](https://twitter.com/cfbolz)是伦敦国王大学的研究员，对动态语言的实现及优化兴趣浓厚。他是 PyPy/RPython 的核心贡献者之一，并为 Prolog, Racket, Smalltalk, PHP 和 Ruby 等语言贡献代码。


# 引子

面向对象开发是现今最主流的编程范式之一，面向对象的思想和变成范式也被大量的编程语言所支持。虽然不痛语言提供的面向对象编程机制表面上非常相似，但是细节上却大相径庭。大多数语言的共同点是拥有对象和继承机制。而『类』，并不是每一种语言都很好的支持它的种种特性。比如，比如对于 `Self` 或者 `JavaScript` 这样的原型程序设计的语言来说没有类的概念，而是对象间直接进行了继承。

了解不同语言的对象模型的不同之处是一件非常有趣的事。这能帮助你发现不同语言的异同之处。这将在我们学习新语言时，运用以往的编程语言经验，快速的学习。

本章将探索学实现一套简单的对象模型。我们将从一个简单的实例和它的类开始，并让它拥有一些方法可以用于访问。这是被 Simula 67 、Smalltalk 等早期 OO（面向对象） 语言所采用的经典面向对象范例。在本章中，这个模型将会被一步步进行扩展，前两个小节我们将展示不同语言的设计思路，最后一小节将讲解如何提高对象模型的性能。最终我们的模型并不是任何一门语言所采用的模型，不过，将认为是一个简化的 Python 对象模型。

本章中使用的代码范例都使用 Python 编写。代码可以运行在 Python 2.7 和 3.4。为了更好的理解对象模型的设计，本章也提供了相应的单元测试。测试代码可以通过 py.test 或者 nose 来运行。

用 Python 作为实现语言并不是最好的，通常一个 VM（或者语言解释器）是由底层语言比如 C 和 C++ 实现的，并在实现过程中对更偏向于扣每一个实现细节从而提高代码效率。而更简单的语言更让我们更容易关注实际的行为表现，而不是扣实现细节。

# Method-Based Model

我们将从实现 Smalltalk 的极简版本开始讲解我么你的对象模型。Smalltalk 是由 Xerox PARC 的Alan Kay's 小组在上世纪 70 年代开发的面向对象语言。它推广了面向对象语言，在今时的众多编程语言中也能看到它的很多特性。Smalltalk 的设计宗旨之一就是『万物皆对象』。如今，Smalltalk 的最接近的思想继承者是 Ruby，Ruby 的语法更像 C 语言，但却保留了大部分 Smalltalk 的对象模型。

本节中的对象模型将具有它们的类和实例，可以读写对象的属性，可以调用对象的方法，
同时允许继承。在开始之前要说的是，这些类的对象都是有属性和方法的普通类。

FYI: 本章节中我使用的『实例』代表『对象（而不是类）』。

开始的一个好方式是编写一个测试用例来指定『将要实现的行为』。本章介绍的所有测试用例都由两部分组成。第一部分由 Python 常用工具类的定义、使用以及更高级的一些特性。第二部门将会使用我们自己编写的对象模型来代替 Python 的常用工具类。

测试用例中，我们需要手动维护 Python 的常用工具类和我们自己编写的对象莫行之间的映射关系。比如，Python 中会使用 `obj.attribute`，而我们自己编写的对象模型的使用是 `obj.read_attr("attribute")`。这种映射在真正的语言实现中可以由语言的解释器或编译器来完成。

本章中对整体的代码的进行进一步简化，我们没有明显的区分『实现对象模型』和『使用对象模型』。在一个真正的语言系统中，这两者通常会以不同的编程语言来实现。

我们从读取和写入对象属性的简单测试开始。

```python
def test_read_write_field():
    # Python code
    class A(object):
        pass
    obj = A()
    obj.a = 1
    assert obj.a == 1

    obj.b = 5
    assert obj.a == 1
    assert obj.b == 5

    obj.a = 2
    assert obj.a == 2
    assert obj.b == 5

    # Object model code
    A = Class(name="A", base_class=OBJECT, fields={}, metaclass=TYPE)
    obj = Instance(A)
    obj.write_attr("a", 1)
    assert obj.read_attr("a") == 1

    obj.write_attr("b", 5)
    assert obj.read_attr("a") == 1
    assert obj.read_attr("b") == 5

    obj.write_attr("a", 2)
    assert obj.read_attr("a") == 2
    assert obj.read_attr("b") == 5
```

这个测试用例使用了三个我们必须实现的东西。`Class` 类和 `Instance` 类分别代表了我们的对象模型的类与实例。其中有两个特殊的类的实例：`OBJECT` 和 `TYPE`，`OBJECT` 代表 Python 中的的根类，`TYPE` 代表 Python 中类的 type。

为了给 `Class` 以及 `Instance` 类的实例提供通用操作支持，他们通过继承一个基类 `Base` 来实现一个共享接口，这个基类暴露了很多方法：

```python
class Base(object):
    """ The base class that all of the object model classes inherit from. """

    def __init__(self, cls, fields):
        """ Every object has a class. """
        self.cls = cls
        self._fields = fields

    def read_attr(self, fieldname):
        """ read field 'fieldname' out of the object """
        return self._read_dict(fieldname)

    def write_attr(self, fieldname, value):
        """ write field 'fieldname' into the object """
        self._write_dict(fieldname, value)

    def isinstance(self, cls):
        """ return True if the object is an instance of class cls """
        return self.cls.issubclass(cls)

    def callmethod(self, methname, *args):
        """ call method 'methname' with arguments 'args' on object """
        meth = self.cls._read_from_class(methname)
        return meth(self, *args)

    def _read_dict(self, fieldname):
        """ read an field 'fieldname' out of the object's dict """
        return self._fields.get(fieldname, MISSING)

    def _write_dict(self, fieldname, value):
        """ write a field 'fieldname' into the object's dict """
        self._fields[fieldname] = value

MISSING = object()
```

`Base` 类实现了对象的类的存储，用一个字典保存对象的属性和其值。现在我们需要去实现 `Class` 和 `Instance` 类。在 `Instance` 的构造器中将会完成类的实例化以及 `fields` 和 `dict` 初始化】。也就是说 `Instance` 仅仅是 没有任何额外功能的 `Base` 的简单子类。

`Class` 的构造器将会获取类名、基类、类的字典以及元类进行构造。对于类来说，属性会在蕾叔实话的时候由用户传入给构造器。构造器也会从基类中获取属性的默认值（这里将会在下一节讲解）。

```python
class Instance(Base):
    """Instance of a user-defined class. """

    def __init__(self, cls):
        assert isinstance(cls, Class)
        Base.__init__(self, cls, {})


class Class(Base):
    """ A User-defined class. """

    def __init__(self, name, base_class, fields, metaclass):
        Base.__init__(self, metaclass, fields)
        self.name = name
        self.base_class = base_class
```

由于类也是一种对象，它们（间接）从 `Base` 继承。 因此，类是一个特殊类的实例：元类。

现在我们基本通过了第一个测试用例。还剩下没有详解的是 `Class` 的两个实例 `TYPE` 和 `OBJECT`。这里我们不使用 Smalltalk 的模型，因为它过于复杂，我们将使用 Python 借鉴的 ObjVlisp（一种元编程语言，可参考《Metaclasses are first class: The ObjVlisp Model》）。

ObjVlisp 模型中，`OBJECT` 和 `TYPE` 是交织在一起的。`OBJECT` 是所有类的基类，这意味着 `OBJECT` 类没有基类。`TYPE` 是 `OBJECT` 的子类，一般来说，每一个类都是 `TYPE` 的实例。特殊情况下，`TYPE` 和 `OBJECT` 都是 `TYPE` 的实例。然而，程序员可以通过继承 `TYPE` 来实现一个新的元类：

```python
# set up the base hierarchy as in Python (the ObjVLisp model)
# the ultimate base class is OBJECT
OBJECT = Class(name="object", base_class=None, fields={}, metaclass=None)
# TYPE is a subclass of OBJECT
TYPE = Class(name="type", base_class=OBJECT, fields={}, metaclass=None)
# TYPE is an instance of itself
TYPE.cls = TYPE
# OBJECT is an instance of TYPE
OBJECT.cls = TYPE
```

为了实现一个新的元类，可以继承 `TYPE`。然而，在本章中我们不这样去做，我们只见得使用 `TYPE` 作为我们每个类的元类。


![图 14.1 - 继承](http://aosabook.org/en/500L/objmodel-images/inheritance.png)

现在第一个测试用例就通过了。第二个测试用例校验了对象属性读写是否正常。这十分简单，很快就能通过用例。

```python
def test_read_write_field_class():
    # classes are objects too
    # Python code
    class A(object):
        pass
    A.a = 1
    assert A.a == 1
    A.a = 6
    assert A.a == 6

    # Object model code
    A = Class(name="A", base_class=OBJECT, fields={"a": 1}, metaclass=TYPE)
    assert A.read_attr("a") == 1
    A.write_attr("a", 5)
    assert A.read_attr("a") == 5
```

# `isinstance` （对象是否是该类实例）校验

到目前为止，我们还没有利用『对象对应类』的特性。下一个测试用例将机械实现 `isinstance`：

```python
def test_isinstance():
    # Python code
    class A(object):
        pass
    class B(A):
        pass
    b = B()
    assert isinstance(b, B)
    assert isinstance(b, A)
    assert isinstance(b, object)
    assert not isinstance(b, type)

    # Object model code
    A = Class(name="A", base_class=OBJECT, fields={}, metaclass=TYPE)
    B = Class(name="B", base_class=A, fields={}, metaclass=TYPE)
    b = Instance(B)
    assert b.isinstance(B)
    assert b.isinstance(A)
    assert b.isinstance(OBJECT)
    assert not b.isinstance(TYPE)
```

要检查一个 `obj` 对象是否是某个类 `cls` 的一个实例，可以通过检查 `cls` 是 `obj` 类的超类还是类本身。要检查一个 `类X` 是否是另一个类的超类，可以通过检查这个类的是否在 `类X` 的继承链上。如果还有其他的类在这个继承链上，那么这些类也是 `类X` 的超类。包括 `类x` 本身在内的一个继承链称为该类的『方法解析顺序』。 通过递归很容易将其计算出：

```python
class Class(Base):
    ...

    def method_resolution_order(self):
        """ compute the method resolution order of the class """
        if self.base_class is None:
            return [self]
        else:
            return [self] + self.base_class.method_resolution_order()

    def issubclass(self, cls):
        """ is self a subclass of cls? """
        return cls in self.method_resolution_order()
```

加上这段代码后，测试用例就可以顺利通过了。

-----

### 译者著：文章有点长，怕大家看着感觉太干，分成了的两篇文章，大家休息一下，继续吧。
### [下一篇：《【500 Lines or Less】-【翻译练习】-【chapter 14】-【简单对象模型】-【第二部分】》](500-Lines-or-Less-14-2)


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)



