---
title: 浅谈ArrayList
date: 2019-05-22 14:46:58
cover: /img/Random-img/22.jpg
categories:
- java
tags:
- 集合
- ArrayList
- 基础知识
---

## 一、ArrayList的继承体系和特点

ArrayList总体继承体系图如下：
![ArrayList继承体系](/img/post-img/19-5-22-1.png)

ArrayList的特点主要有以下几点：

> - ArrayList在内存中分配连续的存储空间，可理解为长度可变的数组。
> -  ArrayList存储元素可以重复，存储顺序和添加顺序一致。
> -  遍历元素和随机访问元素的效率较高，因为和数组一样存在索引。 
> -  添加、删除元素时需移动大量元素，按照内容查询时效率低。

## 二、ArrayList的用法

```java
List<String> list = new ArrayList<String>();//使用多态创建ArrayList，泛型指定该ArrayList只能放String类型的元素
    for(int i=0;i<5;i++)
        list.add("a"+i);//架循环往ArrayList里添加元素
    System.out.println(list);//输出整个ArrayList
    list.add(3, "a0");//在指定索引处添加指定元素
    System.out.println(list);
    list.remove(2);//删除指定索引处的元素
    System.out.println(list);
    System.out.println(list.size());//获取ArrayList的长度并输出
    System.out.println(List.get(3));//拿到指定索引处的元素并输出
    list.removeAll(list);//删除所有元素    
    System.out.println(list);
 
输出结果：
[a0, a1, a2, a3, a4]
[a0, a1, a2, a0, a3, a4]
[a0, a1, a0, a3, a4]
5  a3
[]
```
ArrayList的一些其他常用方法:

`toArray(T [] a)`：把ArrayList转换成数组放到指定数组a里

`contains(Object o)`：判断ArrayList中是否包含指定元素，返回Boolean类型

`isEmpty()`：判断ArrayList按是否为空，返回Boolean类型

## 三、ArrayList的自动扩容机制

java为了实现自动扩容，引入了Capacity和size的概念，用来区别数组的length。为了保证用户增加新的对象，java设置了最小容量（minCapacity），通常情况下，它大于列表对象的数目，所以Capactiy虽然就是底层数组的长度（length），但是对于最终用户来讲，它是没有意义的。而存储着列表对象数量的size才是最终用户所需要的。为了防止用户错误修改，这一属性被设置为private的，不过可以通过size()方法获取。
![size属性](/img/post-img/19-5-22-2.png)

ArrayList的初始容量默认为10：
![初始容量](/img/post-img/19-5-22-3.png)

ArrayList有两个构造方法：
![构造方法1](/img/post-img/19-5-22-4.png)
![构造方法2](/img/post-img/19-5-22-5.png)

这说明Capacity初始值（initialCapacity）可以由用户直接指定或由用户指定的Collection集合存储的对象数目确定，如果没有指定，系统默认为10。而size的被声明为int型变量，默认为0，当用户用指定Collection创建ArrayList时，size值等于initialCapacity。

我们知道ArrayList可以用add()方法添加元素，我们来看一下add()的实现：
![add方法](/img/post-img/19-5-22-6.png)

ensureCapacityInternal()方法的调用主要用来确认数组是否可以放下这一元素，修改elementData数组的指向。ensureCapacityInternal()方法的实现如下：
![ensureCapacityInternal()方法](/img/post-img/19-5-22-7.png)

首先看看数组是否为空，如果数组为空，就将DEFAULT_CAPACITY和minCapacity中较大的一个作为初始大小赋给minCapacity，DEFAULT_CAPACITY就是先前定义的10，minCapacity就是add方法中传入的size+1。

如果数组不为空，就执行ensureExplicitCapacity()方法，其实现如下：
![ensureExplicitCapacity()方法](/img/post-img/19-5-22-8.png)

modCount属性在ArrayList的父类AbstractList中定义，用于存储结构修改次数。

此方法比较minCapacity与elementData.length的大小。当第一次插入值时，由于minCapacity一定大于等于10，而elementData.length 是0，此时调用grow()方法，grow()方法正是ArrayList扩容的核心所在，其实现如下：
![grow()方法](/img/post-img/19-5-22-9.png)

这个方法首先计算出一个容量，大小为oldCapacity + (oldCapacity >> 1)。即elementData数组长度的**1.5倍**。再从minCapacity和这个容量中取较大值作为扩容后的新的数组大小。

新的容量小于数组的最大值MAX_ARRAY_SIZE，可能超过一次能申请的整块内存的大小上限，出现OutOfMemoryError。

如果新的容量大于数组的最大值MAX_ARRAY_SIZE，调用hugeCapacity()方法，其实现如下：
![hugeCapacity()方法](/img/post-img/19-5-22-10.png)

此方法会对minCapacity和MAX_ARRAY_SIZE进行比较，minCapacity 大的话，就将Integer.MAX_VALUE 作为新数组的大小，否则将MAX_ARRAY_SIZE作为数组的大小。最后，就把原来数组的数据复制到新的数组中。调用了Arrays的copyOf方法。内部是System的arraycopy方法，由于是native方法，所以效率较高。

> 通过以上源码我们不难看出，java自动增加ArrayList大小的思路是：**向ArrayList添加对象时，原对象数目加1，如果大于原底层数组长度，则以适当长度新建一个原数组的拷贝，并修改原数组，指向这个新建数组。原数组自动抛弃（java垃圾回收机制会自动回收）。size则在向数组添加对象，自增1。**

综上所述，ArrayList的扩容会产生一个新的数组，将原来数组的值复制到新的数组中。会消耗一定的资源。所以我们初始化ArrayList时，最好可以估算一个初始的大小。

