## 干货分享！Java 多线程编程（锁优化）

大雄 [老九学堂](javascript:void(0);) *2019-04-11*

相信这么努力的你 已经星标了我 

老九学堂 你身边的IT导师



![分割线 卡通人物](https://mmbiz.qpic.cn/mmbiz_png/zUEWQQYPiaWM9wNN4CA5bLzVcgxTmWt1iaYkQocib7HkcDbcib3pS0tb2AJKWh0p2ahBWndY3JfgUqwdCorokN6pTw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



![img](https://mmbiz.qpic.cn/mmbiz_png/JfI4bJ7rK0luNzPHZiaqDmfHFUxObRArWPWzZMia4adA20FWqYJshATruL2buAAaWlIpCLaLrUxHHEgWmTX6ZZxg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



并发环境下进行编程时，需要使用锁机制来同步多线程间的操作，保证共享资源的互斥访问。



加锁会带来性能上的损坏，似乎是众所周知的事情。



然而，加锁本身不会带来多少的性能消耗，性能主要是在线程的获取锁的过程。



如果只有一个线程竞争锁，此时并不存在多线程竞争的情况，那么JVM会进行优化，那么这时加锁带来的性能消耗基本可以忽略。



因此，规范加锁的操作，优化锁的使用方法，避免不必要的线程竞争，不仅可以提高程序性能，也能避免不规范加锁可能造成线程死锁问题，提高程序健壮性。



下面阐述几种锁优化的思路。



***\*
\****

***\*01\****

**
**

**尽量不要锁住方法**



在普通成员函数上加锁时，线程获得的是该方法所在对象的对象锁。此时整个对象都会被锁住。



这也意味着，如果这个对象提供的多个同步方法是针对不同业务的，那么由于整个对象被锁住，一个业务业务在处理时，其他不相关的业务线程也必须wait。



下面的例子展示了这种情况：



LockMethod类包含两个同步方法，分别在两种业务处理中被调用：



> public class LockMethod  {
>
>   public synchronized void busyA() {
>
> ​    for (int i = 0; i < 10000; i++) {
>
> ​      System.out.println(Thread.currentThread().getName() + "deal with bussiness A:"+i);
>
> ​    }
>
>   }
>
>   public synchronized void busyB() {
>
> ​    for (int i = 0; i < 10000; i++) {
>
> ​      System.out.println(Thread.currentThread().getName() + "deal with bussiness B:"+i);
>
> ​    }
>
>   }
>
> }



BusyA是线程类，用来处理A业务，调用的是LockMethod的busyA（）方法：



> public class BusyA extends Thread {
>
>   LockMethod lockMethod;
>
>   void deal(LockMethod lockMethod){
>
> ​    this.lockMethod = lockMethod;
>
>   }
>
> 
>
>   @Override
>
>   public void run() {
>
> ​    super.run();
>
> ​    lockMethod.busyA();
>
>   }
>
> }



BusyB是线程类，用来处理B业务，调用的是LockMethod的busyB（）方法：



> public class BusyB extends Thread {
>
>   LockMethod lockMethod;
>
>   void deal(LockMethod lockMethod){
>
> ​    this.lockMethod = lockMethod;
>
>   }
>
> 
>
>   @Override
>
>   public void run() {
>
> ​    super.run();
>
> ​    lockMethod.busyB();
>
>   }
>
> }



TestLockMethod类，使用线程BusyA与BusyB进行业务处理：



> public class TestLockMethod
>
> 
>
>   public static void main(String[] args) {
>
> ​    LockMethod lockMethod = new LockMethod();
>
> ​    BUSSA bussa = new BUSSA();
>
> ​    BUSSB bussb = new BUSSB();
>
> ​    bussa.deal(lockMethod);
>
> ​    bussb.deal(lockMethod);
>
> ​    bussa.start();
>
> ​    bussb.start();
>
> 
>
>   }
>
> }



运行程序，可以看到在线程bussa 执行的过程中，bussb是不能够进入函数 busyB()的，因为此时lockMethod 的对象锁被线程bussa获取了。



**
**

**02**

**
**

**缩小同步代码块，只锁数据**



有时候为了编程方便，有些人会synchnoized很大的一块代码。



如果这个代码块中的某些操作与共享资源并不相关，那么应当把它们放到同步块外部，避免长时间的持有锁，造成其他线程一直处于等待状态。



尤其是一些循环操作、同步I/O操作。不止是在代码的行数范围上缩小同步块，在执行逻辑上，也应该缩小同步块。

**
**

例如多加一些条件判断，符合条件的再进行同步，而不是同步之后再进行条件判断，尽量减少不必要的进入同步块的逻辑。





**03**

**
**

**锁中尽量不要再包含锁**



这种情况经常发生，线程在得到了A锁之后，在同步方法块中调用了另外对象的同步方法，获得了第二个锁.



这样可能导致一个调用堆栈中有多把锁的请求，多线程情况下可能会出现很复杂、难以分析的异常情况，导致死锁的发生。



下面的代码显示了这种情况：



> synchronized(A){
>
> 
>
>   synchronized(B){
>
>   
>
>    } 
>
> }



或是在同步块中调用了同步方法：



> synchronized(A){
>
> 
>
>   B b = objArrayList.get(0);
>
>   b.method(); //这是一个同步方法
>
> }



解决的办法是跳出来加锁，不要包含加锁：



> {
>
>    B b = null;
>
>   
>
>  synchronized(A){
>
>   b = objArrayList.get(0);
>
>  }
>
>  b.method();
>
> }





**04**

**
**

**将锁私有化，在内部管理锁**



把锁作为一个私有的对象，外部不能拿到这个对象，更安全一些。



对象可能被其他线程直接进行加锁操作，此时线程便持有了该对象的对象锁。



例如下面这种情况：



> class A {
>
>   public void method1() {
>
>   }
>
> }
>
> 
>
> class B {
>
>   public void method1() {
>
> ​    A a = new A();
>
> ​    synchronized (a) { //直接进行加锁
>
> 　　　　　　a.method1();
>
> 
>
> ​    }
>
>   }
>
> }



这种使用方式下，对象a的对象锁被外部所持有，让这把锁在外部多个地方被使用是比较危险的，对代码的逻辑流程阅读也造成困扰。



一种更好的方式是在类的内部自己管理锁，外部需要同步方案时，也是通过接口方式来提供同步操作：



> class A {
>
>   private Object lock = new Object();
>
>   public void method1() {
>
> ​    synchronized (lock){
>
> ​       
>
> ​    }
>
>   }
>
> }
>
> 
>
> class B {
>
>   public void method1() {
>
> ​    A a = new A();
>
> ​    a.method1();
>
>   }
>
> }





**05**



**进行适当的锁分解**



考虑下面这段程序：



> public class GameServer {
>
>  public Map<String, List<Player>> tables = new HashMap<String, List<Player>>();
>
> 
>
>  public void join(Player player, Table table) {
>
>   if (player.getAccountBalance() > table.getLimit()) {
>
>    synchronized (tables) {
>
> ​    List<Player> tablePlayers = tables.get(table.getId());
>
> ​    if (tablePlayers.size() < 9) {
>
> ​     tablePlayers.add(player);
>
> ​    }
>
>    }
>
>   }
>
>  }
>
>  public void leave(Player player, Table table) {/*省略*/} 
>
>  public void createTable() {/*省略*/} 







**通知：**



老九学堂4月线上实战训练营已经开始报名啦！

一个半月，JavaSE从入门到精通。

直播学习最难的Java核心SE所有内容，实时提问解答。

拒绝填鸭式教学，改进学习方法，改变思维模式。

技术大咖带你体验实战开发，积累企业级项目开发经验。

设置结业考试，考试不通过，免费延长学习时间，直到通过为止。

窖头、老九君、大师兄从早9点到晚10点十三个小时，视频、语音、在线答疑。



老九现有会员优惠499元，并且升级为永久会员。

非老九会员，报名送老九永久会员。

**感兴趣的小伙伴可以加小师妹QQ：511233374了解详情哦。**





![img](https://mmbiz.qpic.cn/mmbiz/oBrJUYoj9y7gEfKKXNOFAFuiaGwRhpwNFUpLBCZkaHiasJvc65dIMLpIKtDknB0uwG3lfbTr9wTSkBTMUtcKtpiaw/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





阅读 1886

赞在看2

![img](http://wx.qlogo.cn/mmopen/kk4aicvjd6TY6nDy5cYQfoyOumszdezZn7RcMW7DLia2MSwrKodPZUd4rf2xVenEauZkW2Th9yyKmfBDGJ9cK1bzib0CxNQQS7t/132)

写下你的留言

**精选留言**

- ![img](http://wx.qlogo.cn/mmopen/zUtmPzdCwsYTamN2ohYpviaFXKicmsNvPo2FHmuU5qbKjichZdmE6LtCzaNZiakWiaE4aC9v580ibB0fuYUGvscNw6d1ybyzYhI37f/96)**A-小雨（未命名）**

  2

  

  还没学到这里呢

  作者

  2

  又不要钱 先收着嘛

- ![img](http://wx.qlogo.cn/mmopen/NV4icvWPYAoYRhq7Wbl7ANn3PEMz98lkp3B34ykEHSrF4uWuRiaONG03rq9lfTmrRSSRfMhjOE4Vvk1NajyeXDQQmuFIKEYYJF/96)**²⁰²⁰**

  1

  

  谁能带下我这样的小白入门呢

  作者

  2

  老九网易云课堂免费课程带你飞

- ![img](http://wx.qlogo.cn/mmopen/kk4aicvjd6TbpADWibHcLOVpsfzdMx2CCvrNM4rCf30n3cGvrM9HVib1FibtXvKp7WNhc6MY5ge0BoLT6mazGopj5QTx5ufVOcuT/96)**天高地厚**

  

  收藏收藏，这么干的好东西可不多见啊

- ![img](http://wx.qlogo.cn/mmopen/NV4icvWPYAoYG8ZyWX3bGtXLibTMW9qZvzF2LgOvoCdibaYbZmFkBBav5HG45t07mzpurdOL161oD6rXrmkU1AvpticMhITiaO6de/96)**不言**

  

  死锁了解一下。

  作者

  死锁干货可以安排上了哇

- ![img](http://wx.qlogo.cn/mmopen/kk4aicvjd6Tbqvcw9uWT6SZjhGejn2OjAMx5Rfiariaw71x9Mr25nm6S5efibxtJXg7CGkLiaNhlIdE1o0efW5UITztKNLXfgH3Gy/96)**等到烟火清凉。**

  

  先收藏，再看。

  作者

  不要 先收藏没后续哦