---
title: 【500 Lines or Less】-【翻译练习】-【chapter 14】-【简单对象模型】-【第三部分】
date: 2017-11-22 15:27:19
tags:
	- 译文
	- Python
categories:
	- Python
---

> 原文链接：[A Simple Object Model](http://aosabook.org/en/500L/a-simple-object-model.html)
> 作者信息：[Carl Friedrich Bolz](https://twitter.com/cfbolz)

----

> [上一篇：《【500 Lines or Less】-【翻译练习】-【chapter 14】-【简单对象模型】-【第二部分】》](http://zhoulingyu.com/2017/10/31/500-Lines-or-Less-14-2/)
> 译者注：休息结束，我们继续

# 实例优化

虽然对象模型的在之前的几节中发生了很多行为变化，但在最后一节中，我们将在没有影响任何行为的情况下进行优化。这种优化被称为 maps，并在自编程语言的虚拟机中率先被使用。如今，它仍然是最重要的对象模型优化之一：它被用于 PyPy 和所有现代 JavaScript 虚拟机，如V8（其中优化被称为 *hidden classes*）。

目前的观察：在到目前为止实现的对象模型中，所有实例都使用完整的字典来存储它们的属性。字典是使用哈希映射来实现的，这消耗大量的内存。此外，同一个类的实例将会拥有同样的属性，比如，有一个类 `Point`，它所有的实例都包含同样的属性 `x`、`y`。

maps 优化利用了这一点。它有效地将每个实例的字典分成两部分。一部分存放所有实例共享的 keys。而实例只存储对共享映射的引用和列表中的属性值（比字典更消耗了更少的内存）。

做简单测试用例如下所示：

```python
def test_maps():
    # white box test inspecting the implementation
    Point = Class(name="Point", base_class=OBJECT, fields={}, metaclass=TYPE)
    p1 = Instance(Point)
    p1.write_attr("x", 1)
    p1.write_attr("y", 2)
    assert p1.storage == [1, 2]
    assert p1.map.attrs == {"x": 0, "y": 1}

    p2 = Instance(Point)
    p2.write_attr("x", 5)
    p2.write_attr("y", 6)
    assert p1.map is p2.map
    assert p2.storage == [5, 6]

    p1.write_attr("x", -1)
    p1.write_attr("y", -2)
    assert p1.map is p2.map
    assert p1.storage == [-1, -2]

    p3 = Instance(Point)
    p3.write_attr("x", 100)
    p3.write_attr("z", -343)
    assert p3.map is not p1.map
    assert p3.map.attrs == {"x": 0, "z": 1}
```

<!-- More -->

请注意，这个测试用例不同于我们之前写的测试用例。以前的所有测试都是通过已知的接口来测试类的功能。而此测试用例通过读取内部属性并将其与预定义值进行比较来检查 `Instance` 类的实现细节。

`p1` 包含 `attrs` 的 `map` 存放了 `x` 和 `y` 两个属性，具体在 `p1` 中存放的值分别为 0 和 1。后创建的第二个实例 `p2`，同样在 map 中添加同样的属性。 换句话说，如果添加了不同的属性，其 map 是不通用的。

`Map` 类类似于下：

```python
class Map(object):
    def __init__(self, attrs):
        self.attrs = attrs
        self.next_maps = {}

    def get_index(self, fieldname):
        return self.attrs.get(fieldname, -1)

    def next_map(self, fieldname):
        assert fieldname not in self.attrs
        if fieldname in self.next_maps:
            return self.next_maps[fieldname]
        attrs = self.attrs.copy()
        attrs[fieldname] = len(attrs)
        result = self.next_maps[fieldname] = Map(attrs)
        return result

EMPTY_MAP = Map({})
```

Maps 类拥有两个方法，`get_index` 和 `next_map`。前者用于查找对象存储中属性名称的索引。当一个新的属性被添加到一个对象时，后者被使用。 在这种情况下，对象需要使用 `next_map` 计算不同的 map。该方法使用 `next_maps` 字典来缓存已经创建的 map。这样，具有相同布局的对象也最终使用相同的 `Map` 对象。


![图 14.2 Map 关系](http://aosabook.org/en/500L/objmodel-images/maptransition.png)

使用 maps 的 `Instance` 实现如下所示：

```python
class Instance(Base):
    """Instance of a user-defined class. """

    def __init__(self, cls):
        assert isinstance(cls, Class)
        Base.__init__(self, cls, None)
        self.map = EMPTY_MAP
        self.storage = []

    def _read_dict(self, fieldname):
        index = self.map.get_index(fieldname)
        if index == -1:
            return MISSING
        return self.storage[index]

    def _write_dict(self, fieldname, value):
        index = self.map.get_index(fieldname)
        if index != -1:
            self.storage[index] = value
        else:
            new_map = self.map.next_map(fieldname)
            self.storage.append(value)
            self.map = new_map
```


目前 `Instance` 类将给 `Base` 类传递 `None` 作为字段字典，因为 `Instance` 将会以另一种方式构建存储字典。因此它需要重载 `_read_dict` 和 `_write_dict` 。在实际操作中，我们将重构 `Base `类，使其不再负责存放属性字典。但目前，我们可以仅仅传递一个 `None` 作为参数。

在一个新的实例创建使用 `EMPTY_MAP`，其中没有存放任何对象。为了`_read_dict`，`Instance` 的 map 要提供属性名称的索引，然后返回相应的条目。

写入属性字典有两种情况。第一种是现有属性值的修改，那么就简单的在 map 的列表中修改对应的值就好。而如果对应属性不存在，那么需要进行 map 变换（如上面的图所示一样），将会调用 `next_map` 方法，然后将新的值存放入储存列表中。

这个优化到底实现了什么？一般而言，在具有很多相似结构实例的情况下优化了内存的使用。但这不是一个通用的优化手段：有些时候代码中充斥着结构不同的实例之时，这样优化可能会耗费更大的空间。

在优化动态语言时，这是一个常见问题。一般而言，不太可能找到一种十分通用的方法去优化代码，既其更快，又节省空间。在实践中，所选择的优化适用于通常使用的语言，而对于使用极其动态的功能的程序而言，可能会有相反的作用。

maps 优化另一个有意思的点是，虽然这里只优化内存使用，但是在使用 JIT 技术 的 VM 中，也能提高程序的性能。为了实现这一点，JIT 技术使用 maps 来查找属性在存储空间中的偏移量。然后完全拜托字典查找的方式。

# 潜在扩展

扩展我们的对象模型和使用不同语言的设计选择是很容易的。这里给出一些可能的方向：

- 最简单的是添加更多的特殊方法方法，比如一些 `__init__`, `__getattribute__`, `__set__` 这样容易实现又有趣的方法。

- 扩展模型支持多重继承。为了实现这一点，每一个类都需要一个父类列表。然后 `Class.method_resolution_order` 需要进行修改，以便支持方法查找。可以使用深度优先搜索和删除重复项来计算简单的方法解析顺序。更为复杂的可以采用 C3 算法, 这种算法拥有更好的处理菱形继承结构的能力，并且避免了不可感知的继承模式。

- 一个更为疯狂的想法是切换到原型模式，这需要消除类和实例之间的差别。

# 总结

面向对象编程语言设计的核心是其对象模型的细节。编写一些简单的对象模型原型可以更好地理解现有语言的内部工作原理以及深入了解面向对象语言的设计理念。编写不同的对象模型验证不同对象的设计思路是一个很好的方式，你也必语言实现的更枯燥的部分，比如解析和执行代码。

编写对象模型在实践中也是非常有用的而不仅仅是用作实验。除了作为实验品以外，它们还可以在编写其他语言中所使用。例子有很多：用 C 语言编写的 GObject 模型，用于 GLib 和 其余 Gnome 库中，或者 JavaScript 各种类系统的实现。


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

