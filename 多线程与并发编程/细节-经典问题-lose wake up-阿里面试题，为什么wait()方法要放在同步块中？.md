## 阿里面试题，为什么wait()方法要放在同步块中？

原创 丹霞仙君 [占小狼的博客](javascript:void(0);) *2019-04-12*

点击上方“占小狼的博客”，选择“[设为星标](http://mp.weixin.qq.com/s?__biz=MzU0MzQ5MDA0Mw==&mid=2247484785&idx=1&sn=adfe86f12ffd8ab255e635cac5db67f5&chksm=fb0befe5cc7c66f3a7c5e132f6005135f530d1dbc1788da00382307b1c325ac3fa600ce8f9d8&scene=21#wechat_redirect)”

![img](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04mpCaOPaoDI7Qb1REWWJZWibn41vXdAyiaMQaeQ9iawfibT8qqs9icNWbQgnfnkTzyiaocfXWpghLJZeeSA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

某天我在***的时候，突然有个小伙伴微信上说：“哥，阿里面试又又挂了，被问到为什么wait()方法要放在同步块中，没答出来！”

![img](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04mRCTMdh086iakWYlIPM7BeTbzDgegCKzHnJvCVKdMr4hZicBtWZqrvMAcEHCMmzhrduJKEnH3ECzeQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我顿时觉得**一紧，仔细回顾一下，如果wait()方法不在同步块中，代码的确会抛出异常:

```
public class WaitInSyncBlockTest {
    @Test    public void test() {        try {            new Object().wait();        } catch (InterruptedException e) {            e.printStackTrace();        }    }}
```

结果是：![img](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04mRCTMdh086iakWYlIPM7BeTKyrRqsfpdTqglva1qLkHrr2f25xIbU3lrtbqNgqUAwOHZNdcl2vzSQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

但是，为毛呢？？我也没去了解过。

机智如我立刻假装正在开会忙得不可开交，回了一条：“开会中，等会和你细说。”

![img](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04mRCTMdh086iakWYlIPM7BeTyY5hIvQ8ADPWVWONq5q48AJShYqz5BweAo92AMpd0Ov9zoeIicZc4nQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

经过一番谷歌之后，找到了答案。

## Lost Wake-Up Problem

事情得从一个多线程编程里面臭名昭著的问题"Lost wake-up problem"说起。

这个问题并不是说只在Java语言中会出现，而是会在所有的多线程环境下出现。

假如有两个线程，一个消费者线程，一个生产者线程。生产者线程的任务可以简化成将count加一，而后唤醒消费者；消费者则是将count减一，而后在减到0的时候陷入睡眠：

生产者伪代码：

```
count+1;notify();
```

消费者伪代码：

```
while(count<=0)   wait()
count--
```

熟悉多线程的朋友一眼就能够看出来，这里面有问题。什么问题呢？

生产者是两个步骤：

1. count+1;
2. notify();

消费者也是两个步骤：

1. 检查count值；
2. 睡眠或者减一；

万一这些步骤混杂在一起呢？比如说，初始的时候count等于0，这个时候消费者检查count的值，发现count小于等于0的条件成立；就在这个时候，发生了上下文切换，生产者进来了，噼噼啪啪一顿操作，把两个步骤都执行完了，也就是发出了通知，准备唤醒一个线程。这个时候消费者刚决定睡觉，还没睡呢，所以这个通知就会被丢掉。紧接着，消费者就睡过去了……

![img](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04mRCTMdh086iakWYlIPM7BeTDuzUZkeVG3cr5cnZZuAMyISxQ5zKRgpibkQQEIyf5Fv2EfETibnssEIA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这就是所谓的lost wake up问题。

## 那么怎么解决这个问题呢？

现在我们应该就能够看到，问题的根源在于，消费者在检查count到调用wait()之间，count就可能被改掉了。

这就是一种很常见的竞态条件。

很自然的想法是，让消费者和生产者竞争一把锁，竞争到了的，才能够修改count的值。

于是生产者的代码是:

```
tryLock()
count+1
notify()
releaseLock()
```

消费者的代码是:

```
tryLock()
while(count <= 0)  wait()
count-1
releaseLock
```

注意的是，我这里将两者的两个操作都放进去了同步块中。

现在来思考一个问题，生产者代码这样修改行不行？

```
tryLock()
count+1
notify()
releaseLock()
```

答案是，这样改毫无卵用，依旧会出现lost wake up问题，而且和无锁的表现是一样的。

## 终极答案

所以，我们可以总结到，为了避免出现这种lost wake up问题，在这种模型之下，总应该将我们的代码放进去的同步块中。

Java强制我们的wait()/notify()调用必须要在一个同步块中，就是不想让我们在不经意间出现这种lost wake up问题。

不仅仅是这两个方法，包括java.util.concurrent.locks.Condition的await()/signal()也必须要在同步块中：

```
    private ReentrantLock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();
    @Test    public void test() {        try {            condition.signal();        } catch (Exception e) {            e.printStackTrace();        }    }
```

![img](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04mRCTMdh086iakWYlIPM7BeTq2iaqhzDPm8QOu3Wr3aibd6NS7kKSiaxiaH2KhYW40F2XPcPH9fXvqzEaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

准确的来说，即便是我们自己在实现自己的锁机制的时候，也应该要确保类似于wait()和notify()这种调用，要在同步块内，防止使用者出现lost wake up问题。

Java的这种检测是很严格的。它要求的是，一定要处于锁对象的同步块中。举例来说：

```
    private Object obj = new Object();
    private Object anotherObj = new Object();
    @Test    public void produce() {        synchronized (obj) {            try {                anotherObj.notify();            } catch (Exception e) {                e.printStackTrace();            }        }    }
```

这样是没有什么卵用的。一样出现IllegalMonitorStateException。

## 可以拿去套路面试官的话术

到这里，按照道理来说，就可以结束了。不过既然是面试遇到的问题，我就提供点面试回答的小技巧。

假如面试官问你这个问题了，你最开始不要巴啦啦全部说出来。只需要轻描淡写地说：“这是Java设计者为了避免使用者出现lost wake up问题而搞出来的。”

注意演技，一定要轻描淡写中透露着一丝“我其实就知道lost wake up这个名词，再问就要露馅了”的感觉。

于是面试官肯定会追问：“lost wake up问题是什么？”

这个时候你就可以巴啦啦一大堆了。这个过程你要充满自信，表露出那种睥睨天下这种小问题就别来烦我的气概来。

于是，小手一抖，offer到手。

![img](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04mRCTMdh086iakWYlIPM7BeTOzJA3zWXgEcWvwuKarnCsFyE6FmZFWibQm0rE5u7pBuEORjtEdVMfrg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

------

  

分享一份面试宝典**《Java核心知识点整理.pdf》**，覆盖了JVM、锁、高并发、反射、Spring原理、微服务、Zookeeper、数据库、数据结构等等。

获取方式：关注公众号并回复 **666** 领取，更多内容陆续奉上，敬请期待。





 近期热文：

- ## [LinkedIn 从CMS到G1的调优实战，实现毫秒级响应](http://mp.weixin.qq.com/s?__biz=MzIwMzY1OTU1NQ==&mid=2247485858&idx=1&sn=dad86c9459224bc8a8a2721af2afe3bd&chksm=96cd49eea1bac0f8c15fc1ac7cea55b8de48ec74e6e3a47e954d464561051223010fe14db205&scene=21#wechat_redirect)

- ## [JVM Code Cache空间不足，导致服务性能变慢](http://mp.weixin.qq.com/s?__biz=MzIwMzY1OTU1NQ==&mid=2247485853&idx=1&sn=3d52c9ac77fc237b55676ad0ad7c9f28&chksm=96cd49d1a1bac0c72ce28e365703c5fd0e628bc55cdc1c2df2121ca28deb437eebfa7a4e17c1&scene=21#wechat_redirect)

- ## [Spring StateMachine，教你快速实现一个状态机](http://mp.weixin.qq.com/s?__biz=MzIwMzY1OTU1NQ==&mid=2247485848&idx=1&sn=8dd53ab4001cf833c3820524a0f248ac&chksm=96cd49d4a1bac0c2f51487e16be4314145d6590d3b3f205a26274db3fc92a453176fedc66a87&scene=21#wechat_redirect) 



![img](https://mmbiz.qpic.cn/mmbiz_jpg/8Jeic82Or04mluBobdJvQh1QklGQ7GFPWngZZlRX7MJqTVHDyoAo6fpahaaZwsJ3F6iaKEDpNvekJLDdedCIlF1A/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

 

怼它一下？![img](https://mmbiz.qpic.cn/mmbiz_png/OX4G1a1hUmVfPibic1DcJK3e9AqiawiaPxGfbeLNnAYf1fQ3PmynGGo4kckcictMf5kFY5E3UQfwDZx6ick0dn4PB3Mw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

阅读 6662

赞在看37

![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9jFqejics0IfuGEFHrrOWtFiciaibfEFzWg3QEGbCM48ibtpjUwqSWp7qhgKFPiaynLoPaBtoCNq0b2RkzaVXDZMV8a4a/132)

写下你的留言

**精选留言**

- ![img](http://wx.qlogo.cn/mmopen/Q3auHgzwzM7Mw667ECvCXuGL3EuWR7icbkLnX9luMfInRmyC92s4Nkeu5WDn5k5axHEE1v0jqx3ibFSSt0IHFxTQ/96)**lee**

  3

  

  感觉狼哥公众号很优质

  作者

  35

  把前面两个字去掉？

- ![img](http://wx.qlogo.cn/mmopen/Q3auHgzwzM6CwIxbwb0b84hIKJ4SsHJUsicgOxLjzOmCGfYGS8pcMmnHics1mb6uIYefx0BCtK9e7BrMkgrDTB3A/96)**一方通行**

  9

  

  于是被刨根问题专家组问的底，面试官：你行不行啊小老弟。

- ![img](http://wx.qlogo.cn/mmopen/ajNVdqHZLLAia4ztVtriaMXDzdPnYamYgr9Wqom6ibicoiaYYHufvH9u5Qu7vt2JMHzohFqkZy22u9mDhppStx7Mjmg/96)**½ ²⁰²⁰**

  

  问题来了，怎么用短简的话让面试官了解呢

  作者

  6

  苦练基本功

- ![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9gSQRzj9afosOSgyCW4mLgzcjG5x4xQ1QEUmtYT5TucSDdx7xNiciaib8yciajLPOYELh6REXibzUiaHLu7DpYYSjuxYb/96)**Sirius**

  4

  

  tryLock() while(count<=0){wait()} releaseLock 同一把锁的话恐怕在count=0时就死锁了，不是同一把就和没有同步逻辑一样了

- ![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9h62mHvXsOg3h4NZjck2yyyT26HiaicBSStyL85j2aZa8YDZaIYJ92wawJ9kTRmpdt7cib6A9fbJEcBKcRJAPcEtgA/96)**赵越**

  3

  

  戏精程序员

- ![img](http://wx.qlogo.cn/mmopen/MBb8mySl0e9pm5q2DBu1BmKZ7wCRicfCorGR7VQt6YQwOrYibR6cx8bBSn7pCVvg9FDlHVL6mIVqYakoZjj7bx4SL1ycdlPFE9/96)**如今与你**

  3

  

  我一直以来的理解是，每个锁都对应一个线程阻塞队列，只有在锁的同步块中才知道等待的线程放入那个锁的等待队列，唤醒操作也才会知道去唤醒那个阻塞队列中的线程，Condition用的也是这个原理来实现的指定线程唤醒，每个Condition都对应一个阻塞队列

- ![img](http://wx.qlogo.cn/mmopen/JCfzMXOeFZxWxtHq2cEiaVrW3pNxEQsblBf6m01KBTia7WGHZUFACm9orRooHM6M5SO60jMbiah2ZYicymJtVkfoNA/96)**mc**

  2

  

  直接回答，编译报错不可以吗？？

  作者

  2

  下一个

- ![img](http://wx.qlogo.cn/mmopen/JCfzMXOeFZyulOyb2vCJZiaiceBKFxib5a7d1997coRsQSbPeQINMpJkQYZRUIEYbVjN0kgEsiczibECiaYaSpuOjLjYQPicHTKviawT/96)**九**

  2

  

  ***是啥

  作者

  1

  嘘

- ![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9h62mHvXsOg3p8NJaWdxkrOTRScWZAvmLNFQS3cia8ZuS5NGLbfZiaWE5knmRF0McmKXeLoP1N2KrLIZA9gibDTicP3/96)**Wuyang**

  2

  

  小手一抖，offer 到手

- ![img](http://wx.qlogo.cn/mmopen/MBb8mySl0e850oWlfFeNHzMeUbLYeBiczPJA1icOjOBtU6Sw9lVquwZbnPGHfPicSiby4ysSnF1YOoUZKklGfC8ypzg8TiakEIKzJ/96)**鲸息**

  1

  

  极客时间的专栏，狼哥你之前推的有点过猛了。这篇挺好，赞。

  作者

  1

  哈哈，我也是选择性推合适的，看不上的就不推

- ![img](http://wx.qlogo.cn/mmopen/MBb8mySl0e850oWlfFeNH8nlTHbMEoGfVzwXW4dEztyQk7DyuPOpFiblJs3ZrSe4APbQPcZ81EZbo6l0bsudB2Z6cBPpU2YMg/96)**朝林摆渡**

  

  狼哥，最后一个例子都syn object了，为啥还会抛异常？

  作者

  1

  要同一个对象

- ![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9h62mHvXsOg3miblsDiap6DibeS37hE8vSNxdiaA1uqqDbBiaZ1IFUL0dpo4WXgTkAuiaYcyw3N6vW2XztFSCTzt39Aibo/96)**黄振涛**

  1

  

  不行的那个生产者代码跟行的一样

- ![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9g4YcFTTO3oeuFCich2rWx7mHiaQoeupOa8bXe0Hm2KicnbpfzABRbYkcLw2XegsfCbRibgI7evzyGib622awS44JCJH/96)**独角没有戏**

  

  把Object这个类里面wait、notify两个native方法上的注释先看一遍。 就觉得这个问题很怪，jdk注释解析的是这两个方法是和“监视器对象”相关的，这篇文章好像没有提到。 所以如果能按照jdk所述的方法使用，这个lost wake up问题也就不需要知道了。注释里也没提到这个字眼。并且按jdk原本的意思，不仅wait要在同步块里使用，而且还要包装在loop里，题目只出了一半？

- ![img](http://wx.qlogo.cn/mmopen/MBb8mySl0e850oWlfFeNHxwj70kYvTiapCBjCria1sw8ROdpFNqg2Tju1V2hEfoNibWY9YV9jBfV9GNVpoQQAp6sxcltZKSKhIn/96)**只是路无尽头都是路过**

  

  所以说面试的时候是把自己知道的全说出来，还是留下引子让面试官问🤔🤔🤔

- ![img](http://wx.qlogo.cn/mmopen/MBb8mySl0eicKdwQmOFScnz8g0lA9HkBCbr0sCmjicHwHRvlG31fhfkbd45nxCLj93rvDoIHETheXNzQ9LfpicgKWgflWOFUNPW/96)**张三的锅锅**

  

  终极答案上的第一个代码和终极答案上的第三个代码不是一样的吗？

  作者

  只是强调一下

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RCGkkH27pmvqBQEQAAnk40cPTzF2DBfjMW8SwyI2BlTVpGvgTcKJt6VotzxLxdmWFDMnfF9IGia5w/96)**huxi_2b**

  

  强👍 有些文章颇有R大的风范

  作者

  你这吹的太高了，R大的语言功底是没法比的

- ![img](http://wx.qlogo.cn/mmopen/MBb8mySl0e9bUj8OxSibojdWOCw5ISK7UnMx4Bky0tNibczMJXp81SicpzjoFmt97yEuDwL5uBuatg9Q8loFsTIfu6YdL1uu9Ut/96)**Poison**

  

  这个必须赞，多搞点面试实战

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RXXeFMiaDvdsBb7cowgNVxeRLsa5KZuhPsH6hbp6NjzNP7hvvEsvVfexUqMuiajbs7uDqQMDianewNAQoKJqGeK1X/96)**玩图思瑞佛**

  

  一直以为之所以放到同步块里是因为wait的底层实现是借助于ObjectMonitor的wait方法来实现，不获取锁哪来的ObjectMonitor

- ![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9gPwfeDj5xnt5a0ibicrMtzvwUHxoca5mGHm5ibYvoaJHpn4kURHlo8bTicsQlIntlCoZ16etjsoaw8jHrusy1S5ialg/96)**问风问云**

  

  狼哥。请教个问题，如果同步代码块里面用到了共享变量比如某一个Object，那么进入同步代码块的瞬间，会从主内存读取该对象的属性值吗？同步代码块结束会同步更新到主内存吗？

  作者

  那一瞬间不会，只有读到了某个共享变量，才有可能去主内存读取

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RXXeFMiaDvdsCfNEp09TOKCMCpBKSKW0xX98MSibhFrHud6OLUyK68EYXJJOyNxFHic3HvFkHlnndAAQOSgdzTicou/96)**MaDdTrFa**

  

  图没画完吧？第一个没有表示不是同一把锁  第二个没有表示不是锁同一个对象

  作者

  示意图

- ![img](http://wx.qlogo.cn/mmopen/I2IJrcQraYMymT7fwz2Mc1jMFOHSmZsNicR1Yku3x9TQn9aP9duNUeldY9Dmia7JV4xadyJsNRFCQHVKGaZJGR70DhoHOnic8xD/96)**Amazing**

  

  面试总被如此套路

- ![img](http://wx.qlogo.cn/mmopen/ajNVdqHZLLDGjz83jiaKqpicEUM1uffO0TZTaObe5Q8ibRxprM5XcO21EB2cm3MZFOaOWSCHRqCpdnMhRr570fbaQ/96)**TSN**

  

  演员的自我修养

- ![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9gAxRXB14sUx9TAFznLV37oGDf1ibjgz3mJh0cOHcrB3VTVeBottSL34t7icC1FaD3q2B0waQ1Fibgicw/96)**笨小孩**

  

  不行的那个生产者代码跟行的一样？

  作者

  只是强调一下

- ![img](http://wx.qlogo.cn/mmopen/MBb8mySl0e9ydqJCrWYBgrBSIelnj30uxFAC6NjUtUqialqoEWm5nc0cWLdibw3LhicHiaiaTXH1tzvBhTvyu9icuzcGRaJ3BJzVFF/96)**小天哥哥**

  

  最后的演技部分很有技巧。

- ![img](http://wx.qlogo.cn/mmopen/SIq2e0xicA9h62mHvXsOg3ncnlubwf863eibzw3cYPSPefu9TyZGNXc4Wj1pF1YlCUQDmqT6QO25h7xSNyWBJWhSdiauREaTzCD/96)**z.l**

  

  这样理解可以么 count+1;notify(); 必须是一个原子操作，所以要同步。否则就线程不安全了，包括消费者的代码也一样。

  作者

  也不能这样去理解

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RAepKR4BOcOjrMKWUuib1yicicdkE0jyAAYcIjLHjhcJSVAQOkVGibcP71VbbHsmRHX7OhYcor8XFt5Z7Mw2riaZ401/96)**it_lanxy**

  

  同一个ReetLock对象的try lock就可以了吧

  作者

  你可以试试，验证一下

- ![img](http://wx.qlogo.cn/mmopen/Q3auHgzwzM6ib2d9QHJVbu8zDlvjunA4bT7Oq5gfLKD6svU9XMuEDW1KrOyvPvDVK306R0vvicaNCVBUJbzxbXOwrwYibZFeSQbzrJHYo4FAibU/96)**ℓ 微尘℃**

  

  资料很好！！狼哥牛逼

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RAepKR4BOcOjrMKWUuib1yicicdkE0jyAAYcIjLHjhcJSVAQOkVGibcP71VbbHsmRHX7OhYcor8XFt5Z7Mw2riaZ401/96)**it_lanxy**

  

  狼哥，生产者那么改为啥没用啊，都上锁了啊

  作者

  不是同一把锁

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RXXeFMiaDvdsC0WphpcR2QDFrANTc6mxKmoQZQSmO9TUmCiblQZzkrwlBsB2mIEiafLJefzibibC9ZmedQvLicg7O12x/96)**渔樵**

  

  嗯，我提问是想跟大佬请教下，我对这个问题的理解是不是对的

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RXXeFMiaDvdsC0WphpcR2QDFrANTc6mxKmoQZQSmO9TUmCiblQZzkrwlBsB2mIEiafLJefzibibC9ZmedQvLicg7O12x/96)**渔樵**

  

  是啊，所以只能在同步代码内调用锁对象的notify、wait

  作者

  对的，首先要拿到锁

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RXXeFMiaDvdsL2fXSZUicRNeYRCNxHMXtIcs4Z8wticasUic2UZDuDza2kEQ1KN4iaMpAPN5rSzp2yLibHq8hLHSS3tJ/96)**椰子船长**

  

  trylock那种方式为什么不行？

  作者

  看后面

- ![img](http://wx.qlogo.cn/mmopen/MBb8mySl0e850oWlfFeNH7m1AL0GaG0libyw2nXMaddgVN3ykP4xIww8JtWcy62iacTHt5pflfhIOShicfaMlzLG6ku9Fvy6yp1/96)**近南**

  

  还可以说下线程虚假唤醒  在java并发编程实战都提到过

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RXXeFMiaDvdsC0WphpcR2QDFrANTc6mxKmoQZQSmO9TUmCiblQZzkrwlBsB2mIEiafLJefzibibC9ZmedQvLicg7O12x/96)**渔樵**

  

  可能我没表述清楚，我的意思是按照现在的java实现，调用notify、wait的对象和synchronized锁对象必须要是同一个。 那如果假设notify和wait可以传入指定的锁对象，那是不是显式锁也能用了，和synchronized一样了呢？还是说还会发生lostwakeup...

  作者

  wait方法怎么传入指定的锁对象呢？可以动手验证看看

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5RXXeFMiaDvdsC0WphpcR2QDFrANTc6mxKmoQZQSmO9TUmCiblQZzkrwlBsB2mIEiafLJefzibibC9ZmedQvLicg7O12x/96)**渔樵**

  

  显式锁应该可以解决竞态条件吧，只不过现在的java实现是调用notify、wait的对象和同步代码块的锁对象要一致，如果notify和wait方法能够通过参数指定锁对象，显式锁是不是就可以了？

  作者

  上代码

- ![img](http://wx.qlogo.cn/mmopen/naPHoFY2n5T75QPmxoqt6bXJ0mQEHDcxphUa94zS9TibULVzcOcUhiaZmwTd1G7MFTp7t3VicvQLdoeLTMcsw7uIA/96)**TreeNewBee**

  

  简单的说，wait需要对象锁，你可以用sleep,sleep可以不用对象锁

- ![img](http://wx.qlogo.cn/mmopen/JCfzMXOeFZyKUbEnZXIXeUHcnrGSSkG20Fg2MO9GBKJvyDmNJarq9AOa2vBtg9vu9RLpicO0CxTY6bHyBebicyiaCHkHzliaPZhJ/96)**Ivan**

  

  哈哈哈哈哈演技受教了！

- ![img](http://wx.qlogo.cn/mmopen/JCfzMXOeFZwcmafyhUJDUBYK5ib9tvKog8rCLtHaFW4f8GQJmOBxkvYusoXlsRKZjRHcaVyjEdbiaaYgxXa4XxLEibtXV8tt4yT/96)**darkholder**

  

  小手一抖，offer到手～哈哈

- ![img](http://wx.qlogo.cn/mmopen/ajNVdqHZLLBjLWtgCh2xja7zmP87YfoibOSJgcSlJEYeWLjByJp3nsGbWU5dq2Oz4ZZlcSpXjqYowicpeVUTNDHQ/96)**Ziyear**

  

  666