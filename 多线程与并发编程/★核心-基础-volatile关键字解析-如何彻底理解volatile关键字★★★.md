## 如何彻底理解volatile关键字？

[JAVA葵花宝典](javascript:void(0);) *2019-04-15*

以下文章来源于无敌码农 ，作者无敌码农

[![无敌码农](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6xsn8Z1WFaXAqoVKibRy1eo50q354qyYuQyXibR1RYQBhw/0)**无敌码农**一线互联网公司资深码农一枚，主要专注于互联网系统设计、技术细节经验交流、工程管理、前沿技术发展等方向。在这里作者将结合自身的技术成长经历与思考，并借鉴其他优秀朋友的经验，为大家呈现精彩的技术内容，欢迎关注交流。](https://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049343&idx=2&sn=75dc6c2d60d5d4a5165b8cd7262622d9&chksm=87534cccb024c5da86b90ecc4919831ed34924695d0260fd7c62355e52d636751e7f4edd341a&mpshare=1&scene=24&srcid=&key=ef51a5b0b69d9d3052bc2de7832709d1228d327b4daf562f9a811aee2da2b5045a7a3768fafff54a438a57dc74995b4a1fa56bcfb1cbabdf1cba0b3b44135e60a8ca155caf64dbbb2ba953bf0c0e23e2692d0f7a4ecef0c8b0868ecccfd14c87bf21e592c0cad7f486a03558d04fdb6f83f33051c20cf4252873ab63c73c4268&ascene=14&uin=MTMwOTY3MTQ4Mg%3D%3D&devicetype=Windows+10+x64&version=62090529&lang=zh_CN&exportkey=A92KmyuLuYV1MkXPgiTLBg4%3D&pass_ticket=cZmru81J9saVFU4NF3Up8sapWqwJjFKVm4KKanZudIJIfcbAHCY11rwfNQxOP9xE&wx_header=0&winzoom=1#)

![img](https://mmbiz.qpic.cn/mmbiz_png/l89kosVutomBBhUdGiaFEc5TdZKIdF6G73bN9zTcvYbnqGDiaBQSfOuemS4nWLosEGdEIuYsFB8XNfRFyvCPSfUA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### **导读**

------



最近面试，你又被volatile关键字虐了吗？这个问题，是不是问得有点扎心了！的确，有很多朋友反馈面试中在涉及考察Java并发编程知识的时候，经常会被问到volatile关键字。对于有些公司如果你能回答出volatile关键字的基本作用及原理，如:"**volatile关键字可以实现线程间的可见性，之所以可以实现这一点，原因在于JVM会保证被volatile修饰的变量，在线程栈中被线程使用时都会主动从共享内存(堆内存/主内存)中以实时的方式同步一次；****另一方面，如果线程在工作内存中修改了volatile修饰的变量，也会被JVM要求立马刷新到共享内存中去。因此，即便某个线程修改了该变量，其他线程也可以立马感知到变化从而实现可见性****"**也基本上能够pass这个问题。



但是如果你去阿里这样喜欢刨根问底的公司面试的话，可能这点料就不够用了，因为面试官很可能会问到你更深层次的原理，如果没有彻底理解volatile关键字的话，在这个问题上迟早还是会栽跟头。因此，小码哥决定好好刨一下volatile关键字的根，希望能够对你起到帮助！

#### **初识volatile**

------



下面就让我们循序渐进地逐步了解volatile关键字吧！先了解下volatile的关键字都用在代码的什么地方："**volatile关键字只能修饰类变量和实例变量。****方法参数、局部变量、实例常量以及类常量都是不能用volatile关键字进行修饰的**"。

问题（1）:“为什么volatile关键字只能修饰类变量和实例变量呢？”关于问题，我们可以先进行思考，然后在文章的结尾集中探讨答案。

### 机器硬件CPU&JAVA内存模型

在深入理解volatile关键字之前，让我们先来回顾下**并发问题产生的根本原因**，这一点对于我们理解volatile关键字的存在意义是一个基础性问题。我们知道在计算机系统中所有的运算操作都是由CPU来完成的，而CPU运算需要从内存中加载数据，计算完后再将结果同步回内存，但是由于现代计算机的CPU的处理速度要比内存的访问速度牛逼N倍，如果CPU在进行数据运算时直接访问内存的话，由于内存的访问速度慢，这样就会拖慢CPU的运算效率。



为了解决这一问题，伟大的计算机科学家们就想到了一个办法，通过在CPU和内存之间架设一层缓存，CPU不直接访问物理内存，而是将需要运算的数据从主内存中拷贝一份到缓存，运算的结果也通过缓存同步给主内存。



通过这种方法CPU的运行速度就大大提高了，目前主流的CPU都有L1、L2、L3三级缓存。但是，这样的方式也带来了新的问题，那就是**在多线程情况下同一份主内存中的数据值，会被拷贝多个副本放入CPU的缓存中，如果两个线程同时对一个变量值进行赋值操作的话，就会产生数据不一致的问题**，例如：”变量i的初始值为0，两个线程同时加载到CPU缓存后，同时执行i+1的操作，按照道理说i的值此时应该是变成2，而实际情况主内存的值可能还是1，因为两个线程彼此是不知道对方已经改动了这个变量的值的“。



而为了解决这样一个问题，一些CPU制造商如Intel开发了诸如**MESI协议**这样的**缓存一致性控制协议**来解决CPU缓存与主内存之间的数据不一致问题，其基本操作大概就是在某个线程通过CPU缓存写入主内存时，会通过信号的方式通知其他线程中CPU缓存中的值变为失效，从而让其他线程再次从主内存中同步一份数据到CPU缓存中。



以上关于CPU缓存与内存的介绍，并不是为了探讨关于CPU的原理，而是为了说明并发数据不一致问题产生的基本缘由是什么！同理，JAVA内存模型中的定义中，也进行了类似的设计。在JAVA内存模型中，线程与主内存的关系是，线程并不直接操作主内存，而是通过将主内存中的变量拷贝到自己的工作内存中进行计算，完成后再将变量的值同步回主内存这样的方式进行工作。



JAVA内存模型定义了线程与主内存之间的抽象关系，如下：

- 共享变量（**类变量以及对象的全局实例变量等都是共享变量**）存储于主内存中，每个线程都可以访问，这里的主内存可以看成是堆内存。
- 每个线程都有私有的工作内存，这里的工作内存可以看成是栈内存。
- 工作内存只存储该线程对共享变量的副本。
- 线程不能直接操作主内存，只有先操作了工作内存之后才能通过工作内存写入主内存。

以上关于工作内存及Java内存模型的概述，只是便于我们去理解JVM内存管理机制的一个抽象的概念，物理上并不是具体的存在。从具体情况上来讲因为Java程序是运行在JVM之上的，并没有直接调用到操作系统的底层接口去操作硬件，所以线程操作数据进行运算最终还是通过JVM调用了受操作系统管理的CPU资源去进行计算。而计算中涉及的CPU缓存与主内存的缓存一致性问题，则是操作系统层面的一层抽象，与Java工作内存彧主内存的划分并没有直接关系，它们是不同层次的设计。



如果非要用一张图来进行下类比，以便于大家好理解的话，那就来一张图吧：





![img](https://mmbiz.qpic.cn/mmbiz_jpg/l89kosVutoleGKHkEYVyrtBXibjJxDwPich9xdBAKTUKcOhKsE7UGnNicxOeAE56DHGlyMOD11CMeW0ia92gxnia6FA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



根据图中的描述，Java内存模型的区分的堆、栈内存只是虚拟机对自身使用的物理内存的内部划分，它们对于操作系统管理来说就是一块被JVM使用的物理内存，而这个物理内存如果涉及CPU的运算操作，CPU就会通过硬件指令对数据进行加载运算，最终更改物理内存中相应程序变量所对应的内存区块的值。



### 并发编程三大特性

volatile关键字说到底是Java中对多线程并发问题提供语法机制之一，而要正确地理解Java多线程问题，要求我们必须深刻的理解**“原子性”、“有序性”、“可见性”**这三个非常重要和关键的特性。



**原子性**



所谓原子性是指在一次操作或者多次操作中，要么所有的操作全部都得到执行，要么所有的操作都不执行。这个**原子性与数据库事务的原子性特性是一致的**。Java内存模型只保证了基本的读取和赋值的原子性操作，其他操作均不保证。



简单操作如："**x=10**",这个操作就是原子性的。因为从操作上看执行线程会将x=10这个动作写入工作内存，然后再将其写入主内存；即便在该线程进行数值刷新的过程中，也有其他线程对其进行刷新操作（如x=11）的情况x的最终结果也没有什么不一致的问题，因为最后要么是10，要么是11，两个线程谁先刷新都无所谓，那么在这种情况下我们就说这个操作是原子性的。



其他操作如：**"x++"**，这个操作就是非原子性的。为啥呢？我们来分析下x++这个动作都经过了哪些步骤：



![img](https://mmbiz.qpic.cn/mmbiz_jpg/l89kosVutomAA3ia3nYtmWo986mnrAwu6PxxoK2NnZEsc5bcMQiaPtB52GGKjfw31L1iaZdRlCzAGeStuMA56PT4Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



此时y的正确值应该是6，但是线程A最终将y=2同步给了主内存，从而导致主内存中y的值变成了一个脏数据，从而产生了线程安全问题，所以我们说y++的操作不具备原子性，因为它分了三个步骤来执行一个操作。



从上面的例子可以看到**原子性是一个排他性的特性**，如果需要保证y++具备原子性就需要确保y++动作的三个步骤完成前，不允许其他线程对y变量进行操作。因为Java的内存模型并不确保此类操作的原子性，所以有时候写Java代码时会让人感到代码中处处都充斥着线程不安全的操作，只是现在觉得多数程序员都在从事业务开发，面向的都是数据库编程，加上工程上都是集成了现成的开发框架，所以对于这一点感受并不是特别深刻，只有在少数场景下手动实现多线程编程时才会通过**synchronized**关键字进行加锁同步操作。



然而，这个特性与volatile关键字有什么关系呢？**事实上volatile关键字并不保证被修饰的类变量和实例变量具有原子性。**这是因为被volatile关键字修饰的变量并不具备排他性，关于这一点，我们在下面说完另外两个特性后再分析下原因。



**可见性**



可见性是指，**当一个线程对共享变量进行了修改，那么其他线程可以立刻看到修改后的最新值**。在Java多线程环境下，线程首次读取要操作的变量时，是先到主内存中获取该变量，然后将其放入工作内存，以后关于该变量的操作都是在以工作内存中的变量值为基准的。之后如果要修改该变量的值，也是直接修改工作内存中的变量，最后会在某一时刻将工作内存中该变量的值刷新同步回主内存，之后其他线程就能感知到该变量的变化，实现可见性了！



只是**什么时候将工作内存中的值同步会主内存，这个时间点在自然情况下是不确定的**，所以假设线程A修改了变量的值之后，在正式将其同步会主内存之前，线程B获取了主内存中变量的原先值，而过了一会后线程A刷新了主内存，但是此时主内存中的变量值与线程B工作内存中的变量值已经不一致了，这个时候就出现不可见的问题了！



在Java中本文的主角**volatile关键字就可以解决变量在线程间不可见的问题**。当一个变量被volatile关键字修饰之后，对于共享资源的操作会时刻保持与主内存的数据一致。因为被volatile关键字修饰的变量，如果某个线程对其进行了更改，它就会立马进行一次工作内存刷新同步至主内存的操作；同理，如果某个线程读取volatile关键字修饰的变量，那么该线程返回自己工作内存中的变量时，每次都会被要求从主内存再同步一次到工作内存中。



**除此之外synchronized关键字以及JUC包中提供的显示锁Lock也可以保证可见性。**原因在于它们可以保证在同一时刻只有一个线程获得锁可以操作共享变量，完成工作内存中的运算在释放锁之前会确保工作内存中变更的变量新值会被刷新到主内存中。



**我们再回过头来分析下volatile关键字修饰的变量为什么在保证可见性以后还是不能确保原子性，实现完全的线程安全呢？**



![img](https://mmbiz.qpic.cn/mmbiz_jpg/l89kosVutomAA3ia3nYtmWo986mnrAwu6muqGH8qsGcCqkjMeU3WUHO2249sY6ULVmOHC8vaJ0SMOxL3RpJibkNA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



我们还是以y++举例，这次变量y被volatile修饰了，有什么变化呢？假设**第1步**线程A从主内存拷贝了y的副本到工作内存后，此时线程B直接操作了y＝5这个动作，那么此时线程A中的副本y的值为1就不对了，因为被volatile修饰了，所以在**第2步**线程A使用y进行运算时，会再次从主内存中同步一次y的副本（y＝5），然后线程A执行y=y+1后，会立马执行**第3步**把y的值6立马同步刷新会主内存。



初一看感觉volatile的关键字好像解决了y++这个操作的原子性问题，但实际上我们再看看，如果此时线程A已经执行完第2步了，此时线程B更改了变量y的值，虽然此时线程A知道变量y发生了变化，但是由于操作已经执行完，所以还是只能执行第3步把变量y的值覆盖回主内存，从而又造成了错误数据。



所以从这个例子分析，volatile只是解决了y++第1步和第2步的原子性，并没有解决3个步骤的原子性，所以我们说volatile关键字并不能保证解决原子性问题，就是这个道理！



**有序性**



有序性是指程序代码在执行的过程中要确保有数据依赖关系的代码要有先后顺序。由于代码编译存在**指令重排**的问题，所以程序编写的顺序与最后实际执行的指令可能先后顺序时错乱的，如果代码编写的先后顺序存在数据依赖关系，那么就有可能导致依赖于某条代码指令在它所依赖的代码指令执行之前就被执行了，从而导致程序出现错误的情况。



在Java中的有序性就是要通过对指令重排的干预，避免掉因为指令重排导致的这种程序错误问题。**volatile关键字**就可以通过增加内存屏障的方式禁止指令重排，从而实现程序执行的有序性。除此之外的**synchronized关键字以及JUC包中提供的显示锁Lock也可以保证有序性，**因为同步所以与单线程的情况一样自然能够保证有序性。



此外，Java内存模型本身也会通过一些**happens-before原则**的推导来确保在进行指令重排时程序代码执行的有序性。这里的happens-before原则有：程序次序规则、锁定规则、volatile变量规则、传递规则、线程启动规则、线程中断规则、线程终结规则、对象的终结规则等。

#### **volatile实现机制**

------



通过上面内容的阅读，详细你对volatile关键字已经有了一定深入的了解了，下面我们再深入分析下volatile的实现机制。通过对OpenJDK中的unsafe.cpp源码的分析，会发现被volatile关键字修饰的变量会存在一个“lock:”的前缀。
![img](https://mmbiz.qpic.cn/mmbiz_jpg/l89kosVutomAA3ia3nYtmWo986mnrAwu6H7a4atEGcasVTmJRef5YiaZEHNBKxdey1sZicxyFdfb5Ek4pyvWpUn7w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个实际上相当于是一个内存屏障，该内存屏障会为指令的执行提供如下保障：

- 确保指令重排序时不会将其后面的代码排到内存屏障之前。
- 同样也会确保重排序是不会将其前面的代码排到内存屏障之后。
- 确保在执行到内存屏障修饰的指令时前面的代码全部执行完成。
- 强制将线程的工作内存中值的修改刷新至主内存中。

#### **思考题**

------



以上就是关于volatile关键字的全部要说的内容了！在结束之前，我们先来看看之前文章中提到的一个问题：为什么volatile关键字只能修饰类变量和实例变量？“**关于这个问题类似于就是要回答Java语言的语法设计问题了。****因为volatile关键字是要解决多线程间共享变量的可见性问题的，只有类变量及实例变量才是Java中的共享变量的类型，方法参数因为时线程私有不存在共享的问题，而常量本身的值是固定的所以不需要被volatile这样的机制修饰”。**

▼

往期精彩回顾

▼

[微服务为什么一定要用docker](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049068&idx=1&sn=530ed5ee346befd2ea6714bece98fd32&chksm=875343dfb024cac90570b76a38b1c4f7585d175a9d2fd184ebaf95c6bdac42c27e8a7ec4d880&scene=21#wechat_redirect)

[使用docker部署spring cloud项目详细步骤](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049182&idx=1&sn=d46bbd5d06a73a4a37dcb0d8da8b389b&chksm=87534c6db024c57b6bbf082bd97fad5bde28ce20f89cf6b3dfe867384d41b458a1e5c4d7382a&scene=21#wechat_redirect)

[几道和「堆栈、队列」有关的面试算法题](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049182&idx=2&sn=14a35a0d0c7d0b1b7429992ab07e36df&chksm=87534c6db024c57b456573d3ee417b0e1fe759c2f85bed86b368a86ba2c3e85a7137faf7d53a&scene=21#wechat_redirect)

[在Spring Boot中格式化JSON日期](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049168&idx=1&sn=829ead3661e4725dca062f2021caf7d6&chksm=87534c63b024c575fcdff6f0f9b775b7453c33d1ebf0803ab91c0b4c75dfbc04395c5863ecce&scene=21#wechat_redirect)

[使用windows版Docker并在IntelliJ IDEA使用Docker运行Spring Cloud项目](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049157&idx=1&sn=5b339f8968c74b51b8ce786f0bb9996a&chksm=87534c76b024c560df685ee2e1a902e5c6f1b92ce5a638dbe82515ccd97fc8045ceca743fb0c&scene=21#wechat_redirect)

[Springboot项目的接口防刷](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049145&idx=2&sn=fa191f98b4d8a3e74cba91b5ce16acbc&chksm=87534c0ab024c51cdb8e14f31df01a49c8c725add2567182ba5ca40481d4fb575bfa7b9b36f9&scene=21#wechat_redirect)

[实体与模型之间的映射,就用Mapstruct](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049141&idx=2&sn=005a1392da010954b3db14ead37aad8b&chksm=87534c06b024c51028dfc0773e0f9cb03c132e7424bf7400ecac421b5939677bae53ab71b882&scene=21#wechat_redirect)

[Java高级开发必会的50个性能优化的细节（珍藏版）](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049132&idx=1&sn=3468c65313a1643fecb00169d7027000&chksm=87534c1fb024c5095beb91ef00ad7b44cd689261609cec8e7da765b5c92bec8175481b4a0634&scene=21#wechat_redirect)

[记下来，spring 装配bean的三种方式！](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049114&idx=1&sn=7c8a4325319ee6bc9fd31fc15db90b8f&chksm=87534c29b024c53fcd240f8511ce65a5cf2f4dc233d6b4b771b6cd342f9900a9999913cfa576&scene=21#wechat_redirect)

[厉害！这届码农追星玩出了新花样](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049109&idx=1&sn=33dc89b06119576b269280f476130621&chksm=87534c26b024c530ab58c1d94507a34a209409e0e4f4b54745892a19ce57ab078f215da87ec3&scene=21#wechat_redirect)

[Java生成二维码](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049065&idx=2&sn=9cb669cad49f5c7bbf49dc59ec54d55c&chksm=875343dab024cacc90dc5e19cecf800087fe9dc7fc79f332bed9187b0a4a0ba2169e5b112d29&scene=21#wechat_redirect)

[与 30 家公司过招，得到了这章面试心法](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049106&idx=1&sn=ac465104982aa3e51b8f5e79815a74a2&chksm=87534c21b024c537f903ad3883fc43d45c487eefa2e92d84400ec6d1d00f8e61b818ade152cd&scene=21#wechat_redirect)

[一道让你拍案叫绝的算法题](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049096&idx=1&sn=f7bd569c81bef356a16489253cfe5bec&chksm=87534c3bb024c52d4ebab45c45f0edf5a720df0a72e67298f09d9a20d2e9e3f415c2af49a0b5&scene=21#wechat_redirect)

[了解一下Spring中用了哪些设计模式？这样回答面试官才稳](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649048999&idx=1&sn=ca3efda9a37983d1bd8500f649121e8f&chksm=87534394b024ca826a1f1cdd1256c97fc41b5bce38a72f286fd97019a5a9b0eeb51b2b9b9b88&scene=21#wechat_redirect)

[dubbo 面试18问](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049036&idx=1&sn=3b04d1383043cb8f34a51c3bf9bd8235&chksm=875343ffb024cae95c2b7f742a1e62c403c38deeb46cb2ceb6e00fd4ed1bde83d8400d589b97&scene=21#wechat_redirect)

[拜托！面试请不要再问我Spring Cloud底层原理](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049002&idx=1&sn=fc0428835f97198258c5e670e7e9c16b&chksm=87534399b024ca8fac0d20f8dac0b2da990369820c4acfd635525a641a9292f8428522015692&scene=21#wechat_redirect)

[稳了！Java并发编程71道面试题及答案](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649048995&idx=1&sn=4edf6f841a9ca77356eba5c2b4388152&chksm=87534390b024ca86ece393afea1d460d337b79b0ac916be12b083d094ad1b333595257ec624d&scene=21#wechat_redirect)

[【附答案】Java面试2019常考题目汇总（一）](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649048987&idx=1&sn=51d695e336a15f4f83a040667329c6e9&chksm=875343a8b024cabe3ca4731992a648f75779f83ad65615d2a6a00fd980c449a285d6b8ad8a83&scene=21#wechat_redirect)

[这10道springboot常见面试题你需要了解下](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649048975&idx=1&sn=b0eecd92b2d033647713ecd21216acb1&chksm=875343bcb024caaa849ff589131cc4ff6269619fe1094a71554b79a25e14681ea74c93558711&scene=21#wechat_redirect)

[JVM面试题](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649048958&idx=2&sn=c27caee93cdbdb67642f603a3a4947de&chksm=8753434db024ca5bc19aded90a12cb7cb7f6220d70974a9034c9b7683f9cfd2a500d55d4e7cb&scene=21#wechat_redirect)
[巧用这19条MySQL优化，效率至少提高3倍](http://mp.weixin.qq.com/s?__biz=MzA3ODQ0Mzg2OA==&mid=2649049010&idx=1&sn=3fbd48429e2d384cc9cfb888460f0dff&chksm=87534381b024ca97a7b03c23f0063759dcd88440fa119fc98223ed41ca5fe157aa640174648a&scene=21#wechat_redirect)

![img](https://mmbiz.qpic.cn/mmbiz_jpg/sG1icpcmhbiaDibCb0MpMDoQ657RBycRPEFCtEL0Ktpiakqt3ibT1qbyEpqZjy8PowShHgthU3OicBUfGe3uFqXYXc5Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**长按关注，更多精彩**

阅读 1733

赞在看4

**精选留言**

作者已设置关注后才可以留言

- ![img](http://wx.qlogo.cn/mmopen/A4wWoBrWhypLSJnvf24mlk71ZHQfiaDkdW0zPTqXltW0mjXJl1kHeKKF3GsdATTicN9IicqgCNlnqpbY11xFsbOTCNbNOWOMfSC/96)**......**

  3

  

  我一个菜鸟竟然看懂了，厉害啊作者

- ![img](http://wx.qlogo.cn/mmopen/RfcIZnQV47cF4s7OsNn5KOqwjFZQchmk8k7b2rA1TxsYRYvibSyb1acObtWVx3LSp7Ws5PXHianibKGYjdO69e7BavNMqibeibone/96)**TIME FLIES~**

  

  那就是说syncheonize的有序性并不能解决指令重排问题吗

  作者

  2

  synchronized 是加锁，volatile 禁止指令重排，双重检查再加volatile才是完美的

- ![img](http://wx.qlogo.cn/mmopen/RfcIZnQV47cF4s7OsNn5KOqwjFZQchmk8k7b2rA1TxsYRYvibSyb1acObtWVx3LSp7Ws5PXHianibKGYjdO69e7BavNMqibeibone/96)**TIME FLIES~**

  

  synchronize也能保证有序性  那为什么还要加volatile

  作者

  2

  因为指令重排序影响双重检查加锁

- ![img](http://wx.qlogo.cn/mmopen/RfcIZnQV47cF4s7OsNn5KFMcoRa2ic7dTeibEVN3KYSLC2YmwAtOxGNERC9m53ma4e9mygX29BZ4micLMSeF1E389B2DXnIZA9C/96)**原来**

  

  不错哦

- ![img](http://wx.qlogo.cn/mmopen/A4wWoBrWhyrtbG2llKMtPefhcL2oPPGsr0xXPK3r34zwCepFuD5TIx1yO5mRV2gNzwr6spBJkT3gctBynt4muS5brA1ibIWUm/96)**LW**

  

  666，研究了很久，基本都能看懂，能进阿里不？

- ![img](http://wx.qlogo.cn/mmopen/TqYqpoibCAFSrf8vxN5qvnnDbPqFWbrcibC61dlc33ECNpzL1lPhqfKnooicUUUs1GHvHMT4BC3Q6osXFF2egaia52LZWTqZLMwg/96)**小魔王**

  

  太优秀了，这两天查了查这方面的资料，还拜读了深入理解虚拟机，但还是感觉对线程不安全这块有点模糊，今天终于搞明白了。y++的例子太赞了！

- ![img](http://wx.qlogo.cn/mmopen/A4wWoBrWhyrtbG2llKMtPSGqJ3vS2bhyL6NxXIUojNxWNN52HKVDic8pqHrWvujaxSklQ9o2Picoibkc0ufXWibeESVy50gLnHCT/96)**LhfANdmE**

  

  总结:多线程操作同一个被volatile修饰的变量，首先每个线程会将该变量复制进自己的工作空间，a线程操作完存在自己工作空间，jvm会强制的实时刷新到主内存，如果b线程也要操作该变量，自己的内存空间会再去主内存获取一次最新的值。请问:既然volatile既然保证不了变量的原子性，那么是不是意味着被volatile修饰的多线程计算结果并不是一定安全的？

- ![img](http://wx.qlogo.cn/mmopen/VNMic85jx3X4NibdBS5PZxx1U2lpuJT6qy0P1nJP2IK4LD0fvs2bZb1Jbe8LibIHretTpia3icbtlMDFrKGCfdAgw47H3C8icxwq7z/96)**都尉**

  

  那么一个线程改了共享变量的值，是主动通知其他线程 值变了变了？还是靠人家别的无辜线程自己CAS比较重试呢？还是主内存会主动跪舔其他的无辜线程，告诉他们有人变了变了？

- ![img](http://wx.qlogo.cn/mmopen/RfcIZnQV47cF4s7OsNn5KOqwjFZQchmk8k7b2rA1TxsYRYvibSyb1acObtWVx3LSp7Ws5PXHianibKGYjdO69e7BavNMqibeibone/96)**TIME FLIES~**

  

  主要是看见下面这句 我以为加锁保证有序性和禁止指令重排保证有序性是一个东西。“volatile关键字就可以通过增加内存屏障的方式禁止指令重排，从而实现程序执行的有序性。除此之外的synchronized关键字以及JUC包中提供的显示锁Lock也可以保证有序性，因为同步所以与单线程的情况一样自然能够保证有序性。"

- ![img](http://wx.qlogo.cn/mmopen/Gzp7gKnicTbgu57MUticnD6YqKxVUZ1hicx497fk2LFXuKLD5eiaPBN9KNdEGgKic4YDZF5OcqHJiafdcuibK3yLIF7hUFzBKwQsQYH/96)**小康小康，不慌不慌**

  

  看的云里雾里的