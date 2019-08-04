---
title: ArrayList与LinkList对比
date: 2019-05-04 18:17:32
cover: /img/Random-img/97.jpg
top: true
categories: 
    - 技术文章
    - 集合
tags:
    - java
    - 集合
    - ArrayList
    - LinkedList
---

> 本文简要总结一下java中ArrayList与LinkedList的区别，这在面试中也是常常会问到的一个知识点。

先来看一下ArrayList和LinkedList的关系是怎样的：

![ArrayList与LinkedList的关系图](/img/post-img/List.png)

从继承体系可以看到，ArrayList与LinkedList都是Collection接口下List接口的实现类。可谓是一对双胞胎。

但由于底层数据结构的不同导致ArrayList与LinkedList有本质上的区别。

#### ArrayList与LinkedList的区别

* ArrayList:  
&nbsp;&nbsp;&nbsp;&nbsp;ArrayList是基于**动态数组**的数据结构。
&nbsp;&nbsp;&nbsp;&nbsp;因为是数组，所以ArrayList在初始化的时候，有**初始大小10**，插入新元素的时候，会判断是否需要扩容，扩容的步长是**0.5倍原容量**，扩容方式是利用**数组的复制**，因此有一定的开销；
&nbsp;&nbsp;&nbsp;&nbsp;另外，ArrayList在进行元素插入的时候，需要**移动插入位置之后的所有元素**，位置越靠前，需要位移的元素越多，开销越大，相反，插入位置越靠后的话，开销就越小了，如果在最后面进行插入，那就不需要进行位移；

------

* LinkedList:
&nbsp;&nbsp;&nbsp;&nbsp;内部使用基于**链表**的数据结构实现存储，LinkedList有一个内部类作为存放元素的单元，里面有三个属性，用来存放元素本身以及前后2个单元的引用，另外LinkedList内部还有一个header属性，用来标识起始位置，LinkedList的第一个单元和最后一个单元都会指向header，因此形成了一个**双向**的链表结构。
&nbsp;&nbsp;&nbsp;&nbsp;LinkedList是采用双向链表实现的。所以它也具有链表的特点，每一个元素（结点）的地址不连续，通过引用找到当前结点的上一个结点和下一个结点，即插入和删除效率较高，只需要常数时间，而get和set则较为低效。

> LinkedList的方法和使用和ArrayList大致相同，由于LinkedList是链表实现的，所以额外提供了在头部和尾部添加/删除元素的方法，也没有ArrayList扩容的问题了。另外，ArrayList和LinkedList都可以实现栈、队列等数据结构，但LinkedList本身实现了队列的接口，所以更推荐用LinkedList来实现队列和栈。


##### 总而言之，ArrayList和LinkedList的区别有以下几点：

- &nbsp;&nbsp;&nbsp;ArrayList是实现了基于动态数组的数据结构，而LinkedList是基于链表的数据结构；
- &nbsp;&nbsp;&nbsp;对于随机访问元素，Array获取数据的时间复杂度是O(1),但是要删除数据却是开销很大的，因为这需要重排数组中的所有数据。ArrayList想要**get(int index)** 元素时，直接返回index位置上的元素，而LinkedList需要通过for循环进行查找，虽然LinkedList已经在查找方法上做了优化，比如**index < size / 2**，则从左边开始查找，反之从右边开始查找，但是还是比ArrayList要慢。
- &nbsp;&nbsp;&nbsp;对于添加和删除操作add和remove，LinkedList是更快的。因为LinkedList不像ArrayList一样，不需要改变数组的大小，也不需要在数组装满的时候要将所有的数据重新装入一个新的数组，这是ArrayList最坏的一种情况，时间复杂度是O(n)，而LinkedList中插入或删除的时间复杂度仅为O(1)。
- &nbsp;&nbsp;&nbsp;ArrayList在插入数据时还需要更新索引（除了插入数组的尾部）。 ArrayList想要在指定位置插入或删除元素时，主要耗时的是**System.arraycopy**动作，会移动index后面所有的元素；LinkedList主耗时的是要先通过for循环找到index，然后直接插入或删除。这就导致了两者并非一定谁快谁慢。


> 适用场景

很多场景下都是ArrayList更受欢迎。但有些情况下LinkedList更为合适，比如：

1. 你的应用不会随机访问数据。因为如果你需要LinkedList中的第n个元素的时候，你需要从第一个元素顺序数到第n个数据，然后读取数据。

2. 你的应用有更多的插入和删除元素操作，更少的读取数据。因为插入和删除元素不涉及重排数据，所以它要比ArrayList要快。

 

以上就是关于ArrayList和LinkedList的差别。你需要一个不同步的基于索引的数据访问时，请尽量使用ArrayList。ArrayList很快，也很容易使用。但是要记得要给定一个合适的初始大小，尽可能的减少更改数组的大小。