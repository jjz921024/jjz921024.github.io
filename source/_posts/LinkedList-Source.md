---
title: LinkedList源码阅读笔记
date: 2018-03-11 00:42:43
tags: 
	- Java
	- 源码
---

源码版本JDK1.8.0_121

LinkedList内部是基于双向链表实现的，元素在内存中的存储并不是连续的，通过节点的引用来关联所有元素。优点和ArrayList相反，添加和删除元素比较快，查询和遍历效率低。

## LinkedList类定义
```
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```
从LinkedList类的定义可以看出：

* 支持泛型
* 继承AbstractSequentialList并实现了List接口，所以是一个有序列表
* 实现了Deque接口，可以作为双端队列操作 (能用作双端队列说明也可作为 队列和栈)
* 实现了Cloneable接口，可以被克隆
* 实现了Serializable接口，并**重写**了序列化和反序列化方法

---

## Node节点数据结构
```
    private static class Node<E> {
        E item;  //当前节点的数据
        Node<E> next;  //后一个节点
        Node<E> prev;  //前一个节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

---

## 构造函数和成员变量
```
    // 链表存储元素个数
    transient int size = 0;
    // 首节点
    transient Node<E> first;
    // 尾节点
    transient Node<E> last;

    public LinkedList() {
    }
    
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```
LinkedList有两个构造函数，默认无参构造函数创建一个空链表。带参的构造函数传入一个集合，调用addAll方法将元素插入到链表。

*注意：* LinkedList的带参构造函数调用this()方法后，有可能有其他线程想链表插入数据，所以集合中的元素并不一定从链表首节点开始。

---

## 成员方法
- addAll方法
构造函数中的addAll方法的调用链涉及三个主要函数

```
    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }
    
    // 在index位置插入集合元素
    public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index); //越界检查

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        // 创建第index个节点(即为要插入的第一个节点)的
        // 临时的前置和后置节点 
        Node<E> pred, succ;
        // 在链表尾部插入时，前置节点为last节点，后置节点为null
        if (index == size) {
            succ = null;
            pred = last;
        } else { // 在非尾部插入时，前置节点为第index个节点，后置节点为原来第index个节点的后置
            succ = node(index); //node()方法返回第index个节点
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o; //强转成指定泛型
            // 当前插入节点初始化
            Node<E> newNode = new Node<>(pred, e, null);
            // 如果插入的节点的pred为null，则是首节点
            if (pred == null)
                first = newNode;
            else        
                pred.next = newNode;  // 尾插法，与前置节点相连
            // 一次插入完成后，将当前节点赋值给pred，作为下次插入的前置
            pred = newNode;
        }

        // 集合插入完成后，如果succ为null，则说上面遍历插入的最后一个节点为尾节点
        if (succ == null) {
            last = pred;
        } else {        //否则将遍历的最后一个节点和原链表的index位置相连
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
    
    // 返回index位置上的节点
    Node<E> node(int index) {
        // 折半查询，如果index小于size的一半，从前遍历
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

再看其他add方法
add(E e)功能是在尾部添加元素，调用linkLast方法
add(int index, E element)功能是在index插入，同样调用的主要方法有linkLast和linkBefore

```
    // 在末尾插入，实现和上面addAll中的代码相似
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
    
    // add内部调用 linkBefore(element, node(index));
    // 功能是将e节点插入到succ节点之前
    void linkBefore(E e, Node<E> succ) {
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

- 移除方法
移除方法常用有两个，一个是根据index移除，一个是根据object移除

```
    public E remove(int index) {
        // 注意，这里检查的是有效下标  index < size   index从0开始记
        checkElementIndex(index);
        return unlink(node(index));
    }
    
    public boolean remove(Object o) {
        // 分移除数据是否为null处理
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
    
    // 主要的方法，功能是移除一个节点
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
        
        // 维护前置节点
        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }
        // 维护后置节点
        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```
此外还有没有任何参数的remove，removeFirst，removeLast方法，其都为Deque接口的实现，后面总结。

从上面分析的增删方法可以看出，基于首尾节点的增删操作都是O(1)复杂度；而非首尾操作要调用node()方法遍历链表，所以平均复杂度是O(n)。

- 查询get方法有get(index)，getFirst()，getLast()，
其中get(index)调用node()方法，经过折半优化；getFirst和getLast为Deque接口实现。

- 序列化
LinkedList自己重写序列化方法的原因和ArrayList一样，为了节省空间。如果将整个LinkedList会把Node节点给写入序列化，由于Node是双端链表的数据节点，会导致多浪费2倍的空间。

```
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {

        s.defaultWriteObject();

        s.writeInt(size);

        for (Node<E> x = first; x != null; x = x.next)
            s.writeObject(x.item);
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {

        s.defaultReadObject();

        int size = s.readInt();

        for (int i = 0; i < size; i++)
            linkLast((E)s.readObject());  //尾插
    }
```

-  克隆
LinkedList的克隆也是浅克隆，即只克隆LinkedList并不克隆每个节点。

```
    public Object clone() {
        // 克隆LinkedList本身
        LinkedList<E> clone = superClone();

        // clone的first和last会指向原本链表的first和last
        // 为了下面循环遍历添加元素，先置未null
        clone.first = clone.last = null;
        clone.size = 0;
        clone.modCount = 0;

        for (Node<E> x = first; x != null; x = x.next)
            clone.add(x.item);

        return clone;
    }
    
    private LinkedList<E> superClone() {
        try {
            return (LinkedList<E>) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }
    }
```

---

```
public interface Deque<E> extends Queue<E> {
    
    // ***双端队列的方法***
    void addFirst(E e);
    void addLast(E e);
    E removeFirst();
    E removeLast();
    
    // 插入队列首尾，调用的是addFirst和addLast
    boolean offerFirst(E e);  //调用addFirst
    boolean offerLast(E e);   //调用addLast
   
    // 查看并移除，与remove*区别在于不会报异常
    E pollFirst();
    E pollLast();
    
    // 查看但不移除，不同处在于get*方法得到null会报NoSuchElementException异常，peek*不会
    E getFirst();
    E getLast();
    E peekFirst();
    E peekLast();
    
    boolean removeFirstOccurrence(Object o);
    boolean removeLastOccurrence(Object o);

    // *** Queue methods ***
    boolean add(E e);
    E remove();         //调用removeFirst。移除队首元素，与poll不同处在于移除null会报异常
    
    boolean offer(E e); //添加元素到队尾
    E poll();           //移除队首元素
    E element();        //查看队首元素，null会报异常
    E peek();           //查看队首元素

    // *** Stack methods ***
    void push(E e);  // 压栈
    E pop();         // 出栈
}
```
- Deque接口小结：
队列提供的主要操作有offer、poll和peek，双端操作在方法名后加上后缀First和Last。
双端队列的移除方法有remove\*和poll\*，区别在于遇到null是否报异常；队列的方法同理
双端队列的查询方法有get\*和peek\*，区别在于遇到null是否报异常；队列的方法element和peek同理
队列和栈的**增删**操作调用的是addFirst/addLast/removeFirst/removeLast方法，它们又调用**LinkedList的核心方法link/linkFirst/linkLast/unlink/unlinkFirst/unlinkLast**。

---

迭代器
```
private class ListItr implements ListIterator<E> {
        // 前一个遍历过的节点
        private Node<E> lastReturned;
        // 下次遍历的节点
        private Node<E> next;
        // 下次遍历的节点下标
        private int nextIndex;
        private int expectedModCount = modCount;

        // 构造函数，指定从index开始遍历， 下标：0<=index<size
        ListItr(int index) {
            // index等于size时，next节点为null；下标index比size小1，所以nextIndex等于size
            next = (index == size) ? null : node(index);
            nextIndex = index;
        }

        public boolean hasNext() {
            return nextIndex < size;
        }

        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }
    }
```
ListItr不仅提供从任意节点开始向后变量的功能，也可以向前遍历，移除/修改前次遍历过的元素，在下次遍历指向的next节点前插入元素的功能。

---
参考文章
[JDK8中LinkedList的工作原理剖析][1]


  [1]: https://mp.weixin.qq.com/s/HIWlNFgMf4GbU156lomloA
