---
title:        Python - deque双端队列
date:         2018-05-02
tags:
    - Python
    - 翻译
categories:     Python
---

## class collections.deque([iterable[, maxlen]])

### Deque 双端队列介绍

初始化时，传入一个**可迭代**的数据，将返回一个从左到右的新的deque对象(可以理解为使用`append()`来遍历并添加数据中的元素到队列右端)。如果没有指定一个初始值，则生成一个长度为0的deque。

双端队列（Deque）是堆栈和队列的一般化(读音是“deck”，是“double-end queue”的缩写)。Deque是**线程安全且高效**的，从队列两端添加（append）或弹出（pop）元素的复杂度大约仅**O(1)**。

虽然`list`对象支持类似的操作，但是list的只是对固定长度的列表做优化，在执行改变数据元素位置和列表长度的操作（如`pop(0)`弹出第一个元素，`insert(0,v)`在某个位置插入元素）时，复杂度达到**O(n)**。

如果`maxlen`没有指定或为`None`，那么deque可以增长到任意长度。否则，deque最大长度将限制为`maxlen`。一旦一个有界长度的deque被填满，添加新元素时，相应数目的元素就会从相反的一端被丢弃。有界双端队列提供了类似于`Unix`中的`tail`过滤器的功能。它们对于跟踪事务和一些数据池也很有用，因为只有最近的活动才是程序关心的。

<!--more-->

### Deque 对象支持以下方法：

- append(x)
将元素 x 添加到队列右端

- appendleft(x)
将元素 x 添加到队列左端

- clear()
清空队列，并将队列长度置0

- copy()
浅复制该队列
>Python 3.5 的新特性

- count(x)
计算队列中，值等于 x 的个数
>Python 3.2 的新特性

- extend(iterable)
往队列右端添加可迭代的元素

- extendleft(iterable)
往队列右端添加可迭代的元素。需要注意的是，可迭代的元素集将会以倒序形式体现在最终的队列中：
 ```
 >>> import collections
 >>> d = collections.deque()  
 >>> d.extendleft(range(6))  
 >>> d
 deque([5, 4, 3, 2, 1, 0])
 ```

- index(x[, start[, stop]])
返回元素 x 在队列中的坐标。如果有 start 和 stop 参数，则在以 start 和 stop 为起止索引的子队列中查找该元素 x ，若存在返回元素 x 在完整队列中的坐标。如果元素不在目标队列中，则触发 `ValueError` 错误
>Python 3.5 的新特性

- insert(i, x)
将元素 x 插入到队列的 i 位置。
若插入的位置超过**有界双端队列**长度，则会触发 `IndexError`
>Python 3.5 的新特性

- pop()
弹出（移除并返回）队列最右端的元素。若队列为空，则触发`IndexError`错误

- popleft()
弹出（移除并返回）队列最左端的元素。若队列为空，则触发`IndexError`错误

- remove(value)
移除队列中第一个出现的值为 value 的元素。若不存在这样的元素，则触发`ValueError`错误

- reverse()
反转队列中的元素，并返回`None`
>Python 3.2 的新特性

- rotate(n=1)
将deque视为首尾相连的队列，若 n 为正整数，向右移 n 位。n 默认为 1，即当不指定 n 值时，默认右移 1 位。n 可以取负值，表示向左移 n 位：
 ```
 >>> from collections import deque
 >>> d = deque(range(6))
 >>> d
 deque([0, 1, 2, 3, 4, 5])
 >>> d.rotate()    # 向右移1位
 >>> d
 deque([5, 0, 1, 2, 3, 4])
 >>> d.rotate(-3)    # 向左移3位
 >>> d
 deque([2, 3, 4, 5, 0, 1])
 ``` 
当队列不为空时，右移一位相当于`d.appendleft(d.pop())`，而左移一位相当于`d.append(d.popleft())`

Deque对象还提供了一个只读属性：
- maxlen
如果队列有界，返回队列最大长度，否则返回None
>Python 3.1 的新特性

除了上述的用法，deque同样支持迭代、持久化，能够用 `len(d)`, `reversed(d)`, `copy.copy(d)`, `copy.deepcopy(d)`等方法，也能用 `in` 运算符操作deque对象中的元素，以及下标索引例如`d[-1]`。对两端的索引操作复杂度是**O(1)**，但是当操作中间元素时，复杂度会上升到**O(n)**。想要快速随机访问队列中的元素，用`list`更合适。

从 Python 3.5 开始，deque 支持 `__add__()` (两个deque相加)、`__mul__()` (deque * n 倍)和 `__imul__()` (deque * n 倍后赋值给自身)。

### 方法示例：

```
>>> from collections import deque
>>> d = deque('ghi') # make a new deque with three items
>>> for elem in d: # iterate over the deque's elements
... print(elem.upper())
G
H
I
>>> d.append('j') # add a new entry to the right side
>>> d.appendleft('f') # add a new entry to the left side
>>> d # show the representation of the deque
deque(['f', 'g', 'h', 'i', 'j'])
>>> d.pop() # return and remove the rightmost item
'j'
>>> d.popleft() # return and remove the leftmost item
'f'
>>> list(d) # list the contents of the deque
['g', 'h', 'i']
>>> d[0] # peek at leftmost item
'g'
>>> d[-1] # peek at rightmost item
'i'
>>> list(reversed(d)) # list the contents of a deque in reverse
['i', 'h', 'g']
>>> 'h' in d # search the deque
True
>>> d.extend('jkl') # add multiple elements at once
>>> d
deque(['g', 'h', 'i', 'j', 'k', 'l'])
>>> d.rotate(1) # right rotation
>>> d
deque(['l', 'g', 'h', 'i', 'j', 'k'])
>>> d.rotate(-1) # left rotation
>>> d
deque(['g', 'h', 'i', 'j', 'k', 'l'])
>>> deque(reversed(d)) # make a new deque in reverse order
deque(['l', 'k', 'j', 'i', 'h', 'g'])
>>> d.clear() # empty the deque
>>> d.pop() # cannot pop from an empty deque
Traceback (most recent call last):
    File "<pyshell#6>", line 1, in -toplevel-
        d.pop()
IndexError: pop from an empty deque
>>> d.extendleft('abc') # extendleft() reverses the input order
>>> d
deque(['c', 'b', 'a'])
```

### 应用场景

本节介绍几种deque的具体应用。

有界双端队列提供了一个类似Unix系统下的`tail`命令功能：
```
def tail(filename, n=10):
    'Return the last n lines of a file'
    with open(filename) as f:
        return deque(f, n)
```

deque的另一种使用场景可以是右进左出，用于操作最近添加的几个元素。下面是一个计算相邻几个元素平均值的函数：
```
def moving_average(iterable, n=3):
    # moving_average([40, 30, 50, 46, 39, 44]) --> 40.0 42.0 45.0 43.0
    # http://en.wikipedia.org/wiki/Moving_average
    it = iter(iterable)
    d = deque(itertools.islice(it, n-1))
    d.appendleft(0)
    s = sum(d)
    for elem in it:
        s += elem - d.popleft()
        d.append(elem)
        yield s / n
```

deque的`rotate()`方法提供了一种切片和删除的方法。举个例子，使用纯Python实现`del d[n]`功能（将某个位置的元素弹出）依赖于`rotate()`方法：
```
def delete_nth(d, n):
    d.rotate(-n)
    d.popleft()
    d.rotate(n)
```

将需要删除的第 n 位元素用`rotate()`移至队列最左端，然后用`popleft()`弹出最左端的这个元素，再将剩下的元素移位回去。

再对这个方法进行一些小改动，还可以很容易地实现`Forth`语言中对堆栈的操作方法，例如`dup`, `drop`, `swap`, `over`, `pick`, `rot`, 和 `roll`。

## 翻译自

[1] [collections.deque](https://docs.python.org/3/library/collections.html#deque-objects)