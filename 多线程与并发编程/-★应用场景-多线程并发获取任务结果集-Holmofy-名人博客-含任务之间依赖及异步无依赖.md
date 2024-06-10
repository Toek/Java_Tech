# 多线程并发获取任务结果集

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[洛杉矶的管理局](https://me.csdn.net/qq_18991441) 2018-06-07 16:03:44 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 3554 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 5

分类专栏： [多线程](https://blog.csdn.net/qq_18991441/category_7720115.html)

版权

一般在进行多线程开发时，常用的操作就是ExecutorService来处理任务。任务接口可以是Runnable、Callable。

Runnable的源码如下，表明该任务不支持结果返回，同时不外抛异常。



```
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

Callable的源码如下，表明该任务支持结果返回，同时支持上抛异常。





```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

如果要得到任务结果，可以通过以下方法来进行获取：

**1、Future**



Future封装了如下接口：取消任务、任务状态判断，任务结果获取。

**![img](https://img-blog.csdn.net/20180607150355931?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4OTkxNDQx/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
**

如下是一段使用Future来获取任务结果的demo



```
public static void main(String[] args) {
    Long start = System.currentTimeMillis();
    ExecutorService exs = Executors.newFixedThreadPool(10);
    try {
        List<Integer> list = new ArrayList<>();
        List<Future<Integer>> futureList = new ArrayList<>();
        for(int i=0; i<10; i++) {
            futureList.add(exs.submit(new CallableTask(i + 1)));
        }
        Long getResultStart = System.currentTimeMillis();
        System.out.println("结果归集开始时间="+ LocalDate.now());
        for(Future<Integer> future : futureList) {
            Integer i = future.get();
            System.out.println("任务i="+i+"获取完成!"+LocalDate.now());
            list.add(i);
        }
        System.out.println("list = " + Arrays.toString(list.toArray()));
        System.out.println("总耗时="+(System.currentTimeMillis()-start)+",取结果归集耗时="+(System.currentTimeMillis()-getResultStart));
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        exs.shutdown();
    }
}

static class CallableTask implements Callable<Integer> {
    Integer i;
    public CallableTask(Integer i) {
        super();
        this.i = i;
    }
    @Override
    public Integer call() throws Exception {
        if(i == 1) {
            Thread.sleep(50000);
        }
        System.out.println("task线程："+Thread.currentThread().getName()+"任务i="+i+",完成！");
        return i;
    }
}
```

console结果如下：

![img](https://img-blog.csdn.net/2018060715135587?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4OTkxNDQx/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

结论：线程池在执行任务时，会返回一个Future对象。通过Future.get()能获取到任务结果。但是如果在调用get()时，任务还未执行结束，则会一直阻塞。上述demo通过ExecutorService执行10个任务，获取10个Future对象。从task代码可以看出，如果是第一个任务，则线程睡眠5s。其他任务正常执行。从console的返回可以看出，10个任务是顺序打印出来的。通过线程池工具ExecutorService来执行任务，得到的任务结果就是按照提交顺序来获取的。

2、FutureTask

 FutureTask实现了RunnableFuture，而RunnableFuture继承了Runnable，Future。所以FutureTask延续了Future的特性

3、CompletionService

 其内部是通过阻塞队列（BlockingQueue）+ FuruteTask来实现的。能够实现任务结果按照任务的实际完成顺序来获取，而不是任务的提交顺序。

如下是一段demo代码：



```
public static void main(String[] args) {
    Long start = System.currentTimeMillis();
    ExecutorService exs = Executors.newFixedThreadPool(5);
    try {
        int taskCount = 10;
        List<Integer> list = new ArrayList<>();
        CompletionService<Integer> completionService = new ExecutorCompletionService<Integer>(exs);
        List<Future<Integer>> futureList = new ArrayList<>();
        for(int i=0; i<taskCount; i++) {
            futureList.add(completionService.submit(new Task(i+1)));
        }
        for(int i=0; i<taskCount; i++) {
            Integer result = completionService.take().get();
            System.out.println("任务i=="+result+"完成!"+ LocalDate.now());
            list.add(result);
        }
        System.out.println("list = " + Arrays.toString(list.toArray()));
        System.out.println("总耗时=" + (System.currentTimeMillis()-start));
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        exs.shutdown();
    }
}

static class Task implements Callable<Integer> {
    Integer i;

    public Task(Integer i) {
        super();
        this.i = i;
    }

    @Override
    public Integer call() throws Exception{
        if(i == 1) {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("线程："+Thread.currentThread().getName()+"任务i="+i+",执行完成！");
        return i;
    }
}
```

console结果如下：

![img](https://img-blog.csdn.net/20180607153033652?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4OTkxNDQx/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

结论：同样执行10条任务，这里是通过CompletionService来执行，而ExecutorService。在task里面，如果是第一个任务，则睡眠5s。从console的结果来看，任务结果的打印跟之前的结果不一样了。这里是按照任务完成的实际顺序打印的。

4、CompletableFuture

 前三种方式的实现都是基于Future来进行获取任务结果的。不同的Future之间是无法取得联系的。通过CompletableFuture，能实现多个任务结果之间的一种联系性。

实现了Future、CompletionStage两个接口

4.1、获得单个任务的执行结果。通过supplyAsync接收一个任务，有点类似new Runnable时写的run()实现。supplyAsync还支持第二个参数，该参数表示线程池。如果不指定，则使用默认的ForkJoinPool.commonPool()。



```
CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
    return 300;
});
System.out.println("计算结果:"+completableFuture.get());
```

4.2、两个任务，第一个任务的结果作为第二个任务的入参数，第二个任务要等待第一个任务的执行。通过thenCompose



```
CompletableFuture<Integer> completableFuture1 = CompletableFuture.supplyAsync(() -> {
    return 100;
});

CompletableFuture<String> completableFuture2 = completableFuture1.thenCompose(result -> CompletableFuture.supplyAsync(() -> {
    return  result + ":" + "200";
}));
System.out.println(completableFuture2.get());
```

4.3、两个任务，将两个任务的结果合并起来，这两个任务不分先后顺序执行。通过thenCombine



```
CompletableFuture<Integer> completableFuture1 = CompletableFuture.supplyAsync(() -> {
    return 100;
});

CompletableFuture<Integer> completableFuture2 = completableFuture1.thenCombine(
        CompletableFuture.supplyAsync(() -> {
            return 2000;
        }),
        (result1, result2) -> result1 + result2);

System.out.println(completableFuture2.get());
```

4.4、任务完成后，添加监听函数。通过thenAccept



```
CompletableFuture<Integer> completableFuture1 = CompletableFuture.supplyAsync(() -> {
    return 100;
});
//注册完成事件
completableFuture1.thenAccept(result -> System.out.println("task1 done, result = " + result));
System.out.println(completableFuture1.get());
```



总结：



|                              | Futrue                                  | FutureTask                                                   | CompletionService                                            | **CompletableFuture**                                     |
| ---------------------------- | --------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------------- |
| ***\*原理\****               | Futrue接口                              | 接口RunnableFuture的唯一实现类，RunnableFuture接口继承自Future<V>+Runnable: | 内部通过阻塞队列+FutureTask接口                              | JDK8实现了Future<T>, CompletionStage<T>2个接口            |
| ***\*多任务并发执行\****     | 支持                                    | 支持                                                         | 支持                                                         | 支持                                                      |
| ***\*获取任务结果的顺序\**** | 支持任务完成先后顺序                    | 未知                                                         | 支持任务完成的先后顺序                                       | 支持任务完成的先后顺序                                    |
| ***\*异常捕捉\****           | 自己捕捉                                | 自己捕捉                                                     | 自己捕捉                                                     | 源生API支持，返回每个任务的异常                           |
| ***\*建议\****               | CPU高速轮询，耗资源，可以使用，但不推荐 | 功能不对口，并发任务这一块多套一层，不推荐使用。             | 推荐使用，没有JDK8**CompletableFuture**之前最好的方案，没有质疑 | ***\*API极端丰富，配合流式编程，速度飞起，推荐使用！\**** |

 上表来源：https://www.cnblogs.com/dennyzhangdd/p/7010972.html