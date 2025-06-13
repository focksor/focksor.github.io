# Python 删除已导入的模块


## 概述

本文主要讨论 Python 应用如何在 `import` 一个模块之后 `de-import/un-import` 此模块。

## 背景

在使用 Python 的过程中，我们很多时候需要 import 一个库来帮助自己完成一些代码逻辑。

然而，import 一个库意味着会引入额外的内存使用，而在一些内存受限的环境（如嵌入式设备上），内存是很珍贵的，如果只是为了一个使用频次很低的功能引入一个库会多少显得有些奢侈。

因此，Python 应用如何在 import 一个库并使用完成后 "de-import" ，在某些时候显得尤为重要。

笔者查阅了互联网上的一些资料，均未能达到释放内存的效果，因此写下了这篇文档。

## How To

### 判断规则

判断一个库是否已释放的最重要标准是引入库前和释放库后的内存用量相差不大，这样才能起到释放内存的作用。如果只是代码中无法使用库及相关接口，但是内存还是贮留在系统中，那意义就不大了。

综上，判断一个库是否已经被释放的步骤如下：
1. 记录此时 Python 应用内存用量为 A
2. 引入一个库
3. 使用该库进行一些操作
4. 释放该库
5. 记录此时 Python 应用内存用量为 B
6. 如果 B 与 A 相差不大，则认为该库已完成释放

为了方便描述，我们在下文中均以导入 `asyncio` 模块为例（因为它足够大，且不用额外安装）。

### 使用 del 释放 

我们知道，已导入的库会索引于 `sys.modules` 中，所以是不是可以直接删除索引中这个库就可以了呢？

在 [How to de-import a Python module? - Stack Overflow](https://stackoverflow.com/questions/1668223/how-to-de-import-a-python-module)  这篇帖子中，有人就是这样问的，让我们试一下：

```python
# file: de_import_with_del.py
import gc
import sys

with open('/proc/self/status') as f:
    print('init', ''.join(line for line in f if line.startswith('VmRSS')).strip())

import asyncio

with open('/proc/self/status') as f:
    print('imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())

del asyncio
del sys.modules['asyncio']
gc.collect()

with open('/proc/self/status') as f:
    print('de-imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())
```

执行结果如下：
```shell
$ python de_import_with_del.py 
init VmRSS:         9856 kB
imported VmRSS:    20092 kB
de-imported VmRSS:         20092 kB
```

可以看到，这样并不能真正释放已导入模块的内存。

不过，这篇帖子下面还贴了另一个邮件[[Tutor] How to un-import modules?](https://mail.python.org/pipermail/tutor/2006-August/048596.html)，上面提到有一种更进阶的方式进行模块释放，其原理是不仅解除了对模块的引用，还解除了其子模块的使用。

这听起来好像很有道理，让我们再试试：
```python
# file: de_import_deep_del.py
import gc


def delete_module(modname, paranoid=None):
    from sys import modules
    try:
        thismod = modules[modname]
    except KeyError:
        raise ValueError(modname)
    these_symbols = dir(thismod)
    if paranoid:
        try:
            paranoid[:]  # sequence support
        except:  # noqa: E722
            raise ValueError('must supply a finite list for paranoid')
        else:
            these_symbols = paranoid[:]
    del modules[modname]
    for mod in modules.values():
        try:
            delattr(mod, modname)
        except AttributeError:
            pass
        if paranoid:
            for symbol in these_symbols:
                if symbol[:2] == '__':  # ignore special symbols
                    continue
                try:
                    delattr(mod, symbol)
                except AttributeError:
                    pass


with open('/proc/self/status') as f:
    print('init', ''.join(line for line in f if line.startswith('VmRSS')).strip())

import asyncio  # noqa: E402

with open('/proc/self/status') as f:
    print('imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())

del asyncio
delete_module('asyncio')
gc.collect()

with open('/proc/self/status') as f:
    print('de-imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())
```

执行结果如下：
```shell
$ python de_import_deep_del.py 
init VmRSS:         9984 kB
imported VmRSS:    20352 kB
de-imported VmRSS:         20352 kB
```

很可惜，还是不行。


### 使用 deimport 库

[probml/deimport](https://github.com/probml/deimport) 库宣称可以释放已导入的库，让我们试试：

```python
# file: demo_of_using_deimport.py
import gc
from deimport.deimport import deimport


with open('/proc/self/status') as f:
    print('init', ''.join(line for line in f if line.startswith('VmRSS')).strip())

import asyncio

with open('/proc/self/status') as f:
    print('imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())

deimport(asyncio)
gc.collect()

with open('/proc/self/status') as f:
    print('de-imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())

```

执行结果如下：
```shell
$ python demo_of_using_deimport.py 
init VmRSS:        15340 kB
imported VmRSS:    24044 kB
de-imported VmRSS:         24044 kB
```

很可惜，还是不行。从其源码可以看到，它相比上面我们介绍的 del 方法无非是多删除了 `globals()` 内的引用。


### 另辟蹊径

回过头来想一下，从运行时中删除一个已导入模块的难点在哪？笔者觉得主要有两点：
1. 导入模块产生了很多变量和字节码，这些对象可能在很多地方都有引用
2. 一般来说，导入一个稍微大点的模块就会不可避免地导入该模块的依赖模块。这会导致我们在导入一个模块的时候实际上导入了多个模块，而我们有不可能去一一识别然后删除

问题 1 倒是还有解决的余地——我们只需要查看 gc 中的相关引用情况，然后一一删除就好。而问题 2 就比较棘手了，我们很难在导入一个模块时去一一分析其依赖模块的导入，依赖模块是否被别的模块依赖……等等。

让我们把思路放宽一下，另辟蹊径地想一下：什么一定要删除模块呢，不引入不就好了？我们知道一个进程 fork 出来的子进程会继承主进程的资源，而子进程结束并不会影响主进程的运行。那是不是可以先创建一个子进程，然后在这个子进程中完成脏活累活，最后销毁该子进程——这样是不是主进程并不用导入模块，而又利用该模块完成了预定任务？让我们试试。

#### 使用 fork+pipe

代码如下：

```python
import os
import pickle


def run_in_other_process(func, *args, **kwargs):
    """
    Run a function in another process using os.fork and os.pipe.
    """
    read_fd, write_fd = os.pipe()
    pid = os.fork()

    if pid == 0:  # Child process
        os.close(read_fd)
        try:
            result = func(*args, **kwargs)
            with os.fdopen(write_fd, 'wb') as wf:
                wf.write(pickle.dumps(result))
        except Exception as e:
            with os.fdopen(write_fd, 'wb') as wf:
                wf.write(pickle.dumps(e))
        os._exit(0)
    else:  # Parent process
        os.close(write_fd)
        with os.fdopen(read_fd, 'rb') as rf:
            result = pickle.loads(rf.read())
        if isinstance(result, Exception):
            raise result
        return result


def a_task_using_asyncio(a):
    with open('/proc/self/status') as f:
        print('task init', ''.join(line for line in f if line.startswith('VmRSS')).strip())
    import asyncio  # noqa: F401
    with open('/proc/self/status') as f:
        print('imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())
    return a ** a


with open('/proc/self/status') as f:
    print('init', ''.join(line for line in f if line.startswith('VmRSS')).strip())

assert run_in_other_process(a_task_using_asyncio, 3) == 27

with open('/proc/self/status') as f:
    print('task done', ''.join(line for line in f if line.startswith('VmRSS')).strip())

```

运行结果：
```shell
$ python demo_of_using_fork+pipe.py 
init VmRSS:        17120 kB
task init VmRSS:           11896 kB
imported VmRSS:    21624 kB
task done VmRSS:           17248 kB
```


#### 使用 fork+socketpair

代码如下：
```python
import os
import pickle
import socket


def run_in_other_process(func, *args, **kwargs):
    """
    Run a function in another process using os.fork and socket.socketpair.
    """
    parent_sock, child_sock = socket.socketpair()
    pid = os.fork()

    if pid == 0:  # Child process
        parent_sock.close()
        try:
            result = func(*args, **kwargs)
            child_sock.sendall(pickle.dumps(result))
        except Exception as e:
            child_sock.sendall(pickle.dumps(e))
        finally:
            child_sock.close()
        os._exit(0)
    else:  # Parent process
        child_sock.close()
        result = pickle.loads(parent_sock.recv(4096))
        parent_sock.close()
        if isinstance(result, Exception):
            raise result
        return result


def a_task_using_asyncio(a):
    with open('/proc/self/status') as f:
        print('task init', ''.join(line for line in f if line.startswith('VmRSS')).strip())
    import asyncio  # noqa: F401
    with open('/proc/self/status') as f:
        print('imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())
    return a ** a


with open('/proc/self/status') as f:
    print('init', ''.join(line for line in f if line.startswith('VmRSS')).strip())

assert run_in_other_process(a_task_using_asyncio, 3) == 27

with open('/proc/self/status') as f:
    print('task done', ''.join(line for line in f if line.startswith('VmRSS')).strip())

```


运行结果：
```shell
$ python demo_of_using_fork+socketpair.py 
init VmRSS:        17492 kB
task init VmRSS:           12204 kB
imported VmRSS:    21676 kB
task done VmRSS:           17620 kB
```


#### 使用 multiprocessing

代码如下：
```python
import multiprocessing


def run_in_other_process(func, *args, **kwargs):
    """
    Run a function in another process.
    """
    with multiprocessing.Pool(1) as pool:
        result = pool.apply(func, args, kwargs)
    return result


def a_task_using_asyncio(a):
    with open('/proc/self/status') as f:
        print('task init', ''.join(line for line in f if line.startswith('VmRSS')).strip())
    import asyncio  # noqa: F401
    with open('/proc/self/status') as f:
        print('imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())
    return a ** a


with open('/proc/self/status') as f:
    print('init', ''.join(line for line in f if line.startswith('VmRSS')).strip())

assert run_in_other_process(a_task_using_asyncio, 3) == 27

with open('/proc/self/status') as f:
    print('task done', ''.join(line for line in f if line.startswith('VmRSS')).strip())

```

运行结果：
```shell
$ python demo_of_using_multiprocessing.py 
init VmRSS:        18008 kB
task init VmRSS:           15216 kB
imported VmRSS:    22256 kB
task done VmRSS:           20440 kB
```


#### 使用 ProcessPoolExecutor

代码如下：
```python
from concurrent.futures import ProcessPoolExecutor


def run_in_other_process(func, *args, **kwargs):
    """
    Run a function in another process.
    """
    with ProcessPoolExecutor(max_workers=1) as executor:
        future = executor.submit(func, *args, **kwargs)
        try:
            return future.result()
        except Exception:
            raise


def a_task_using_asyncio(a):
    with open('/proc/self/status') as f:
        print('task init', ''.join(line for line in f if line.startswith('VmRSS')).strip())
    import asyncio  # noqa: F401
    with open('/proc/self/status') as f:
        print('imported', ''.join(line for line in f if line.startswith('VmRSS')).strip())
    return a ** a


with open('/proc/self/status') as f:
    print('init', ''.join(line for line in f if line.startswith('VmRSS')).strip())

assert run_in_other_process(a_task_using_asyncio, 3) == 27

with open('/proc/self/status') as f:
    print('task done', ''.join(line for line in f if line.startswith('VmRSS')).strip())

```

运行结果：
```shell
$ python demo_of_using_ProcessPoolExecutor.py 
init VmRSS:        20692 kB
task init VmRSS:           15484 kB
imported VmRSS:    22396 kB
task done VmRSS:           20820 kB
```

### 总结

从上个章节，我们可以看到各个方法主进程在执行任务的前后内存用量变化非常少（应该是启动子进程以及进程间通信带来的开销），而主要的内存用量都体现在子进程上了——由于子进程在执行完任务之后会释放，所以完全没有关系。

当然，你可能注意到使用有些方法初始化的内存会比较大，这是因为导入用于创建子进程以及进行进程间通信的库也会消耗相当量的内存。你可以根据自己项目中导入了哪些库、任务传递数据的格式和实际的工程情况，决定你使用什么方法实现 `run_in_other_process` 中关于子进程启动和进程间通信的部分。


## 资源

1. 本文档所有相关文件和脚本均已上传到 [Github](https://github.com/focksor/notebook_demo__python_de_import_package)
2. [How to de-import a Python module? - Stack Overflow](https://stackoverflow.com/questions/1668223/how-to-de-import-a-python-module)
3. [[Tutor] How to un-import modules?](https://mail.python.org/pipermail/tutor/2006-August/048596.html)

