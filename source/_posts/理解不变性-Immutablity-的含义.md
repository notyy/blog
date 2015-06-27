title: 理解不变性(Immutablity)的含义
date: 2015-06-26 17:26:20
tags:
- scala
- 函数式编程
categories:
- 技术
---
最近有位《深入理解scala》的读者对书中关于不变性的问题有些不理解。主要是关于这两段代码：
![不变性](http://7u2h31.com1.z0.glb.clouddn.com/immutable.jpg)
> 第二章2.3选择不变性中的例子应该是错的，为了说明不变性的优点将HashMap替换成ImmutableHashMap，但是变量本身还是可变的，在多线程环境下可能导致变量读取到的是旧值

<!-- more -->

其实不可变解决的不是“旧值”的问题，而是解决读到“错误的值”的问题。 这一点在作者这个例子里面体现的不是很明显，是因为在lookup里面看上去只是执行了一个简单的原子操作。 假设`currentIndex.get(k)`的实现是一段复杂的代码，其中会判断当前集合的值，做一些什么操作，

在这种情况下，如果lookup不加synchronized,那么有可能在执行的过程中currentIndex同时被insert修改，就可能导致lookup发生错误。
而不可变版本里的lookup不需要sychronized，是因为lookup返回了一个当时的currentIndex的不可变副本，在lookup执行过程中insert里对currentIndex的修改不会对不可变副本造成任何影响。

因此在大量读、少量写的情况下不可变版本的性能就会高得多。
