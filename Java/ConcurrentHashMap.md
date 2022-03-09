# ConcurrentHashMap

## 1 ConcurrentHashMap 1.7

ConcurrentHashMap 是 Java1.5 中引用的一个**线程安全的支持高并发**的 HashMap 集合类。

![](../asset/17CM.png)

如图所示：由 Segment 数组、HashEntry 组成，Segment 的个数是 **16** 个，源码如下所示：

```java
//Segment 数组，存放数据时首先需要定位到具体的 Segment 中。
final Segment<K,V>[] segments;
transient Set<K> keySet;
transient Set<Map.Entry<K,V>> entrySet;

//Segment 是 ConcurrentHashMap 的一个内部类
static final class Segment<K,V> extends ReentrantLock implements Serializable {
       private static final long serialVersionUID = 2249069246763182397L;       
       //真正存放数据的桶
       transient volatile HashEntry<K,V>[] table;
       transient int count;
       transient int modCount;
       transient int threshold;
       final float loadFactor;      
}
```

ConcurrentHashMap 采用了**分段锁**技术， Segment 继承于 ReentrantLock，每当一个线程占用锁访问一个 Segment 时，不会影响到其他的 Segment。

### 1.1 构造参数

```java
public ConcurrentHashMap() {
      //DEFAULT_INITIAL_CAPACITY 默认初始化容量 16
      //DEFAULT_LOAD_FACTOR 默认负载因子 0.75f
      //DEFAULT_CONCURRENCY_LEVEL 默认并发级别 16
      this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}


@SuppressWarnings("unchecked")
public ConcurrentHashMap(int initialCapacity,float loadFactor, int concurrencyLevel) {
    //校验参数
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    //校验并发级别大小，大于 1 << 16，重置为 65536
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    int sshift = 0;
    int ssize = 1;
    //这个循环可以找到 concurrencyLevel 之上最近的 2的次方值
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }   
    //segmentShift 偏移量
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    //Segment 中的类似于 HashMap 的容量至少是2或者2的倍数
    while (cap < c)
        cap <<= 1;
    // 创建 Segment 数组，设置 segments[0]
    Segment<K,V> s0 = new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0);
    this.segments = ss;
}
```

### 1.2 put

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          
         (segments, (j << SSHIFT) + SBASE)) == null)
        //如果 Segment 为空，则初始化，实际通过 key 定位到 Segment
        s = ensureSegment(j);
    //在 Segment 中 put
    return s.put(key, hash, value, false);
}

//Segment.java
inal V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //1.tryLock():尝试获取锁
    HashEntry<K,V> node = tryLock() ? null :
        //2.scanAndLockForPut():自旋获取锁
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        // CAS 获取 index 坐标的值
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

上面有两个比较重要的方法 tryLock() 和 canAndLockForPut()，意思就是尝试获取锁，如果获取失败肯定就有其他线程存在竞争，则利用 scanAndLockForPut() 自旋获取锁。

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
		//键值对的hash值定位到数组tab的第一个键值对
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;
        int retries = -1; 
		//线程尝试通过CAS获取锁
        while (!tryLock()) {
            HashEntry<K,V> f; 
            if (retries < 0) {
                if (e == null) {
                    if (node == null) 
						//初始化键值对，next指向null
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    retries = 0;
                else
                    e = e.next;
            }
			//超过最大自旋次数，阻塞
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
			//头节点发生变化，重新遍历
            else if ((retries & 1) == 0 &&
                    (f = entryForHash(this, hash)) != first) {
                e = first = f;
                retries = -1;
            }
        }
        return node;
    }
```

1. 通过 key ，获取当前 Segment 并定位到 HashEntry。
2. 遍历 HashEntry，如果不为空则判断传入的 key 和当前遍历的 key 是否相等，相等则覆盖旧的 value。
3. 不为空则新建 HashEntry 并加入到 Segment 中，同时会先判断是否需要扩容。
4. 最后解除 scanAndLockForPut() 中所获取当前 Segment 的锁。

### 1.3 get

```java
public V get(Object key) {
    Segment<K,V> s; 
    HashEntry<K,V>[] tab;
    //hash 运算
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

将 Key 通过 Hash 之后定位到具体的 Segment ，再通过一次 Hash 定位到具体的元素上。

 HashEntry 中的 value 是用 volatile 关键词修饰的，保证内存可见性，每次获取时都是最新值。

### 1.4 rehash

```java
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    //老容量
    int oldCapacity = oldTable.length;
    //新容量，扩大两倍
    int newCapacity = oldCapacity << 1;
    //新的扩容阀值 
    threshold = (int)(newCapacity * loadFactor);
    //创建新的数组
    HashEntry<K,V>[] newTable = (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            //计算新的位置
            int idx = e.hash & sizeMask;
            if (next == null)   
                //如果当前位置还不是链表，只是一个元素，直接赋值
                newTable[idx] = e;
            else { //链表
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;               
                for (HashEntry<K,V> last = next; last != null; last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                //lastRun 后面的元素位置都是相同的，直接作为链表赋值到新位置。
                newTable[lastIdx] = lastRun;
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    //遍历剩余元素，头插法到指定 k 位置。
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    //头插法插入新的节点
    int nodeIndex = node.hash & sizeMask; 
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

## 2 ConcurrentHashMap 1.8

1.7 已经解决了并发问题，并且能支持 N 个 Segment 这么多次数的并发，但是查询遍历链表效率太低，ConcurrentHashMap 1.8 抛弃 **Segment 分段锁**，而采用了  CAS + synchronized 来保证并发安全性。数据结构**Segment 数组 + HashEntry 数组 + 链表**也变成 **Node 数组 + 链表 / 红黑树**。



![](../asset/next.png)



### 2.1 put 

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            //数组桶为空，初始化数组桶（自旋+CAS)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //CAS操作得到对应 table 中元素
            if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))
                break;  
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // 链表
                    if (fh >= 0) {
                        binCount = 1;
                        //循环加入新的或者覆盖节点
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        //红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

- 根据 key 计算出 hashcode 。
- 判断是否需要进行初始化。
- `f` 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
- 如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容。
- 如果都不满足，则利用 synchronized 锁写入数据。
- 如果数量大于 `TREEIFY_THRESHOLD` 则要转换为红黑树。

### 2.2 get

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    //key 所在的 hash 位置
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        //如果指定位置元素存在，头结点hash值相同
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                //key hash 值相等，key值相同，直接返回元素 value
                return e.val;
        }
        else if (eh < 0)
            //头结点hash值小于0，说明正在扩容或者是红黑树，find查找
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            //链表，遍历查找
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

1. 根据 hash 值计算位置。
2. 查找到指定位置，如果头节点就是要找的，直接返回它的 value.
3. 如果头节点 hash 值小于 0 ，说明正在扩容或者是红黑树，查找之。
4. 如果是链表，遍历查找之。

## 初始化数据结构时的线程安全

就算有多个线程同时进行put操作，在初始化数组时使用了乐观锁CAS操作来决定到底是哪个线程有资格进行初始化，其他线程均只能等待。

用到的并发技巧：

- volatile变量（sizeCtl）：它是一个标记位，用来告诉其他线程这个坑位有没有人在，其线程间的可见性由volatile保证。
- CAS操作：CAS操作保证了设置sizeCtl标记位的原子性，保证了只有一个线程能设置成功

## put操作的线程安全

由于其减小了锁的粒度，若Hash完美不冲突的情况下，可同时支持n个线程同时put操作，n为Node数组大小，在默认大小16下，可以支持最大同时16个线程无竞争同时操作且线程安全。当hash冲突严重时，Node链表越来越长，将导致严重的锁竞争，此时会进行扩容，将Node进行再散列，下面会介绍扩容的线程安全性。总结一下用到的并发技巧：

- 减小锁粒度：将Node链表的头节点作为锁，若在默认大小16情况下，将有16把锁，大大减小了锁竞争（上下文切换），就像开头所说，将串行的部分最大化缩小，在理想情况下线程的put操作都为并行操作。同时直接锁住头节点，保证了线程安全
- Unsafe的getObjectVolatile方法：此方法确保获取到的值为最新。

## 扩容操作的线程安全

ConcurrentHashMap运用各类CAS操作，将扩容操作的并发性能实现最大化，在扩容过程中，就算有线程调用get查询方法，也可以安全的查询数据，若有线程进行put操作，还会协助扩容，利用sizeCtl标记位和各种volatile变量进行CAS操作达到多线程之间的通信、协助，在迁移过程中只锁一个Node节点，即保证了线程安全，又提高了并发性能。

## 计数中用到的并发技巧

- 利用CAS递增baseCount值来感知是否存在线程竞争，若竞争不大直接CAS递增baseCount值即可，性能与直接baseCount++差别不大。
- 若存在线程竞争，则初始化计数桶，若此时初始化计数桶的过程中也存在竞争，多个线程同时初始化计数桶，则没有抢到初始化资格的线程直接尝试CAS递增baseCount值的方式完成计数，最大化利用了线程的并行。此时使用计数桶计数，分而治之的方式来计数，此时两个计数桶最大可提供两个线程同时计数，同时使用CAS操作来感知线程竞争，若两个桶情况下CAS操作还是频繁失败（失败3次），则直接扩容计数桶，变为4个计数桶，支持最大同时4个线程并发计数，以此类推…同时使用位运算和随机数的方式"负载均衡"一样的将线程计数请求接近均匀的落在各个计数桶中。