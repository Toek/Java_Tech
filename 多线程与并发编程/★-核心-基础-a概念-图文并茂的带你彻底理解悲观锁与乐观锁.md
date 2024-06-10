## 图文并茂的带你彻底理解悲观锁与乐观锁                 

原创                                                                    安静的boy                                                             [                        Hollis                      ](javascript:void(0);)                       **Hollis**                               ![img]()                               微信号                               hollischuang功能介绍                               Hollis，一个对Coding有着独特追求的人。现任阿里巴巴技术专家，《程序员的三门课》联合作者。本公众号专注分享Java相关技术干货！我已加入“维权骑士”(rightknights.com)的版权保护计划                                                                                 *2019-05-05*

收录于话题

这是一篇介绍悲观锁和乐观锁的入门文章。旨在让那些不了解悲观锁和乐观锁的小白们弄清楚什么是悲观锁，什么是乐观锁。不同于其他文章，本文会配上相应的图解让大家更容易理解。通过该文，你会学习到如下的知识。



***1***



**锁（Lock）**

在介绍悲观锁和乐观锁之前，让我们看一下什么是锁。

锁，在我们生活中随处可见，我们的门上有锁，我们存钱的保险柜上有锁，是用来保护我们财产安全的。

程序中也有锁，当多个线程修改共享变量时，我们可以给修改操作上锁（syncronized）。

当多个用户修改表中同一数据时，我们可以给该行数据上锁（行锁）。因此，锁其实是在并发下控制多个操作的顺序执行，以此来保证数据安全的变动。

 并且，锁是一种保证数据安全的机制和手段，而并不是特定于某项技术的。悲观锁和乐观锁亦是如此。本篇介绍的悲观锁和乐观锁是基于数据库层面的。

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5Kric0aHYZda9XQicRAs7Gvxgo5p5wXTVib3X91UhbnSrrYmTHGTOraYWa1WwG9ibfWhHSbWcKv6Xtyfw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



***2\***



**悲观锁**

悲观锁**（Pessimistic Concurrency Control）**，第一眼看到它，相信每个人都会想到这是一个悲观的锁。没错，它就是一个悲观的锁。

那这个悲观体现在什么地方呢？悲观是我们人类一种消极的情绪，对应到锁的悲观情绪，悲观锁认为被它保护的数据是极其不安全的，每时每刻都有可能变动，一个事务拿到悲观锁后（可以理解为一个用户），其他任何事务都不能对该数据进行修改，只能等待锁被释放才可以执行。

数据库中的行锁，表锁，读锁，写锁，以及syncronized实现的锁均为悲观锁。

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5Kric0aHYZda9XQicRAs7GvxgTY8iagh7fufLbINXshpJyKCgMpV1delknxzHGLYzosicUWOlMYZOoH9A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这里再介绍一下什么是数据库的表锁和行锁，以免有的同学对后面悲观锁的实现看不明白。


我们经常使用的数据库是mysql，mysql中最常用的引擎是Innodb，Innodb默认使用的是行锁。3

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5Kric0aHYZda9XQicRAs7GvxgfCMZTZ9uEM1viakDtEjBrLXcZb7GttHwwBHL6iccfoYvRQCNFV1pianTw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 【悲观锁概念扩展深化】

#### 【Ref:"★-核心-基础-java对象结构-中的锁状态-锁粒度优化-- 偏向锁、轻量级锁、自旋锁、重量级锁(转载)-概念比较.md"】

悲观锁是就是悲观思想，即认为写多，遇到并发写的可能性高，每次去拿数据的时候都认为别人会修改，所以每次在读写数据的时候都会上锁，这样别人想读写这个数据就会block直到拿到锁。java中的悲观锁就是Synchronized,AQS框架下的锁则是先尝试cas乐观锁去获取锁，获取不到，才会转换为悲观锁，如RetreenLock。



***3\***



**乐观锁**

与悲观相对应，乐观是我们人类一种积极的情绪。乐观锁（Optimistic Concurrency Control）的“乐观情绪”体现在，它认为数据的变动不会太频繁。因此，它允许多个事务同时对数据进行变动。

 但是，乐观不代表不负责，那么怎么去负责多个事务顺序对数据进行修改呢？

乐观锁通常是通过在表中增加一个版本(version)或时间戳(timestamp)来实现，其中，版本最为常用。

事务在从数据库中取数据时，会将该数据的版本也取出来(v1)，当事务对数据变动完毕想要将其更新到表中时，会将之前取出的版本v1与数据中最新的版本v2相对比，如果v1=v2，那么说明在数据变动期间，没有其他事务对数据进行修改，此时，就允许事务对表中的数据进行修改，并且修改时version会加1，以此来表明数据已被变动。

如果，v1不等于v2，那么说明数据变动期间，数据被其他事务改动了，此时不允许数据更新到表中，一般的处理办法是通知用户让其重新操作。不同于悲观锁，乐观锁是人为控制的。

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5Kric0aHYZda9XQicRAs7GvxgddfYRfiaGVJnQhib2ZXa1meE4rKIqmIP6bv8V3iaJzjKlOkjZy7JSzNtg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 【乐观锁概念扩展深化】

#### 【Ref:"★-核心-基础-java对象结构-中的锁状态-锁粒度优化-- 偏向锁、轻量级锁、自旋锁、重量级锁(转载)-概念比较.md"】

乐观锁是一种乐观思想，即认为读多写少，遇到并发写的可能性低，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，采取在写时先读出当前版本号，然后加锁操作（比较跟上一次的版本号，如果一样则更新），如果失败则要重复读-比较-写的操作。

java中的乐观锁基本都是通过CAS操作实现的，CAS是一种更新的原子操作，比较当前值跟传入值是否一样，一样则更新，否则失败。





***4\***



**如何实现**

经过上面的学习，我们知道悲观锁和乐观锁是用来控制并发下数据的顺序变动问题的。那么我们就模拟一个需要加锁的场景，来看不加锁会出什么问题，并且怎么利用悲观锁和乐观锁去解决。

场景：A和B用户最近都想吃猪肉脯，于是他们打开了购物网站，并且找到了同一家卖猪肉脯的>店铺。下面是这个店铺的商品表goods结构和表中的数据。

| id   | name   | num  |
| ---- | ------ | ---- |
| 1    | 猪肉脯 | 1    |
| 2    | 牛肉干 | 1    |


从表中可以看到猪肉脯目前的数量只有1个了。在不加锁的情况下，如果A，B同时下单，就有可能导致超卖。

**悲观锁解决**

利用悲观锁的解决思路是，我们认为数据修改产生冲突的概率比较大，所以在更新之前，我们显示的对要修改的记录进行加锁，直到自己修改完再释放锁。加锁期间只有自己可以进行读写，其他事务只能读不能写。

A下单前先给猪肉脯这行数据（id=1）加上悲观锁（行锁）。此时这行数据只能A来操作，也就是只有A能买。B想买就必须一直等待。

当A买好后，B再想去买的时候会发现数量已经为0，那么B看到后就会放弃购买。

那么如何给猪肉脯也就是id=1这条数据加上悲观锁锁呢？我们可以通过以下语句给id=1的这行数据加上悲观锁

```
select num from goods where id = 1 for update;
```

下面是悲观锁的加锁图解

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5Kric0aHYZda9XQicRAs7Gvxg2ibU1ia4UotXsAm1Cia2tvAQe65GZ8TsnlzVSZBFW0Ak87IHNkriaiciayBA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们通过开启mysql的两个会话，也就是两个命令行来演示。
1、事务A执行命令给id=1的数据上悲观锁准备更新数据

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5Kric0aHYZda9XQicRAs7GvxgXq6WqdH2M3Jycf6Rx3DnxjpGRyR0iczocEMkQvQCRhmNggMSjaT9Fag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



这里之所以要以begin开始，是因为mysql是自提交的，所以要以begin开启事务，否则所有修改将被mysql自动提交。

2、事务B也去给id=1的数据上悲观锁准备更新数据

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KJNeM89lUxibYRPNZOHYiaAleEQjCTTueRGWgugZs379ZiaiaEptuF5DXlZc0V56dxzaA8ygLgtJfwPg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



我们可以看到此时事务B再一直等待A释放锁。如果A长期不释放锁，那么最终事务B将会报错，这有兴趣的可以去尝试一下。

3、接着我们让事务A执行命令去修改数据，让猪肉脯的数量减一，然后查看修改后的数据，最后commit,结束事务。

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KJNeM89lUxibYRPNZOHYiaAlDcbKhnjk6NDmXWzdl86KsFScoZkbia1b0NLJESC6BmCtgIzJrVgthhw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



我们可以看到，此时最后一个猪肉脯被A买走，只剩0个了。


4、当事务A执行完第3步后，我们看事务B中出现了什么

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KJNeM89lUxibYRPNZOHYiaAlib4DbiaKInBdy1d47DVYicMEd8JQz2KVoEv6mC3a7HzneXQO9uk6BfSJQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



我们看到由于事务A释放了锁，事务B就结束了等待，拿到了锁，但是数据此时变成了0，那么B看到后就知道被买走了，就会放弃购买。


通过悲观锁，我们解决了猪肉脯购买的问题。

**乐观锁解决** 

下面，我们利用乐观锁来解决该问题。上面乐观锁的介绍中，我们提到了，乐观锁是通过版本号version来实现的。所以，我们需要给goods表加上version字段，表变动后的结构如下：

| id   | name   | num  | version |
| ---- | ------ | ---- | ------- |
| 1    | 猪肉脯 | 1    | 0       |
| 1    | 牛肉干 | 1    | 0       |

使用乐观锁的解决思路是，我们认为数据修改产生冲突的概率并不大，多个事务在修改数据的之前先查出版本号，在修改时把当前版本号作为修改条件，只会有一个事务可以修改成功，其他事务则会失败。

A和B同时将猪肉脯(id=1下面都说是id=1)的数据查出来，然后A先买，A将id=1和version=0作为条件进行数据更新，即将数量-1，并且将版本号+1。

此时版本号变为1。A此时就完成了商品的购买。最后B开始买，B也将id=1和version=0作为条件进行数据更新，但是更新完后，发现更新的数据行数为0，此时就说明已经有人改动过数据，此时就应该提示用户重新查看最新数据购买。

下面是乐观锁的加锁图解  

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KJNeM89lUxibYRPNZOHYiaAlCzHOQoHGFNf52aZpL3phhor488HeKLJPlgPHozbABbxvBPnJ09xbmA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们还是通过开启mysql的两个会话，也就是两个命令行来演示。

1、事务A执行查询命令，事务B执行查询命令，因为两者查询的结果相同，所以下面我只列出一个截图。

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KJNeM89lUxibYRPNZOHYiaAlJocOVVvFcxIIhm6IKTpPVohx8mBzIxJQWxzcpxHAEyGZl79P99zVMA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



此时A和B均获取到相同的数据

2、事务A进行购买更新数据，然后再查询更新后的数据。

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KJNeM89lUxibYRPNZOHYiaAlTdKooQkiaqnniaRApRoQO4GnXldn3ia1Zl22s06pCBbibEFIyhFnhIbEhg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



我们可以看到事务A成功更新了数据和版本号。



事务B再进行购买更新数据，然后我们看影响行数和更新后的数据 

![图片描述](https://mmbiz.qpic.cn/mmbiz_png/6fuT3emWI5KJNeM89lUxibYRPNZOHYiaAl1NVib5dVPlPaB9YGESc8KYNwazhnztAlKKV7XgXD2YK4x71bVWFVR0Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



可以看到最终修改行数为0，数据没有改变。此时就需要我们告知用户重新处理。



***5\***



**优缺点**

下面我们介绍下乐观锁和悲观锁的优缺点以便我们分析他们的应用场景，这里我只分析最重要的优缺点，也是我们要记住的。

**悲观锁**



- 优点：悲观锁利用数据库中的锁机制来实现数据变化的顺序执行，这是最有效的办法
- 缺点：一个事务用悲观锁对数据加锁之后，其他事务将不能对加锁的数据进行除了查询以外的所有操作，如果该事务执行时间很长，那么其他事务将一直等待，那势必影响我们系统的吞吐量。

**
**

**乐观锁**

**
**

- 优点：乐观锁不在数据库上加锁，任何事务都可以对数据进行操作，在更新时才进行校验，这样就避免了悲观锁造成的吞吐量下降的劣势。

- 缺点：乐观锁因为是通过我们人为实现的，它仅仅适用于我们自己业务中，如果有外来事务插入，那么就可能发生错误。

  

  

***6\***



**应用场景**



悲观锁：因为悲观锁会影响系统吞吐的性能，所以适合应用在写为居多的场景下。



乐观锁：因为乐观锁就是为了避免悲观锁的弊端出现的，所以适合应用在读为居多的场景下。



参考资料 

https://chenzhou123520.iteye.com/blog/1860954 

https://baike.baidu.com/item/乐观锁/7146502



本文来自作者投稿，原作者：安静的boy，Hollis做了简单的修改及排版。转载请注明作者及出处。



![动态黑色音符](https://mmbiz.qpic.cn/mmbiz_gif/6fuT3emWI5JdsbFN9yGeibj0I2DgXCfuW10VfoHooX8s9K6N0t8GGtoRfeicDo9WSoJSO5pVDMjUb1Xd6aIaZpicQ/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)



[Java工程师成神之路系列文章](http://mp.weixin.qq.com/s?__biz=MzI3NzE0NjcwMg==&mid=2650122991&idx=1&sn=350ef209d0677ba8a5e8e2968eaa4874&chksm=f36bb7cec41c3ed895a796f88139143c8a2881c6d16c4d35b831ba3fc53a477923ef79e80f3b&scene=21#wechat_redirect)

在 GitHub 更新中，欢迎关注，欢迎star。

 ![img](https://mmbiz.qpic.cn/mmbiz_jpg/6fuT3emWI5ITEgCp0uQdM0cZSVQd0shMGgUfezj8QRflooGGQf6szoHn9icAoY8WA399hSlIpvibJMnib7kcx6rtg/640?wxfrom=5&wx_lazy=1&wx_co=1)

直面Java第234期：缓存一致性就是Java并发编程中的可见性吗？

成神之路第015期：设计模式：单例模式

深入并发第008期：到底什么是计算机内存模型？

**- MORE | 更多精彩文章 -**

- [我博客中带密码的文章怎么访问？](http://mp.weixin.qq.com/s?__biz=MzI3NzE0NjcwMg==&mid=2650123847&idx=1&sn=1a321c0f2ff65e4162398092609d68da&chksm=f36bb366c41c3a705893ba54a8ede46e67a98bc29a0d42268415be63408d3f40d8349638a249&scene=21#wechat_redirect)
- [腾讯面试题：一条SQL语句执行得很慢的原因有哪些？](http://mp.weixin.qq.com/s?__biz=MzI3NzE0NjcwMg==&mid=2650123840&idx=1&sn=a3965b6732c4aae2e738ea5838a38cb4&chksm=f36bb361c41c3a7787f9b109f99f0ab39f175b527047bfc8dd8f4dc1e00d9d3f2de934ddc385&scene=21#wechat_redirect)
- [Spring Cloud Alibaba 发布 GA 版本，Spring 再发贺电](http://mp.weixin.qq.com/s?__biz=MzI3NzE0NjcwMg==&mid=2650123832&idx=2&sn=dff9291657a1d4ee427d884e805dac7b&chksm=f36bb319c41c3a0f380d61bcc21bcfb7a36dff0b250c0ad6ca1bb16c0a173913172a69205ae1&scene=21#wechat_redirect)
- [面试问烂的 MySQL 四种隔离级别，看完吊打面试官！](http://mp.weixin.qq.com/s?__biz=MzI3NzE0NjcwMg==&mid=2650123822&idx=2&sn=839363e712e2d0069ee3c673cd3686e6&chksm=f36bb30fc41c3a1924f1b3a587da7b9db6af2f1614459cf770f749a24f817d1687e34746251c&scene=21#wechat_redirect)