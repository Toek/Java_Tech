# Java多线程复习与巩固(九)--volatile关键字与CAS操作

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[Holmofy](https://me.csdn.net/Holmofy) 2017-06-27 23:16:48 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 4498 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 2

分类专栏： [[JAVA\]](https://blog.csdn.net/holmofy/category_6902466.html) [|--JUC](https://blog.csdn.net/holmofy/category_7456442.html) [Java多线程复习与巩固](https://blog.csdn.net/holmofy/category_9269457.html)

版权

前一篇文章中提到原子操作，也许大家和我一样很好奇为什么`AtomicInteger.increment`方法能保证原子性，而简单的`++`运算却不能保证原子性。这篇文章我们就从`AtomicInteger`类下手分析源码，来了解一下原子操作的实现原理，但是分析源码之前需要来一段小小的前奏。

# CPU内存架构

现代计算机都是多处理机CPU，每个核心(Core)都有一套寄存器，CPU访问寄存器的速度是最快的，但是访问RAM内存速度相对来说要慢很多，所以为了解决寄存器与内存速度的不协调问题，每个CPU内核都会有一级或多级高速缓存(Cache)：

![CPU内存架构](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI3MjI1NTI1MTY1?x-oss-process=image/format,png)

当两个线程同时运行的时候，可能会出现下面的情况：两个线程同时使用一个共享变量，会在Cache中缓存该变量，当一个线程修改共享变量时，Cache未能及时将修改的值放回RAM，导致另一个线程不能读取修改后的值。

![线程共享变量出现的问题](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI3MjI1NjEyNjQ4?x-oss-process=image/format,png)

# volatile关键字的作用

前面讲CPU内存架构就是为了说明`volatile`关键字的作用：用来保证对变量修改后，能立即写回主存，从而保证共享变量的修改对所有线程是可见的。[JVM语言规范](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5)将该特性称为`happens-before`。

另外，在[Java官方教程](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html)中讲“原子操作”时，提到平常写代码遇到的最简单的原子操作：

- 对引用变量(不是引用的对象)和大多数基本类型变量(除了long和double)的读写操作都是原子性的。

  > 为什么long和double除外呢，我个人是这么理解的：因为long和double是8个字节长的，如果程序运行在32位的机器上，JVM需要执行更多的操作来实现long和double的运算。所以JVM **不能保证** long和double类型读写操作的原子性。

- 对于声明了`volatile`的所有变量(包括long和double)的读写操作都是原子性的。

从上面的说明我们可以了解到：`volatile`关键字修饰的所有变量读写操作都是原子性的。那么是不是意味着对`volatile`修饰的`int`值进行`++`操作也是原子性的。答案是否定的，`volatile`不能保证`++`，`--`操作的原子性，这里所说的读写操作仅仅是指“取值”和“赋值”操作。我们可以对之前的例子进行简单修改，来证明这个说法：

```java
public class ThreadCommunicate {
    static class Counter {
        private volatile int value;

        public void increment() {
            value++;
        }

        public void decrement() {
            value--;
        }

        public int value() {
            return this.value;
        }
    }

    static class IncrementTask implements Runnable {
        public void run() {
            for (int i = 0; i < 10000; i++) {
                counter.increment();
            }
        }
    }

    static class DecrementTask implements Runnable {
        public void run() {
            for (int i = 0; i < 10000; i++) {
                counter.decrement();
            }
        }
    }

    private static Counter counter = new Counter();

    public static void main(String[] args) throws InterruptedException {
        Thread i = new Thread(new IncrementTask());
        Thread d = new Thread(new DecrementTask());
        i.start();
        d.start();
        i.join();
        d.join();
        System.out.println(counter.value());
    }
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445
```

运行结果仍然很难得到`0`。

所以最终的结论是：**对于所有`volatile`修饰的变量，它们的取值和赋值操作是原子性的。**

> 更多关于volatile关键字的解释可以参考：
> https://stackoverflow.com/questions/1063133/usage-of-volatile-specifier-in-c-c-java/1065150
> https://en.wikipedia.org/wiki/Volatile_(computer_programming)

# AtomicInteger中`volatile`的使用

下面就让我们看看AtomicInteger中是怎么使用volatile关键字的

```java
// AtomicInteger的get和set也是原子操作
public class AtomicInteger extends Number implements java.io.Serializable {
    ...

    private volatile int value;

    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    public AtomicInteger() {
    }

    public final int get() {
        return value;
    }

    public final void set(int newValue) {
        value = newValue;
    }
    ...
    // 其他代码下面会继续分析
}
1234567891011121314151617181920212223
```

# [CAS 比较并交换](https://en.wikipedia.org/wiki/Compare-and-swap)

CAS (compare and swap) 比较并交换，就是将内存值与预期值进行比较，如果相等才将新值替换到内存中，并返回true表示操作成功；如果不相等，则直接返回false表示操作失败。

可以用以下C++伪代码来表示该操作：

```c++
// compare and swap
bool cas(int* address, int expect, int update){
    if(*address == expect){
        *address = update;
        return true;
    }else{
        return false;
    }
}
123456789
```

> 上面的代码只是伪代码，无法实现原子性，CAS操作大多都是靠CPU原语来实现，比如intel x86的`cmpxchg`指令就是CAS原语(compare and exchange)，该指令会在后面源码分析中遇到。

在`java.util.concurrent.aotmic`包中的每个类中都有`compareAndSet`方法，这个方法就是CAS操作的最好体现：

```java
// 这里以AtomicInteger为例：expect表示预期值，update表示将要更新的值
public final boolean compareAndSet(int expect, int update) {
    // this和valueOffset就是用来获取value字段的地址
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
12345
```

> CAS操作经常被用来实现无锁数据结构，在`java.util.concurrent`包中就有很多这样的数据结构：`ConcurrentLinkedQueue`、`ConcurrentLinedDeque`、`ConcurrentHashMap`、`ConcurrentSkipListMap`、`ConcurrentSkipListSet`

# 源码分析CAS操作

说了半天都没有说到`getAndIncrement`和`getAndDecrement`的实现！接下来就来分析分析这几个方法的源码：

```java
...

private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        // 获取value字段的偏移量,这个偏移量类似于C语言中的指针值
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}
private volatile int value;

...

public final int getAndSet(int newValue) {
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
public final boolean weakCompareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
public final int getAndDecrement() {
    return unsafe.getAndAddInt(this, valueOffset, -1);
}
public final int getAndAdd(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta);
}
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
public final int decrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
}
public final int addAndGet(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}

...
123456789101112131415161718192021222324252627282930313233343536373839404142434445
```

> 该类中都是调用`sun.misc.Unsafe`类，该类通过暴露一些从Java意义上来说“不安全”的功能给Java层代码，让JDK能够更多的使用Java代码来实现一些原本是平台相关的、需要使用native语言（例如C或C++）才可以实现的功能。该类只适合在JDK内部使用，开发者不应该调用。

下面看一下Unsafe的相关源码：

> 源码可以从[这里](http://download.csdn.net/detail/holmofy/9839016)下载

```java
    public native long objectFieldOffset(Field f);

    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, v + delta));
        return v;
    }

    public final int getAndSetInt(Object o, long offset, int newValue) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, newValue));
        return v;
    }

    public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);
123456789101112131415161718192021
```

上面两个方法最终都是调用`compareAndSwapInt`方法来修改原来的值。该方法调用C++层JVM的源码。

> 可以查看JVM源码中的`Atomic.hpp`、`Atomic.inline.hpp`等相关文件内容，里面会有几个`cmpxchg`重载函数(compare and exchange 比较并交换)。该方法就是JVM原子操作的底层实现
> 这里有JVM的实现[Hotspot的源码下载](http://download.csdn.net/detail/holmofy/9873419)

由于每个平台架构实现都不一样所以无法把所有的实现代码都列举出来，这里只看`atomic_windows_x86`的相关内容：

```c
// 如果是多处理机系统，添加0xF0前缀
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:

inline jint Atomic::cmpxchg    (jint exchange_value, volatile jint* dest, jint compare_value) {
  // 判断系统是否为多处理机系统(MP MultiProcessor)
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}
1234567891011121314151617
```

> 这里直接在C++中内嵌x86的汇编指令。其中`_emit`是汇编伪指令，cmpxchg指令就是硬件级别的比较并交换。LOCK_IF_MP检测系统是否为多处理器，如果是多处理器系统则加0xF0前缀，让cmpxchg指令原子执行，否则直接执行cmpxchg指令。

下图是[Intel开发手册](https://software.intel.com/en-us/articles/intel-sdm)对`cmpxchg`指令的描述

![cmpxchg指令](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI3MjMxMTM5MTc0?x-oss-process=image/format,png)

- `cmpxchg`指令有两个操作数，第一个操作数是内存地址，第二个操作数是交换的值。
- `cmpxchg`同时需要一个accumulator寄存器，如果是`x86_64`架构CPU的就是64位的RAX寄存器，如果是`x86`架构的CPU就是32位的EAX寄存器(`x86_64`是向下兼容的，RAX的低32位就是EAX)，该寄存器中存储进行比较的预期值。`cmpxchg`指令可以对8位(AL)、16位(AX)、32位(EAX)、64位(RAX)进行CAS操作。上图红框就是操作对应的伪代码。
  ![x86_64 累计计数寄存器](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI3MjMxMzMxNTQy?x-oss-process=image/format,png)
- `cmpxchg`指令可以加LOCK前缀(0xF0)来保证`cmpxchg`指令原子性执行。

# CAS操作的[ABA问题](https://en.wikipedia.org/wiki/ABA_problem)

**CAS操作只有内存值与预期值相等才会更新内存中的值**，所有CAS操作可能会出现这种现象：原来内存值为A，线程1和线程2都获取该值，然后线程1使用CAS将内存值修改为B，然后又使用CAS将内存值修改回A；这时线程2使用CAS对内存值进行修改时发现内存值仍然是A，然后线程2修改成功。这种现象是“ABA问题”，也称“调包问题”。

大多数情况下ABA问题并不会对程序造成什么影响，但在某些情况下ABA问题将会产生很严重的问题：比如一个链表`head -> a -> b -> c`，线程1已经知道`a.next=b`，现在要删除a节点，需要将head.next由原来的a变成b，此时会有CAS操作：`head.compareAndSet(a,b)`，在执行CAS之前线程2将b节点删除了，此时b节点变成游离状态，而线程1并不知道，CAS成功后导致c节点也被无故的删除了。在Java中被无故删除的c节点会被垃圾回收机制回收，但在C/C++中这就造成了内存泄漏了。

类似于这样的情况大多发生在链式数据结构上，为了解决这个问题我们可以使用前一篇文章中提到的`AtomicMarkableReference`和`AtomicStampedReference`这两个类。

下面是我在github上看到的一个使用`AtomicStampedReference`的例子：

使用AtomicStampedReference实现无锁二叉搜索树：https://github.com/arunmoezhi/LockFreeBST

# 参考文章：

同步原语：https://software.intel.com/en-us/articles/choosing-between-synchronization-primitives

Java内存模型：https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html

happends-before原则：http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5

Intel开发者手册：http://www.intel.cn/content/www/cn/zh/architecture-and-technology/64-ia-32-architectures-software-developer-manual-325462.html 、 https://software.intel.com/en-us/articles/intel-sdm