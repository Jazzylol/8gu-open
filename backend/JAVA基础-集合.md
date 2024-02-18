##### 集合类图

![](https://pdai.tech/images/java_collections_overview.png)



![](https://images2015.cnblogs.com/blog/818454/201610/818454-20161010004545490-287699251.jpg)

##### List

ArrayList  数组结构，可自动扩容，扩容容量是1.5倍，遍历迅速，增删慢点，线程不安全

Vector：和arrayList类似，但是线程安全，主要是直接加锁的

LinkedList：双向链表实现，zhi可以顺序访问，但是可以快速插入删除元素

##### Set

TreeSet：可以排序 + 去重的Set 红黑树实现，查询时间复杂度  O(logn) ，支持有序性的操作。

HashSet：去重，基于hash表实现，底层用的HashMap

LinkedHashSet ： 底层是LinkedHashMap包装实现，底层双向链表

##### Queue

Queue：队列顶层接口

ArrayQueue：数组实现，现成非安全，队列

PriorityQueue：基于优先级堆的无界优先级队列，实现不是同步，线程不安全。队列元素按照自然顺序排序，或者按照指定Comparator排序，禁止null元素，元素必须都要具备可比较性。

ConcurrentLinkedQueue：无界 非阻塞 线程安全队列，在多线程下的首选队列，实现比较精巧，可重点阅读源码，按照FIFO原则存储元素。

##### Deque

接口，继承Queue接口，double ended queue 的缩写，表示双端队列，支持在两端插入和移除元素。没有严格要求禁止插入null元素，但是最好不要插入null元素，因为null是用来作为特殊值来指示双端队列为空

ArrayDeque：Deque接口的数组实现，数组实现的双端队列，没有容量限制。线程非安全，不支持多个线程并发访问，禁止null元素，次类用作堆栈比`Stack`快，在用作队列的时候比`LinkedList`快

LinkedBlockingDeque：基于已链接节点的、任选范围的阻塞双端队列。构造器可选范围，防止过度膨胀，如果未指定容量则为`Integer.MAX_VALUE`

##### BlockingQueue

阻塞队列接口

ArrayBlockingQueue：BlockingQueue的数组实现，有界。按照FIFO对元素进行排序，头部是队列中存在时间最长的元素。一旦创建就不可以改编队列大小，队列支持对线程的进行公平性排序的可选公平策略，默认是非公平，公平策略会降低吞吐量。

LinkedBlockingQueue：BlockingQueue的链表实现。可指定容量，按照FIFO顺序对元素进行排序，头部是在队列中时间最长的元素，吞吐量一般高于基于数组的队列。

PriorityBlockingQueue：无界阻塞队列，队列禁止null元素，线程安全，PriorityQueue阻塞版本，需要对象具有可比较性。

SynchronousQueue：无界无缓冲等待队列，isEmpty()方法一直返回true，生产者线程对其插入操作put必须等待消费者的移除操作take，反过来一样。队列头元素是第一个排队要插入数据的**线程**，而不是要交换的数据。数据是在配对的生产者和消费者线程之间传递的，并不会将数据缓冲到队列中。可以理解为生产者和消费者等待对方，握手然后一起离开。

DelayQueue：无界阻塞队列，BlockingQueue的实现。内部元素必须是先`Delayed`接口，只有延迟期满才能从队列中提取元素。队列头部是延迟期满保存最久的元素，禁止null元素。（推荐查看）



##### BlockingDeque

双端阻塞队列，接口

LinkedBlockingDeque：基于已链接节点的、任选范围的阻塞双端队列。构造器可选范围，防止过度膨胀，如果未指定容量则为`Integer.MAX_VALUE` 

##### Map

顶层接口 map

Dictionary：抽象类，意义被Map取代，HashTable的父类。

HashTable：线程安全类 KV结构，禁止null键值。

Properties：Properties 类表示了一个持久的属性集。Properties 可保存在流中或从流中加载。属性列表中每个键及其对应值都是一个字符串。

SortedMap：提供关于键排序的Map，映射是根据键的自然顺序排序，或者使用指定的Comparator，所有的键都必须是可比较的。

TreeMap：SortedMap 实现类，TreeSet的内部实现类，基于红黑树实现，可以对键进行排序的Map，可用于实现一致性哈希算法。

HashMap：高效版本HashTable,线程不安全，多线程的时候，put方法容易造成循环链表导致get的时候死循环。允许null键值。

ConcurrentHashMap:线程安全版本的HashMap,支持高并发，主要原因锁分离，减少请求锁的频率，减少持有锁的时间，读写分离，用volitile保持线程之间的可见性。

LinkedHashMapHashMap的子类，在HashMap的基础上保留了插入顺序，在迭代的时候以插入顺序进行迭代。实现原理是在HashMap保存元素的基础上，重新定义了HashMap中的Entry，不仅保存当前对象的引用还持有有before和after两个元素的引用，从而形成了双向链表。

WeakHashMap：以弱键实现的基于哈希表的map。map中的键不再使用的时候将自动移除，更精确地说，对于一个给定的键，其映射的存在并不阻止垃圾回收器对该键的回收，通常我们使用短时间内就过期的缓存时最好使用weakHashMap。



