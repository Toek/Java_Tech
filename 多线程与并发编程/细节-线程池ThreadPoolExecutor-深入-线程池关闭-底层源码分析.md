## [ThreadPoolExecutor 优雅关闭线程池的原理.md](https://www.cnblogs.com/xiaoheike/p/11185453.html)



也可参考这篇文章：

https://blog.csdn.net/wangmx1993328/article/details/80648238 



## 经典关闭线程池代码

```java
ExecutorService executorService = Executors.newFixedThreadPool(10);

executorService.shutdown();
while (!executorService.awaitTermination(10, TimeUnit.SECONDS)) {
    System.out.println("线程池中还有任务在处理");
}
```

## shutdown 做了什么？

先上源码

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 线程安全性检查
        checkShutdownAccess();
        // 更新线程池状态为 SHUTDOWN
        advanceRunState(SHUTDOWN);
        // 尝试关闭空闲线程
        interruptIdleWorkers();
        // 空实现
        onShutdown();
    } finally {
        mainLock.unlock();
    }
    // 尝试中止线程池
    tryTerminate();
}
```

每个方法都有特定的目的，其中 `checkShutdownAccess()` 和 `advanceRunState(SHUTDOWN)`比较简单，所以这里不再描述了，而 `interruptIdleWorkers()` 和 `tryTerminate()`。

### interruptIdleWorkers 做了什么？

关闭当前空闲线程。

onlyOne = true：至多关闭一个空闲worker，可能关闭0个。

onlyOne = false：遍历所有的worker，只要是空闲的worker就关闭。

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            // 尝试获得锁，如果获得锁则表明该worker是空闲状态
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

`w.tryLock()`：该方法并不会阻塞，尝试一次如果不成功就返回false，成功则放回true。

在方法 `runWorker` 中一旦work获得了任务，就会调用调用了 `w.lock()`，从而倘若 worker 是无锁状态，就是空闲状态。

因此 Worker 之所以继承 `AbstractQueuedSynchronizer` 实际上是为了达到用无锁，有锁标识worker空闲态与忙碌态，从而方便控制worker的销毁操作。

### tryTerminate 做了什么？

这个只是尝试将线程池的状态置为 `TERMINATE` 态，如果还有worker在执行，则尝试关闭一个worker。

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) || // 是运行态
            runStateAtLeast(c, TIDYING) ||	// 是 TIDYING 或者 TERMINATE 态
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())) // SHUTDOWN 态且队列中有任务
            return;
        if (workerCountOf(c) != 0) {
            // 还存在 worker，则尝试关闭一个worker
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 通过CAS，先置为 TIDYING 态
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                // TIDYING 态
                try {
                    // 空方法
                    terminated();
                } finally {
                    // 最终更新为 TERMINATED 态
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

通过如下的方法简单做一些过滤。因为状态是从 TIDYING 态往后才到 TERMINATE 态。从这个过滤条件，可以看出如果是STOP态也是会通过的。不过如果线程池到了STOP应该就不会再使用了吧，所以也是不会有什么影响。

```java
if (isRunning(c) || // 是运行态
runStateAtLeast(c, TIDYING) ||	// 是 TIDYING 或者 TERMINATE 态
(runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())) // SHUTDOWN 态且队列中有任务
	return;
```

如果该线程池中有worker，则中止1个worker。通过上面的过滤条件，到了这一步，那么肯定是 SHUTDOWN 态（忽略 STOP 态），并且任务队列已经被处理完成。

```java
if (workerCountOf(c) != 0) {
    // 还存在 worker，则尝试关闭一个worker
    interruptIdleWorkers(ONLY_ONE);
    return;
}
```

如果该线程池中有多个worker，终止1个worker之后 tryTerminate() 方法就返回了，那么剩下的worker在哪里被处理的呢？（看后面的解答）

假设线程池中的worker都已经关闭并且队列中也没有任务，那么后面的代码将会将线程池状态置为 TERMINATE 态。`terminate()` 是空实现，用于有需要的自己实现处理，线程池关闭之后的逻辑。

## awaitTermination 做了什么

这个方法只是判断当前线程池是否为 TERMINATED 态，如果不是则睡眠指定的时间，如果睡眠中途线程池变为终止态则会被唤醒。这个方法并不会处理线程池的状态变更的操作，纯粹是做状态的判断，所以得要在循环里边做判断。

```java
public boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (;;) {
            if (runStateAtLeast(ctl.get(), TERMINATED))	// 是否为 TERMINATED 态
                return true;
            if (nanos <= 0)
                return false;
            nanos = termination.awaitNanos(nanos);	// native 方法，睡一觉
        }
    } finally {
        mainLock.unlock();
    }
}
```

## 问题

从 shutdown() 方法源码来看有很大概率没有完全线程池，而awaitTermination() 方法则只是判断线程池状态，并没有关闭线程池状态，那么剩下的worker什么时候促发关闭呢？关键代码逻辑在一个worker被关闭之后，触发了哪些事情。

### runWorker

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // 省略其他不需要的代码
		   try{
               task.run();
            } finally {
                task = null;
                w.unlock();
            }
        }
        // 用于标识是否是task.run报错
        completedAbruptly = false;
    } finally {
        // 线程关闭的时候调用
        processWorkerExit(w, completedAbruptly);
    }
}
```

`completedAbruptly`：用于标识是否是task.run报错，如果值为 TRUE，则是task.run 异常；反之，则没有发生异常

从上面的核心代码来看，当一个worker被关闭之后会调用 processWorkerExit() 方法。看看它做了什么。

### processWorkerExit 做了什么

对一个worker退出之后做善后工作，比如统计完成任务数，将线程池的关闭态传播下去，根据条件补充 worker。

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
	// 如果有worker则尝试关闭一个，否则置为TERMINATE态
    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

线程池关闭的关键就在于一个worker退出之后，会调用 `tryTerminate()` 方法，将退出的信号传递下去，这样其他的线程才能够被依次处理，最后线程池会变为 TERMINATE 态。

[好文要顶](javascript:void(0);) [关注我](javascript:void(0);) [收藏该文](javascript:void(0);) [![img](https://common.cnblogs.com/images/icon_weibo_24.png)](javascript:void(0);) [![img](https://common.cnblogs.com/images/wechat.png)](javascript:void(0);)

[![img](https://pic.cnblogs.com/face/sample_face.gif)](https://home.cnblogs.com/u/xiaoheike/)

[xiaoheike](https://home.cnblogs.com/u/xiaoheike/)
[关注 - 7](https://home.cnblogs.com/u/xiaoheike/followees/)
[粉丝 - 17](https://home.cnblogs.com/u/xiaoheike/followers/)

[+加关注](javascript:void(0);)