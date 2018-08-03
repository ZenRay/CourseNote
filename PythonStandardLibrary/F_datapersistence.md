**目录**

[TOC]

数据的持久存储和交换，包括内存中对象和持久保存格式之间的来回转换，以及处理转换后数据的存储。主要由两个模块可以将对象转换为一种可以传输和存储的格式——这个转换的过程是序列化（**Serializing**）。其中 `pickle` 多用于存储，这是因为它可以集成到其他可以存储序列化数据的标准库中（例如 `shelve`）。`JSON` 是另一个常用于基于 web 的应用（**web-based application**），这是因为它和现有的 web 服务存储工具整合良好。

一旦内存对象被转换为可以被存储的格式，下一步就是需要确认怎么存储数据。如果数据不需要以某种方式索引，那么依序先后写入序列化对象的简单平面文件（**flat-file**，这类文件主要是文件格式，且文件之间是独立的。例如 `CSV` 文件）就很适用。`Python` 中有一组模块可以在一个简单的数据库中存储键值对，需要索引查找时会使用某种变种的 `DBM` 格式。

在使用 `DBM` 格式方式中，最直接的的方式是使用 `shelve`，访问一个打开的 `shelve` 文件可以通过类似字典 `API` 来进行访问。而且保存到数据库的对象会自动 `pickle` 并保存，而无须调用其他方法。但是也存在一个缺点，使用默认接口时没有办法预测将要使用哪一个 `DBM` 格式，所以在涉及到移植性时需要将用模块中特定的类来确保选用特定的格式。

对于已经使用 `JSON` 数据的 web 应用，`json` 和 `dbm` 提供了另一种持久化机制。直接使用 `dbm` 比 `shelve` 需要多一点工作，因为 `DBM` 数据库中键和值必须是字符串，并且值在访问时不会自动重新创建。



## 1. `pickle` 对象序列化

一旦数据被序列化，你就可以把它写入到文件、socket、管道等等中。之后可以读取这个文件，反序列化这些数据来构造具有相同值的新对象。

```python
"ADDITEMS", "APPEND", "APPENDS", "BINBYTES", "BINBYTES8", "BINFLOAT", "BINGET", "BININT", "BININT1", "BININT2", "BINPERSID", "BINPUT", "BINSTRING", "BINUNICODE", "BINUNICODE8", "BUILD", "DEFAULT_PROTOCOL", "DICT", "DUP", "EMPTY_DICT", "EMPTY_LIST", "EMPTY_SET", "EMPTY_TUPLE", "EXT1", "EXT2", "EXT4", "FALSE", "FLOAT", "FRAME", "FROZENSET", "FunctionType", "GET", "GLOBAL", "HIGHEST_PROTOCOL", "INST", "INT", "LIST", "LONG", "LONG1", "LONG4", "LONG_BINGET", "LONG_BINPUT", "MARK", "MEMOIZE", "NEWFALSE", "NEWOBJ", "NEWOBJ_EX", "NEWTRUE", "NONE", "OBJ", "PERSID", "POP", "POP_MARK", "PROTO", "PUT", "PickleError", "Pickler", "PicklingError", "PyStringMap", "REDUCE", "SETITEM", "SETITEMS", "SHORT_BINBYTES", "SHORT_BINSTRING", "SHORT_BINUNICODE", "STACK_GLOBAL", "STOP", "STRING", "TRUE", "TUPLE", "TUPLE1", "TUPLE2", "TUPLE3", "UNICODE", "Unpickler", "UnpicklingError", "_Framer", "_Pickler", "_Stop", "_Unframer", "_Unpickler", "__all__", "__builtins__", "__cached__", "__doc__", "__file__", "__loader__", "__name__", "__package__", "__spec__", "_compat_pickle", "_dump", "_dumps", "_extension_cache", "_extension_registry", "_getattribute", "_inverted_registry", "_load", "_loads", "_test", "_tuplesize2code", "bytes_types", "codecs", "compatible_formats", "decode_long", "dispatch_table", "dump", "dumps", "encode_long", "format_version", "io", "islice", "load", "loads", "maxsize", "pack", "partial", "re", "sys", "unpack", "whichmodule"
```

这里可以使用 `dumps` 和 `loads` 方法，它们两者是直接将对象进行序列化和反序列化。需要注意⚠️反序列化后的数据与源数据相等，但是并不是之前的对象。

```python
import pickle
import pprint

data1 = [{'a': 'A', 'b': 2, 'c': 3.0}]
# 这里是直接打印出了序列化对象和原始对象
print('DATA:', end=' ')
pprint.pprint(data)

data_string = pickle.dumps(data)
print('PICKLE: {!r}'.format(data_string)

# output
DATA: [{'a': 'A', 'b': 2, 'c': 3.0}]
PICKLE: b'\x80\x03]q\x00}q\x01(X\x01\x00\x00\x00cq\x02G@\x08\x00
\x00\x00\x00\x00\x00X\x01\x00\x00\x00bq\x03K\x02X\x01\x00\x00\x0
0aq\x04X\x01\x00\x00\x00Aq\x05ua.'

# 下面主要是为了说明序列化和反序列化后数据的差异
print('BEFORE: ', end=' ')
pprint.pprint(data1)

data1_string = pickle.dumps(data1)

data2 = pickle.loads(data1_string)
print('AFTER : ', end=' ')
pprint.pprint(data2)

print('SAME? :', (data1 is data2))
print('EQUAL?:', (data1 == data2))

# output
BEFORE:  [{'a': 'A', 'b': 2, 'c': 3.0}]
AFTER :  [{'a': 'A', 'b': 2, 'c': 3.0}]
SAME? : False
EQUAL?: True
```

⚠️：`pickle` 的文档清晰的表明它不提供安全保证。实际上，反序列化后可以执行任意代码，所以慎用 `pickle`来作为内部进程通信或者数据存储，也不要相信那些你不能验证安全性的数据。请参阅 `hmac`模块，它提供了一个以安全方式验证序列化数据源的示例。

### 1.1 流序列化处理

`pickle`  除了提供  `dumps` 和  `loads` ，还提供了非常方便的函数用于操作文件流。支持同时写多个对象到同一个流中，然后在不知道有多少个对象或不知道它们有多大时，能够从这个流中读取到这些对象。需要注意另外的两个方法 `dump` 和 `load` ，它们是将序列化对象保存到文件或者从文件中读取内容。

下面的例子中使用两个 `BytesIO` 缓冲区来模拟流。一个接收序列化对象，另一个通过 `load` 方法读取第一个的值。一个简单的数据库格式也可以使用序列化来存储对象。 `shelve` 模块就是按照这个方式来处理数据。

```python
import io
import pickle
import pprint


class SimpleObject:

    def __init__(self, name):
        self.name = name
        self.name_backwards = name[::-1]
        return


data = []
data.append(SimpleObject('pickle'))
data.append(SimpleObject('preserve'))
data.append(SimpleObject('last'))

# Simulate a file.
out_s = io.BytesIO()

# Write to the stream
for o in data:
    print('WRITING : {} ({})'.format(o.name, o.name_backwards))
    pickle.dump(o, out_s)
    out_s.flush()

# Set up a read-able stream
in_s = io.BytesIO(out_s.getvalue())

# Read the data
while True:
    try:
        o = pickle.load(in_s)
    except EOFError:
        break
    else:
        print('READ    : {} ({})'.format(
            o.name, o.name_backwards))
        
# output
WRITING : pickle (elkcip)
WRITING : preserve (evreserp)
WRITING : last (tsal)
READ    : pickle (elkcip)
READ    : preserve (evreserp)
READ    : last (tsal)
```

除了用于存储数据，序列化在用于内部进程通信时也是非常灵活的。比如，使用  `os.fork()` 和 `os.pipe()` ，可以建立一些工作进程，它们从一个管道中读取任务说明并把结果输出到另一个管道。操作这些工作池、发送任务和接受返回的核心代码可以复用，因为任务和返回对象不是一个特殊的类。如果使用管道或者套接字，就不要忘记在序列化每个对象后刷新它们，并通过它们之间的连接将数据推送到另外一端。查看`multiprocessing` 模块构建一个可复用的任务池管理器

### 1.2 对象重构问题

当一个自定义类的对象被保存后，如果没有将类引入加载方法的所在进程的命名空间中，将出现报错。所以为了解决这个问题需要将需要的类、方法等 `import` 到相应的文件中

```python
# pickle_dump_to_file_1.py
import pickle
import sys


class SimpleObject:

    def __init__(self, name):
        self.name = name
        l = list(name)
        l.reverse()
        self.name_backwards = ''.join(l)


if __name__ == '__main__':
    data = []
    data.append(SimpleObject('pickle'))
    data.append(SimpleObject('preserve'))
    data.append(SimpleObject('last'))

    filename = sys.argv[1]

    with open(filename, 'wb') as out_s:
        for o in data:
            print('WRITING: {} ({})'.format(
                o.name, o.name_backwards))
            pickle.dump(o, out_s)
            
# 运行脚本
$ python3 pickle_dump_to_file_1.py test.dat

WRITING: pickle (elkcip)
WRITING: preserve (evreserp)
WRITING: last (tsal)
```

上面是将文件保存 `dump` 到了 `test.dat`，下面的方法是直接进行 `load`

```python
# pickle_load_from_file_1.py
import pickle
import pprint
import sys

filename = sys.argv[1]

with open(filename, 'rb') as in_s:
    while True:
        try:
            o = pickle.load(in_s)
        except EOFError:
            break
        else:
            print('READ: {} ({})'.format(
                o.name, o.name_backwards))
                
# 运行脚本
$ python3 pickle_load_from_file_1.py test.dat

Traceback (most recent call last):
  File "pickle_load_from_file_1.py", line 15, in <module>
    o = pickle.load(in_s)
AttributeError: Can't get attribute 'SimpleObject' on <module '_
_main__' from 'pickle_load_from_file_1.py'>
```

上面的就是因为没有相应的类，而出现报错。修正这个问题需要在脚本中加入 `from pickle_dump_to_file_1 import SimpleObject` 即可

### 1.3 不可`pickle` 对象

不是所有对象都可以被序列化的，如套接字、文件句柄、数据库连接或其他运行时状态可能依赖于操作系统或其他进程的对象，则无法有效的存储下来。如果对象具有不可 `pickle` 属性，可以定义 `__getstate__` 和 `__setstate__` 方法来返回实例在被序列化时的状态。

这个 `__getstate__` 方法须返回对象，它包含该对象内部状态。一种便捷的方式是使用字典来表达状态，字典的值可以是任意可序列化的对象。当对象通过 `pickle` 加载时，状态会被存储，并且传递给 `__setstate__` 方法。下面是使用一个单独的 `State` 对象存储 `MyClass` 的内部状态。当 `MyClass` 的实例反序列化时，会给  `__setstate__` 传入一个 `State` 的实例去初始化新的对象。⚠️如果 `__getstate__()` 返回值是 false，则 `__setstate__()` 在对象反序列化时不会被调用。

```python
import pickle


class State:

    def __init__(self, name):
        self.name = name

    def __repr__(self):
        return 'State({!r})'.format(self.__dict__)


class MyClass:

    def __init__(self, name):
        print('MyClass.__init__({})'.format(name))
        self._set_name(name)

    def _set_name(self, name):
        self.name = name
        self.computed = name[::-1]

    def __repr__(self):
        return 'MyClass({!r}) (computed={!r})'.format(
            self.name, self.computed)

    def __getstate__(self):
        state = State(self.name)
        print('__getstate__ -> {!r}'.format(state))
        return state

    def __setstate__(self, state):
        print('__setstate__({!r})'.format(state))
        self._set_name(state.name)


inst = MyClass('name here')
print('Before:', inst)

dumped = pickle.dumps(inst)

reloaded = pickle.loads(dumped)
print('After:', reloaded)

# output
MyClass.__init__(name here)
Before: MyClass('name here') (computed='ereh eman')
__getstate__ -> State({'name': 'name here'})
__setstate__(State({'name': 'name here'}))
After: MyClass('name here') (computed='ereh eman')
```

## 2. `shelve` ——对象的持久存储

`shelve` 模块使用一种类字典的 `API`，可以持久存储可 `pickle` 的任意 `Python` 对象，对不需要关系数据库时，该模块特别有用。

```python
"BsdDbShelf", "BytesIO", "DbfilenameShelf", "Pickler", "Shelf", "Unpickler", "_ClosedDict", "__all__", "__builtins__", "__cached__", "__doc__", "__file__", "__loader__", "__name__", "__package__", "__spec__", "collections", "open"
```

### 2.1 创建一个新的 `shelf`

最简单的使用 `shelve` 模块的方式是通过 `DbfilenameShelf` 类—— 该类使用 `dbm` 来存储数据。你可以直接使用该类，或者调用 `shelve.open()` 方法。

```python
import shelve

with shelve.open('test_shelf.db') as s:
    s['key1'] = {
        'int': 10,
        'float': 9.5,
        'string': 'Sample data',
    }
    
# 再次访问数据，打开 shelf 并且像字典一样使用它
with shelve.open('test_shelf.db') as s:
    existing = s['key1']

print(existing)

# output
{'int': 10, 'float': 9.5, 'string': 'Sample data'}
```

`dbm`模块不支持多个应用同时写入同一数据库。如果你确定客户端不会修改 `shelf`， 请传入 `flag='r'` 来指定 `shelve` 以只读方式打开数据库。当数据库以只读方式打开时，使用者又尝试着更改数据库，这将引起一个访问出错的异常。这一异常类型依赖于在创建数据库时被 `dbm`选择的数据库模块。

```python
import shelve

with shelve.open('test_shelf.db', flag='r') as s:
    print('Existing:', s['key1'])
    try:
        s['key1'] = 'new value'
    except dbm.error as err:
        print('ERROR: {}'.format(err))
        
# output
Existing: {'int': 10, 'float': 9.5, 'string': 'Sample data'}
ERROR: cannot add item to database
```

### 2.2 写回——`write back`

默认情况下，`shelf` 不会跟踪可变对象的修改，这样如果存储在 `shelf` 中的一个元素内容有变化，`shelf` 必须通过再次存储整个元素来显式更新。下面的例子中， `key1` 中存在变化但是并没有被存储，所以重新打开 `shelf` 时，修改并没有体现。

```python
import shelve

with shelve.open('test_shelf.db') as s:
    print(s['key1'])
    s['key1']['new_value'] = 'this was not here before'

with shelve.open('test_shelf.db', writeback=True) as s:
    print(s['key1'])
    
# output
{'int': 10, 'float': 9.5, 'string': 'Sample data'}
{'int': 10, 'float': 9.5, 'string': 'Sample data'}
```

对于 `shelf` 中存储的可变对象，为了自动捕获其修改，打开 `shelf` 时可以启动写回，需要启动协会标志使得 `shelf` 使用内存中缓存记住从数据库获取的所有对象。`shelf` 关闭时每个缓存对象也写回到数据库。⚠️需要开启协会标志。

```python
import shelve
import pprint

with shelve.open('test_shelf.db', writeback=True) as s:
    print('Initial data:')
    pprint.pprint(s['key1'])

    s['key1']['new_value'] = 'this was not here before'
    print('\nModified:')
    pprint.pprint(s['key1'])

with shelve.open('test_shelf.db', writeback=True) as s:
    print('\nPreserved:')
    pprint.pprint(s['key1'])
    
# output
Initial data:
{'float': 9.5, 'int': 10, 'string': 'Sample data'}

Modified:
{'float': 9.5,
 'int': 10,
 'new_value': 'this was not here before',
 'string': 'Sample data'}

Preserved:
{'float': 9.5,
 'int': 10,
 'new_value': 'this was not here before',
 'string': 'Sample data'}
```

尽管这样会减少程序员犯错的机会，并且能够使对象持久存储特命，但是并非所有情况下都需要使用写回模式。打开 `shelf` 时使用缓存也会消耗额外的内存，它关闭时也会暂停其他各个缓存对象写回到数据库，这样就会使应用 **速度变慢**。所有缓存的对象都要写回数据库，因为 **区分它们是否有修改**。如果应用读取的数据对于写的数据，写回会影响性能并且没有太大的意义。

此外，上面的例子全都使用了默认的 shelf 实现。使用 `shelve.open`  方法而非直接使用 `shelf` 实现，这是常见用法，特别是在使用哪种数据库存储数据无关紧要的时候。然而在某些时候，需要关关注数据库格式的时候，通常就会直接使用 `DbfilenameShelf` 或者 `BsdDbShelf` ，甚至通过 `Shelf` 子类来定制化解决相应的问题。

## 3. `dbm` —— `Unix` 键值数据库

作用是 `dbm` 提供了针对 `DBM-style` 、 `string-keyed` 数据库的类似字典的接口。`dbm` 是一个前端 `DBM` 式数据库，这个数据库能利用简单的字符串值作为键来访问包含字符串的记录。其中使用 `whichdb` 方法会识别数据库，并使用合适的模块来打开数据库。它被用作 `shelve` 的后端，`shelf` 使用 `pickle` 将对象存储在一个 `DBM` 数据库中。

```python
"__all__", "__builtins__", "__cached__", "__doc__", "__file__", "__loader__", "__name__", "__package__", "__path__", "__spec__", "_defaultmod", "_modules", "_names", "error", "io", "ndbm", "open", "os", "struct", "sys", "whichdb"
```

### 3.1 数据库类型

`Python` 提供了多个模块来访问 `DBM` 数据库，默认的实现取决于当前系统上可用的库以及 编译 `Python` 时使用选项。与特定实现分离的接口允许 `Python` 程序与其他语言的程序交换数据，且不会自动在可用格式之间切换，也可编写在多个平台上运行的便携式数据文件。

#### 3.1.1 `dbm.gnu`

`dbm.gnu` 是来自 `GNU` 项目中 `dbm` 库版本的接口。 它实现方式与这里描述的其他 `DBM` 相同，但在 `open` 方法中提供的 `flags`标志进行了一些更改。除了标准的 `'r'`, `'w'`, `'c'`, `'n'` 标志， `dbm.gnu.open()` 还提供了：

* `'f'` 表示使用 **fast** 模式打开数据库。这种模式下，写入数据库不是同步的
* `'s'` 表示使用 **synchronized** 模式打开数据库。这种模式下，对数据库所做的更改会立即写入文件，而不是在数据库关闭或其他同步操作时进行延迟写入
* `'u'` 表示不加锁地打开数据库

#### 3.1.2 `dbm.ndbm`

`dbm.ndbm` 模块提供了一个接口来 `Unix` 上 `ndbm` 方式的执行 `dbm` 格式，它依赖于在编译期间模块的配置。模块的 `library` 属性会识别出 `configure` 库的名称然后确认出扩展模块是什么时候进行编译的。

#### 3.1.3 `dbm.dump`

当没有其他实现可用时， `dbm.dumb` 模块为 `DBM` 的 `API` 提供了一个可移植的后备实现。 使用 `dbm.dumb` 不需要外部依赖，但比其他大多数实现要慢。

### 3.2 创建一个新数据库

新数据库的存储格式，会按照下面的顺序查找：

1. `dbm.gnu`
2. `dbm.ndbm`
3. `dbm.dump`

`open` 函数可以接收一些标志来控制如何管理数据库文件。必要时，创建一个新的数据库，可以用 `‘c’`，而使用 `‘n’` 则会创建一个新数据库而覆盖现有的文件

```python
# dbm_new.py
import dbm

with dbm.open('/tmp/example.db', 'n') as db:
    db['key'] = 'value'
    db['today'] = 'Sunday'
    db['author'] = 'Doug'
    
# 上面的脚本使文件总会重新初始化
```

使用 `whichdb`  方法可以报告数据库创建的类型

```python
import dbm

print(dbm.whichdb('/tmp/example.db'))

# 这里的数据库依赖上面的数据库建立
# output
dbm.ndbm
```

### 3.3 打开现有数据库

要打开现有数据库，需要使用 `‘r’` 或者 `‘w’` 标志，这样会把现有数据库自动传给 `whichdb` 识别，因此只要一个文件可以识别，就会使用一个适当的模式来打开这个文件。一旦打开了文件，`db` 是一个类字典对象，并支持所有常用的方法。

```python
import dbm

with dbm.open('/tmp/example.db', 'r') as db:
    print('keys():', db.keys())
    for k in db.keys():
        print('iterating:', k, db[k])
    print('db["author"] =', db['author'])
    
# output
keys(): [b'key', b'today', b'author']
iterating: b'key' b'value'
iterating: b'today' b'Sunday'
iterating: b'author' b'Doug'
db["author"] = b'Doug'
```

在该数据库中，需要注意⚠️ 键只能是字符串，而值只能是字符串或者 `None`

>  



## 参考

1. 数据序列化

   将结构化数据转换成允许以共享或存储的格式，可恢复其原始结构的概念

2. [Pickle: An interesting stack language.](http://peadrop.com/blog/2007/06/18/pickle-an-interesting-stack-language/) – by Alexandre Vassalotti

3. [**PEP 3154**](https://www.python.org/dev/peps/pep-3154) – Pickle protocol version 4

4. [feedcache](https://bitbucket.org/dhellmann/feedcache) -  `feedcache` 模块使用 `shelve` 做为默认的存储选项。

5. [shove](http://pypi.python.org/pypi/shove/) - Shove 使用更多的后端格式来实现类似的 API 。