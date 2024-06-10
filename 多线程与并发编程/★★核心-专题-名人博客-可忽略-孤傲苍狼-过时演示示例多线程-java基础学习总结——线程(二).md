# [孤傲苍狼](https://www.cnblogs.com/xdp-gacl/)

只为成功找方法，不为失败找借口！

## [java基础学习总结——线程(二)](https://www.cnblogs.com/xdp-gacl/p/3634382.html)

## 一、线程的优先级别

![image-20200905222439049](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200905222439049.png)



**线程优先级别的使用范例：**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 package cn.galc.test;
 2 
 3 public class TestThread6 {
 4     public static void main(String args[]) {
 5         MyThread4 t4 = new MyThread4();
 6         MyThread5 t5 = new MyThread5();
 7         Thread t1 = new Thread(t4);
 8         Thread t2 = new Thread(t5);
 9         t1.setPriority(Thread.NORM_PRIORITY + 3);// 使用setPriority()方法设置线程的优先级别，这里把t1线程的优先级别进行设置
10         /*
11          * 把线程t1的优先级(priority)在正常优先级(NORM_PRIORITY)的基础上再提高3级 
12          * 这样t1的执行一次的时间就会比t2的多很多 　　　　
13          * 默认情况下NORM_PRIORITY的值为5
14          */
15         t1.start();
16         t2.start();
17         System.out.println("t1线程的优先级是：" + t1.getPriority());
18         // 使用getPriority()方法取得线程的优先级别，打印出t1的优先级别为8
19     }
20 }
21 
22 class MyThread4 implements Runnable {
23     public void run() {
24         for (int i = 0; i <= 1000; i++) {
25             System.out.println("T1：" + i);
26         }
27     }
28 }
29 
30 class MyThread5 implements Runnable {
31     public void run() {
32         for (int i = 0; i <= 1000; i++) {
33             System.out.println("===============T2：" + i);
34         }
35     }
36 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

run()方法一结束，线程也就结束了。

## 二、线程同步

![image-20200905222532166](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200905222532166.png)



**synchronized关键字的使用范例：**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 package cn.galc.test;
 2 
 3 public class TestSync implements Runnable {
 4     Timer timer = new Timer();
 5 
 6     public static void main(String args[]) {
 7         TestSync test = new TestSync();
 8         Thread t1 = new Thread(test);
 9         Thread t2 = new Thread(test);
10         t1.setName("t1");// 设置t1线程的名字
11         t2.setName("t2");// 设置t2线程的名字
12         t1.start();
13         t2.start();
14     }
15 
16     public void run() {
17         timer.add(Thread.currentThread().getName());
18     }
19 }
20 
21 class Timer {
22     private static int num = 0;
23 
24     public/* synchronized */void add(String name) {// 在声明方法时加入synchronized时表示在执行这个方法的过程之中当前对象被锁定
25         synchronized (this) {
26             /*
27              * 使用synchronized(this)来锁定当前对象，这样就不会再出现两个不同的线程同时访问同一个对象资源的问题了 只有当一个线程访问结束后才会轮到下一个线程来访问
28              */
29             num++;
30             try {
31                 Thread.sleep(1);
32             } catch (InterruptedException e) {
33                 e.printStackTrace();
34             }
35             System.out.println(name + "：你是第" + num + "个使用timer的线程");
36         }
37     }
38 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**线程死锁的问题：**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 package cn.galc.test;
 2 
 3 /*这个小程序模拟的是线程死锁的问题*/
 4 public class TestDeadLock implements Runnable {
 5     public int flag = 1;
 6     static Object o1 = new Object(), o2 = new Object();
 7 
 8     public void run() {
 9         System.out.println(Thread.currentThread().getName() + "的flag=" + flag);
10         /*
11          * 运行程序后发现程序执行到这里打印出flag以后就再也不往下执行后面的if语句了 
12          * 程序也就死在了这里，既不往下执行也不退出
13          */
14 
15         /* 这是flag=1这个线程 */
16         if (flag == 1) {
17             synchronized (o1) {
18                 /* 使用synchronized关键字把对象01锁定了 */
19                 try {
20                     Thread.sleep(500);
21                 } catch (InterruptedException e) {
22                     e.printStackTrace();
23                 }
24                 synchronized (o2) {
25                     /*
26                      * 前面已经锁住了对象o1，只要再能锁住o2，那么就能执行打印出1的操作了 
27                      * 可是这里无法锁定对象o2，因为在另外一个flag=0这个线程里面已经把对象o1给锁住了 
28                      * 尽管锁住o2这个对象的线程会每隔500毫秒睡眠一次，可是在睡眠的时候仍然是锁住o2不放的
29                      */
30                     System.out.println("1");
31                 }
32             }
33         }
34         /*
35          * 这里的两个if语句都将无法执行，因为已经造成了线程死锁的问题 
36          * flag=1这个线程在等待flag=0这个线程把对象o2的锁解开， 
37          * 而flag=0这个线程也在等待flag=1这个线程把对象o1的锁解开 
38          * 然而这两个线程都不愿意解开锁住的对象，所以就造成了线程死锁的问题
39          */
40 
41         /* 这是flag=0这个线程 */
42         if (flag == 0) {
43             synchronized (o2) {
44                 /* 这里先使用synchronized锁住对象o2 */
45                 try {
46                     Thread.sleep(500);
47                 } catch (InterruptedException e) {
48                     e.printStackTrace();
49                 }
50                 synchronized (o1) {
51                     /*
52                      * 前面已经锁住了对象o2，只要再能锁住o1，那么就能执行打印出0的操作了 可是这里无法锁定对象o1，因为在另外一个flag=1这个线程里面已经把对象o1给锁住了 尽管锁住o1这个对象的线程会每隔500毫秒睡眠一次，可是在睡眠的时候仍然是锁住o1不放的
53                      */
54                     System.out.println("0");
55                 }
56             }
57         }
58     }
59 
60     public static void main(String args[]) {
61         TestDeadLock td1 = new TestDeadLock();
62         TestDeadLock td2 = new TestDeadLock();
63         td1.flag = 1;
64         td2.flag = 0;
65         Thread t1 = new Thread(td1);
66         Thread t2 = new Thread(td2);
67         t1.setName("线程td1");
68         t2.setName("线程td2");
69         t1.start();
70         t2.start();
71     }
72 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　解决线程死锁的问题最好只锁定一个对象，不要同时锁定两个对象

**生产者消费者问题：**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 package cn.galc.test;
 2 
 3 /*    范例名称：生产者--消费者问题
 4  *     源文件名称：ProducerConsumer.java
 5  *    要  点：
 6  *        1. 共享数据的不一致性/临界资源的保护
 7  *        2. Java对象锁的概念
 8  *        3. synchronized关键字/wait()及notify()方法
 9  */
10 
11 public class ProducerConsumer {
12     public static void main(String args[]){
13             SyncStack stack = new SyncStack();
14             Runnable p=new Producer(stack);
15             Runnable c = new Consumer(stack);
16             Thread p1 = new Thread(p);
17             Thread c1 = new Thread(c);
18             
19             p1.start();
20             c1.start();
21     }
22 }
23 
24 
25 class SyncStack{  //支持多线程同步操作的堆栈的实现
26     private int index = 0;
27     private char []data = new char[6];    
28     public synchronized void push(char c){
29         if(index == data.length){
30         try{
31                 this.wait();
32         }catch(InterruptedException e){}
33         }
34         this.notify();
35         data[index] = c;
36         index++;
37     }
38     public synchronized char pop(){
39         if(index ==0){
40             try{
41                 this.wait();
42             }catch(InterruptedException e){}
43         }
44         this.notify();
45         index--;
46         return data[index];
47     }
48 }
49 
50 
51 class  Producer implements Runnable{
52     SyncStack stack;    
53     public Producer(SyncStack s){
54         stack = s;
55     }
56     public void run(){
57         for(int i=0; i<20; i++){
58             char c =(char)(Math.random()*26+'A');
59             stack.push(c);
60             System.out.println("produced："+c);
61             try{                                    
62                 Thread.sleep((int)(Math.random()*1000)); 
63             }catch(InterruptedException e){
64             }
65         }
66     }
67 }
68 
69 
70 class Consumer implements Runnable{
71     SyncStack stack;    
72     public Consumer(SyncStack s){
73         stack = s;
74     }
75     public void run(){
76         for(int i=0;i<20;i++){
77             char c = stack.pop();
78             System.out.println("消费："+c);
79             try{                                       
80                 Thread.sleep((int)(Math.random()*1000));
81             }catch(InterruptedException e){
82             }
83         }
84     }
85 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

分类: [java基础总结](https://www.cnblogs.com/xdp-gacl/category/563690.html)

标签: [java基础总结](https://www.cnblogs.com/xdp-gacl/tag/java基础总结/)

[好文要顶](javascript:void(0);) [关注我](javascript:void(0);) [收藏该文](javascript:void(0);) [![img](https://common.cnblogs.com/images/icon_weibo_24.png)](javascript:void(0);) [![img](https://common.cnblogs.com/images/wechat.png)](javascript:void(0);)

[![img](https://pic.cnblogs.com/face/289233/20160221223904.png)](https://home.cnblogs.com/u/xdp-gacl/)

[孤傲苍狼](https://home.cnblogs.com/u/xdp-gacl/)
[关注 - 92](https://home.cnblogs.com/u/xdp-gacl/followees/)
[粉丝 - 20211](https://home.cnblogs.com/u/xdp-gacl/followers/)

[+加关注](javascript:void(0);)

16

0

[« ](https://www.cnblogs.com/xdp-gacl/p/3633936.html)上一篇： [java基础学习总结——线程(一)](https://www.cnblogs.com/xdp-gacl/p/3633936.html)
[» ](https://www.cnblogs.com/xdp-gacl/p/3634409.html)下一篇： [java基础学习总结——流](https://www.cnblogs.com/xdp-gacl/p/3634409.html)