---
title: 单例模式整理
date: 2018-07-12 12:31:45
tags:
        - 设计模式
        - Java
---

# 单例模式

**单例模式** 

保证一个类在内存中只有一个实例，并提供一个全局访问点

- 饿汉式

```
class Singleton{
   /**
     在类内创建对象  
     静态，保证内存中只有一个
     私有，进行封装通过方法调用
    */
   private static Singleton s = new Singleton();  
                                            
   private Singleton(){}                       //私有化构造函数，对外不能创建对象

   public static Singleton getInstance(){      //公共静态方法，用类名调用
        return s;                              //Singleton s1 = Singleton.getInstance();
   }
}
```

- 懒汉式
```
class Singleton{

    private static Singleton s;   //懒汉模式，声明时未实例化

    private Singleton(){}

    public static Singleton getInstance(){
        if(s == null)           //存在线程安全问题
            s = new Singleton();
        return s;
    }
}
```
未实例化时，可能存在多个线程都通过判空s==null的情况，因此存在线程安全问题。

- 双重检查加锁  
改进懒汉式线程安全问题，在创建对象前加同步锁Synchronized

```
class Singleton{

    private volatile static Single s = null; 
    
    private Singleton(){}
    
    public static Single getInstance(){
        if(s==null){                  
            synchronized (Single.class) {
                if(s==null){      
                     s = new Single();
                }
            }
        }
        return s;
    }
}
```

1. 第一次判空防止实例化后，每次调用getInstance方法都会同步，提高性能

2. 第二次判空防止未实例化时，多个线程通过第一次判断，进入等待。其中一个线程创建了对象后，另一个线程也能再一次创建对象，无法保证单例

3. volatile关键字防止因为JVM的指令重排而产生线程安全问题，保证指令执行的顺序

    >如：instance = new Singleton，会被编译器编译成如下JVM指令：
    >
    >memory = allocate();    //1：分配对象的内存空间
    >ctorInstance(memory);   //2：初始化对象
    >instance = memory;      //3：设置instance指向刚分配的内存地址
    >
    >但是这些指令顺序并非一成不变，有可能会经过JVM和CPU的优化，指令重排成下面的顺序：
    >memory = allocate();    //1：分配对象的内存空间
    >instance = memory;      //3：设置instance指向刚分配的内存地址
    >ctorInstance(memory);   //2：初始化对象
    >
    >当线程A执行完1,3时，instance对象还未完成初始化，但已经不再指向null。此时如果线程B抢占到CPU资源，执
    >行if（instance == null）的结果会是false，从而返回一个没有初始化完成的instance对象。

    >加入volatile关键字时，生成的汇编代码会多出一个lock前缀指令，lock前缀指令实际上相当于一个内存屏障(也叫内存栅栏)，内存屏障会提供3个功能:
    >1.它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面，即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
    >2.它会强制将对缓存的修改操作立即写入主存；
    >3.如果是写操作，它会导致其他CPU中对应的缓存行无效
    >**volatile不保证原子性，保证可见性和部分有序性**


- 静态内部类
```
public class Singleton {
    private Singleton() {}

    public static Singleton getInstance(){
        return SingletonHolder.instance;
    }

    private static class SingletonHolder{
        private static Singleton instance = new Singleton();
    }  
}
```

1. instance对象是通过在调用getInstance方法时才实例化对象，通过ClassLoader的加载机制实现懒加载(加载外部类时，并不加载内部类)

2. 线程安全，但能利用反射破坏单例

```
//获得构造器
Constructor con = Singleton.class.getDeclaredConstructor();
//设置为可访问
con.setAccessible(true);
//构造两个不同的对象
Singleton singleton1 = (Singleton)con.newInstance();
Singleton singleton2 = (Singleton)con.newInstance();
//验证是否是不同对象
System.out.println(singleton1.equals(singleton2));
```

- 单元素枚举
```
public enum ResourceEnum {
    INSTANCE;
    
    private Resource instance;
    
    ResourceEnum() {
        instance = new Resource();
    }
    public Resource getInstance() {
        return instance;
    }
}

// 需要单例的资源
class Resource{
}
```
调用方法：ResourceEnum.INSTANCE.getInstance()

1. 枚举类的构造方法限制为私有，在访问枚举实例时会执行构造方法

2. 每个枚举实例都是static final类型，只能被实例化一次

3. 单例对象在枚举类加载时进行初始化，并非懒加载

3. 能防止反射破坏单例

4. 保证反序列化结果为同一对象

    >JVM在序列化时，只是将枚举对象的name属性输出到结果中，反序列化时通过java.lang.Enum的valueOf()方法根据名字查找枚举对象。因此反序列化后的对象和序列化前的对象实例相同。
    >其他单例模型要做到这点，必须实现readResolve()方法




