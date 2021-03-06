---
layout: post
title:  "Python中列表的常用操作"
categories: Python基础
tags: Python
author: 肖邦
---

* content
{:toc}

> 记录总结 Python 列表常用的操作




## 一、迭代列表时如何访问列表下标索引
普通版本：
```python
items = [8, 23, 45]
for index in range(len(items)):
    print(index, ">>>", items[index])
0 >>> 8
1 >>> 23
2 >>> 45
```
优雅版：
```python
for index, item in enumerate(items):
    print(index, ">>>", item)
0 >>> 8
1 >>> 23
2 >>> 45
```
`enumerate`还可以指定元素的第一个元素从几开始，默认是0，也可以指定从1开始：
```python
for index, item in enumerate(items, start=1):
    print(index, ">>>", item)
1 >>> 8
2 >>> 23
3 >>> 45
```
## 二、append与extend方法有什么区别
`append`表示把某个数据当做新元素追加到列表的最后面，它的参数可以是任意对象
```python
x = [1, 2, 3]
y = [4, 5]
x.append(y)
print(x)
>>>
[1, 2, 3, [4, 5]]
```
`extend`的参数必须是一个可迭代对象，表示把该对象里面的所有元素逐个地追加到列表的后面
```python
x = [1, 2, 3]
y = [4, 5]
x.extend(y)
print(x)
>>>
[1, 2, 3, 4, 5]

# 等价于：
for i in y:
    x.append(i)
```

## 三、检查列表是否为空
普通版：
```python
if len(items) == 0:
    print("空列表")
# 或者
if items == []:
    print("空列表")
```
优雅版：
```python
if not items:
    print("空列表")
```

## 四、如何理解切片
切片用于获取列表中指定范的子集，语法非常简单。
```python
items[start:end:step]
```
从 start 到 end-1 位置之间的元素。step 表示步长，默认为1，表示连续获取，如果 step 为 2 就表示每隔一个元素获取。
```python
a = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

>>> a[3:8] # 第3到第8位置之间的元素
[4, 5, 6, 7, 8]

>>> a[3:8:2] # 第3到第8位置之间的元素，每隔一个元素获取
[4, 6, 8]

>>> a[:5]   # 省略start表示从第0个元素开始
[1, 2, 3, 4, 5]

>>> a[3:]  # 省略end表示到最后一个元素
[4, 5, 6, 7, 8, 9, 10]

>>> a[::]  # 都省略相当于拷贝一个列表，这种拷贝属于浅拷贝
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

## 五、如何拷贝一个列表对象
第一种方法：
```python
new_list = old_list[:]
```
第二种方法：
```python
new_list = list(old_list)
```
第三种方法：
```python
import copy
# 浅拷贝
new_list = copy.copy(old_list)
# 深拷贝
new_list = copy.deepcopy(old_list)
```

## 六、如何获取列表中最后一个元素
索引列表中的元素不仅支持正数还支持负数，正数表示从列表的左边开始索引，负数表示从列表的右边开始索引，获取最后一个元素有两种方法。
```python
>>> a = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
>>> a[len(a)-1]
10
>>> a[-1]
10
```

## 七、如何对列表进行排序
列表排序有两种方式，一种是列表自带的方式 sort，一种是内建函数 sorted。复杂的数据类型可通过指定 key参数进行排序。由字典构成的列表，根据字典元素中的age字段进行排序：
```python
items = [{'name': 'Homer', 'age': 39},
         {'name': 'Bart', 'age': 10},
         {"name": 'cater', 'age': 20}]
items.sort(key=lambda item: item.get("age"))
print(items)
>>>
[{'age': 10, 'name': 'Bart'}, {'age': 20, 'name': 'cater'}, {'age': 39, 'name': 'Homer'}]
```
列表有 sort方法，用于对原列表进行重新排序，指定 key 参数，key 是匿名函数，item 是列表中的字典元素，我们根据字典中的age进行排序，默认是按升序排列，指定 reverse=True 按降序排列
```python
items.sort(key=lambda item: item.get("age"), reverse=True)
>>>
[{'name': 'Homer', 'age': 39}, {'name': 'cater', 'age': 20}, {'name': 'Bart', 'age': 10}]
```
如果不希望改变原列表，而是生成一个新的有序列表对象，那么可以内置函数 sorted ，该函数返回新列表
```python
items = [{'name': 'Homer', 'age': 39},
         {'name': 'Bart', 'age': 10},
         {"name": 'cater', 'age': 20}]
new_items = sorted(items, key=lambda item: item.get("age"))
print(items)
>>>
[{'name': 'Homer', 'age': 39}, {'name': 'Bart', 'age': 10}, {'name': 'cater', 'age': 20}]
print(new_items)
>>>
[{'name': 'Bart', 'age': 10}, {'name': 'cater', 'age': 20}, {'name': 'Homer', 'age': 39}]
```

## 八、如何移除列表中的元素
删除列表中的元素有三种方式，remove 移除某个元素，而且只能移除第一次出现的元素
```python
>>> a = [0, 2, 2, 3]
>>> a.remove(2)
>>> a
[0, 2, 3]

# 如果要移除的元素不在列表中，则抛出 ValueError 异常
>>> a.remove(7)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: list.remove(x): x not in list·
```
del 根据指定的位置移除某元素
```python
>>> a = [3, 2, 2, 1]
# 移除第一个元素
>>> del a[1]
[3, 2, 1]

# 当超出列表的下表索引时，抛出IndexError的异常
>>> del a[7]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IndexError: list assignment index out of range
```
pop 与del 类似，但是 pop 方法可以返回移除的元素
```python
>>> a = [4, 3, 5]
>>> a.pop(1)
3
>>> a
[4, 5]

# 同样，当超出列表的下表索引时，抛出IndexError的异常
>>> a.pop(7)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IndexError: pop index out of range
```

## 九、如何连接两个列表
```python
listone = [1, 2, 3]
listtwo = [4, 5, 6]
mergedlist = listone + listtwo
print(mergelist)
>>>
[1, 2, 3, 4, 5, 6]
```
列表实现了 + 的运算符重载，使得 + 不仅支持数值相加，还支持两个列表相加，只要你实现了对象的 __add__操作，任何对象都可以实现 + 操作，例如：
```python
class User(object):
    def __init__(self, age):
        self.age = age

    def __repr__(self):
        return 'User(%d)' % self.age

    def __add__(self, other):
        age = self.age + other.age
        return User(age)


user_a = User(10)
user_b = User(20)

c = user_a + user_b

print(c)

>>>
User(30)
```

## 十、如何随机获取列表中的某个元素
```python
import random
items = [8, 23, 45, 12, 78]

>>> random.choice(items)
78
>>> random.choice(items)
45
>>> random.choice(items)
12
```

## 十一、两个列表获取交集、并集、差集的方法

* 获取两个 list 的交集
  ```
  result = list(set(a).intersection(set(b)))
  ```

* 获取两个 list 的并集

  ```
  result = list(set(a).union(set(b)))
  ```

* 获取两个 list 的差集

  ```
  result = list(set(b).difference(set(a)))
  # 只在 b 中有，而 a 中没有。
  ```
