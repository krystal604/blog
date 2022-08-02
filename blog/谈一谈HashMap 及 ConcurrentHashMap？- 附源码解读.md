HashMap
一、HasMap概括
HashMap是一种采取K-V结构的高校存储的数据结构，通过hash算法使得其最优复杂度在O(1)。
HashMap的基本数据结构在jdk1.7是数组+链表，jdk1.8后为数组加链表加红黑树。

二、通过源码解读HashMap基本数据结构
要想了解HashMap的基本实现，那我们先看HashMap类中的静态常量。

	//初始容量
	static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
	//最大容量 
    static final int MAXIMUM_CAPACITY = 1 << 30;
	//扩容因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
	//转化为红黑树的最小链表长度
    static final int TREEIFY_THRESHOLD = 8;
	//红黑树退化为链表的最小长度
    static final int UNTREEIFY_THRESHOLD = 6;
	//转化为红黑树的最小数组容量
    static final int MIN_TREEIFY_CAPACITY = 64;

HashMap数组初始化容量为16
扩容因子为0.75，即数组被填满75%会进行扩容
当链表长度大于8数组容量大于64 链表会转化为红黑树
当红黑树结点少于6会退化为链表
看完常量我们再来开put方法
put方法是比较能全面的理解HashMap数据结构的
先看put方法

public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

很多人可能都不知道忽视了 put方法还是有返回值的，当一个Key的值被更新时他的就val值会被返回。
put方法看起来很简单就是直接调用了putVal方法，putVal方法后面的两个参数对初级面试来说意义不大 就是控制是否返回旧值的。

那我们继续看putVal方法

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
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
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }


这里就是整个put流程的核心了，也狠生动的解读了数组加链表加红黑树。
putVal朴实无华，没有用到什么设计模式，一把if梭哈到低。那就按照分支结构从上到下解读一下。

上来第一个判断就是table是否为空 这个table是什么
transient Node<K,V>[] table;
1
这个table就是一个全局变量，他是一个Node数组，这就是我们常说的HashMap中的数组
来看一下Node

static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;  
    }
1
2
3
4
5
6
我们可以看到Node是一个静态内部类 (我这里只保留了内部变量减少篇幅）除了他的hash值K、V以外 还有一个next指针，所以说我们的HashMap中的链表是单链表。

回到putVal方法中
if ((tab = table) == null || (n = tab.length) == 0)
n = (tab = resize()).length;
当数组为空的时候会调用resize方法对数组初始化，所以说hashmap的数组是懒加载机制

往下说putVal的方法
if ((p = tab[i = (n - 1) & hash]) == null)
这句很简单就是此时数组对应的这个位置如果为空的话，直接在上面新建一个Node，跟链表和红黑树显然就没什么关系了；
否则的话那就要跟链表和数组扯上关系了。

下面是一个三向的分支 展开说
如果信加入的这个结点就是当前数组index的头结点，那么直接替换且返回就好了
第二个分支用了一个关键字instanceof他是用来判断是不是同一个类型的，就是判断我们当前数组index是否已经进化成为红黑树了，如果是那么就调用红黑树的那套逻辑（不多说了，就是左旋右旋什么的）
第三个分支就只用一种情况了，那么就是当前是一个链表，这里写的很简练但核心意思一样就是便利这个链表，他们的hash一定都是一样的，要不断判断equals，如果equals相等则替换返回，否则加在链表尾端；
说完了这些，可能发现下面还有一个if (e != null) { // existing mapping for key
这个就是控制如果被替换了要不要返回旧值的

最后通过对putVal的解读，已经基本讲清楚了为什么以及什么是 链表加数组加红黑树了

同时这里还会引申出一个问题,HashMap是线程安全的嘛？
答案是不安全的。
原因很简单，不多赘述了，在插入新值时 寻找对应位置和向对应位置加入新节点这个操作 不是原子操作，如果并发的两个线程都找到了同一个位置且都向这一个位置添加新值，那么就会发生覆盖！
如果要保证线程安全，可以使用ConcurrentHashMap 或 HashTable；后面会详细说ConcurrentHashMap 。

顺便也简单说一下resize方法吧
HashMap扩容是二倍扩容 ， 会新建一个数组在resize方法中有一个for循环里面嵌套了一个do-while循环，for循环就是便利我们的数组，do-while遍历下面的链表，将所有元素都迁移至新的table中。

ConcurrentHashMap
ConcurrentHashMap概括
ConcurrentHashMap是线程安全的HashMap，同时HashTable也是一个线程安全的HashMap，但是HashTable是一个遗留类，HashTable锁的是全表所以并发度只用1。但是ConcurrentHashMap在1.8中锁的是链表头，所以理论上数组有多大并发度就有多大。

ConcurrentHashMap如何保重线程安全
先说putVal方法
附上源码有点多，不想看直接看解释也行。

final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
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

流程上与 HashMap区别并不大
当对应数组index为空时，会用CAS进行加锁
当对应数组index为链表或红黑树时，会直接使用synchronized锁头

在这里要提 jdk1.7是使用的分段锁
jdk1.7是使用大表套小表ConcurrentHashMap下最多有16个 Segment，
锁的是 Segment ， 所以最大并发量只有16.

static class Segment<K,V> extends ReentrantLock implements Serializable
1
这里可以看到 Segment 继承了ReentrantLock 所以锁是使用到的ReentrantLock

说完putVal再说resize
ConcurrentHashMap的resize方法是支持多个线程共同协助完成的，使用到的也是CAS锁
