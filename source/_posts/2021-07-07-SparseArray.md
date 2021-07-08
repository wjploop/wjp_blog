---
title: SparseArray
date: 2021-07-07 16:18:29
tags: Android
categories:
---

给自己一个自认为熟悉的感念，表达出来能得到多少分？

是的，我认为我对 `SparseArray` 熟悉，可当我要表述这个数据结构时，我该怎么表达呢？

### 首先，描述使用范围，对比HashMap

`SparseArray` 适用于key-value结构的数据，相比于Java基础库提供的通用的 `HashMap`，
`SparseArray` 限制了key只能为int类型，可以节省存储空间，在数据范围小时存取效率优于 `HashMap`。

为何省空间？

其key直接使用了基本类型int，不同于HashMap存储时经过装箱拆箱使用Integer对象，而且，当确定value也是一种基本类型时，
可以使用对相应得SparseXXXArray,比如对于 <Integer,Integer>，可以使用 `SparseIntArray`，这样value也可以存储
在基本类型里面，而基本类型所占内存是远小于对象的。

`SparseArray`内部有两个数组，`mKeys`, `mValues`, key直接升序放置在 `mKeys`数组中，其插入过程：
```
key -> binarySearch -> index 
    put key in mKeys[index]
    put value in mValues[index]
```
将key转换成index是一次二分查找得过程，其要求key在数组中有序的。

而 HashMap其内部可以看作有一个 `tables`，其类型是一个Entry<Key,Value>，插入过程：

```
key -> hash -> index
     h = key.hashCode()
     hash = h ^ ( h >>> 16) // 将高16位的比特位与地位异或，高16位相当于取非了，从而把高位的信息分配到低位上，这样做的原因是，最后我们可能
     // 利用到的是地位的信息，（假设 n = table.length <= pow(2,16)，那么在转移成index时，我们仅仅利用到了hash的低16位）
     ensure hash < n ,(n = table.length)
     index = hash & (n - 1)   // 依赖n为2的k次方
     // 理想情况，table[index]没有元素，即无hash冲突
     // 冲突时，转为链表，其实存储时，tables存储的就是链表节点，甚至为树节点
     // 存在类存在这样的继承关系，Entry > Node > TreeNode
```

两者都存在从 key 到 index的过程，理论上，可以看到查找时间复杂度，前者为 O(logN)，后者为 O(1)，似乎后者更胜一筹。 但应了解到，这里说的时间
复杂度是简化分析的结果，忽略了复杂度前面的常量，前者常量可能是50，后者可能是1，当数据量N比较小时，实际上是前者更快一点的。这一点，做题时也有所
体会，有时理论复杂度低的解法表现更慢，原因是用例N低了。 

### 源码分析

#### 结构

```java
public class SparseArray<E> implements Cloneable {
    // 删除时用于标记某坑位，避免每次删除时都要挪动元素
    private static final Object DELETED = new Object();
    // 是否有可能清理DELETED垃圾
    private boolean mGarbage = false;

    // 关键两个数组
    private int[] mKeys;
    private Object[] mValues;
    private int mSize;

```

#### put

```java
    
    public void put(int key, E value) {
        // 寻找到这个key应该放在哪？
        // 这个二分查找，特别之处在于当找不到时，返回该插入下标的取非（不是相反数吧，取反+1才是，不纠结）
        // 取非必然为负数，因为首位0改为1了,这样用负数表示不能存在该key，而且还把该插入的下标变相地保存了下来
        // 是我孤陋寡闻了
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            // 存在已放置的 key 相同，直接替换（并没有HashMap的替换与否的策略）
            mValues[i] = value;
        } else {
            // 转为该插入下的下标
            i = ~i;
            
            // 发现居然那个坑是无用的，直接替换，免除数组插入的困扰
            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }

            // 坑位使用完了，且可能存在垃圾，尝试回收下
            if (mGarbage && mSize >= mKeys.length) {
                // 思路就是移除mValues数组中DELETED无效元素，有效元素往前移
                // JVM老年代垃圾清除算法思路（更追求省内存，碎片整体）
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }

            // 将key插入到mKeys的i位置上
            // mSize + 1 == mKeys.length 时，扩容 currentSize <= 4 ? 8 : currentSize * 2，创建新的数组复制原来的元素
            // 即使不需要扩容，也需要移动 i 位置之后元素后移一位
            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
    }
```

#### 查询,无需整理

通过key来查询，不需要整理

```java

    public E get(int key) {
        return get(key, null);
    }

    public E get(int key, E valueIfKeyNotFound) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        // 利用标记，我们确认某个key不存在，或已经被删除
        if (i < 0 || mValues[i] == DELETED) {
            return valueIfKeyNotFound;
        } else {
            return (E) mValues[i];
        }
    }
    
```


#### 查询，需整理

发现依赖了index的查询需要gc,当然还包括插入时也得gc
```
    public int size() {
        if (mGarbage) {
            gc();
        }
        return mSize;
    }
    
    public int keyAt(int index) {
        if (mGarbage) {
            gc();
        }
        return mKeys[index];
    }
```

### remove

可以欣赏的一点就是，频繁删除元素后，不会马上整理Values数组，而时延后到查询/插入时才会整理，整理就是清除无用的DELETED坑位

```java
    public void remove(int key) {
        delete(key);
    }

    public void delete(int key) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            if (mValues[i] != DELETED) {
                mValues[i] = DELETED;
                mGarbage = true;
            }
        }
    }
    public E removeReturnOld(int key) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
        if (i >= 0) {
            if (mValues[i] != DELETED) {
                final E old = (E) mValues[i];
                mValues[i] = DELETED;
                mGarbage = true;
                return old;
            }
        }
        return null;
    }

```

### 后记

另外，SparseXXXMap都要求Key必须为int，若是key不是int，可以用ArrayMap，大略看了下，相似度挺高的，大致是将key转为hash作为key来排序。


好了，发现自己表达起来确实很着急~
对比大佬的[博文](https://juejin.cn/post/6844903961963528199#heading-10)

别人家写的，虽然自己也能感受到的，但表达出来还是很困难。

引用大佬总结的三个关键字，双数组，二分查找，DELETED标记





