- ## 深入理解 Java 多线程核心知识：跳槽面试必备

  原创 Colie [GitChat精品课](javascript:void(0);) *2018-03-31*

  ![img](https://mmbiz.qpic.cn/mmbiz_gif/wvkocF2MXjUB0iaia5nk3IQJuVKG6CZ1xKkNDpFBGygGxdWibF4gcIGEEicicedRqKrW1JIAX72mYV2zMcxJO5siaFcA/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

  

  多线程相对于其他 Java 知识点来讲，有一定的学习门槛，并且了解起来比较费劲。在平时工作中如若使用不当会出现数据错乱、执行效率低（还不如单线程去运行）或者死锁程序挂掉等等问题，所以掌握了解多线程至关重要。

  

  本文从基础概念开始到最后的并发模型由浅入深，讲解下线程方面的知识。

  

  ### 

  **概念梳理**

  ###  

  本节我将带大家了解多线程中几大基础概念。

  

  #### 

  

  并发与并行

  

  ####  

  并行，表示两个线程同时做事情。

  

  并发，表示一会做这个事情，一会做另一个事情，存在着调度。单核 CPU 不可能存在并行（微观上）。

  

  ![img](https://mmbiz.qpic.cn/mmbiz_png/wvkocF2MXjXVUSGvp7ricK0QNVicDNcjicqgrrt2jQQJMSfrNfyiaSoHvxdkdas1Zd45RdFWakiaBWECjS0ISJ51icYg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  ####  

  #### 

  

  临界区

  

  

  临界区用来表示一种公共资源或者说是共享数据，可以被多个线程使用。但是每一次，只能有一个线程使用它，一旦临界区资源被占用，其他线程要想使用这个资源，就必须等待。

  

  ![img](https://mmbiz.qpic.cn/mmbiz_png/wvkocF2MXjXVUSGvp7ricK0QNVicDNcjicqJ4Zia0wmHZdIpqX1XRZibX52VKexLfHX68icvv5SuJoZ9hxJUQkdiaNm9Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  ####  

  #### 

  

  阻塞与非阻塞

  

  

  阻塞和非阻塞通常用来形容多线程间的相互影响。比如一个线程占用了临界区资源，那么其它所有需要这个资源的线程就必须在这个临界区中进行等待，等待会导致线程挂起。这种情况就是阻塞。

  

  此时，如果占用资源的线程一直不愿意释放资源，那么其它所有阻塞在这个临界区上的线程都不能工作。阻塞是指线程在操作系统层面被挂起。阻塞一般性能不好，需大约8万个时钟周期来做调度。

  

  非阻塞则允许多个线程同时进入临界区。

  

  #### 

  

  死锁

  

  ####  

  死锁是进程死锁的简称，是指多个进程循环等待他方占有的资源而无限的僵持下去的局面。

  

  ![img](https://mmbiz.qpic.cn/mmbiz_png/wvkocF2MXjXVUSGvp7ricK0QNVicDNcjicqLe2hUVFFr2194RvUbEQGw10Wotia5mfibk3lR0hrBApVNZzDHn5ia3ib2g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  ####  

  #### 

  

  活锁

  

  

  假设有两个线程1、2，它们都需要资源 A/B，假设1号线程占有了 A 资源，2号线程占有了 B 资源；由于两个线程都需要同时拥有这两个资源才可以工作，为了避免死锁，1号线程释放了 A 资源占有锁，2号线程释放了 B 资源占有锁；此时 AB 空闲，两个线程又同时抢锁，再次出现上述情况，此时发生了活锁。

  

  简单类比，电梯遇到人，一个进的一个出的，对面占路，两个人同时往一个方向让路，来回重复，还是堵着路。

  

  如果线上应用遇到了活锁问题，恭喜你中奖了，这类问题比较难排查。

  

  #### 

  

  饥饿

  

  ####  

  饥饿是指某一个或者多个线程因为种种原因无法获得所需要的资源，导致一直无法执行。

  

  ### 

  **线程的生命周期**

  

  在线程的生命周期中，它要经历创建、可运行、不可运行几种状态。

  

  #### 

  

  创建状态

  

  ####  

  当用 new 操作符创建一个新的线程对象时，该线程处于创建状态。

  

  处于创建状态的线程只是一个空的线程对象，系统不为它分配资源。

  

  #### 

  

  可运行状态

  

  ####  

  执行线程的 start() 方法将为线程分配必须的系统资源，安排其运行，并调用线程体——run()方法，这样就使得该线程处于可运行状态（Runnable）。

  

  这一状态并不是运行中状态（Running），因为线程也许实际上并未真正运行。

  

  #### 

  

  不可运行状态

  

  ####  

  当发生下列事件时，处于运行状态的线程会转入到不可运行状态：

  - 调用了 sleep() 方法；
  - 线程调用 wait() 方法等待特定条件的满足；
  - 线程输入/输出阻塞；
  - 返回可运行状态；
  - 处于睡眠状态的线程在指定的时间过去后；
  - 如果线程在等待某一条件，另一个对象必须通过 notify() 或 notifyAll() 方法通知等待线程条件的改变；
  - 如果线程是因为输入输出阻塞，等待输入输出完成。

  ###  

  ### 

  **线程的优先级**

  ####  

  #### 

  

  线程优先级及设置

  

  

  线程的优先级是为了在多线程环境中便于系统对线程的调度，优先级高的线程将优先执行。一个线程的优先级设置遵从以下原则：

  - 线程创建时，子继承父的优先级；
  - 线程创建后，可通过调用 setPriority() 方法改变优先级；
  - 线程的优先级是1-10之间的正整数。

  ####  

  #### 

  

  线程的调度策略

  

  

  线程调度器选择优先级最高的线程运行。但是，如果发生以下情况，就会终止线程的运行：

  - 线程体中调用了 yield() 方法，让出了对 CPU 的占用权；
  - 线程体中调用了 sleep() 方法，使线程进入睡眠状态；
  - 线程由于 I/O 操作而受阻塞；
  - 另一个更高优先级的线程出现；
  - 在支持时间片的系统中，该线程的时间片用完。

  ###  

  ### 

  **单线程创建方式**

  ###  

  单线程创建方式比较简单，一般只有两种方式：继承 Thread 类和实现 Runnable 接口；这两种方式比较常用就不在 Demo 了，但是对于新手需要注意的问题有：

  

  - 不管是继承 Thread 类还是实现 Runable 接口，业务逻辑是写在 run 方法里面，线程启动的时候是执行 start() 方法；
  - 开启新的线程，不影响主线程的代码执行顺序也不会阻塞主线程的执行；
  - 新的线程和主线程的代码执行顺序是不能够保证先后的；
  - 对于多线程程序，从微观上来讲某一时刻只有一个线程在工作，多线程目的是让 CPU 忙起来；
  - 通过查看 Thread 的源码可以看到，Thread 类是实现了 Runnable 接口的，所以这两种本质上来讲是一个；

  

  PS：平时在工作中也可以借鉴这种代码结构，对上层调用来讲提供更多的选择，作为服务提供方核心业务归一维护

  

  ### 

  **为什么要用线程池**

  

  通过上面的介绍，完全可以开发一个多线程的程序，为什么还要引入线程池呢。主要是因为上述单线程方式存在以下几个问题：

  

  - 线程的工作周期：线程创建所需时间为 T1，线程执行任务所需时间为 T2，线程销毁所需时间为 T3，往往是 T1+T3 大于 T2，所有如果频繁创建线程会损耗过多额外的时间；
  - 如果有任务来了，再去创建线程的话效率比较低，如果从一个池子中可以直接获取可用的线程，那效率会有所提高。所以线程池省去了任务过来，要先创建线程再去执行的过程，节省了时间，提升了效率；
  - 线程池可以管理和控制线程，因为线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控；
  - 线程池提供队列，存放缓冲等待执行的任务。

  

  大致总结了上述的几个原因，所以可以得出一个结论就是在平时工作中，如果要开发多线程程序，尽量要使用线程池的方式来创建和管理线程。

  

  通过线程池创建线程从调用 API 角度来说分为两种，一种是原生的线程池，另外该一种是通过 Java 提供的并发包来创建，后者比较简单，后者其实是对原生的线程池创建方式做了一次简化包装，让调用者使用起来更方便，但道理都是一样的。所以搞明白原生线程池的原理是非常重要的。

  

  ### 

  **ThreadPoolExecutor**

  

  通过 ThreadPoolExecutor 创建线程池，API 如下所示：

  ```
  public ThreadPoolExecutor(int corePoolSize,                          int maximumPoolSize,                          long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue); 
  ```

  先来解释下其中的参数含义（如果看的比较模糊可以大致有个印象，后面的图是关键）。

  - corePoolSize
  - 核心池的大小。

  

  在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了 prestartAllCoreThreads() 或者 prestartCoreThread() 方法，从这两个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建 corePoolSize 个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到 corePoolSize 后，就会把到达的任务放到缓存队列当中。

  - maximumPoolSize

  

  线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程。

  - keepAliveTime

  

  表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于 corePoolSize 时，keepAliveTime 才会起作用，直到线程池中的线程数不大于 corePoolSize，即当线程池中的线程数大于 corePoolSize 时，如果一个线程空闲的时间达到 keepAliveTime，则会终止，直到线程池中的线程数不超过 corePoolSize。

  

  但是如果调用了 allowCoreThreadTimeOut(boolean) 方法，在线程池中的线程数不大于 corePoolSize 时，keepAliveTime 参数也会起作用，直到线程池中的线程数为0。

  - unit

  参数 keepAliveTime 的时间单位。

  - workQueue

  

  一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下这几种选择：ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue。

  - threadFactory

  

  线程工厂，主要用来创建线程。

  - handler

  

  表示当拒绝处理任务时的策略，有以下四种取值：

  1. ThreadPoolExecutor.AbortPolicy：丢弃任务并抛出 RejectedExecutionException 异常；
  2. ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常；
  3. ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）；
  4. ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务。

  

  上面这些参数是如何配合工作的呢？请看下图：

  

  ![img](https://mmbiz.qpic.cn/mmbiz_png/wvkocF2MXjXVUSGvp7ricK0QNVicDNcjicqP5ymo3ud9ebia8Nxw8qz9dh4fkaKDtianFicnaMb5DB6YD5hvLYwzqliaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  

  注意图上面的序号。

  

  简单总结下线程池之间的参数协作分为以下几步：

  1. 线程优先向 CorePool 中提交；
  2. 在 Corepool 满了之后，线程被提交到任务队列，等待线程池空闲；
  3. 在任务队列满了之后 corePool 还没有空闲，那么任务将被提交到 maxPool 中，如果 MaxPool 满了之后执行 task 拒绝策略。

  

  流程图如下：

  

  ![img](https://mmbiz.qpic.cn/mmbiz_png/wvkocF2MXjXVUSGvp7ricK0QNVicDNcjicqDran4sGjAWPlm6GT8NjsL7R1TCgJrMyfmDFYGXmOle8Vyqxl2ohUgA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  

  以上就是原生线程池创建的核心原理。除了原生线程池之外并发包还提供了简单的创建方式，上面也说了它们是对原生线程池的一种包装，可以让开发者简单快捷的创建所需要的线程池。

  

  ### 

  **Executors**

  ####  

  #### 

  

  newSingleThreadExecutor

  

  

  创建一个线程的线程池，在这个线程池中始终只有一个线程存在。如果线程池中的线程因为异常问题退出，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

  

  #### 

  

  newFixedThreadPool

  

  

  创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

  

  #### 

  

  newCachedThreadPool

  

  

  可根据实际情况，调整线程数量的线程池，线程池中的线程数量不确定，如果有空闲线程会优先选择空闲线程，如果没有空闲线程并且此时有任务提交会创建新的线程。在正常开发中并不推荐这个线程池，因为在极端情况下，会因为 newCachedThreadPool 创建过多线程而耗尽 CPU 和内存资源。

  

  #### 

  

  newScheduledThreadPool

  

  ####  

  此线程池可以指定固定数量的线程来周期性的去执行。比如通过 scheduleAtFixedRate 或者 scheduleWithFixedDelay 来指定周期时间。

  PS：另外在写定时任务时（如果不用 Quartz 框架），最好采用这种线程池来做，因为它可以保证里面始终是存在活的线程的。

  

  ### 

  **推荐使用 ThreadPoolExecutor 方式**

  ###  

  在阿里的 Java 开发手册时有一条是不推荐使用 Executors 去创建，而是推荐去使用 ThreadPoolExecutor 来创建线程池。

  

  这样做的目的主要原因是：使用 Executors 创建线程池不会传入核心参数，而是采用的默认值，这样的话我们往往会忽略掉里面参数的含义，如果业务场景要求比较苛刻的话，存在资源耗尽的风险；另外采用 ThreadPoolExecutor 的方式可以让我们更加清楚地了解线程池的运行规则，不管是面试还是对技术成长都有莫大的好处。

  

  改了变量，其他线程可以立即知道。保证可见性的方法有以下几种：

  - volatile

  

  加入 volatile 关键字的变量在进行汇编时会多出一个 lock 前缀指令，这个前缀指令相当于一个内存屏障，内存屏障可以保证内存操作的顺序。当声明为 volatile 的变量进行写操作时，那么这个变量需要将数据写到主内存中。

  

  由于处理器会实现缓存一致性协议，所以写到主内存后会导致其他处理器的缓存无效，也就是线程工作内存无效，需要从主内存中重新刷新数据。

  

  

  

  

  

  

  

  **阅读全文请**

  **扫描下方二维码，**

  **还可以加入读者圈与作者聊天~：**

  **
  **

  ![img](https://mmbiz.qpic.cn/mmbiz_png/wvkocF2MXjXVUSGvp7ricK0QNVicDNcjicqzyCOZ4y9evL5V8PMxZKk10cbibtxKQbnU0xQPYjVOqR3Riax3tl0t2gQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  

  

  

  

  

  

  

  

  

  

  阅读 1.3万

  赞46在看

  ![img](http://wx.qlogo.cn/mmopen/Ko1bNiaWNDOzUg9xwcndRfdLoxFH5kWovk6gFQ14DodVzuAibIu6LIzvSicDID3MvMBcUZ0ySnibLX0oFIL9FbqlEejcqgZmfQ1m/132)

  写下你的留言

  **精选留言**

  - ![img](http://wx.qlogo.cn/mmopen/y7HVKG5l1AmicKvl1Nw4uibfNBUqD3D1tyib1dIonF8hX4leIRrWX27m8jc9pv0pNDeZw42hmCziaJ1bRkH7lfsBhSAQTicXHIgI6/96)**吴敏强**

    7

    

    写的非常好，最近也在学习这方面知识，希望多出这方面理论知识，分系列就更好，从简单一直到高级。

  - ![img](http://wx.qlogo.cn/mmopen/JGq7V1cYd5OfiaT1KyZyV9mtd42ickFgTFwdAhTOAcQZusMIuEnTibP7VEEeu17fBXpyviaE2mk2GUEJzmyM9VNAfWic4o6SRJZbV/96)**邵志鹏**

    5

    

    炼数成金的PPT

  - ![img](http://wx.qlogo.cn/mmopen/7SPO0mRJt6CibiaYhIeGX1q1drF9SO2sa75onIicPSexng16whicQnIp6W7mxkG9lklCSrj3s6o3OXYYIxQWGajCEA/96)**guokun**

    2

    

    用net，表示线程操作很简单

  - ![img](http://wx.qlogo.cn/mmopen/bazvcZyra5J3ndrG3ZWjR0mAAfeXDxvY5uwMNDmibbiczPkrxLLiaCcibBrKGuH1CPkuwVkW5lCATg6WbibvcnD9P4fWfgqKSO4JG/96)**Ankermaker**

    2

    

    稳

  - ![img](http://wx.qlogo.cn/mmopen/Ko1bNiaWNDOyPSELp7ddibQiaMECg2wIB31E6I3vPyt6vybocFezrg4XZRssw6uqPrIVHFIXy9N8Sbe1X6YeAhRSLbWicict8yZjZ/96)**tomcat**

    1

    

    你好，请问核心线程和非核心线程的区别是什么？为什么等待队列没满的时候线程进入等待，而满了之后就进入了非线程执行？这样是否会导致等待队列的线程一直得不到资源去执行？

    作者

    1

    可以扫描二维码，在读者圈和作者直接交流

  - ![img](http://wx.qlogo.cn/mmopen/Ko1bNiaWNDOzJtYQW31bOgHBX1CBQK36G2II5UJmrX59r7hSIzBIKQLbTyf69ibpIib1Ej3UVn5vbZcibTRzsNxUIyZgicto8Znpp/96)**明月**

    

    那个任务队列没有指定大小，怎么确定它满了的呢？

  - ![img](http://wx.qlogo.cn/mmopen/ajNVdqHZLLBYY8Z3tuzNHGgX84dJ28XbxU1pmBQ4jww74yYbia8fQQgxLCIJpicLibkfcgF5icgrt6Wm1IhT7mqLXQ/96)**Go golang**

    

    66666 牛b 我一个php 都能看懂 。通俗易懂

  - ![img](http://wx.qlogo.cn/mmopen/bazvcZyra5Jp731uFkDia8v2WNOhkF04xHy0wUZ8ibmLpxdCBIuXibNlTQTkIQgicBGQkw7w9FMXgbjKm1LNAf3IHm9dlicJKe1Jk/96)**一伯**

    

    即使不是计算机专业的朋友们也好好好补习一下操作系统

  - ![img](http://wx.qlogo.cn/mmopen/Ko1bNiaWNDOyPSELp7ddibQpjic04JDzdOEOicbkTs4cREfeNwT6NTkTgbSQWQ4gPOSgU7mTCQibEdVhT0NA9ibK2MhRKhBy2y59lI/96)**阿呆**

    

    买本书学下

  - ![img](http://wx.qlogo.cn/mmopen/UGJJKEvjnTHT4uU8wVlQvanuEG2SZjMXic2Bq8Dhb8zMwF3GPHPdcDKkElD4yLhK7ua5NguRW1iapvBmxNLFcYQWr6foC1Zcsf/96)**Rt**

    

    还好学过操作系统