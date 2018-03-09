---
title: ArrayList源码阅读笔记
date: 2018-03-08 22:17:41
tags: 
	- Java 
	- 源码
---

源码版本JDK1.8.0_131

ArrayList内部是基于数组的动态管理来实现的，容量能自动增长，数组占据内存一块连续的存储空间，对于下标随机访问和遍历非常高效。

## ArrayList类定义
```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
从ArrayList类的定义可以看出：

* 支持泛型
* 继承AbstractList并实现了List接口，因此具有基本的增删查改功能
* 实现了RandomAccess接口，具有随机读写的功能。该接口是个标识接口
* 实现了Cloneable接口，可以被克隆
* 实现了Serializable接口，并**重写**了序列化和反序列化方法

---

## 成员变量
```
    // 数组的默认容量
    private static final int DEFAULT_CAPACITY = 10;

    // 空数组，用于带指定容量参数的构造函数
    private static final Object[] EMPTY_ELEMENTDATA = {};

    // 空数组，用于无参构造函数
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    // 数据存放的数组对象， elementData.length即为数组容量
    // 在序列化过程中不参与序列化，由重写的序列化方法实现具体序列化过程
    transient Object[] elementData; // non-private to simplify nested class access

    // 数组中存放元素的个数， 应小于数组容量
    private int size;
    
    // 数组最大可存放元素个数
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

---

## 构造函数
ArrayList有三种构造函数
```
    //1.指定容量就分配指定容量大小的数组
    //若指定大小为0用EMPTY_ELEMENTDATA赋值
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
    
    //2.默认构造函数，用DEFAULTCAPACITY_EMPTY_ELEMENTDATA赋值
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    //3.传入一个集合作为数组的数据
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

---

## 主要成员方法
### (1)添加
add方法有两种重载，一个是在数组末尾添加，一个是在指定位置添加元素

add(E e)的调用链涉及5个方法, 依次如下：

```
    public boolean add(E e) {
        // 确定数组容量能否再添加一个元素
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;  // 将元素赋值给扩容后的size位置上，并size加1
        return true;
    }
    
    // 确定数组的容量
    private void ensureCapacityInternal(int minCapacity) {
        // 如果数组为空，容量不能小于默认容量10
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }
    
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;  // 表明对数组进行了结构性修改，遍历时涉及
        
        // 数组容量不足最小需求容量时，进行扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        // 扩容长度是增加原来数组的一半大小，1.5倍扩容
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        
        // 判断扩容后是否符合最小容量，还不足则直接扩容至最小需求
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 判断扩容后大小是否超过上限
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
       
        // 复制原数组，并扩容数组大小至newCapacity
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0)
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
```
    public void add(int index, E element) {
        rangeCheckForAdd(index);  //判断下标是否在0到size之间

        ensureCapacityInternal(size + 1); 
        // 将elementData数组从index开始的size-index个元素往后移1位
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        //在指定位置插入新元素
        elementData[index] = element;
        size++;
    }
```
*modCount*变量是父类AbstrcatList中的属性，表示list结构化修改的次数。于遍历器的fail-fast机制相关。
由于要对数组元素整体移动，因此在指定位置插入的操作比较耗性能。

addAll方法也有两种重载，与add方法相似。将集合追加到末尾和将集合插入到指定位置。
*(实现方法基本相同，我不想这篇文章太过累赘，相似的代码就不帖出来了)*

ArrayList每次在增加元素时，都会*ensureCapacityInternal*方法来确保容量足够。在容量不够时会调用*Arrays.copyOf*方法将原数组拷贝到新数组中，这是一个非常耗性能的操作。因此建议在确定元素数量时才使用ArraysList，否则建议使用LindedList。

### (2)移除
remove方法也有2种重载，一个是移除指定下标的元素，一个是指定元素移除其第一次出现
```
    public E remove(int index) {
        rangeCheck(index);  // 越界检查

        modCount++;        // 移除操作也是对数组的结构性改动，不允许发生在遍历过程中
        E oldValue = elementData(index);

        int numMoved = size - index - 1; // 需要移动的元素个数
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // size减1，并让旧数组的末尾元素GC

        return oldValue;
    }
```
```
    public boolean remove(Object o) {
        // 分是否为null两种情况，过程大致一致
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    
    // 不进行越界检查，不返回被移除的元素
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; 
    }
```
批量删除
```
    // 删除与指定集合c中相同的元素
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

    // 保留与指定集合c中相同的元素
    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }

    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            // 遍历elementData数组
            for (; r < size; r++)
                // 通过compement来决定是否保留元素
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```
此外，还有removeRange方法将指定范围内的元素移除。


### (3)修剪
作用是将数组对象的capacity减小到size长度，减少ArrayList对象占用的内存。
```
    /**
     * Trims the capacity of this <tt>ArrayList</tt> instance to be the
     * list's current size.  An application can use this operation to minimize
     * the storage of an <tt>ArrayList</tt> instance.
     */
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```

### (4)包含
```
    // 利用indexOf方法判断是否包含某个元素
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    // 返回元素所在的下标，不存在则返回-1
    // 与remove(Object o)方法相似
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```
与indexOf方法相似的lastIndexOf方法，作用是从最后一个元素开始向前遍历。

### (5)克隆方法
Arraylist的克隆是浅克隆，即只产生ArrayList对象的副本，并不赋值元素本身
```
    public Object clone() {
        try {
            // 调用父类的clone方法，产生ArrayList对象副本
            ArrayList<?> v = (ArrayList<?>) super.clone();
            // 对elementData数组进行拷贝(浅拷贝)
            // 将elementData的地址赋值给副本的elementData
            v.elementData = Arrays.copyOf(elementData, size);
            // 基本类型的值会拷贝，所以副本的modCount要置零
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
```


### (6)数组拷贝方法
Arrays.copyOf方法参数含义：（原数组，拷贝个数），返回拷贝的数组。
System.arraycopy方法，该方法是一个本地方法，通过调用系统的C/C++方法实现，实现数组之间的移动和复制。
```
    public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
    }
    
    public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }

    public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```

## 序列化机制
ArrayList实现了Serializable接口，本身具备序列化的功能。那为什么ArrayList中存储数据的elementData数组要用transient修饰，并且重写*writeObject*和*readObject*方法？

分析源码，ArrayList重写的序列化方法其实就是把size和elementData数组中**不为null**的元素逐个写到流中，反序列化时在逐个读取。这么做的原因是因为数组每个扩容时，极端情况下会产生原来数组长度一半的为null的元素。在序列化时把这部分null排除出去，有助于提高性能

```
    // wirteObject方法，for循环中遍历size大小而不是elementData.length
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }
```

## 迭代机制
在进行ArrayList遍历时，可以调用iterator()方法返回一个迭代器，使用迭代器可以进行遍历操作。
```
    public Iterator<E> iterator() {
        return new Itr();
    }
    
    private class Itr implements Iterator<E> 
```

iterator方法返回的Itr实例是ArrayList的内部类，实现了Iterator接口。

成员变量
```
    int cursor;       // 指向迭代器下一个值的位置
    int lastRet = -1; // 指向迭代器最后取出元素的位置，没进行遍历时为-1
    int expectedModCount = modCount; //记录初始化迭代器时modCount的值
```
ArrayList的迭代器使用fail-fast机制，在调用add和remove方法时会使modCount++，记录ArrayList发生结构性修改的次数。在调用迭代器的next方法时会检查modCount与expectedModCount是否相等，不等则抛出ConcurrentModificationException异常。从而保证迭代器和ArrayList中**数据一致性**。

成员方法
```
    public E next() {
        checkForComodification();
        // 当前要迭代的位置
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        // 记录lastRet，返回元素
        return (E) elementData[lastRet = i];
    }
    
    // 迭代器内部的remove方法，移除当前遍历到的元素(lastRet指向的)
    public void remove() {
        // 还没进行遍历
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            // 调用的是ArrayList的remove方法
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            // lastRet置为-1，所以不能连续remove
            lastRet = -1;
            // 不会报异常
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
    
```


---

## 总结
- ArrayList非线程安全，只能应用在单线程环境下。
- 多线程情况下，JDK提供有Vector、Collections.SynchronizedList(List<E> list)。推荐JDK并发包的CopyOnWriteArrayList，Guava和Apache Common等提供的线程安全的List。

---

 第一次写技术博客，想法由来已久，与其说是技术博客，不如算是对自己知识的回顾与总结。
 写的过程中，发现很久知识已经生疏，整理的时候又有新的认识。
 过程中也查了很多资料，借鉴了下面两篇文章，在这里向作者表达感谢。
## 参考资料
[微信公众号：我是攻城师][1]
[tinylcy.me][2]


  [1]: https://mp.weixin.qq.com/s/SPSxbuaduIG9rayYfRXhaQ
  [2]: http://tinylcy.me/2016/ArrayList%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/
