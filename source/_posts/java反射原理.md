---
title: java反射原理
date: 2019-05-25 11:26:22
cover: /img/Random-img/85.jpg
categories: 
- java
tags:
- java
- 反射
---

## 一、什么是反射？
&emsp;&emsp;简单来说，反射可以帮助我们在动态运行的时候，对于任意一个类，可以获取其所有的方法（包括public、protected、private和默认状态的），所有的变量（包括public、protected、private和默认状态的）。

## 二、反射机制的原理

### 1. 一个类正常被执行的流程

&emsp;&emsp;.java --> .class --> JVM运行期系统 --> 操作系统 --> 物理硬件
&emsp;&emsp;.首先在编译期，一个java源文件（.java文件）通过编译器（javac指令）编译后，成为.class字节码文件。
&emsp;&emsp;在运行期间，字节码文件以字节流的形式传输到类加载器，类加载器结合本地类库验证字节码的正确性，验证完成后将字节码文件交由java虚拟机来执行，JVM中的解释器和即时编译器对字节码文件进行处理，最终变为虚拟机可以识别的java运行系统，最后将文件放入操作系统，操作系统在物理硬件上运行这个类文件。
&emsp;&emsp;反射就是在运行期间不知道是哪一个类被编译了，但是在类加载器中包含这个类的所有信息，所以可以动态类加载器中的字节码文件，从而获取整个类的源信息。
![反射原理](/img/post-img/19-5-26-1.png)

### 2. 类加载机制

&emsp;&emsp;以 Person person = new Person(); 为例，在new对象时，JVM将.class文件加载到方法区，创建一个Class对象，方法区存放类的所有信息，例如类名、访问修饰符、方法、属性、构造函数、静态变量、静态块等，但不会初始化。然后在堆里开辟一块空间，将new出的对象放入此空间，在调用构造方法时，方法区中的所有非静态成员属性拷贝到堆内存中，并初始化为缺省值。并持有非静态方法的引用。然后将person对象放入栈空间中，这个对象指向堆内存的地址。
![类加载机制](/img/post-img/19-5-26-2.png)

## 二、反射的基本使用

反射常用API：类（Class）、属性（Field）、方法（Method）、构造器（Constructor）

- 三种获取类的Class对象的方式：**通过.class获取**，**通过classForName()方法获取**，**通过类的实例获取**
- 两种获取类属性的方法，**getFields()** 和 **getDeclaredFields()** 方法。前者只能获取类中的公有属性以及父类中的公有属性，后者可以获取类中所有的属性，但**不包括父类的属性**。
- 获取类中方法的方法，**getMethods()** 和**getDeclaredMethods()**，同样，前者只能获取到类中的公有方法以及从父类继承来的公有方法，后者可以获取当前类中定义的所有方法，不区分访问权限，不包括父类方法。
- 获取类中的构造方法信息的方法：**getConstructors()** 和**getDeclaredConstructors()**，前者只获取公有构造方法，后者获取所有构造方法。

详细测试用例：
Human.java:

``` java
public class Human {
    public String sex;
    protected String height;
}
```

Person.java:
```java
import java.io.Serializable;

public class Person extends Human implements Serializable {

    private Integer personId;
    public String personName;
    protected static int personAge;

    public Person(){ 
	}
    private Person(int personId,String personName){
    }
    protected Person(String personName){
    }
	
    public Integer getPersonId() {
        return personId;
    }

    public void setPersonId(Integer personId) {
        this.personId = personId;
    }

    public String getPersonName() {
        return personName;
    }

    private void setPersonName(String personName) {
        this.personName = personName;
    }

    public int getPersonAge() {
        return personAge;
    }

}

```

ReflectDemo.java:
```java
import org.junit.Test;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.Arrays;

public class ReflectDemo {
    /**
     * 三种获取类的Class对象的方式
     */
    @Test
    public void testCreateClass() throws Exception {
        //通过.class获取
        System.out.println(Person.class);
        //通过Class对象中的classForName()获取
        Class<?> aClass = Class.forName("com.xzy.reflect.Person");
        System.out.println(aClass);
        //通过类的实例获取
        Person person = new Person();
        System.out.println(person.getClass());
    }

    /**
     * 获取一个类中的属性
     */
    @Test
    public void testClassForField(){
        //获取类的Class对象
        Class<Person> clazz = Person.class;
        //获取类中的所有属性
        //getFields()方法获取类中所有public属性,包括父类的共有属性
        Field[] fields = clazz.getFields();
        for (Field field:fields){
            System.out.println(field);
        }

        System.out.println("======================================");

        //只获取当前类中所有定义的属性，不区分访问权限
        Field[] fields1 = clazz.getDeclaredFields();
        for (Field field:fields1){
            System.out.println(field);
            //各部分分开获取
            System.out.println("访问修饰符："+Modifier.toString(field.getModifiers()));
            System.out.println("属性名称："+field.getType().getSimpleName());
            System.out.println("属性名："+field.getName());
        }
    }
    /**
     * 获取方法的所有信息
     */
    @Test
    public void getClassForMethod() throws Exception {
        //获取Class对象
        Class<Person> clazz = Person.class;
        //获取所有public修饰的方法
        Method[] methods = clazz.getMethods();
        for (Method method:methods){
            System.out.println(method);
        }
        System.out.println("===============================");
        //获取当前类中所有方法
        Method[] declaredMethods = clazz.getDeclaredMethods();
        for (Method method:declaredMethods){
            System.out.println(method);
            System.out.println("访问修饰符："+Modifier.toString(method.getModifiers()));
            System.out.println("返回类型："+method.getReturnType().getSimpleName());
            System.out.println("方法名："+method.getName());
            System.out.println("参数个数："+method.getParameterCount());
            System.out.println("参数类型："+Arrays.toString(method.getParameterTypes()));
        }
        System.out.println("=======================================");
        //获取单个方法
        Method method = clazz.getDeclaredMethod("setPersonName", String.class);
        //通过反射实例化一个对象
        Person instance = clazz.newInstance();
        //开启方法访问权限
        method.setAccessible(true);
        //通过反射钓鱼用实例中对应的方法
        method.invoke(instance,"张三");
        //获取值,注意需要再获取getPersonName方法
        method = clazz.getDeclaredMethod("getPersonName");
        //因为getPersonName方法是public，所以不需要开启访问权限，直接拿值
        Object value = method.invoke(instance);
        System.out.println(value);

    }
    /**
     * 获取类中的构造方法信息
     */
    @Test
    public void getClassForConstructor() throws Exception {
        //获取Class对象
        Class<Person> clazz = Person.class;
        //获取所有public修饰的构造方法
        Constructor<?>[] constructors = clazz.getConstructors();
        for (Constructor constructor:constructors){
            System.out.println(constructor);
        }
        System.out.println("=====================================");
        //获取类中所有的定义的构造方法
        Constructor<?>[] declaredConstructors = clazz.getDeclaredConstructors();
        for (Constructor constructor:declaredConstructors){
            System.out.println(constructor);
            System.out.println("访问修饰符："+Modifier.toString(constructor.getModifiers()));
            System.out.println("构造方法名："+constructor.getName());
            System.out.println("参数列表："+Arrays.toString(constructor.getParameterTypes()));
            System.out.println("-------------------------");
        }
        //获取类中指定的构造方法
        Constructor<Person> constructor = clazz.getDeclaredConstructor(int.class, String.class);
        constructor.setAccessible(true);
        Person instance = constructor.newInstance(19, "李四");
        System.out.println(instance);
    }
    /**
     * 利用反射暴力访问类中的私有属性
     */
    @Test
    public void testAccessPrivate() throws Exception {
        //获取Class对象
        Class<Person> clazz = Person.class;
        //获取到Person对象的实例
        Object instance = clazz.newInstance();
        //单独获取到personId字段
        Field personIdField = clazz.getDeclaredField("personId");
        //设置字段访问权限
        personIdField.setAccessible(true);
        //调用字段对应的set方法
        personIdField.set(instance,23);
        //获取刚刚设置的字段中的值
        Object value = personIdField.get(instance);
        System.out.println(value);
    }
}
```