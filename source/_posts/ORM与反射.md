---
title: ORM与反射
date: 2019-05-25 11:29:01
cover: /img/Random-img/91.jpg
categories:
- java
tags:
- java
- 反射
---

## ORM（Object Relation Mapping）

**对象关系模型**
关系模型（数据库表）：   表、字段、字段类型
对象模型（java实体类）：类、属性、属性类型
通过ORM框架解除模型之间的阻抗。

## 利用反射实现ORM中的准备预编译SQL语句

1. 自定义注解：

Column.java:
```java
package com.xzy.ORM.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)//指定标注位置为属性上
@Retention(RetentionPolicy.RUNTIME)//指定在运行期间有效
public @interface Column {
    //配置映射字段名称
    String name();
}
```
PK.java:

```java
package com.xzy.ORM.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)//指定标注位置为字段上
@Retention(RetentionPolicy.RUNTIME)//指定在运行期间有效
public @interface PK {
    //是否自动增长，默认为true
    boolean isAuto() default true;
}
```
table.java:

```java
package com.xzy.ORM.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 注意：
 * 1、注解标注的位置
 * 2、注解作用域
 */
@Target(ElementType.TYPE)//指定标注位置为类上
@Retention(RetentionPolicy.RUNTIME)//指定在运行期间有效
public @interface Table {
    String name();
}
```

2. 实体类：User.java

``` java
package com.xzy.ORM.entity;

import com.xzy.ORM.annotation.Column;
import com.xzy.ORM.annotation.PK;
import com.xzy.ORM.annotation.Table;

import java.io.Serializable;

/**
 * 用户实体类
 */
@Table(name = "users")
public class User implements Serializable {

    @PK(isAuto = false)
    @Column(name = "user_id")
    private Integer id;

    @Column(name = "user_name")
    private String name;

    private String email;
    private String country;

    @Column(name = "user_password")
    private String password;


    public void setId(int id) {
        this.id = id;
    }

    public Integer getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getCountry() {
        return country;
    }

    public void setCountry(String country) {
        this.country = country;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```
3. Dao层实现SQL语句：

UserDao.java:

``` java
package com.xzy.ORM.dao;

public interface UserDao {
    public int save(Object obj) throws Exception;
}
```
UserDaoImpl.java:

``` java
package com.xzy.ORM.dao;

import com.xzy.ORM.annotation.Column;
import com.xzy.ORM.annotation.Table;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;

public class UserDaoImpl implements UserDao {
    @Override
    public int save(Object obj) throws Exception {
        //加载驱动

        //获取连接

        //准备预编译SQL语句
        String sql = prepareSQL(obj);

        //通过连接对象创建预编译对象

        return 0;
    }

    /**
     * 拼装SQL语句的方法
     * 如果用@Table注解的使用注解表名，没有的话使用实体类作为表名
     *
     * @param obj
     * @return
     */
    private String prepareSQL(Object obj) throws Exception {
        StringBuffer stringBuffer = new StringBuffer("INSERT INTO ");
        //获取obj中定义的Class对象
        Class<?> clazz = obj.getClass();
        String tablename = clazz.getSimpleName().toUpperCase();
        //判断是否使用@Table注解
        Table table = clazz.getAnnotation(Table.class);
        if (table != null) {
            //获取@Table中的表名
            tablename = table.name().toUpperCase();
        }
        stringBuffer.append(tablename + "(");
        //获取当前操作类中所有定义的字段
        Field[] fields = clazz.getDeclaredFields();
        //定义存储字段对应的值的集合
        List<Object> valueList = new ArrayList<>();
        for (Field field:fields){
            //获取默认的字段名
            String firldName = field.getName().toUpperCase();
            //判断是否使用@Column注解
            Column column = field.getAnnotation(Column.class);
            if (column!=null){
                //获取映射的字段名称
                firldName = column.name().toUpperCase();
            }
            field.setAccessible(true);
            //获取字段的值
            Object value = field.get(obj);
            if (value!=null){
                stringBuffer.append(firldName+",");
                valueList.add(value);
            }
        }
        //删除最后一个逗号
        stringBuffer.deleteCharAt(stringBuffer.length()-1);
        stringBuffer.append(") VALUES (");
        //设置值
        for (Object  value:valueList){
            //如果是字符串，为其加上单引号
            Object temp = value;
            if (temp instanceof String){
                temp = "'"+value+"'";
            }
            stringBuffer.append(temp+",");
        }
        //删除最后一个逗号
        stringBuffer.deleteCharAt(stringBuffer.length()-1);
        stringBuffer.append(")");

        System.out.println(stringBuffer);
        return null;
    }
}
```
4. 启动类：App.java

``` java
package com.xzy.ORM.app;

import com.xzy.ORM.dao.UserDaoImpl;
import com.xzy.ORM.entity.User;

public class App {
    public static void main(String[] args) {
        //创建一个User对象实例
        User user = new User();
        user.setId(3);
        user.setName("Jack");
        user.setEmail("jack@google.com");
        user.setCountry("US");
        user.setPassword("12546");
        UserDaoImpl userDao = new UserDaoImpl();
        //输出一条正常的SQL语句
        try {
            userDao.save(user);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```