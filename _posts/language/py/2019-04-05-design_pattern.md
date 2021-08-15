---
layout:     post
rewards: false
title:      设计模式 for python
categories:
    - py
---

一直觉得设计模式是很高大上的东西，但其实有时自己用了也不知道，面试时被问得一愣一愣的，看一下书才发现，自己用过这么多。**当然说不出==没用过╮(╯_╰)╭**

23种设计模式，分为三类：创造性 (creational)，结构性(structural)和行为性(behavioral)。

- creational

  创建对象

  Factory,Builder,Prototype,Singleton

- structural

  处理类与对象间的组合

  Adapter,Decorator,Bridge,Facade,flyweight, model-view-controller (MVC), and proxy.

- behavioral

  类与对象交互中的职责分配

  Chain of Responsibility,Command, Observer,State，Interpreter, Strategy, Mememto, Iterator, and Template

  

# creational

## Factory

简化对象创建过程,将**创建对象的代码与使用它的代码分离**(client 端不知对象是怎样创建的)来降低维护应用程序的复杂性。

###  factory method

输入一些参数到一个单独函数得到想要的对象

- **集中了对象(逻辑上相似的对象)创建**，使得跟踪对象变得更加容易，可以创建多个工厂方法，屏蔽产品的具体实现，调用者只关心产品的接口。
- 优化应用性能和内存。工厂方法只有在绝对必要时才能通过创建新对象来提高性能和内存使用率。当我们使用直接类实例化创建对象时，每次创建新对象时都会分配额外的内存
- 作为一种创建类模式，在任何需要生成复杂对象的地方，都可以使用工厂方法模式。有一点需要注意的地方就是复杂对象适合使用工厂模式，而**简单对象，特别是只需要通过 new 就可以完成创建的对象，无需使用工厂模式。如果使用工厂模式，就需要引入一个工厂类，会增加系统的复杂度。**

```py

import json
import xml.etree.ElementTree as etree

class JSONConnector:
    def __init__(self, filepath):
        self.data = dict()
        with open(filepath, mode='r', encoding='utf8') as f:
            self.data = json.load(f)

    @property
    def parsed_data(self):
        return self.data


class XMLConnector:
    def __init__(self, filepath):
        self.tree = etree.parse(filepath)

    @property
    def parsed_data(self):
        return self.tree


def connection_factory(filepath):
    """ 工厂方法 """
    if filepath.endswith('json'):
        connector = JSONConnector
    elif filepath.endswith('xml'):
        connector = XMLConnector
    else:
        raise ValueError('Cannot connect to {}'.format(filepath))
    return connector(filepath)
```

### abstract factory
抽象工厂是工厂方法的（逻辑）组

是围绕一个超级工厂创建其他工厂。**该超级工厂又称为其他工厂的工厂**。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

接口是负责创建一个相关对象的工厂，不需要显式指定它们的类。每个生成的工厂都能按照工厂模式提供对象。

- When use abstract factory not factory method
> 从更简单的工厂方法开始，当工厂方法越来越多的时候

```python
class Frog:
    def __init__(self, name):
        self.name = name

    def __str__(self):
        return self.name

    def interact_with(self, obstacle):
        """ 不同类型玩家遇到的障碍不同 """
        print('{} the Frog encounters {} and {}!'.format(
            self, obstacle, obstacle.action()))


class Bug:
    def __str__(self):
        return 'a bug'

    def action(self):
        return 'eats it'


class FrogWorld:
    def __init__(self, name):
        print(self)
        self.player_name = name

    def __str__(self):
        return '\n\n\t----Frog World -----'

    def make_character(self):
        return Frog(self.player_name)

    def make_obstacle(self):
        return Bug()


class Wizard:
    def __init__(self, name):
        self.name = name

    def __str__(self):
        return self.name

    def interact_with(self, obstacle):
        print('{} the Wizard battles against {} and {}!'.format(
            self, obstacle, obstacle.action()))


class Ork:
    def __str__(self):
        return 'an evil ork'

    def action(self):
        return 'kill it'


class WizardWorld:
    def __init__(self, name):
        print(self)
        self.player_name = name

    def __str__(self):
        return '\n\n\t------ Wizard World -------'

    def make_character(self):
        return Wizard(self.player_name)

    def make_obstacle(self):
        return Ork()


class GameEnvironment:
    """ 抽象工厂，根据不同的玩家类型创建不同的角色和障碍 (游戏环境)
    这里可以根据年龄判断，成年人返回『巫师』游戏，小孩返回『青蛙过河』游戏"""
    def __init__(self, factory):
        self.hero = factory.make_character()
        self.obstacle = factory.make_obstacle()

    def play(self):
        self.hero.interact_with(self.obstacle)
                

def main():
    name = input("Hello. What's your name? ")
    valid_input = False
    while not valid_input:
       valid_input, age = validate_age(name)
    game = FrogWorld if age < 18 else WizardWorld
    environment = GameEnvironment(game(name))
    environment.play()
```


## Builder

我们想要创建一个由多个部分组成的对象，并且需要一步一步地完成组合。使用于**一些基本部件不会变，而其组合经常变化的时候。**

- factory diff
> 构建器模式在**多个步骤**中创建对象,当工厂模式立即返回创建的对象时，在构建器模式中，客户端代码**显式要求**导向器在需要时返回最终对象
>
> 与工厂模式的区别是：**建造者模式更加关注与零件装配的顺序。**




```python
# Director
class Director(object):
    def __init__(self):
        self.builder = None
 
    def construct_building(self):
        self.builder.new_building()
        self.builder.build_floor()
        self.builder.build_size()
 
    def get_building(self):
        return self.builder.building
 
 
# Abstract Builder
class Builder(object):
    def __init__(self):
        self.building = None
 
    def new_building(self):
        self.building = Building()
 
 
# Concrete Builder
class BuilderHouse(Builder):
    def build_floor(self):
        self.building.floor = 'One'
 
    def build_size(self):
        self.building.size = 'Big'
 
 
class BuilderFlat(Builder):
    def build_floor(self):
        self.building.floor = 'More than One'
 
    def build_size(self):
        self.building.size = 'Small'
 
 
# Product
class Building(object):
    def __init__(self):
        self.floor = None
        self.size = None
 
    def __repr__(self):
        return 'Floor: %s | Size: %s' % (self.floor, self.size)
 
 
# Client
if __name__ == "__main__":
    director = Director()
    director.builder = BuilderHouse()
    director.construct_building()
    building = director.get_building()
    print(building)
    director.builder = BuilderFlat()
    director.construct_building()
    building = director.get_building()
    print(building)

```

## Prototype

在原有的对象基础上创建对象 copy出两个独立的副本

`copy.deepcopy`

适用：

- 资源优化场景
- 类初始化需要消化非常多的资源
- 性能和安全要求的场景
- 一个对象多个修改者的场景


## Singleton
需要创建一个且只有一个对象时，它很有用。维护一个全局状态等

**主要解决：**一个全局使用的类频繁地创建与销毁。

**何时使用：**当您想控制实例数目，节省系统资源的时候。

metaclass
> A metaclass is the class of a class. A class defines how an instance of the class (i.e. an object) behaves while a metaclass defines how a class behaves. A class is an instance of a metaclass.


```python
class SingletonType(type):
       _instances = {}
       def __call__(cls, *args, **kwargs):
           if cls not in cls._instances:
               cls._instances[cls] = super(SingletonType,
               cls).__call__(*args, **kwargs)
           return cls._instances[cls]
```

# 结构性 structural

## Adapter

解决接口不兼容,在不修改原有类代码的情况下实现新的功能

- 适配器模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。
- 继承或依赖

优点

- 可以让任何两个没有关联的类一起运行。 
- 提高了类的复用。
- 增加了类的透明度。 
- 灵活性好

缺点

- 过多地使用适配器，因此如果不是很有必要，可以不使用适配器，而是直接对系统进行重构。 
- 适配器不是在详细设计时添加的，而是解决正在服役的项目的问题。

```python
class Adapter:
    def __init__(self, obj, adapted_methods):
        """ 不使用继承，使用__dict__属性实现适配器模式 """
        self.obj = obj
        self.__dict__.update(adapted_methods)

    def __str__(self):
        return str(self.obj)
```

## Decorator

动态地给一个对象添加一些额外的职责

对象添加新功能方式： 直接给对象所属的类添加方法，继承，组合，接口（Adapter适配器）

装饰类和被装饰类可以独立发展，不会相互耦合，装饰模式是继承的一个替代模式，装饰模式可以动态扩展一个实现类的功能。

```python
class foo(object):
    def f1(self):
        print("original f1")
 
    def f2(self):
        print("original f2")
 
 
class foo_decorator(object):
    def __init__(self, decoratee):
        self._decoratee = decoratee
 
    def f1(self):
        print("decorated f1")
        self._decoratee.f1()
 
    def __getattr__(self, name):
        return getattr(self._decoratee, name)
 
u = foo()
v = foo_decorator(u)
v.f1()
v.f2()
```

## Bridge

多个对象共享实现

将抽象部分与它的实现部分分离，使它们都可以独立地变化。

- 实现接口

```python
class ResourceContentFetcher(metaclass=abc.ABCMeta):
       """
       Define the interface for implementation classes that fetch content.
       """
       @abc.abstractmethod
       def fetch(path):
            pass
            
class File(ResourceContentFetcher):
    def fetch(path):
          xxx
```

## Composite

组合模式（Composite Pattern）

是用于把一组相似的对象当作一个单一的对象。组合模式依据树形结构来组合对象，用来表示部分以及整体层次。这种类型的设计模式属于结构型模式，它创建了对象组的树形结构。

您想表示对象的部分-整体层次结构（树形结构）。 2、您希望用户忽略组合对象与单个对象的不同，用户将统一地使用组合结构中的所有对象。

**优点：** 1、高层模块调用简单。 2、节点自由增加。

**缺点：**在使用组合模式时，其叶子和树枝的声明都是实现类，而不是接口，违反了依赖倒置原则。

**使用场景：**部分、整体场景，如树形菜单，文件、文件夹的管理。









## Facade

外观设计模式有助于我们隐藏系统的内部复杂性，并通过简化的界面仅向客户端公开必要的内容

引入facade 将这个子系统与客户以及其他的子系统分离，可以提高子系统的独立性和可移植性。

使用facade模式定义子系统中每层的入口点。如果子系统之间是相互依赖的，你可以让它们仅通过facade进行通讯，从而简化了它们之间的依赖关系。

```python
class A:
    def run():
        pass
class B:
    def run():
        pass

#  Facade
class Facade:
    def __init__(self):
         self.run_list=[A(),B()]
    def run(self):
        [r.run() for r in self.run_list]

Facade().run()
```

## Flyweight

对象创建带来的性能和内存占用问题,实现对象复用从而改善资源使用,对象之间引入数据共享来最小化内存使用并提高性能的技术。

flyweight是一个共享对象，它包含与状态无关的，不可变的（也称为内部）数据。
可变（也称为外在）数据不应该是flyweight的一部分，因为这是无法共享的信息，因为它对每个对象不同。如果flyweight需要外部数据，则应由客户端代码明确提供。

```python
import weakref


class Card(object):
    """The object pool. Has builtin reference counting"""
    _CardPool = weakref.WeakValueDictionary()

    """Flyweight implementation. If the object exists in the
    pool just return it (instead of creating a new one)"""

    def __new__(cls, value, suit):
        obj = Card._CardPool.get(value + suit, None)
        if not obj:
            obj = object.__new__(cls)
            Card._CardPool[value + suit] = obj
            obj.value, obj.suit = value, suit
        return obj

    # def __init__(self, value, suit):
    #     self.value, self.suit = value, suit

    def __repr__(self):
        return "<Card: %s%s>" % (self.value, self.suit)


if __name__ == '__main__':
    # comment __new__ and uncomment __init__ to see the difference
    c1 = Card('9', 'h')
    c2 = Card('9', 'h')
    print(c1, c2)
    print(c1 == c2)
    print(id(c1), id(c2))
```

## MVC

关注点分离（SoC）原理。 SoC原则背后的想法是将应用程序拆分为不同的部分，其中每个部分都涉及一个单独的问题。

模型，控制器，视图 数据访问层，业务逻辑层，表示层

模型是核心组成部分，包含并管理应用程序的（业务）逻辑，数据，状态和规则。视图是模型的直观表示。视图仅显示数据;它没有处理它。
控制器是模型和视图之间的链接/粘合剂。模型和视图之间的所有通信都通过控制器进行。

不修改模型的情况下使用多个视图，为了实现模型与视图之间的分离，每个视图通常都需要自己的控制器。如果模型直接与特定视图通信，我们将无法使用多个视图

## Proxy

代理 访问实际对象之前执行重要操作

- remote proxy 访问远程对象就像本地访问一样 
- virtual proxy 延迟初始化来推迟创建计算昂贵的对象，直到实际需要它为止,带来显着的性能改进
- protection/protective proxy 用于控制对敏感对象的访问
- smart(reference) proxy 在访问对象时执行额外操作。此类操作的示例是引用计数和线程安全检查。

protection/protective proxy
```python
import time


class SalesManager:
    def work(self):
        print("Sales Manager working...")

    def talk(self):
        print("Sales Manager ready to talk")


class Proxy:
    def __init__(self):
        self.busy = 'No'
        self.sales = None

    def work(self):
        print("Proxy checking for Sales Manager availability")
        if self.busy == 'No':
            self.sales = SalesManager()
            time.sleep(2)
            self.sales.talk()
        else:
            time.sleep(2)
            print("Sales Manager is busy")


if __name__ == '__main__':
    p = Proxy()
    p.work()
    p.busy = 'Yes'
    p.work()
```
virtual proxy
```python
class LazyProperty:
    """ 用描述符实现延迟加载的属性 """
    def __init__(self, method):
        self.method = method
        self.method_name = method.__name__

    def __get__(self, obj, cls):
        if not obj:
            return None
        value = self.method(obj)
        print('value {}'.format(value))
        setattr(obj, self.method_name, value)
        return value


class Test:
    def __init__(self):
        self.x = 'foo'
        self.y = 'bar'
        self._resource = None

    @LazyProperty
    def resource(self):    # 构造函数里没有初始化，第一次访问才会被调用
        print('initializing self._resource which is: {}'.format(self._resource))
        self._resource = tuple(range(5))    # 模拟一个耗时计算
        return self._resource


t = Test()
print(t.x)
print(t.y)
# 访问LazyProperty, resource里的print语句只执行一次，实现了延迟加载和一次执行
print(t.resource)
print(t.resource)
```



# behavioral

类与对象交互中的职责分配

## Chain of Responsibility

使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。
将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。

在责任连模式中，我们把消息发送给一系列对象的首个节点，
对象可以选择**处理消息**或者**向下一个对象传递**,只有对消息感兴趣的节点处理。用来解耦发送者和接收者。

如果所有请求都可以由单个处理元素处理，则责任链模式不是非常有用，除非我们真的不知道哪个元素。
这种模式的价值在于它提供的解耦。客户端和所有处理元素之间没有多对多的关系（对于处理元素和所有其他处理元素之间的关系也是如此），
客户端只需要知道如何与开始进行通信(链头)。


处理请求
```python
class Handler:
    def successor(self, successor):
        self.successor = successor


class ConcreteHandler1(Handler):
    def handle(self, request):
        if 0 < request <= 10:
            print("in handler1")
        else:
            self.successor.handle(request)


class ConcreteHandler2(Handler):
    def handle(self, request):
        if 10 < request <= 20:
            print("in handler2")
        else:
            self.successor.handle(request)


class ConcreteHandler3(Handler):
    def handle(self, request):
        if 20 < request <= 30:
            print("in handler3")
        else:
            print('end of chain, no handler for {}'.format(request))


class Client:
    def __init__(self):
        h1 = ConcreteHandler1()
        h2 = ConcreteHandler2()
        h3 = ConcreteHandler3()

        h1.successor(h2)
        h2.successor(h3)

        requests = [2, 5, 14, 22, 18, 3, 35, 27, 20]
        for request in requests:
            h1.handle(request)


if __name__ == "__main__":
    client = Client()

```

event
```python
class Event:
    def __init__(self, name):
        self.name = name

    def __str__(self):
        return self.name


class Widget:

    """Docstring for Widget. """

    def __init__(self, parent=None):
        self.parent = parent

    def handle(self, event):
        handler = 'handle_{}'.format(event)
        if hasattr(self, handler):
            method = getattr(self, handler)
            method(event)
        elif self.parent:
            self.parent.handle(event)
        elif hasattr(self, 'handle_default'):
            self.handle_default(event)


class MainWindow(Widget):
    def handle_close(self, event):
        print('MainWindow: {}'.format(event))

    def handle_default(self, event):
        print('MainWindow: Default {}'.format(event))


class SendDialog(Widget):
    def handle_paint(self, event):
        print('SendDialog: {}'.format(event))


class MsgText(Widget):
    def handle_down(self, event):
        print('MsgText: {}'.format(event))


def main():
    mw = MainWindow()
    sd = SendDialog(mw)    # parent是mw
    msg = MsgText(sd)

    for e in ('down', 'paint', 'unhandled', 'close'):
        evt = Event(e)
        print('\nSending event -{}- to MainWindow'.format(evt))
        mw.handle(evt)
        print('Sending event -{}- to SendDialog'.format(evt))
        sd.handle(evt)
        print('Sending event -{}- to MsgText'.format(evt))
        msg.handle(evt)

if __name__ == "__main__":
    main()

```

## Command

把一个操作封装成一个对象。命令模式可以控制命令的执行时间和过程，在不同的时刻指定、排列和执行请求，复杂的命令还可以分组

支持取消操作。Command的Excute 操作可在实施操作前将状态存储起来，在取消操作时这个状态用来消除该操作的影响。

```python
import os
 
class MoveFileCommand(object):
    def __init__(self, src, dest):
        self.src = src
        self.dest = dest
 
    def execute(self):
        self()
 
    def __call__(self):
        print('renaming {} to {}'.format(self.src, self.dest))
        os.rename(self.src, self.dest)
 
    def undo(self):
        print('renaming {} to {}'.format(self.dest, self.src))
        os.rename(self.dest, self.src)
 
 
if __name__ == "__main__":
    command_stack = []
 
    # commands are just pushed into the command stack
    command_stack.append(MoveFileCommand('foo.txt', 'bar.txt'))
    command_stack.append(MoveFileCommand('bar.txt', 'baz.txt'))
 
    # they can be executed later on
    for cmd in command_stack:
        cmd.execute()
 
    # and can also be undone at will
    for cmd in reversed(command_stack):
        cmd.undo()
```

## Observer

一对多的依赖关系,当一个对象的状态发生改变时, 所有依赖于它的对象都得到通知并被自动更新。e.g 事件驱动

```python
class Publisher:
    def __init__(self):
        self.observers = []

    def add(self, observer):
        if observer not in self.observers:
            self.observers.append(observer)
        else:
            print('Failed to add : {}').format(observer)

    def remove(self, observer):
        try:
            self.observers.remove(observer)
        except ValueError:
            print('Failed to remove : {}').format(observer)

    def notify(self):
        [o.notify_by(self) for o in self.observers]

class DefaultFormatter(Publisher):
    def __init__(self, name):
        super().__init__()
        self.name = name
        self._data = 0

    def __str__(self):
        return "{}: '{}' has data = {}".format(
            type(self).__name__, self.name, self._data)

    @property
    def data(self):
        return self._data

    @data.setter
    def data(self, new_value):
        try:
            self._data = int(new_value)
        except ValueError as e:
            print('Error: {}'.format(e))
        else:
            self.notify()    # data 在被合法赋值以后会执行notify


class HexFormatter:
    """ 订阅者 """
    def notify_by(self, publisher):
        print("{}: '{}' has now hex data = {}".format(
            type(self).__name__, publisher.name, hex(publisher.data)))


class BinaryFormatter:
    """ 订阅者 """
    def notify_by(self, publisher):
        print("{}: '{}' has now bin data = {}".format(
            type(self).__name__, publisher.name, bin(publisher.data)))


if __name__ == "__main__":
    df = DefaultFormatter('test1')
    print(df)
    print()
    hf = HexFormatter()
    df.add(hf)
    df.data = 3
    print(df)

    print()
    bf = BinaryFormatter()
    df.add(bf)
    df.data = 21
    print(df)
```

## State

面向对象编程（OOP）侧重于维护彼此交互的对象的状态。在解决许多问题时模拟状态转换的非常方便的工具被称为有限状态机

状态机是一个抽象机器，它有两个关键组件，即**状态**和**转换**。状态机在任何时间点只能有一个活动状态。转换是从当前状态切换到新状态。
- 状态是系统的当前（活动）状态。
- 转换是从一种状态切换到另一种状态

状态机的一个很好的特性是它们可以表示为图形（称为状态图），其中每个状态是一个节点，每个转换是两个节点之间的边缘。


[state_machine](https://github.com/jtushman/state_machine)

```python
@acts_as_state_machine
class Person():
    name = 'Billy'

    sleeping = State(initial=True)
    running = State()
    cleaning = State()

    run = Event(from_states=sleeping, to_state=running)
    cleanup = Event(from_states=running, to_state=cleaning)
    sleep = Event(from_states=(running, cleaning), to_state=sleeping)

    @before('sleep')
    def do_one_thing(self):
        print "{} is sleepy".format(self.name)

    @before('sleep')
    def do_another_thing(self):
        print "{} is REALLY sleepy".format(self.name)

    @after('sleep')
    def snore(self):
        print "Zzzzzzzzzzzz"

    @after('sleep')
    def big_snore(self):
        print "Zzzzzzzzzzzzzzzzzzzzzz"

person = Person()
print person.current_state == Person.sleeping       # True
print person.is_sleeping                            # True
print person.is_running                             # False
person.run()
print person.is_running                             # True
person.sleep()

# Billy is sleepy
# Billy is REALLY sleepy
# Zzzzzzzzzzzz
# Zzzzzzzzzzzzzzzzzzzzzz

print person.is_sleeping                            # True
```

## Interpreter

给定一个语言，定义它的文法的一种表示，并定义一个解释器，这个解释器使用该表示来解释语言中的句子

当有一个语言需要解释执行, 并且你可将该语言中的句子表示为一个抽象语法树时，可使用解释器模式。而当存在以下情况时该模式效果最好：

该文法简单对于复杂的文法, 文法的类层次变得庞大而无法管理。此时语法分析程序生成器这样的工具是更好的选择。它们无需构建抽象语法树即可解释表达式, 这样可以节省空间而且还可能节省时间。

效率不是一个关键问题最高效的解释器通常不是通过直接解释语法分析树实现的, 而是首先将它们转换成另一种形式。例如，正则表达式通常被转换成状态机。但即使在这种情况下, 转换器仍可用解释器模式实现, 该模式仍是有用的。


## Strategy

定义一系列的算法,把它们一个个**封装**起来, 并且根据某些条件使它们可相互替换。本模式使得算法可独立于使用它的客户而变化。

每当我们希望能够动态和透明地应用不同的算法时，策略就是可行的方法

## Memento
对象的内部状态快照

- Memento，一个包含基本状态存储和检索功能的简单对象
- Originator，一个获取和设置Memento实例的值的对象
- Caretaker，存储和检索所有以前创建Memento实例

`pickle`

## Template

该模式侧重于消除代码冗余。我们的想法是，我们应该能够在不改变其结构的情况下重新定义算法的某些部分。

模板设计模式侧重于消除代码重复。如果我们注意到算法中存在具有结构相似性的可重复代码，
我们可以将算法的不变（公共）部分保留在模板方法/函数中，并在动作/钩子方法/函数中移动变体（不同）部分。

```python
def dots_style(msg):
    msg = msg.capitalize()
    msg = '.' * 10 + msg + '.' * 10
    return msg


def admire_style(msg):
    msg = msg.upper()
    return '!'.join(msg)



def generate_banner(msg, style=dots_style):
    print('-- start of banner --')
    print(style(msg))
    print('-- end of banner --\n\n')


def main():
    msg = 'happy coding'
    [generate_banner(msg, style) for style in (dots_style, admire_style)]

if __name__ == "__main__":
    main()

```