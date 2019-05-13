---
title: Java集合框架整理(四)——PriorityQueue、DelayQueue源码分析
tag: Java
---

# PriorityQueue、DelayQueue源码分析

本文将会讲解两个Queue的源码，分别是PriorityQueue和DelayQueue

## PriorityQueue源码分析

> 具有优先级的Queue
> 利用数组实现的Queue

直接进入正题吧，先看PriorityQueue的继承结构吧

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
    ...
}
```

PriorityQueue是通过继承AbstractQueue实现的，并实现了Serializable，这都没啥好说的，关于AbstractQueue的源码，还请先看[**Java集合框架整理**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(四)——PriorityQueue、DelayQueue源码分析/)

接着看看变量和构造方法吧

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
    private static final long serialVersionUID = -7720805057305804111L;
    private static final int DEFAULT_INITIAL_CAPACITY = 11;  //默认初始化容量大小为11
    transient Object[] queue;  //通过数组实现的队列，元素存放的地方，不可序列化
    private int size = 0;  //元素个数
    private final Comparator<? super E> comparator;
    transient int modCount = 0; // non-private to simplify nested class access
    //默认实例化时容量为默认值值11，没有比较方式
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }
    //指定容量大小实例化
    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }
    //指定比较方式（如何判定优先级）实例化
    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }
    //最后会重载到这个构造方法
    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }
    //通过instanceof来判断时SortedSet类型、PriorityQueue类型还是其他的Collection，后面两个就是分别对应参数的构造方法
    //这个方法相对于其他类的实现相对要多一点，因为涉及到comparator，SortedSet
    @SuppressWarnings("unchecked")
    public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }
    //直接通过PriorityQueue实例化
    @SuppressWarnings("unchecked")
    public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }
    //直接通过SortedSet实例化
    @SuppressWarnings("unchecked")
    public PriorityQueue(SortedSet<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initElementsFromCollection(c);
    }
    ...
}
```

PriorityQueue有多个构造方法，默认就是使用默认的容量11和不指定comparator进行实例化，当然也可以指定优先级的判定。然后又有根据Collection、PriorityQueue、SortedSet进行实例化的，不过`public PriorityQueue(Collection<? extends E> c)`已经包括了后面两种，所以我们着重看一下这个吧。
这个构造方法会对Collection进行一个判别，如果是SortedSet类型的就采用对应的处理方式，因为我们知道SortedSet也是一个有序的集合，所以也有comparator；如果是PriorityQueue，那就不用说了，就是一个类型的；如果是其他的Collection，那就说明没有comparator，就是一个普通的集合，最后放到队列里，元素就没有优先级

接着看看这三种情况下，到底怎么处理元素的吧

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
    ...
    //处理PriorityQueue类型
    private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
        if (c.getClass() == PriorityQueue.class) {
            this.queue = c.toArray();
            this.size = c.size();
        } else {
            initFromCollection(c);
        }
    }
    //处理元素数据
    private void initElementsFromCollection(Collection<? extends E> c) {
        Object[] a = c.toArray();
        // If c.toArray incorrectly doesn't return Object[], copy it.
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, a.length, Object[].class);
        int len = a.length;
        //检验元素是否存在null
        if (len == 1 || this.comparator != null)
            for (int i = 0; i < len; i++)
                if (a[i] == null)
                    throw new NullPointerException();
        this.queue = a;
        this.size = a.length;
    }
    //其他Collection类型
    private void initFromCollection(Collection<? extends E> c) {
        initElementsFromCollection(c);
        heapify();
    }
    //采用堆排序进行排序
    private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
    //进行元素的调整，k是待插入的位置，x是插入的值，进行堆调整（下沉）
    private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }
    //默认的比较方式compareTo
    private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        int half = size >>> 1;        // loop while a non-leaf
        while (k < half) {
            int child = (k << 1) + 1; // assume left child is least
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                c = queue[child = right];
            if (key.compareTo((E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = key;
    }
    //采用comparator比较方式
    private void siftDownUsingComparator(int k, E x) {
        int half = size >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                comparator.compare((E) c, (E) queue[right]) > 0)
                c = queue[child = right];
            if (comparator.compare(x, (E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = x;
    }
    ...
}
```

针对是不是SortedSet、PriorityQueue以及其他的Collection采取了不同的处理方式，因为SortedSet和PriorityQueue都是已经排好序了的，所以只需要检验一下元素，转为的数组是否符合Object[]和赋值comparator就可以了；而对于那些不是有序的Collection，在处理元素时就需要处理排序的问题，这里采用的就是堆排序（**利用了二叉堆的性质进行实现的**），然后根据是否指定了comparator来进行不同的排序，没有指定comparator时采用默认的比较方式（`compareTo`比较)，而指定了comparable就用指定的进行比较，进行堆调整

接着看看几种常用的操作吧

### add

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
    ...
    //调用offer进行添加 入队
    public boolean add(E e) {
        return offer(e);
    }

    public boolean offer(E e) {
        //先对null进行检验，因为涉及排序所以不允许null的存在
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        //是否不够容量，不够就要扩容
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e);  //假设插到最后，然后进行位置调整
        return true;
    }

    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;  //最大值
    //真正扩容的地方
    private void grow(int minCapacity) {
        int oldCapacity = queue.length;
        // Double size if small; else grow by 50%
        int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1));
        // overflow-conscious code
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        queue = Arrays.copyOf(queue, newCapacity);
    }
    //最大容量调整
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    //插入并进行元素调整，从后向前遍历整个数组进行插入（上浮）
    private void siftUp(int k, E x) {
        if (comparator != null)
            siftUpUsingComparator(k, x);
        else
            siftUpComparable(k, x);
    }
    //使用compareTo
    @SuppressWarnings("unchecked")
    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = key;
    }
    //有comparator
    @SuppressWarnings("unchecked")
    private void siftUpUsingComparator(int k, E x) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (comparator.compare(x, (E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = x;
    }
}
```

在需要扩容的时候，根据原来的容量是否小于64来进行两种方式的扩容，旧容量小于64，则在原来的基础上扩容旧容量+2，新的容量也就是2*旧容量+2；如果旧容量不是小于64，则在原来的基础上扩容一半，新的容量也就是1.5*旧容量。最大容量的调整跟其他集合类似

`siftUp(int k, E x)`在要插入的时候（这里的k就是队列中最后一个元素的位置），需要找到插入的位置，通过从后向前遍历数组，根据不同的比较方式，确定插入元素的位置，同时将比较过的元素向后移动，这样就顺利插入了。入队只能在队尾，出队只能第一个元素出队

### poll

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
    ...
    //出队
    public E poll() {
        if (size == 0)
            return null;
        int s = --size;
        modCount++;
        E result = (E) queue[0];
        E x = (E) queue[s];
        queue[s] = null;
        //如果不是空队，需要调整队列，在下标为0待插入x（最后一个元素），然后进行堆调整
        if (s != 0)
            siftDown(0, x);
        return result;
    }

    public boolean remove(Object o) {
        int i = indexOf(o);
        if (i == -1)
            return false;
        else {
            removeAt(i);
            return true;
        }
    }

    //移除某个位置的元素，然后进行堆调整
    private E removeAt(int i) {
        // assert i >= 0 && i < size;
        modCount++;
        int s = --size;
        if (s == i) // removed last element
            queue[i] = null;  //最后一个被移除直接置空
        else {  //不是最后一个的话
            E moved = (E) queue[s];
            queue[s] = null;
            siftDown(i, moved);  //将最后一个元素插入到i的位置，进行堆调整（下沉）
            if (queue[i] == moved) {  //如果i的位置是moved要移动的元素（也就是前面说的最后一个元素，说明没有调整成功，换种方式调整）
                siftUp(i, moved);  //继续调整二叉堆（上浮）
                if (queue[i] != moved)
                    return moved;
            }
        }
        return null;
    }
}
```

在删除某个元素的时候，可能会导致`siftDown()`调整二叉堆不成功，就需要换`siftUp()`进行调整，两者只会有一个执行，这两个函数是用来保证最大值或最小值处于顶端的

关于下沉、上浮的二叉堆调整参考[【Java源码】PriorityQueue源码剖析及其应用](https://blog.csdn.net/xidiancoder/article/details/77850535)

### peek

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
    ...
    public E peek() {
        return (size == 0) ? null : (E) queue[0];
    }
    ...
}
```

### 其他方法

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
    ...
    //获取位置
    private int indexOf(Object o) {
        if (o != null) {
            for (int i = 0; i < size; i++)
                if (o.equals(queue[i]))
                    return i;
        }
        return -1;
    }

    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

    public void clear() {
        modCount++;
        for (int i = 0; i < size; i++)
            queue[i] = null;
        size = 0;
    }
    ...
}
```

这个方法跟其他的集合逻辑都差不多，就不再多言

## DelayQueue源码分析

> 可重入锁ReentrantLock实现线程安全
> 用于根据delay时间排序的优先级队列
> 没有实现Serializable接口，不可序列化

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    //重入锁，实现同步
    private final transient ReentrantLock lock = new ReentrantLock();
    //内部采用优先级队列实现
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    //优化阻塞通知的线程，就是一个标志
    private Thread leader = null;
    //用于阻塞和通知的条件变量
    private final Condition available = lock.newCondition();

    public DelayQueue() {}

    public DelayQueue(Collection<? extends E> c) {
        this.addAll(c);
    }
    ...
}
```

DelayQueue继承自AbstractQueue，实现BlockingQueue（有关BlockingQueue可见[PriorityBlockingQueue、ArrayBlockingQueue源码分析](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(四)——PriorityQueue、DelayQueue源码分析/)），BlockingQueue就是一些api的定义

DelayQueue通过全局变量重入锁lock——ReetrantLock来实现上锁和同步，实现线程安全，同时，内部采用PriorityQueue来实现元素的存储。默认构造器没有什么参数

在看后面的方法之前，还需要看Delayed是什么，因为要使用DelayQueue的元素必须是实现Delayed的

```java
Delayedpublic interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}
```

Delayed是继承自Comparable的，这是为了方便内部使用PriorityQueue要用的，然后又定义的了getDelay方法获取delay，这两个方法是必须实现的

接着看常用的方法吧

### 入队

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    ...
    //内部调用offer方法
    public boolean add(E e) {
        return offer(e);
    }
    //添加元素
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        //获取锁，上锁
        lock.lock();
        try {
            //添加到PriorityQueue中
            q.offer(e);
            //如果刚放入的元素是第一个，之前是空队列，现在不是，则唤醒其他线程，当前没有线程占用
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
    //添加
    public void put(E e) {
        offer(e);
    }

    public boolean offer(E e, long timeout, TimeUnit unit) {
        return offer(e);
    }
    ...
}
```

最终都是通过offer来入队，并且通过PriorityQueue进行操作（具体方法在前面），然后多了获取锁这一步

### 出队

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    ...
    //出队
    public E poll() {
        final ReentrantLock lock = this.lock;
        //获取锁，上锁
        lock.lock();
        try {
            E first = q.peek();  //获取头
            //如果是空队或者延长还没到期，则返回null
            if (first == null || first.getDelay(NANOSECONDS) > 0)
                return null;
            else
                return q.poll();  //否则，出队
        } finally {
            lock.unlock();
        }
    }

    //有一定时间的出队，也可能导致阻塞
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);  //计算超时等待时间
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();  //可中断获取锁
        try {
            for (;;) {
                E first = q.peek();  //获取队头元素
                //如果是空队
                if (first == null) {
                    //如果是空队且超时等待时间已过（<=0），返回null
                    if (nanos <= 0)
                        return null;
                    else
                        nanos = available.awaitNanos(nanos);  //超时等待时间还没有过，则继续等待 超时等待时间 这么长的时间
                } else {
                    //有元素，计算其delay时延
                    long delay = first.getDelay(NANOSECONDS);
                    //delay<=0，到了时间，直接出队
                    if (delay <= 0)
                        return q.poll();
                    //超时等待时间到了，但元素的delay时延还没有到，则返回null
                    if (nanos <= 0)
                        return null;
                    first = null; // don't retain ref while waiting
                    //如果超时等待时间没到，且小于delay时延，则继续等待 超时等待时间 这么长的时间，或者没有线程占用
                    if (nanos < delay || leader != null)
                        nanos = available.awaitNanos(nanos);
                    else {
                        //如果超时等待时间没到，但大于delay时延，就获取当前线程，等待delay时间，获取元素
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;  //将leader标志量指定为当前线程，防止其他线程在抢断
                        try {
                            long timeLeft = available.awaitNanos(delay);  //等待delay时间
                            nanos -= delay - timeLeft;  //剩余超时等待时间
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
    ...
}
```

直接出队很简单，获取锁然后通过PriorityQueue直接出队就行。
稍微复杂一点的就是有一定等待时间的出队（超时等待时间），所以就会比直接出队多了几种情况（并且这个方法可能会阻塞）：当超时等待时间到了的话，就直接返回null；如果超时等待时间没到，就会与元素的delay时间进行比较，看是否等待多长时间，当超时等待时间小于delay时间时（超时等待时间到会比元素的delay时间先到），这时就只能等到超时等待时间的时长；如果超时等待时间大于元素的delay时间（delay时间先到，这时还在等待，可以获取元素），就等待delay长的时间，然后线程占用，获取元素；如果在此之前，已经有线程在占用了，那么不管超时等待时间与delay的时间如何，此时的线程只能等待

### 获取队头

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    ...
    //获取队头元素，可能会导致阻塞
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();  //可中断获取锁
        try {
            //死循环，一直获取头元素
            for (;;) {
                E first = q.peek();
                //空队，通知唤醒线程
                if (first == null)
                    available.await();
                else {
                    //不是空队就计算时延
                    long delay = first.getDelay(NANOSECONDS);
                    //已经过期，直接出队
                    if (delay <= 0)
                        return q.poll();
                    //不是空队（有元素）且还没有过期，则要阻塞
                    first = null; // don't retain ref while waiting
                    //如果有线程占有，通知其他线程等待
                    if (leader != null)
                        available.await();
                    else {
                        //没有线程占有，就当前线程占有，且等待delay的时间获取元素
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;  //获取完释放，不再占有
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
    //获取元素
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();  //获取锁，通过优先级队列直接获取
        try {
            return q.peek();
        } finally {
            lock.unlock();
        }
    }
    ...
}
```

其中take方法可能会引起阻塞，涉及等待，会有死循环一直获取，所以可能会导致阻塞。这个情况跟前面出队类似，重要区别就是take方法后面只有判断是否正有线程占用

### 其他方法

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    ...
    //获取大小，得到锁后直接获取
    public int size() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return q.size();
        } finally {
            lock.unlock();
        }
    }
    //获取过期的元素
    private E peekExpired() {
        // assert lock.isHeldByCurrentThread();
        E first = q.peek();
        return (first == null || first.getDelay(NANOSECONDS) > 0) ?
            null : first;
    }
    //获取所有过期的元素，放入集合c中，返回过期元素的个数
    public int drainTo(Collection<? super E> c) {
        if (c == null)
            throw new NullPointerException();
        if (c == this)
            throw new IllegalArgumentException();
        final ReentrantLock lock = this.lock;
        lock.lock();  //获取锁
        try {
            int n = 0;
            //通过peekExpired方法获取队头过期的元素，添加到集合c中，并移除，依次循环获取所有的队头过期元素
            for (E e; (e = peekExpired()) != null;) {
                c.add(e);       // In this order, in case add() throws.
                q.poll();
                ++n;
            }
            return n;
        } finally {
            lock.unlock();
        }
    }
    //比上一个方法多了一个条件，c有最大个数限制
    public int drainTo(Collection<? super E> c, int maxElements) {
        if (c == null)
            throw new NullPointerException();
        if (c == this)
            throw new IllegalArgumentException();
        if (maxElements <= 0)
            return 0;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int n = 0;
            for (E e; n < maxElements && (e = peekExpired()) != null;) {
                c.add(e);       // In this order, in case add() throws.
                q.poll();
                ++n;
            }
            return n;
        } finally {
            lock.unlock();
        }
    }
    //清空队
    public void clear() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.clear();
        } finally {
            lock.unlock();
        }
    }
    //总是返回int的最大值，因为DelayQueue是无界的
    public int remainingCapacity() {
        return Integer.MAX_VALUE;
    }
    ...
    //获取锁，通过PriorityQueue移除
    public boolean remove(Object o) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return q.remove(o);
        } finally {
            lock.unlock();
        }
    }
    ...
}
```

## 总结

- PriorityQueue
    1. 实现了Serialilzable，可序列化
    2. 默认容量是11，可指定Comparator，又默认的Comparable
    3. 数组实现，内部采用二叉堆的数据结构
    4. 可以SortedSet进行初始化，还可以PriorityQueue和其他Collection
    5. 每次添加、删除都会调整二叉堆的结构（上浮、下沉）
    6. 每次扩容先会针对64这个界值进行判断，旧容量小于64，会在原来的基础上扩容旧容量+2（新容量也就是2*旧容量+2）；如果不小于64，则在原来的基础上扩容一半（新容量也就是*旧容量）
- DelayQueue
    1. 通过可重入锁ReentrantLock实现线程安全
    2. 内部采用优先级队列PriorityQueue实现
    3. 没有实现Serializable，不可序列化
    4. 采用一个Thread类型的leader对象用来标识是否被线程占用
    5. DelayQueue的元素必须实现Delayed接口并实现其中的方法
    6. DelayQueue是无界的，获取容量的时候总会返回Integer.MAX_VALUE
    7. 可以通过drainTo方法获取所有过期的元素
    8. 大部分操作都是通过获取锁然后再进行操作，但take方法和带有超时限制的poll方法是可中断的锁，也可能会导致阻塞