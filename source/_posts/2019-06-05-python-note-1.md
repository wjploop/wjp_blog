---
title: python-note_1
date: 2019-06-05 09:04:13
tags: python
categories: python
---

### 面向对象

构造方法：
```python
class Student(object):  //继承自object 

    def __init__(self,name,score):  //注意类中方法，必须传递一个形参self
        self.name = name    
        self.score = score        

```

访问限制：

```python
"" 与java对应的public, default, protected, private

class Student():
    _name  //约定，仍可从外部直接访问，但依照约定，提示使用者应该将其视为私有变量
    __name  //解释器将其编译成 _Student__name, 使用者不能直接使用__name来访问了，可以说，将门槛提更高来告知限制
    
```

继承多态：
```python
class Animal():
    
    def run(self):
        print('go go go ...`)
        

class Duck(Animal):
    
    def run(self):
        print('duck run slowly')
     
```
Duck 可以拥有父类的成员，概念和java一样   
只不过的，作为动态类型的python，在方法调用时，假设要传入一个Animal类型来使用其run方法。  
java要求其参数必须是Animal的类型实例，而python不做要求，只要其对象实例拥有run方法既可。

判断对象是否是某个类的实例，可以使用 isInstance(object,Class) 来判断object是否是Class的类型


获取对象信息：

使用type(), 判断对象类型。  

```python
>>> type(123)
<class 'int'>

>>> type('123')
<class `str'>

class Animal():
    pass 

a = Animal()

>>> type(a) 

<class '__main__.Animal'>   //即一个main线上的Animal类型

额外的，还可以判断一个对象是否是函数类型。

import types

def fn():
    pass

type(fn) == types.FunctionType  //return True
```

使用isInstance(obj,cls) 判断obj是否是cls类型


使用 dir() 列出一个对象的所有属性和方法，返回一个字符串的list
```python
>>> dir('123')
['__add__', '__class__', '__contains__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__',
 '__getattribute__', '__getitem__', '__getnewargs__', '__gt__', '__hash__', '__init__', '__init_subclass__',
  '__iter__', '__le__', '__len__', '__lt__', '__mod__', '__mul__', '__ne__', '__new__', '__reduce__', 
  '__reduce_ex__', '__repr__', '__rmod__', '__rmul__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__',
   'capitalize', 'casefold', 'center', 'count', 'encode', 'endswith', 'expandtabs', 'find', 'format', 'format_map', 
   'index', 'isalnum', 'isalpha', 'isascii', 'isdecimal', 'isdigit', 'isidentifier', 'islower', 'isnumeric', 
   'isprintable', 'isspace', 'istitle', 'isupper', 'join', 'ljust', 'lower', 'lstrip', 'maketrans', 'partition', 
   'replace', 'rfind', 'rindex', 'rjust', 'rpartition', 'rsplit', 'rstrip', 'split', 'splitlines', 'startswith', 
   'strip', 'swapcase', 'title', 'translate', 'upper', 'zfill']

```
有很多内置方法，比如__add_ ，字符串拼接时，就是调用了这个方法，使用上，直接使用 '+' ，重载操作符。  
由此，可以联想到，在我们自定义的类中，也可以重载这些方法。

为了方便，判断一个对象是否某方法或属性，我们可以用 getattr(), setattr(), hasattr()

比如： 
```python

def readImage(fp):      //判断对象fp是否read方法，若是有读取数据返回，否则返回None
    if(hasattr(fp, 'read'):
        return readData(fp)
    return None
```


类属性，实例属性

感觉这个有点神坑啊

```python

class Student():        //定义了一个Student类型，并赋予了一个类属性 name
    name = 'student'
    
s = Student();  //创建一个实例

>>> s.name      //通过实例，可以访问到类属性
student

>>> s.name = 'wolf' //定义一个对象属性，而且与类属性重名
>>> s.name  //访问该属性，实例属性优先于类属性。（实例属性屏蔽类属性）
wolf

```
实例属性，尽量不要定义跟类属性一样了
