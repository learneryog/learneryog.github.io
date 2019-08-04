---
title: hashCode和equals方法
date: 2019-05-21 16:15:37
cover: /img/Random-img/21.jpg
categories: 
     - 技术文章
tags:
     - java
     - 基础知识点
---

hashCode和equals方法是Object类中的两个常用方法。其定义如下：
```java
// hashCode()方法默认是native方法：
public native int hashCode();

// equals(obj)默认比较的是内存地址：
public boolean equals(Object obj) {
    return (this == obj);
}
```

> hashCode()方法有三个关注点：
> - 关注点1：主要是这个hashCode方法对哪些类是有用的，并不是任何情况下都要使用这个方法，（不使用时根本就没有必要覆写此方法），而是当涉及到像HashMap、HashSet(他们的内部实现中使用到了hashCode方法)等与hash有关的一些类时，才会使用到hashCode方法。
 >- 关注点2：推荐按照这样的原则来设计，即 当equals(object)相同时，hashCode()的返回值也要尽量相同，当equals(object)不相同时，hashCode()的返回没有特别的要求，但是也要尽量不相同以获取好的性能。
 > - 关注点 3：默认的hashCode实现一般是内存地址对应的数字，所以不同的对象，hashCode()的返回值是不一样的。

在这种缺省实施情况下，只有它们引用真正同一个对象时这两个引用才是相等的。同样，Object提供的hashCode()的缺省实施通过将对象的内存地址对映于一个整数值来生成。

### 源码分析

HashMap内部是由Entry<K,V>类型的数组table来存储数据，其初始状态如下：

``` java
static final Entry<?,?>[] EMPTY_TABLE = {};
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
```
```java
static class Entry<K,V> implements Map.Entry<K,V>{
    final K key;
	V value;
	Entry<K,V> next;
	// key的hashCode方法的返回值经过hash运算得到的值
	int hash;
	
	/**
	* Create new entry.
	*/
	Entry(int h,K k,V ,v,Entry<K,V> n){
	    value = v;
		next = n;
		key = k;
		hash = h;
	}
}
```

HashMap的存储结构如下图所示：
![hashmap结构](/img/post-img/19-5-21-2.png)

图中的每一个方格就表示一个Entry<K,V>对象，其中的横向则构成一个Entry<K,V>[] table数组，而竖向则是由Entry<K,V>的next属性形成的链表。

HashMap在添加元素(put)时的第一步就是计算该元素的key的hash值，用到如下方法：
![hash方法](/img/post-img/19-5-21-3.png)

HashMap对于key的重复性判断是基于两个内容的判断，一个就是hash值是否一样（会演变成key的hashCode是否一样），另一个就是equals方法是否一样（引用一样则肯定一样）。`e.hash == hash && ((k = e.key) == key || key.equals(k))`。

hashCode重写的原则：**当equals方法返回true，则两个对象的hashCode必须一样**。

equals()方法在get()方法中的使用：
![get方法](/img/post-img/19-5-21-4.png)
![getNode方法](/img/post-img/19-5-21-5.png)

### 为什么要同时覆写HashCode()和equals() ?

- 由于作为key的对象将通过计算其hashCode来确定与之对应的value的位置，因此任何作为key的对象都必须实现 hashCode和equals方法。
- hashCode和equals方法继承自根类Object，如果你用自定义的类当作key的话，要相当小心，按照散列函数的定义，如果两个对象相同，即`obj1.equals(obj2)=true`，则它们的hashCode必须相同，但如果两个对象不同，则它们的 hashCode不一定不同，如果两个不同对象的hashCode相同，这种现象称为冲突，冲突会导致操作哈希表的时间开销增大，hashCode()方法目的纯粹用于提高效率，所以尽量定义好的 hashCode()方法，能加快哈希表的操作。

> 如果相同的对象有不同的hashCode，对哈希表的操作会出现意想不到的结果（期待的get方法返回null），要避免这种问题，只需要牢记一条：**要同时覆写（重载）equals方法和hashCode方法，而不要只写其中一个。**