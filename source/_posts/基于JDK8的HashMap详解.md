---
title: 基于JDK8的HashMap详解
date: 2019-05-22 15:55:48
cover: /img/Random-img/25.jpg
categories:
- java
tags:
- HashMap
- 基础知识
top: true
---

### 摘要

HashMap是程序员使用频率较高的一种用于映射（键值对）处理的数据类型，随着JDK（Java Development Kit）版本的更新，HashMap也在不断被优化。其中JDK1.8在HashMap底层引入了红黑树的数据结构并对其扩容进行了优化等。本文将结合JDK1.7与JDK1.8对HashMap进行分析，浅析HashMap在JDK1.8中的改进。

## 一、HashMap的继承体系和特点

#### 1. HashMap的继承体系

java.util.Map是java为数据结构中的映射定义的接口，实现此接口的有四个常用类，分别是HashMap、HashTable、LinkedHashMap和TreeMap，他们的继承关系如下：
![HashMap继承体系](/img/post-img/19-5-22-11.png)

#### 2. HashMap及相关类的特点：

 - **HashMap**：它根据键（key）的HashCode值存储数据，大多数情况下可以直接定位到它的值，因此具有很快的访问速度，但遍历顺序却是不确定的。HashMap最多允许一个元素的键为null，允许多个元素的值为null。HashMap是非线程安全的，即可以有多个线程同时访问HashMap，可能导致数据的前后访问不一致。可以用Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者直接使用引入了分段锁的ConcurrentHashMap。
 - **HashTable**：HashTable是遗留类，很多映射的常用功能与HashMap类似，不同的是HashTable继承自Dictionary类，并且是线程安全的，任一时刻只允许一个线程访问HashTable，并发性不如ConcurrentHashMap。新代码中不建议使用HashTable，因为在不需要线程安全的场合可以用HashMap代替，需要线程安全的场合可以用ConcurrentHashMap替代。
 - **LinkedHashMap**：LinkedHashMap是HashMap的一个子类，它保存了元素的插入顺序，在用Iterator遍历LinkedHashMap时，先得到的元素肯定是先插入的，也可以在构造时带参数，按照访问次序排序。
 - **TreeMap**：TreeMap实现了SortedMap接口，能够将它的元素根据键排序，默认根据键的升序排序，也可以指定排序的比较器。当用Iterator遍历TreeMap时，得到的结果是排过序的，如果需要排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap时传入自定义的Comparator，否则会在运行时抛出java.lang.ClassCastException类型的异常。
> 通过以上比较，我们了解到HashMap是java中Map家族的普通一员，因为它是满足大多数场景的使用条件，所以HashMap是使用最频繁的一个。下面我们将结合HashMap源码，从存储结构、常用方法、扩容以及安全性等方面深入理解HashMap的工作原理。

## 二、HashMap的内部实现

#### 1. HashMap是什么——存储结构

从结构来讲，HashMap是数组+链表+红黑树（JDK1.8新增红黑树部分）实现的，数组table存放类型为Node<K,V>的结点，该结点向外引申出单向链表，当链表长度大于8时，转换为红黑树。如下图所示：
![存储结构](/img/post-img/19-5-22-12.png)

Node [] table数组被称为哈希桶数组。那么Node到底是什么？JDK1.8中对Node<K,V>的定义如下：
```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;//用来定为数组索引位置
        final K key;
        V value;
        Node<K,V> next;//链表的下一个Node
        Node(int hash, K key, V value, Node<K,V> next) { ··· }
        public final K getKey() { ··· }
        public final V getValue() { ··· }
        public final String toString() { ··· }
        public final int hashCode() { ··· }
        public final V setValue(V newValue) { ··· }
        public final boolean equals(Object o) { ··· }
    }
```
Node是HashMap的一个内部类，实现了Map.Entry接口，本质上就是一个映射（键值对），上图中的每个蓝色椭圆就表示一个Node的对象。

HashMap使用哈希表存储的。为解决哈希冲突，哈希表可以采用开放地址法和链地址法等方法，java中的HashMap采用的是链地址法。所谓链地址法，简单来说就是数组加链表的组合，每个数组元素上都有一个链表结构，将数据被hash后得到的数字作为数组下标，将该元素添加到对应下标下的链表中。
举例说明，假如程序在执行以下代码：
`map.put("通信","张三");`
系统会“通信”这个key的hashCode方法（该方法适用于任何java对象）得到其HashCode值，再通过Hash算法的**后两步运算**（高位运算和取模运算）来定位该键值对的存储位置。有时两个key会定位到相同的位置，表示发生了**哈希碰撞**。hash算法计算结果越分散均匀，发生哈希碰撞的概率就越小，HashMap的存储效率就越高。这就会出现以下问题，如果哈希桶数组足够大，即使差的hash算法分布也比较均匀，相反如果哈希桶数组很小，即使好的hash算法也很难避免出现较多碰撞。因此就需要在时间成本和空间成本之间有所权衡，即根据实际情况设定合适大小的哈希桶数组，并设计好的hash算法来减少哈希碰撞。那么通过什么方式来控制HashMap使哈希桶数组占用空间又少有能降低hash碰撞的几率呢？答案就是**好的hash算法加扩容机制**。

在理解Hash和扩容流程之前，我们得先了解下HashMap的几个字段。从HashMap的默认构造函数源码可知，构造函数就是对以下几个字段进行初始化，源码如下：

```java
int threshold; //所能容纳的key-value对极限
final float loadFactor；//负载因子
int modCount；
int size；
```
首先，Node[] table的初始长度length（默认值是**16**），loadFactor为**负载因子**（默认是**0.75**），threshold是HashMap所能容纳的最大数据量的Node个数。`threshold=length*loadFactor`。也就是说，在数组定义好长度后，负载因子越大，所能容纳的键值对的个数越多。

结合负载因子的定义公式可知，threshold就是在此loadFactor和length对应下允许的最大元素数目，超过这个数目就重新resize，扩容后的HashMap容量是之前的两倍。默认的负载因子是0.75，是对空间和时间效率的一个平衡选择，建议大家不要修改，除非在时间和空间比较特殊的情况下，如果内存空间很多而又对时间效率要求很高，可以降低负载因子loadFactor的值，相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子loadFactor的值，这个值可以大于1。

size这个字段很好理解，就是HashMap中**实际存在的键值对数量**。注意和table的长度length、容纳最大键值对数量threshold的区别。而modCount字段主要用来记录HashMap内部结构发生变化的次数，主要用于迭代的快速失败。强调，内部结构发生变化指的是结构发生变化，例如put新键值对，但是某个key对应的value值被覆盖不属于结构变化。

在HashMap中，哈希桶数组table的长度length大小必须为2的n次方，这是一种非常规的设计，常规的设计是把桶的大小设计为素数。相对来说素数导致冲突的概率要小与合数，Hashtable初始化桶大小为11，就是桶大小设计为素数的应用。HashMap采用这种非常规设计，主要是为了在**取模和扩容时做优化**，同时为了减少冲突，HashMap定位哈希桶索引位置时，也加入了**高位参与运算**的过程。

这里存在一个问题，即使负载因子和hash算法设计的再合理，也免不了会出现拉链过长的情况，一旦出现拉链过长，则会严重影响HashMap的性能。于是，在JDK1.8版本中，对数据结构做了进一步的优化，引入了红黑树。而当**链表长度大于8时**，链表就转换为红黑树，利用红黑树快速增删改查的特点提高HashMap的性能，其中会用到红黑树的插入、删除、查找等算法。本文不再对红黑树展开讨论，想了解很多红黑树数据结构的工作原理可以参考另一篇博文深入理解红黑树。

#### 2. HashMap能干什么——功能实现

HashMap的内部功能实现很多，本文主要从根据key获取哈希桶数组索引位置、put方法的详细执行过程、扩容过程这三个具有代表性的点深入展开。

- **根据key确定哈希桶数组索引位置**

不管是增加、删除还是查找键值对，定位到哈希桶数组的位置都是很关键的第一步。前面说到HashMap的结构是数组加链表的组合，所以我们当然希望HashMap里的元素分布尽量均匀些。HashMap定位数组索引位置，直接决定了HashMap的离散性能。我们先看一下源码对hash算法的实现：
![hash方法](/img/post-img/19-5-22-13.png)

`h=key.hashCode()` 为算法第一步，取hashCode值；

`h^(h>>>16)` 为算法第二步，高位参与运算；

我们注意到在JDK1.7的源码中有这样一个方法：

``` java
static int indexFor(int h,int length){
    return h&(length-1);
}
```
其中的`h&(length-1)`为hash算法的第三步，取模运算，这个方法在JDK1.8中没有出现，但是实现原理是相同的。

`h&(table.length-1)`这个方法非常巧妙，因为一般我们能想到的计算索引方法就是将hash值对table数组长度进行取模运算，但模运算的消耗还是比较大的，而HashMap底层的数组长度总是2的n次方，这是HashMap在速度上的优化。当length总是2的n次方的时候，h&(length-1)等价于取模运算，但&比%具有更高的效率。

JDK1.8中，优化了高位运算的算法，通过将hash的高16位与低16位进行异或运算，主要是从速度、功效、质量上考虑的，这么做可以在table的length很小的时候也能考虑到高低位都参与到运算中，同时不会有太大的开销。

具体计算过程如下：
![高位运算过程](/img/post-img/19-5-22-14.png)

- **HashMap的put过程**

可以通过下图来辅助理解HashMap的put过程：
![put流程](/img/post-img/19-5-22-15.png)

JDK1.8中HashMap的put方法源码如下：
![put源码](/img/post-img/19-5-22-16.png)

此方法将对key进行hash运算，得到hash值。

``` java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //第一步，判断table是否为空，如果为空，创建
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //第二步，计算index，并对null做处理
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //第三步，如果key结点已经存在，直接覆盖value
            if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //第四步，判断该链是否为红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //该链为链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    //如果链表为空，直接插入
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果链表长度大于8，转换为红黑树进行操作
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //key已经存在，直接覆盖value
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //第五步，超过最大容量，扩容 
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

- **HashMap的扩容机制**

扩容（resize）就是重新计算容量，向HashMap里不停的添加元素，而当HashMap对象内部的数组无法装载更多的元素时，对象就需要扩大数组长度，以便能装入更多的元素。当然java里的数组是无法自动扩容的，方法是使用一个新的数组代替已有的容量小的数组，就像我们用一个小桶装水，如果想装更多的水，就得换更大的水桶。

JDK1.8中对扩容引入了红黑树，较为复杂，为了便于理解我们仍然使用JDK1.7的源码来分析，本质上区别不大，但易于理解：

``` java
void resize(int newCapacity){//传入新的容量
    Entry[] oldTable = table;//引用扩容前的Entry数组
    int oldCapacity = oldTable.length;
    if(oldCapacity == MAXIMUM_CAPACITY){//扩容前的数组大小如果已经达到最大（2^30）了
        threshold = Integer.MAX_VALUE;//修改阈值为int的最大值（2^31-1），这样以后就不会扩容了
        return;
}
 
Entry[] newTable = new Entry[newCapacity];//初始化一个新的Entry数组
transfer(newTable);//将数据转移到新的Entry数组里
table=newTable;//HashMap的table属性引用新的Entry数组
thteshold=(int)(newCapacity*loadFactor);//修改阈值
}
 
void transfer(Entry[] newtable){
    Entry[] src = table;//src引用了旧的Entry数组
    int newCapacity = newTable.length;
    for(int j=0;j<src.length;j++){//遍历旧的Entry数组
        Entry<K,V> e = src[j];//取得旧Entry数组的每个元素
        if(e!=null){
            src[j]=null;//释放旧Entry数组的对象引用（for循环后，旧的Entry数组不再引用任何对象）
            do{
                Entry<K,V> next = e.next;
                int i = indexFor(e.hash,newCapacity);//重新计算每个元素在数组中的位置
                e.next = newTable[i];//标记[1]
                newTable[i]=e;//将元素放在数组上
                e=next;//访问下一个Entry链上的元素
            }while(e!=null);
        }
    }
}
```

newTable[i]的引用赋给了e.next，也就是使用了单链表的头插方式，同一位置上的新元素总会被放到链表的头部位置；这样先放在一个索引上的元素终会被放到Entry链的尾部（如果发生了Hash冲突的话）。这一点和JDK1.8有区别，下文详解。在旧数组中同一条Entry链上的元素，通过重新计算索引位置后，有可能被放到了新数组的不同位置上。

下面举个例子说明一下扩容过程。假设我们的hash算法就是简单的用key mod一下数组的长度。其中的哈希桶数组table的size=2，所以key=3、7、5，put顺序依次为5、7、3.在mod 2以后都冲突在table[1]这里了。假设负载因子loadFactor=1，即当键值对的实际大小size大于table的实际大小时扩容。接下来的三个步骤是哈希桶数组resize成4，然后所有的Node重新rehash的过程。
![put举例](/img/post-img/19-5-22-17.png)

下面我们讲解下JDK1.8做了哪些优化。经过观察可以发现，我们使用的是2次幂的扩展（长多扩为原来的2倍），所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置上，因此我们在扩充HashMap的时候，不需要像JDK1.7那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引不变，是1的话索引变成“原索引+oldCapacity”。可以借助下图来理解：
![扩容](/img/post-img/19-5-22-18.png)

这个设计非常巧妙，既省去了重新计算hash值的时间，同时，由于新增的1位是0还是1可以认为是随机的，因此resize的过程均匀的吧之前的冲突的结点分散到新的bucket了，这一块就是JDK1.8新增的优化点。有一点注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。有兴趣的读者可以研究下JDK1.8的resize源码，如下：

``` java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //超过最大值就不再扩容了
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //没超过最大值，就扩为原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //设置新的resize上限
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            //把每个bucket都移到新的buckets中
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

## 三、小结

- [x] 扩容是一个特别耗性能的操作，所以当程序员在使用HashMap的时候，估算map的大小，初始化的时候给一个大致的数值，避免map进行频繁的扩容。
- [x] 负载因子是可以修改的，也可以大于1，但是建议不要轻易修改，除非情况非常特殊。
- [x] HashMap是线程不安全的，不要在并发的环境中同时操作HashMap，建议使用ConcurrentHashMap。
- [x] JDK1.8引入红黑树大程度优化了HashMap的性能

