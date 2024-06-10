# [THREADLOCAL设计模式 .](https://www.cnblogs.com/eason-chan/p/3731193.html)

线程安全问题的由来

　　在传统的Web开发中,我们处理Http请求最常用的方式是通过实现Servlet对象来进行Http请求的响应.Servlet是J2EE的重要标准之一,规定了Java如何响应Http请求的规范.通过HttpServletRequest和HttpServletResponse对象,我们能够轻松地与Web容器交互.

　　当Web容器收到一个Http请求时,Web容器中的一个主调度线程会从事先定义好的线程中分配一个当前工作线程,将请求分配给当前的工作线程,由该线程来执行对应的Servlet对象中的service方法.当这个工作线程正在执行的时候,Web容器收到另外一个请求,主调度线程会同样从线程池中选择另外一个工作线程来服务新的请求.Web容器本身并不关心这个新的请求是否访问的是同一个Servlet实例.因此,我们可以得出一个结论:**对于同一个Servlet对象的多个请求**,**Servlet的service方法将在一个多线程的环境中并发执行**.所以,Web容器默认采用**单实例（单Servlet实例）多线程**的方式来处理Http请求.这种处理方式**能够减少新建Servlet实例的开销**,**从而缩短了对Http请求的响应时间**.但是,这样的处理方式会导致变量访问的线程安全问题.也就是说,Servlet对象并不是一个线程安全的对象.

下面的测试代码将证实这一点：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
01.package com.qingdao.proxy;  
02.  
03.public class ThreadSafeTestServlet extends HttpServlet {  
04.    // 定义一个实例变量，并非一个线程安全的变量   
05.    private int counter = 0;  
06.  
07.    public void doGet(HttpServletRequest req, HttpServletResponse resp)  
08.            throws ServletException, IOException {  
09.        doPost(req, resp);  
10.    }  
11.  
12.    public void doPost(HttpServletRequest req, HttpServletResponse resp)  
13.            throws ServletException, IOException {  
14.        // 输出当前Servlet的信息以及当前线程的信息   
15.        System.out.println(this + ":" + Thread.currentThread());  
16.        // 循环，并增加实例变量counter的值   
17.        for (int i = 0; i < 5; i++) {  
18.            System.out.println("Counter = " + counter);  
19.            try {  
20.                Thread.sleep((long) Math.random() * 1000);  
21.                counter++;  
22.            } catch (InterruptedException exc) {  
23.            }  
24.        }  
25.    }  
26.}  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

输出结果为:

```
sample.SimpleServlet@11e1bbf:Thread[http-8081-Processor23,5,main] 
Counter = 60 
Counter = 61 
Counter = 62 
Counter = 65 
Counter = 68 
Counter = 71 
Counter = 74 
Counter = 77 
Counter = 80 
Counter = 83 

sample.SimpleServlet@11e1bbf:Thread[http-8081-Processor22,5,main] 
Counter = 61 
Counter = 63 
Counter = 66 
Counter = 69 
Counter = 72 
Counter = 75 
Counter = 78 
Counter = 81 
Counter = 84 
Counter = 87 

sample.SimpleServlet@11e1bbf:Thread[http-8081-Processor24,5,main] 
Counter = 61 
Counter = 64 
Counter = 67 
Counter = 70 
Counter = 73 
Counter = 76 
Counter = 79 
Counter = 82 
Counter = 85 
Counter = 88
```

 

 

```

```

通过上面的输出,我们可以得出以下三个Servlet对象的运行特性:**1. Servlet对象是一个无状态的单例对象（Singleton）**,**因为我们看到多次请求的this指针所打印出来的hashcode值都相同2. Servlet在不同的线程（线程池）中运行**,**如http-8081-Processor22和http-8081-Processor23等输出值可以明显区分出不同的线程执行了同一段Servlet逻辑代码**.**3. Counter变量在不同的线程中共享**,**而且它的值被不同的线程修改**,**输出时已经不是顺序输出**.**也就是说**,**其他的线程会篡改当前线程中实例变量的值**,**针对这些对象的访问不是线程安全的**.

 

 

 

**ThreadLocal模式的实现机理**

```

```


在JDK的早期版本中,提供了一种解决多线程并发问题的方案: java.lang.ThreadLocal类.ThreadLocal类在维护变量时,实际使用了当前线程（Thread）中的一个叫做ThreadLocalMap的独立副本,每个线程可以独立修改属于自己的副本而不会互相影响,从而隔离了线程和线程,避免了线程访问实例变量发生冲突的问题.
ThreadLocal本身并不是一个线程,而是通过操作当前线程（Thread）中的一个内部变量来达到与其他线程隔离的目的.之所以取名为ThreadLocal,所期望表达的含义是其操作的对象是线程（Thread）的一个本地变量.如果我们看一下Thread的源码实现,就会发现这一变量,代码如下:

```
01.public class Thread implements Runnable {  
02.    // 这里省略了许多其他的代码   
03.    ThreadLocal.ThreadLocalMap threadLocals = null;  
04.} 
```

这是JDK中Thread源码的一部分,从中我们可以看出ThreadLocalMap跟随着当前的线程而存在.不同的线程Thread,拥有不同的ThreadLocalMap的本地实例变量,这也就是“副本”的含义.接下来我们再来看看ThreadLocal.ThreadLocalMap是如何定义的,以及ThreadLocal如何来操作它

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
01.public class ThreadLocal<T> {  
02.  
03.    // 这里省略了许多其他代码   
04.  
05.    // 将value的值保存于当前线程的本地变量中   
06.    public void set(T value) {  
07.        // 获取当前线程   
08.        Thread t = Thread.currentThread();  
09.        // 调用getMap方法获得当前线程中的本地变量ThreadLocalMap   
10.        ThreadLocalMap map = getMap(t);  
11.        // 如果ThreadLocalMap已存在，直接使用   
12.        if (map != null)  
13.            // 以当前的ThreadLocal的实例作为key，存储于当前线程的   
14.            // ThreadLocalMap中，如果当前线程中被定义了多个不同的ThreadLocal   
15.            // 的实例，则它们会作为不同key进行存储而不会互相干扰   
16.            map.set(this, value);  
17.        else  
18.            // ThreadLocalMap不存在，则为当前线程创建一个新的   
19.            createMap(t, value);  
20.    }  
21.  
22.    // 获取当前线程中以当前ThreadLocal实例为key的变量值   
23.    public T get() {  
24.        // 获取当前线程   
25.        Thread t = Thread.currentThread();  
26.        // 获取当前线程中的ThreadLocalMap   
27.        ThreadLocalMap map = getMap(t);  
28.        if (map != null) {  
29.            // 获取当前线程中以当前ThreadLocal实例为key的变量值   
30.            ThreadLocalMap.Entry e = map.getEntry(this);  
31.            if (e != null)  
32.                return (T) e.value;  
33.        }  
34.        // 当map不存在时，设置初始值   
35.        return setInitialValue();  
36.    }  
37.  
38.    // 从当前线程中获取与之对应的ThreadLocalMap   
39.    ThreadLocalMap getMap(Thread t) {  
40.        return t.threadLocals;  
41.    }  
42.  
43.    // 创建当前线程中的ThreadLocalMap   
44.    void createMap(Thread t, T firstValue) {  
45.        // 调用构造函数生成当前线程中的ThreadLocalMap   
46.        t.threadLocals = new ThreadLocalMap(this, firstValue);  
47.    }  
48.  
49.    // ThreadLoaclMap的定义   
50.    static class ThreadLocalMap {  
51.        // 这里省略了许多代码   
52.    }  
53.}  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```

```

从上述代码中,我们看到了ThreadLocal类的大致结构和进行ThreadLocalMap的操作.我们可以从中得出以下的结论:**1. ThreadLocalMap变量属于线程（Thread）的内部属性**,**不同的线程（Thread）拥有完全不同的ThreadLocalMap变量**.**2. 线程（Thread）中的ThreadLocalMap变量的值是在ThreadLocal对象进行set或者get操作时创建的**.**3. 在创建ThreadLocalMap之前**,**会首先检查当前线程（Thread）中的ThreadLocalMap变量是否已经存在**,**如果不存在则创建一个；如果已经存在**,**则使用当前线程（Thread）已创建的ThreadLocalMap**.**4. 使用当前线程（Thread）的ThreadLocalMap的关键在于使用当前的ThreadLocal的实例作为key进行存储**ThreadLocal模式,至少从两个方面完成了数据访问隔离,有了横向和纵向的两种不同的隔离方式,ThreadLocal模式就能真正地做到线程安全：**纵向隔离** —— **线程（Thread）与线程（Thread）之间的数据访问隔离**.**这一点由线程（Thread）的数据结构保证**.**因为每个线程（Thread）在进行对象访问时**,**访问的都是各自线程自己的ThreadLocalMap**.**横向隔离** —— **同一个线程中**,**不同的ThreadLocal实例操作的对象之间的相互隔离**.**这一点由ThreadLocalMap在存储时**,**采用当前ThreadLocal的实例作为key来保证**.

 

 

深入比较TheadLocal模式与synchronized关键字

　　ThreadLocal模式synchronized关键字都用于处理多线程并发访问变量的问题,只是二者处理问题的角度和思路不同.

　　1）ThreadLocal是一个java类,通过对当前线程中的局部变量的操作来解决不同线程的变量访问的冲突问题.所以,ThreadLocal提供了线程安全的共享对象机制,每个线程都拥有其副本.

　　2）Java中的synchronized是一个保留字,它依靠JVM的锁机制来实现临界区的函数或者变量的访问中的原子性.在同步机制中,通过对象的锁机制保证同一时间只有一个线程访问变量.此时,被用作“锁机制”的变量时多个线程共享的.

　　同步机制(synchronized关键字)采用了“以时间换空间”的方式,提供一份变量,让不同的线程排队访问.而ThreadLocal采用了“以空间换时间”的方式,为每一个线程都提供一份变量的副本,从而实现同时访问而互不影响

 

**ThreadLocal模式的核心元素**

**要完成ThreadLocal模式,其中最关键的地方就是创建一个任何地方都可以访问到的ThreadLocal实例（也就是执行示意图中的菱形部分）.**而这一点,我们可以通过类的静态实例变量来实现,这个用于承载静态实例变量的类就被视作是一个共享环境.我们来看一个例子,如代码清单如下所示：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
01.public class Counter {  
02.     //新建一个静态的ThreadLocal变量，并通过get方法将其变为一个可访问的对象    
03.     private static ThreadLocal<Integer> counterContext = new ThreadLocal<Integer>(){  
04.         protected synchronized Integer initialValue(){  
05.             return 10;  
06.         }  
07.     };  
08.     // 通过静态的get方法访问ThreadLocal中存储的值   
09.     public static Integer get(){  
10.         return counterContext.get();  
11.     }  
12.     // 通过静态的set方法将变量值设置到ThreadLocal中   
13.     public static void set(Integer value) {    
14.         counterContext.set(value);    
15.     }   
16.     // 封装业务逻辑，操作存储于ThreadLocal中的变量     
17.     public static Integer getNextCounter() {    
18.         counterContext.set(counterContext.get() + 1);    
19.         return counterContext.get();    
20.     }    
21. }  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
01.public class ThreadLocalTest extends Thread {  
02.     public void run(){  
03.         for(int i = 0; i < 3; i++){    
04.             System.out.println("Thread[" + Thread.currentThread().getName() + "],counter=" + Counter.getNextCounter());    
05.         }    
06.     }  
07. }  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
01.public class Test {  
02.     public static void main(String[] args) {  
03.         ThreadLocalTest testThread1 = new ThreadLocalTest();    
04.         ThreadLocalTest testThread2 = new ThreadLocalTest();    
05.         ThreadLocalTest testThread3 = new ThreadLocalTest();    
06.         testThread1.start();    
07.         testThread2.start();    
08.         testThread3.start();  
09.     }  
10. }  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

输出结果:

Thread[Thread-2],counter=11
Thread[Thread-2],counter=12
Thread[Thread-2],counter=13
Thread[Thread-0],counter=11
Thread[Thread-0],counter=12
Thread[Thread-0],counter=13
Thread[Thread-1],counter=11
Thread[Thread-1],counter=12
Thread[Thread-1],counter=13

 

上面的输出结果也证实了,counter的值在多线程环境中的访问是线程安全的.从对例子的分析中我们可以再次体会到,T**<u>hreadLocal模式最合适的使用场景</u>**:**在同一个线程（Thread）的不同开发层次中共享数据.**

　　从上面的例子中,我们可以简单总结出实现ThreadLocal模式的两个主要步骤:
　　**1. 建立一个类**,**并在其中封装一个静态的ThreadLocal变量**,**使其成为一个共享数据环境.
　　2. 在类中实现访问静态ThreadLocal变量的静态方法（设值和取值）.**

　　建立在ThreadLocal模式的实现步骤之上,ThreadLocal的使用则更加简单.在线程执行的任何地方，我们都可以通过访问共享数据类中所提供的ThreadLocal变量的设值和取值方法安全地获得当前线程中安全的变量值.
　　这两个步骤,我们之后会在Struts2的实现中多次提及,读者只要能充分理解ThreadLocal处理多线程访问的基本原理，就能对Struts2的数据访问和数据共享的设计有一个整体的认识.

　　讲到这里,我们回过头来看看ThreadLocal模式的引入,到底对我们的编程模型有什么重要的意义呢?

　　**结论 ：使用ThreadLocal模式**,**可以使得数据在不同的编程层次得到有效地共享**,

　　这一点,是由ThreadLocal模式的实现机理决定的.因为实现ThreadLocal模式的一个重要步骤,就是构建一个静态的共享存储空间.从而使得任何对象在任何时刻都可以安全地对数据进行访问.

　　**结论 使用ThreadLocal模式,可以对执行逻辑与执行数据进行有效解耦**

　　**这一点是ThreadLocal模式给我们带来的最为核心的一个影响**,**因为在一般情况下,Java对象之间的协作关系，主要通过参数和返回值来进行消息传递,这也是对象协作之间的一个重要依赖**,**而ThreadLocal模式彻底打破了这种依赖关系,通过线程安全的共享对象来进行数据共享,可以有效避免在编程层次之间形成数据依赖**,**这也成为了XWork事件处理体系设计的核心.**

ThreadLocal 多线程共享变量 的隔离
线程内部有ThreadLocalMap 属性 ，而ThreadLocalMap是ThreadLocal的静态内部类
横向 线程与线程之间通过Thread线程内部ThradLocalMap 隔离；
纵向 变量与变量之间通过不同ThreadLocal对象隔离
ThreadLocalMap 内部以this指针根据当前ThreadLocal对象为键存储，实现多个变量的隔离

![img](https://images0.cnblogs.com/i/618007/201405/160022554536920.jpg)

分类: [JAVA](https://www.cnblogs.com/eason-chan/category/565206.html)

[好文要顶](javascript:void(0);) [关注我](javascript:void(0);) [收藏该文](javascript:void(0);) [![img](https://common.cnblogs.com/images/icon_weibo_24.png)](javascript:void(0);) [![img](https://common.cnblogs.com/images/wechat.png)](javascript:void(0);)

[![img](https://pic.cnblogs.com/face/sample_face.gif)](https://home.cnblogs.com/u/eason-chan/)

[Eason_Chan](https://home.cnblogs.com/u/eason-chan/)
[关注 - 1](https://home.cnblogs.com/u/eason-chan/followees/)
[粉丝 - 4](https://home.cnblogs.com/u/eason-chan/followers/)

[+加关注](javascript:void(0);)

0

0

[« ](https://www.cnblogs.com/eason-chan/p/3709969.html)上一篇： [转 Spring @Transactional 声明式事务管理 getCurrentSession](https://www.cnblogs.com/eason-chan/p/3709969.html) 
[» ](https://www.cnblogs.com/eason-chan/p/3731747.html)下一篇： [PB中游标的使用](https://www.cnblogs.com/eason-chan/p/3731747.html)