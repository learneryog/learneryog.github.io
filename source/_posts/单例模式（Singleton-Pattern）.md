---
title: 单例模式（Singleton Pattern）
date: 2019-05-21 16:57:07
cover: /img/Random-img/52.jpg
categories:
- 设计模式
tags:
- 设计模式
---

单例模式是最简单的设计模式之一，这种设计模式是一种创建型的模式，提供了创建对象的最佳方式。

单例模式顾名思义就是一个类只允许创建一个实例，因此它只涉及到一个单一的类，并且这个类要完成自己创建自己的实例的工作，并保证能且只能创建一个实例。这个类还需要提供一个访问这个实例的方法。

接下来我们分析一下单例模式的多种实现方式。（以下代码均为Java实现，若读者有兴趣可自行用其他语言实现）

## 一、懒汉模式（延迟加载）

```java
public class Singleton{
    //定义一个未初始化的私有静态对象，外部不可访问
    private static Singleton singleton;
 
    //构造私有，保证外部不可创建实例
    private Singleton() {
    }
 
    //提供公有方法获取实例
    public synchronized static Single newInstance() {
        if (singleton== null) {
		    //如果singleton为空就为其创建实例
            singleton= new Singleton();
        }
		//将创建好的实例返回
        return singleton;
    }
}
```
> 懒汉形式是标准的单例实现形式，它的延迟加载体现在newInstance方法里的判断，方法加了synchronized关键字可以避免多线程问题，但会影响程序性能。

## 二、饿汉模式（贪婪加载）

``` java
public class Singleton {
    //直接创建私有静态实例
    private static Singleton singleton= new Singleton();
 
    //构造方法私有保证外部不可创建实例
    private singleton() {
    }
 
    public static Singleton newInstance() {
	    //将创建好的实例返回
        return singleton;
    }
}
```
> 在单例对象声明的时候就直接初始化对象，可以避免多线程问题，但是如果对象初始化比较复杂，会导致程序初始化缓慢。

## 三、双重锁检查

``` java
public class Singleton {
    //静态对象用volatile关键字修饰
    private volatile static Singleton singleton;
 
    private Singleton() {
    }
 
    public static Singleton newInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

> 这个是懒汉形式的加强版，将synchronized关键字移到了newInstance方法里面，同时将singleton对象加上volatile关键字，这种方式既可以避免多线程问题，又不会降低程序的性能。但volatile关键字也有一些性能问题，不建议大量使用。

## 四、Lazy initialization holder class

``` java
public class Singleton {
    private static class SingletonHolder {
        private static Singleton singleton = new Singleton();
    }
 
    private Singleton() {
    }
 
    public static Singleton newInstance() {
        return SingletonHolder.singleton;
    }
}
```

> 这种方法创建了一个内部静态类，通过内部类的机制使得单例对象可以延迟加载，同时内部类相当于是外部类的静态部分，所以可以通过jvm来保证其线程安全。这种形式比较推荐。


还有一些其他的方法有待学习探索，正所谓学无止境，与君共勉。

