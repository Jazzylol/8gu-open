##### 为什么需要多线程

因为要提高系统资源利用率

##### 多线程出问题的本质

多线程出问题的本质是 操作共享变量

##### 线程实现安全的方法

1. 互斥同步  比如各种reentrantlock   sync等互斥锁
2. 非阻塞同步  比如cas，各种原子类，  ABA可以使用版本号解决
3. 线程私有，比如 threadlocal

##### JAVA是如何解决并发的

主要是靠JMM，JMM又成为JAVA内存模型，核心就是 JMM规范了JVM如何提供一些禁用cpu缓存以及编译优化来避免并发问题。这些问题主要两个角度

角度1

1. Volatile  ，synchronized  和 final
2. 规定了 happen before规则

角度2

JMM规定一系列规范，解决了并发的 可见性，原子性，有序性。

原子性：java中保证了 数字复制给变量 是原子性的，++  这种不是原子的

可见性：java中提供了 volatile 关键词来保证可见性。当一个共享变量被修改后，它会保证修改的值会立刻被更新到内存中。

有序性：在java中 提供了 volatile  来保证一定的有序性，更重要的是 提供了 sync 和 lock来保证了有序性，很显然 sync和lock保证了 包含的代码的有序性，另外再JMM层面的 happen before规则来保证有序性

##### synchronized实现原理

sync修饰静态方法，锁对象为类的class对象，修饰成员方法，锁对象为 this，也可以指定锁对象。

sync优点是 会自动释放锁，不需要手动释放，抛出异常也可会自动释放，但是无法设置超时时间

实现方式：

基于底层 monitor enter 和 monitor exit指令实现，当获取到锁的时候  进入monitor enter，然后释放锁的时候 monitor  exit。

可重入原理：

monitor监视器 每次已获取锁的线程再次申请锁的时候 会把monitor+1  等待退出的时候 monitor -1  ，monitor计数器=0的时候 锁释放

Happen before规则

对于一个object 的 解锁一定 happen  before于 object的加锁



##### Volatile的作用是什么 ？ 

volatile 主要作用是

1. 防止重排序  比如new Object  正常顺序是 分配内存空间。初始化对象。将内存空间的地址赋值给对应的引用。volatile可以禁止这几个操作重新排序
2. 实现可见性 ，两个线程高速去修改共享变量，加了volatile可以保障更准确的修改被立刻发现
3. 保证变量的单次原子性 ，共享的long和double变量的为什么要用volatile? 因为long和double两种数据类型的操作可分为高32位和低32位两部分，因此普通的long或double类型读/写可能不是原子的。因此，鼓励大家将共享的long和double变量设置为volatile类型，这样能保证任何情况下对long和double的单次读/写操作都具有原子性。目前各种平台下的商用虚拟机都选择把 64 位数据的读写操作作为原子操作来对待，因此我们在编写代码时一般不把long 和 double 变量专门声明为 volatile多数情况下也是不会错的。

##### volatile实现原理

volatile 变量的内存可见性是基于内存屏障(Memory Barrier)实现，内存屏障，又称内存栅栏，是一个 CPU 指令。在程序运行时，为了提高执行性能，编译器和处理器会对指令进行重排序，JMM 为了保证在不同的编译器和 CPU 上有相同的结果，通过插入特定类型的内存屏障来禁止+ 特定类型的编译器重排序和处理器重排序，插入一条内存屏障会告诉编译器和 CPU：不管什么指令都不能和这条 Memory Barrier 指令重排序。

在反编译代码中可以看到  volatile 有 lock 前缀的指令在多核处理器下会引发两件事情:

- 将当前处理器缓存行的数据写回到系统内存。
- 写回内存的操作会使在其他 CPU 里缓存了该内存地址的数据无效。



##### JAVA中的锁简单介绍下 ？

java中提供了非常多的锁，因为在不同的场景下有不同的使用方法，展现出比较高的效率。java中通常是按照是否含有某个特性来定义分类各种锁。

举例来说 

###### 线程是否锁住资源

1. 悲观锁
2. 乐观锁

###### 锁住同步资源失败，是否阻塞

1. 阻塞
2. 不阻塞  自旋锁

###### 多个线程竞争同步资源的时候有没有细节区别（针对 sync锁的，JVM慢慢演进的 sync锁优化

1. 不锁住资源，其他线程失败会重试   无锁

2. 同一个线程，执行同步资源代码的时候 是否自动获取资源  偏向锁

3. 多个线程竞争资源的时候，没有获取到资源的线程，自选等待锁   轻量级锁

4. 多个线程竞争资源的时候，没获取到资源的线程，阻塞等待唤醒

###### 多个线程竞争的时候是否要排队

1. 排队，公平锁
2. 不排队，先插队，再排队，非公平锁

###### 一个线程在一个同步流程中是否可以多次获取同一把锁

1. 可以，可重入锁
2. 不可以  不可重入锁

###### 多个线程是否可以共享同一把锁

1. 可以，共享锁
2. 不可以 排他锁

##### 线程池原理说下

```JAVA
public  ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler)
```

corePoolSize：核心线程数量，创建任务的时候，没到corePoolSize 则直接创建线程执行任务

maximumPoolSize：最大线程数量，如果是无界队列则这个参数无效，因为到达 corePoolSize 后续再有任务会丢到队列里。

keepAliveTime：线程任务执行完以后，存活时间。只有 大于 corePoolSize 才会回收

workQueue用来保存等待被执行的任务的阻塞队列. 在JDK中提供了如下阻塞队列: 

- `ArrayBlockingQueue`: 基于数组结构的有界阻塞队列，按FIFO排序任务；
- `LinkedBlockingQueue`: 基于链表结构的阻塞队列，按FIFO排序任务，吞吐量通常要高于ArrayBlockingQueue；
- `SynchronousQueue`: 一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue；
- `PriorityBlockingQueue`: 具有优先级的无界阻塞队列；

`threadFactory `创建线程的工厂，通过自定义的线程工厂可以给每个新建的线程设置一个具有识别度的线程名。默认为DefaultThreadFactory

`handler ` 线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4种策略:

- `AbortPolicy`: 直接抛出异常，默认策略；
- `CallerRunsPolicy`: 用调用者所在的线程来执行任务；
- `DiscardOldestPolicy`: 丢弃阻塞队列中靠最前的任务，并执行当前任务；
- `DiscardPolicy`: 直接丢弃任务

##### Executors 提供了几种工具类？为啥一定要new

###### newFixedThreadPool

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>());
}
```

线程池的线程数量达corePoolSize后，即使线程池没有可执行任务时，也不会释放线程。

FixedThreadPool的工作队列为无界队列LinkedBlockingQueue(队列容量为Integer.MAX_VALUE), 这会导致以下问题:

- 线程池里的线程数量不超过corePoolSize,这导致了maximumPoolSize和keepAliveTime将会是个无用参数
- 由于使用了无界队列, 所以FixedThreadPool永远不会拒绝, 即饱和策略失效

###### newSingleThreadExecutor

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

初始化的线程池中只有一个线程，如果该线程异常结束，会重新创建一个新的线程继续执行任务，唯一的线程可以保证所提交任务的顺序执行.

由于使用了无界队列, 所以SingleThreadPool永远不会拒绝, 即饱和策略失效

###### newCachedThreadPool

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```

线程池的线程数可达到Integer.MAX_VALUE，即2147483647，内部使用SynchronousQueue作为阻塞队列； 和newFixedThreadPool创建的线程池不同，newCachedThreadPool在没有任务执行时，当线程的空闲时间超过keepAliveTime，会自动释放线程资源，当提交新任务时，如果没有空闲线程，则创建新线程执行任务，会导致一定的系统开销； 执行过程与前两种稍微不同:

- 主线程调用SynchronousQueue的offer()方法放入task, 倘若此时线程池中有空闲的线程尝试读取 SynchronousQueue的task, 即调用了SynchronousQueue的poll(), 那么主线程将该task交给空闲线程. 否则执行(2)
- 当线程池为空或者没有空闲的线程, 则创建新的线程执行任务.
- 执行完任务的线程倘若在60s内仍空闲, 则会被终止. 因此长时间空闲的CachedThreadPool不会持有任何线程资源.

线程池 shutdown shutdownNow区别

1. shutdown正在执行的任务 不会中断，shutdown now 所有线程全部中断
2. shutdownNow会把 未执行任务返回

##### JUC工具类

CAS 

UNSAFE

LockSupport

AQS

ReentrantLock

ReentrantReadWriteLock

ConcurrentHashMap

CopyOnWriteArrayList

ConcurrentLinkedQueue

BlockingQueue

ScheduledThreadPoolExecutor

Fork/Join

CountDownLatch

CyclicBarrier

ThreadLocal





