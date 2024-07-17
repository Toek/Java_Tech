# 线程池的三种队列区别：SynchronousQueue、LinkedBlockingQueue 和ArrayBlockingQueue

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

线程池的几个核心参数讲解【文心一言】

线程池是一种基于池化技术管理线程的工具，它能够提高系统的响应速度，减少线程创建和销毁的开销，并控制系统中并发线程的数量。线程池的核心参数主要包括以下几个：

### 1. **corePoolSize（核心线程数）**

- **定义**：线程池的基本大小，即即使没有空闲线程，在创建新线程前，线程池也会尝试复用已有的空闲线程。

- 特点

  ：

  - 核心线程会一直存在，即使没有任务执行。
  - 当线程数小于核心线程数时，即使有空闲线程，线程池也会创建新线程来执行任务，直到线程数达到核心线程数。
  - 除非设置了`allowCoreThreadTimeOut`（且其值为true），否则核心线程不会因为空闲而被销毁。

### 2. **maximumPoolSize（最大线程数）**

- **定义**：线程池允许创建的最大线程数。

- 特点

  ：

  - 当工作队列已满，且已创建的线程数小于最大线程数时，线程池会再创建新的线程来执行任务。
  - 线程池中的线程数量不会超过这个值。

### 3. **keepAliveTime（线程存活时间）**

- **定义**：线程池中的非核心线程闲置后，保持存活的时间。

- 特点

  ：

  - 如果在这段时间内，线程没有被复用，那么线程池会销毁这些线程（直到线程数等于核心线程数，如果`allowCoreThreadTimeOut`为true，则直到线程数等于零）。
  - 这个参数仅对非核心线程有效。

### 4. **unit（时间单位）**

- **定义**：`keepAliveTime`的时间单位。

- 常见单位

  ：

  - `TimeUnit.SECONDS`（秒）
  - `TimeUnit.MILLISECONDS`（毫秒）

### 5. **workQueue（工作队列）**

- **定义**：用于存放待执行的任务的队列。

- 常见队列类型

  ：

  - `ArrayBlockingQueue`：基于数组的有界阻塞队列，按FIFO（先进先出）原则对元素进行排序。
  - `LinkedBlockingQueue`：基于链表的无界阻塞队列（虽然说是无界，但实际上会受到JVM可用内存的限制）。
  - `SynchronousQueue`：一个不缓存任务的阻塞队列。每个插入操作必须等到另一个线程调用移除操作才可插入，否则插入一直阻塞。
  - `PriorityBlockingQueue`：一个具有优先级的无限阻塞队列。

### 6. **ThreadFactory（线程工厂）**

- **定义**：用于设置新创建线程的工厂，创建新线程的时候会用到。
- **用途**：可以通过线程工厂给每个创建出来的线程设置更有意义的名字、设置是否为守护线程等。

### 7. **Handler（拒绝处理策略）**

- **定义**：当队列和线程池都满了时，用于处理新提交任务的策略。

- 常见策略

  ：

  - `AbortPolicy`：直接抛出异常。
  - `CallerRunsPolicy`：在调用者所在线程来执行任务。
  - `DiscardPolicy`：不处理，直接丢弃。
  - `DiscardOldestPolicy`：丢弃队列里最靠前最近的任务，并执行当前任务。

### 总结

线程池的核心参数共同协作，确保线程池能够有效地管理线程资源，提高系统的并发处理能力。通过合理配置这些参数，可以优化线程池的性能，满足不同的应用场景需求。







置顶 [无名后生](https://me.csdn.net/qq_26881739) 2018-07-10 11:39:28 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 25123 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 34

分类专栏： [java](https://blog.csdn.net/qq_26881739/category_7416135.html) [多线程](https://blog.csdn.net/qq_26881739/category_7581145.html) 文章标签： [线程池](https://www.csdn.net/gather_22/MtTaEg0sMTEwMjYtYmxvZwO0O0OO0O0O.html)[多线程](https://www.csdn.net/gather_2d/MtTaEg0sMzM0MjItYmxvZwO0O0OO0O0O.html)[java](https://www.csdn.net/gather_24/NtTaIg5sMzYyLWJsb2cO0O0O.html)

版权



```
ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) 
```

使用方法：

### **1.SynchronousQueue**

```java
private static ExecutorService cachedThreadPool = new ThreadPoolExecutor(4, Runtime.getRuntime().availableProcessors() * 2, 0, TimeUnit.MILLISECONDS, new SynchronousQueue<>(), r -> new Thread(r, "ThreadTest"));
```

SynchronousQueue没有容量，是无缓冲等待队列，是一个不存储元素的阻塞队列，会直接将任务交给消费者，必须等队列中的添加元素被消费后才能继续添加新的元素。

拥有公平（FIFO）和非公平(LIFO)策略，非公平侧罗会导致一些数据永远无法被消费的情况？

使用SynchronousQueue阻塞队列一般要求maximumPoolSizes为无界(Integer.MAX_VALUE)，避免线程拒绝执行操作。

 

### 2.**LinkedBlockingQueue**

```java
private static ExecutorService cachedThreadPool = new ThreadPoolExecutor(4, Runtime.getRuntime().availableProcessors() * 2, 0, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(), r -> new Thread(r, "ThreadTest"));
```

LinkedBlockingQueue是一个无界缓存等待队列。当前执行的线程数量达到corePoolSize的数量时，剩余的元素会在阻塞队列里等待。（所以在使用此阻塞队列时maximumPoolSizes就相当于无效了），每个线程完全独立于其他线程。生产者和消费者使用独立的锁来控制数据的同步，即在高并发的情况下可以并行操作队列中的数据。

注：这个队列需要注意的是，虽然通常称其为一个无界队列，但是可以人为指定队列大小，而且由于其用于记录队列大小的参数是int类型字段，所以通常意义上的无界其实就是队列长度为 Integer.MAX_VALUE，且在不指定队列大小的情况下也会默认队列大小为 Integer.MAX_VALUE，等同于如下：

```java
private static ExecutorService cachedThreadPool = new ThreadPoolExecutor(4, Runtime.getRuntime().availableProcessors() * 2, 0, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(Integer.MAX_VALUE), r -> new Thread(r, "ThreadTest"));
```

###  

### 3.**ArrayBlockingQueue**

```java
 private static ExecutorService cachedThreadPool = new ThreadPoolExecutor(4, Runtime.getRuntime().availableProcessors() * 2, 0, TimeUnit.MILLISECONDS, new ArrayBlockingQueue<>(32), r -> new Thread(r, "ThreadTest"));
```

ArrayBlockingQueue是一个有界缓存等待队列，可以指定缓存队列的大小，当正在执行的线程数等于corePoolSize时，多余的元素缓存在ArrayBlockingQueue队列中等待有空闲的线程时继续执行，当ArrayBlockingQueue已满时，加入ArrayBlockingQueue失败，会开启新的线程去执行，当线程数已经达到最大的maximumPoolSizes时，再有新的元素尝试加入ArrayBlockingQueue时会报错。

 