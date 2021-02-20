---
title: SharedPreference & DataStore
tags:
---

# SharedPreference 共享偏好配置

关键词: 

* 轻量级
* 全量更新
* 引发ANR的原因
* 不支持跨进程

## 关于"轻量级"理解

存储简单的键值对，若是存储复杂的结构，比如Json文件，可以

## 创建的SP实例会一直保存在内存中

> file ContextImpl.java

context.getSharePreference(name,mode) 

根据当前Context，使用packageInfo，将`name` >> `file`

context.getSharedPreference(file,mode)

维持`cache` ,  Map<File, SharePreferenceImpl> ，我们创建SP实例会缓存到内存中

实际上缓存结构是 package > file > SP

## 异步创建SP,获取数据需要加载SP完成

构建SharedPreferenceImpl，会起一个线程来从磁盘加载数据，这样构建Sp就可以快速返回了，不会堵塞调用该方法的线程。   

```
SharedPreferenceImpl(File file, int mode){
    ...
    // 起一个线程来读取磁盘数据，这样创建Sp就可以快速返回了
    startLoadFromDisk();
}
```

但是呢？ 即使将加载数据的任务委托到了另一个线程加载，加载还是需要时间的，调用`getSp()`之后，并不能立即之后使用来取值设值。  
故，把获取SP，使用SP两个动作放在不同时间点就有必要了。
