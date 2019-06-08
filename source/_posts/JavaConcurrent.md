---
title: Java并发编程——理论基础
date: 2019-05-15 21:15:50
tags:
        - Java
---

# Java并发编程学习笔记——理论基础

## 并发的三个核心问题
- 分工
- 同步
- 互斥  
![并发编程核心问题][1]

微观层面  
1. CPU增加缓存，以均衡与内存的速度差异。  
    缓存带来可见性问题  
    可见性：一个线程对共享变量的修改，另外一个线程能够立即看到

2. 操作系统增加进程线程，分时复用CPU，均衡与IO设备的速度差异。  
    线程切换带来原子性问题   
    原子性：一个或多个操作在CPU执行过程中不被中断的特性
    > 例如：count += 1； 32位机器对long型变量 都不是原子性操作

3. 编译器优化指令执行次序，使缓存更加合理地利用。  
    编译优化带来有序性问题
    > 例如： 双重检查创建单例对象  

宏观层面  
1. 安全性  => 互斥、锁  
    数据竞争：多个线程访问未加锁的共享资源  
    竟态条件：程序的执行结果依赖线程的执行顺序  
    
2. 活跃性 => 资源公平分配，公平锁，等待队列  
    死锁，活锁，饥饿

3. 性能问题  
    使用无锁的数据结构和算法，减少持有锁的时间

    > 指标  
    吞吐量：指**单位时间**内能处理的请求数量  
    延时：指发出请求到收到响应的时间  
    并发量：指**同时**能处理的请求数量

---

## Java内存模型对可见性和有序性的处理

### Happens-Before规则  

1. 程序次序规则  
    在一个线程中，按照程序的顺序，前面的操作happens-before于后续的操作

2. volatile变量规则  
    对volatile变量的写操作happens-before于后续对该变量的读操作 (一个线程写volatile变量后，其他线程总是可见的)  

3. 管程锁定规则  
    对一个锁的解锁操作happens-before于后续对这个锁的加锁操作 (一个线程在锁中的操作，解锁后其他线程都是可见的)  

4. 线程启动规则  
    线程A调用线程B的start方法时，线程B能够看到线程A在启动线程B前的操作  

5. 线程终止规则  
    线程A得到线程B完成(调用线程B的join方法)，当线程B完成返回后，线程A能够看到线程B的操作(对于共享变量而言)

    **Happens-Before是有传递性的**

    实现方式： 内存屏障(Memory Barrier)；对编译器而言，内存屏障会限制指令重排序；对处理器而言，内存屏障会使缓存刷新

---

## 锁对原子性问题的处理

**加锁的本质是在锁对象的对象头中写入当前线程id**  

**原子性的本质：多个资源间有一致性要求，操作的中间状态对外不可见**

```
class SafeCals {
    long value = 0L;
    long get() {  // 但get方法的可见性没保证到
        return value;
    }
    synchronized void addOne() { //多线程执行addOne方法能保证可见性
        value += 1;
    }
}
```
解决方法：get方法加锁synchronized，或value变量加volatile关键字

```
    // 加的是SafeCals.class的锁，加不同的锁也无法保证可见性
    synchronized static long get() { 
        return value;
    }
```

```
    // JVM逃逸分析后，sync代码会被优化掉
    void addOne() {
        synchronized(new Object()) {
            value += 1;
        }
    }
```

### 锁和受保护资源的关系

**受保护资源和锁的关系应该是N：1 (不能多把锁保护一个资源)**

```
class Account {
    private int balance;
    
    // 有并发问题，this对象只能保护自己的balance字段
    // 保护不了target对象的balance字段
    synchronzied void transfer(Account target, int amt) {
        ...
    }
}
```
解决方法：创建Acount是传入**同一个**Object对象作为锁, 或者使用Account.class作为锁(但有性能问题，所有账户的转账transfer操作都会变成串行的)  

> 不能用this.balance这类可变对象作为锁，例如Integer，String，Boolean

### 死锁的处理方式
```
void transfer(Account target, int amt) {
    synchronzied(this) {
        synchronzied(targer) {   
            // 锁的粒度细，但先后加锁不同的对象，有可能产生死锁
            ...
        }
    }
}
```

当以下四个条件同时满足时，才会发生死锁  
- 互斥： 共享资源X和Y只能被一个线程占有  (目的)
- 占有且等待： 线程占有X在等待Y时，不会释放X
- 不可抢占： 其他线程不能抢占线程占有的资源
- 循环等待： 两个线程相互等待对方占有的资源

#### 破坏占有且等待条件  

一次申请所有资源
```
// 创建一个单例的类，负责一次申请所有资源
class Allocator {
    private List<Object> als = new ArrayList<>(); // 存放表示已申请到的资源
    synchronzied boolean apply(Object from, Object to) {
        // list中有对象，表明锁已经被其他线程申请了
        if (als.contain(from) || als.contain(to)) 
            return false;
        else {
            als.add(from);
            als.add(to);
        }
        return true;
    }
}
```

#### 破坏不可抢占条件
&emsp;要求获取资源的线程能主动解锁 (synchronzied原语做不到主动释放资源)

#### 破坏循环等待条件
&emsp;对资源进行排序，申请资源时按顺序申请  (相对来说成本小)

### 等待-通知机制
```
while(!allocator.apply(from, to));   // 不停的请求锁
---
    // 改进后
    synchronzied void apply(Object from, Object to) {
        while (als.contain(from) || als.contain(to)) 
            try {
                wait();
            } catch(Exception e) {
            }
        als.add(from);
        als.add(to);
    }

    // 释放资源
    synchronzied void free() {
        als.remove(from);
        als.remove(to);
        notifyAll();
    }
```

----

## 管程
&emsp;管程：管理共享变量以及对共享变量的操作过程，让其支持并发

### 管程对互斥的处理方式  
&emsp;将共享变量及其操作封装起来(类似Java)，同一时间只允许一个线程进入管程执行  

### 管程对同步的处理方式

**同步意味着有条件控制**  
![MESA管程模型][2]  
Hasen/Hoare/MESA三种管程模型的核心区别: 当条件满足后，通知线程的方式不同 (假如当线程T2使线程T1等待的条件满足，线程T1和T2如何执行)  

- Hasen: 要求notify方法放在最后。T2通知完T1后，T2结束，T1执行
- Hoare：T2通知完T1后，T2阻塞，T1立马执行，T1执行完再唤醒T2  
  - 多了阻塞操作，本质是中断当前线程
- MESA：T2通知完T1后，T2接着执行，T1从条件变量等待队列进入到入口等待队列
  - 好处是notify不用放在最后，也无阻塞操作
  - 副作用是当T1执行时，条件有可能变化，因此需要轮询执行条件

    ```
    // MESA管程模型的编程范式
    while(条件不满足) {
        wait();
    }
    ```

Java内置的管程synchronized对MESA模型进行了精简，只有一个条件变量。  

wati()方法只有MESA模型有超时时间的参数。因为notify后，是将等待的线程放入入口等待队列，不一定有机会执行，所以要设超时时间。Hasen/Hoare模型都是notify后，等待的线程肯定能执行到。

----

```
Thread th = Thread.currentThread();
while(true) {
    if (th.isInterrupted()) {
        break;
    }
    ...
    try {
        Thread.sleep(100);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
// 当线程被中断后，不能退出while循环
// 因为抛出异常后会清理中断标志，因此执行不到break语句 (线程大部分时间在执行sleep)
```
修改方式：  
1. 在catch中重置中断标志， Thread.currentThread().interrupt();  
2. try包在while循环外
3. 在catch中break;

----

### 逸出
1. 静态方式若操作了静态变量就会有线程安全问题
2. 尽管方法内部考虑了线程安全，但方法的参数是引用类型时，也会产生线程安全问题 

  [1]: https://github.com/jjz921024/jjz921024.github.io/raw/master/images/concurrent/javaconcurrent.png
  [2]: https://github.com/jjz921024/jjz921024.github.io/raw/master/images/concurrent/guancheng.png
