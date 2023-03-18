<div align="center">
    <h1>
        Java八股整理
    </h1>
</div>

## Thread

### 线程暂停的方法

sleep、yield、join、stop、interrupt

Sleep，会导致**正在执行的线程**进入**阻塞状态TIME_WAITING**

yield：会导致**正在执行的线程**进入**就绪状态RUNNABLE**，**当前正在执行的线程**暂停一次，允许其他线程执行，不阻塞，线程进入就绪状态，如果没有其他等待执行的线程，这个时候当前线程就会马上恢复执行。

Join，会导致**调用它的线程**进入**阻塞状态TIME_WAITING**，而**调用此方法的线程**强制执行，也就是变成**运行状态**

stop：强迫线程停止执行，**已过时**。不推荐使用。

interrupt：停止一个线程执行，但允许该线程料理一下自己的后事。



### sleep和wait的区别

对于sleep()方法，我们首先要知道该方法是属于**Thread类**中的。而wait()方法，则是属于**Object类**中的。

sleep()方法导致了程序暂停执行指定的时间，让出CPU给其他线程，但是他的监控状态依然保持者，当指定的时间到了又会自动恢复运行状态。

在调用sleep()方法的过程中，线程**不会释放对象锁**。

而当调用wait()方法的时候，线程**会放弃对象锁**，进入等待此对象的等待锁定池，只有针对此对象**调用notify()**方法后本线程才进入对象锁定池准备

从使用角度看，sleep是Thread线程类的方法，而wait是Object顶级类的方法。

sleep可以在任何地方使用，而wait只能在同步方法或者同步块中使用。

![Sleep VS Wait](https://img-blog.csdnimg.cn/2019050816141738.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDA5MDE1,size_16,color_FFFFFF,t_70)



### ThreadLocal

ThreadLocal 顾名思义“线程本地变量”，对应到 Java 代码就是线程私有变量，可以把它理解为对于同一个变量，在不同的线程包含不同的副本，并且各个副本之间相互独立

```java
public class UserHolder {
    private static final ThreadLocal<UserDTO> tl = new ThreadLocal<>();

    public static void saveUser(UserDTO user){
        tl.set(user);
    }

    public static UserDTO getUser(){
        return tl.get();
    }

    public static void removeUser(){
        tl.remove();
    }
}
```

ThreadLocal 的使用主要基于 **get()、set() 方法**.这里首先获取当前线程对象，根据线程对象获取 ThreadLocalMap 对象，之后以 this 为 key，value 为值，存入 map。这里 this 指代当前 ThreadLocal 对象

```java
public void set(T value) {
	// 获取调用 set() 方法的线程
    Thread t = Thread.currentThread();
    // 获取一个 map
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue()
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```



### 线程池

线程池的优点

+ 降低系统资源消耗，通过**重用已存在的线程**，降低线程创建和销毁造成的消耗；
+ 提高系统响应速度，当有任务到达时，无需等待新线程的创建便能立即执行；
+ 方便线程**并发数的管控**，线程若是无限制的创建，不仅会额外消耗大量系统资源，更是占用过多资源而阻塞系统或oom等状况，从而降低系统的稳定性。线程池能有效管控线程，统一分配、调优，提供资源使用率；
+ 更强大的功能，线程池提供了**定时**、定期以及可控线程数等功能的线程池，使用方便简单。

Java中的**四种**线程池

+ **newCachedThreadPool**：
  创建一个**可缓存的无界线程池**，如果线程池长度超过处理需要，可灵活回收空线程，若无可回收，则新建线程。当线程池中的线程空闲时间超过60s，则会自动回收该线程，当任务超过线程池的线程数则创建新的线程，线程池的大小上限为**Integer.MAX_VALUE**,可看作无限大。

  ```java
  /**
   * 可缓存无界线程池测试
   * 当线程池中的线程空闲时间超过60s则会自动回收该线程，核心线程数为0
   * 当任务超过线程池的线程数则创建新线程。线程池的大小上限为Integer.MAX_VALUE，
   * 可看做是无限大。
   */
  @Test
  public void cacheThreadPoolTest() {
      // 创建可缓存的无界线程池，可以指定线程工厂，也可以不指定线程工厂
      ExecutorService executorService = Executors.newCachedThreadPool(new testThreadPoolFactory("cachedThread"));
      for (int i = 0; i < 10; i++) {
          executorService.submit(() -> {
              print("cachedThreadPool");
              System.out.println(Thread.currentThread().getName());
          }
          );
      }
  }
  ```

+ **newFixedThreadPool**：

  创建一个指定大小的线程池，可控制线程的最大并发数，超出的线程会在LinkedBlockingQueue阻塞队列中等待

  一般指定线程池大小为CPU线程整数倍

+ **newScheduledThreadPool：**

  创建一个定长的线程池，可以指定线程池核心线程数，支持定时及周期性任务的执行

  ```java
  /**
   * 创建定时周期执行的线程池测试
   *
   * schedule(Runnable command, long delay, TimeUnit unit)，延迟一定时间后执行Runnable任务；
   * schedule(Callable callable, long delay, TimeUnit unit)，延迟一定时间后执行Callable任务；
   * scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)，延迟一定时间后，以间隔period时间的频率周期性地执行任务；
   * scheduleWithFixedDelay(Runnable command, long initialDelay, long delay,TimeUnit unit)，与scheduleAtFixedRate()方法很类似，
   * 但是不同的是scheduleWithFixedDelay()方法的周期时间间隔是以上一个任务执行结束到下一个任务开始执行的间隔，而scheduleAtFixedRate()方法的周期时间间隔是以上一个任务开始执行到下一个任务开始执行的间隔，
   * 也就是这一些任务系列的触发时间都是可预知的。
   * ScheduledExecutorService功能强大，对于定时执行的任务，建议多采用该方法。
   *
   * 作者：张老梦
   * 链接：https://www.jianshu.com/p/9ce35af9100e
   * 来源：简书
   * 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
   */
  @Test
  public void scheduleThreadPoolTest() {
      // 创建指定核心线程数，但最大线程数是Integer.MAX_VALUE的可定时执行或周期执行任务的线程池
      ScheduledExecutorService executorService = Executors.newScheduledThreadPool(5, new testThreadPoolFactory("scheduledThread"));
  
      // 定时执行一次的任务，延迟1s后执行
      executorService.schedule(new Runnable() {
          @Override
          public void run() {
              print("scheduleThreadPool");
              System.out.println(Thread.currentThread().getName() + ", delay 1s");
          }
      }, 1, TimeUnit.SECONDS);
  
  
      // 周期性地执行任务，延迟2s后，每3s一次地周期性执行任务
      executorService.scheduleAtFixedRate(new Runnable() {
          @Override
          public void run() {
              System.out.println(Thread.currentThread().getName() + ", every 3s");
          }
      }, 2, 3, TimeUnit.SECONDS);
  
  
      executorService.scheduleWithFixedDelay(new Runnable() {
          @Override
          public void run() {
              long start = new Date().getTime();
              System.out.println("scheduleWithFixedDelay 开始执行时间:" +
                      DateFormat.getTimeInstance().format(new Date()));
              try {
                  Thread.sleep(5000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
              long end = new Date().getTime();
              System.out.println("scheduleWithFixedDelay执行花费时间=" + (end - start) / 1000 + "m");
              System.out.println("scheduleWithFixedDelay执行完成时间："
                      + DateFormat.getTimeInstance().format(new Date()));
              System.out.println("======================================");
          }
      }, 1, 2, TimeUnit.SECONDS);
  
  }
  ```

+ **newSingleThreadExecutor**：

  创建一个单线程化的线程池，它只有一个线程，用仅有的一个线程来执行任务，保证所有的任务按照指定顺序（FIFO，LIFO，优先级）执行，所有的任务都保存在队列LinkedBlockingQueue中，等待唯一的单线程来执行任务。

| 工厂方法                | corePoolSize | maximumPoolSize   | keepAliveTime | workQueue           |
| ----------------------- | ------------ | ----------------- | ------------- | ------------------- |
| newCachedThreadPool     | 0            | Integer.MAX_VALUE | 60s           | SynchronousQueue    |
| newFixedThreadPool      | nThreads     | nThreads          | 0             | LinkedBlockingQueue |
| newSingleThreadExecutor | 1            | 1                 | 0             | LinkedBlockingQueue |
| newScheduledThreadPool  | corePoolSize | Integer.MAX_VALUE | 0             | DelayedWorkQueue    |



## HashMap

### HashMap的扩容机制

- capacity 即容量，默认16。
- loadFactor 加载因子，默认是0.75
  - 底层是因为泊松分布 我实在是 完全不懂 可以看[从泊松分布谈起HashMap为什么默认扩容因子是0.75 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/396019103)
  - 如果加载因子过小，那么扩容门槛低，扩容频繁，这虽然能使元素存储得更稀疏，有效避免了哈希冲突发生，同时操作性能较高，但是会占用更多的空间。
  - 如果加载因子过大，那么扩容门槛高，扩容不频繁，虽然占用的空间降低了，但是这会导致元素存储密集，发生哈希冲突的概率大大提高，从而导致存储元素的数据结构更加复杂（用于解决哈希冲突），最终导致操作性能降低。
  - 还有一个因素是为了提升扩容效率。因为`HashMap`的容量（`size`属性，构造函数中的`initialCapacity`变量）有一个要求：它一定是2的幂。所以加载因子选择了0.75就可以保证它与容量的乘积为整数。
- threshold 阈值。阈值=容量*加载因子。默认12。当元素数量超过阈值时便会触发扩容

#### JDK7中的扩容机制

JDK7的扩容机制相对简单，有以下特性：

- 空参数的构造函数：以默认容量、默认负载因子、默认阈值初始化数组。内部数组是**空数组**。
- 有参构造函数：根据参数确定容量、负载因子、阈值等。
- 第一次put时会初始化数组，其容量变为**不小于指定容量的2的幂数**。然后根据负载因子确定阈值。
- 如果不是第一次扩容，则 **新容量=旧容量×2** ， **负载因子新阈值=新容量×负载因子** 。

#### JDK8的扩容机制

JDK8的扩容做了许多调整。

HashMap的容量变化通常存在以下几种情况：

1. 空参数的构造函数：实例化的HashMap默认内部数组是null，即没有实例化。第一次调用put方法时，则会开始第一次初始化扩容，长度为16。
2. 有参构造函数：用于指定容量。会根据指定的正整数找到**不小于指定容量的2的幂数**，将这个数设置赋值给**阈值**（threshold）。第一次调用put方法时，会将阈值赋值给容量，然后让 阈值容量负载因子阈值=容量×负载因子 。（因此并不是我们手动指定了容量就一定不会触发扩容，超过阈值后一样会扩容！！）
3. 如果不是第一次扩容，则容量变为原来的2倍，**阈值也变为原来的2倍**。*（容量和阈值都变为原来的2倍时，负载因子还是不变）*

此外还有几个细节需要注意：

- 首次put时，先会触发扩容（算是初始化），然后存入数据，然后判断是否需要扩容；
- 不是首次put，则不再初始化，直接存入数据，然后判断是否需要扩容；

由于数组的容量是以2的幂次方扩容的，那么一个Entity在扩容时，新的位置要么在**原位置**，要么在**原长度+原位置**的位置。



### HashMap为什么线程不安全

首先HashMap是**线程不安全**的，其主要体现：

1.在jdk1.7中，在多线程环境下，扩容时会造成环形链或数据丢失。

2.在jdk1.8中，在多线程环境下，会发生数据覆盖的情况。

https://cloud.tencent.com/developer/article/1631902

JDK7

```java
 1    void transfer(Entry[] newTable, boolean rehash) {
 2         int newCapacity = newTable.length;
 3         for (Entry<K,V> e : table) {
 4             while(null != e) {
 5                 Entry<K,V> next = e.next;
 6                 if (rehash) {
 7                     e.hash = null == e.key ? 0 : hash(e.key);
 8                 }
 9                 int i = indexFor(e.hash, newCapacity);
10                 e.next = newTable[i];
11                 newTable[i] = e;
12                 e = next;
13             }
14         }
15     }
```

在对table进行扩容到newTable后，需要将原来数据转移到newTable中，注意10-12行代码，这里可以看出在转移元素的过程中，使用的是头插法，也就是链表的顺序会翻转，这里也是形成死循环的关键点。

当一个线程阻塞在11行而另一个线程完成重hash之后会形成环形链表

JDK8

在jdk1.8中对HashMap进行了优化，在发生hash碰撞，不再采用头插法方式，而是直接插入链表尾部，因此不会出现环形链表的情况，但是在多线程的情况下仍然不安全，这里我们看jdk1.8中HashMap的put操作源码：

```java
 1  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
 2                    boolean evict) {
 3         Node<K,V>[] tab; Node<K,V> p; int n, i;
 4         if ((tab = table) == null || (n = tab.length) == 0)
 5             n = (tab = resize()).length;
 6         if ((p = tab[i = (n - 1) & hash]) == null) // 如果没有hash碰撞则直接插入元素
 7             tab[i] = newNode(hash, key, value, null);
 8         else {
 9             Node<K,V> e; K k;
10             if (p.hash == hash &&
11                 ((k = p.key) == key || (key != null && key.equals(k))))
12                 e = p;
13             else if (p instanceof TreeNode)
14                 e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
15             else {
16                 for (int binCount = 0; ; ++binCount) {
17                     if ((e = p.next) == null) {
18                         p.next = newNode(hash, key, value, null);
19                         if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
20                             treeifyBin(tab, hash);
21                         break;
22                     }
23                     if (e.hash == hash &&
24                         ((k = e.key) == key || (key != null && key.equals(k))))
25                         break;
26                     p = e;
27                 }
28             }
29             if (e != null) { // existing mapping for key
30                 V oldValue = e.value;
31                 if (!onlyIfAbsent || oldValue == null)
32                     e.value = value;
33                 afterNodeAccess(e);
34                 return oldValue;
35             }
36         }
37         ++modCount;
38         if (++size > threshold)
39             resize();
40         afterNodeInsertion(evict);
41         return null;
42 }
```

这是jdk1.8中HashMap中put操作的主函数， 注意第6行代码，如果没有hash碰撞则会直接插入元素。**如果线程A和线程B同时进行put操作，刚好这两条不同的数据hash值一样，并且该位置数据为null，所以这线程A、B都会进入第6行代码中。**假设一种情况，线程A进入后还未进行数据插入时挂起，而线程B正常执行，从而正常插入数据，然后线程A获取CPU时间片，此时线程A不用再进行hash判断了，问题出现：线程A会把线程B插入的数据给**覆盖**，发生线程不安全。

这里只是简要分析下jdk1.8中HashMap出现的线程不安全问题的体现，后续将会对java的集合框架进行总结，到时再进行具体分析。



### ConcurrentHashMap的底层原理

[ConcurrentHashMap底层结构和原理详解](https://blog.csdn.net/qq_45408390/article/details/122189726)



