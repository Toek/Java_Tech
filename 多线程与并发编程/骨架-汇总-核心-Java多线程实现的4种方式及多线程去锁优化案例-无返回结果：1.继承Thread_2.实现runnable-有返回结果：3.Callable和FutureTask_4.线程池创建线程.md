# java多线程、FutureTask的用法及两种常用的使用场景

![img](https://csdnimg.cn/release/phoenix/template/new_img/reprint.png)

[ntotl](https://me.csdn.net/ntotl) 2018-07-10 21:28:34 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 1159 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 1

https://blog.csdn.net/ntotl/article/details/80992274 

分类专栏： [java](https://blog.csdn.net/ntotl/category_6107691.html)



### Java多线程实现的方式有四种

- **1.继承Thread类，重写run方法**
- **2.实现Runnable接口，重写run方法，实现Runnable接口的实现类的实例对象作为Thread构造函数的target**
- **3.通过Callable和FutureTask创建线程**
- **4.通过线程池创建线程**

前面两种可以归结为一类：无返回值，原因很简单，通过重写run方法，run方式的返回值是void，所以没有办法返回结果 

后面两种可以归结成一类：有返回值，通过Callable接口，就要实现call方法，这个方法的返回值是Object，所以返回的结果可以放在Object对象中



```
//继承Thread类实现多线程
class MyThread extends Thread{
    private int index;
    public MyThread(int index){
        this.index = index;
    }
    public void run(){
        System.out.println("子线程：" + index + ",开始！");
    }
}
//开启多线程
for(int i = 0;i < 10;i++){
    final int index = i;
    //实现runnable接口实现多线程
    new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("子线程：" + index + ",开始！");
        }
    }).start();

    new MyThread(index).start();

}
```

**下面重点介绍\**FutureTask的用法：\****

FutureTask可用于异步获取执行结果或取消执行任务的场景。通过传入Runnable或者Callable的任务给FutureTask，直接调用其run方法或者放入线程池执行，之后可以在外部通过FutureTask的get方法异步获取执行结果，因此，FutureTask非常适合用于耗时的计算，主线程可以在完成自己的任务后，再去获取结果。另外，FutureTask还可以确保即使调用了多次run方法，它都只会执行一次Runnable或者Callable任务，或者通过cancel取消FutureTask的执行等。

\1. FutureTask执行多任务计算的使用场景

利用FutureTask和ExecutorService，可以用多线程的方式提交计算任务，主线程继续执行其他任务，当主线程需要子线程的计算结果时，在异步获取子线程的执行结果。



```
ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
List<FutureTask<Integer>> futureTaskList = new ArrayList<FutureTask<Integer>>();
for(int i = 0;i < 10 ;i ++){
    final int index = i;
    FutureTask<Integer> ft = new FutureTask<Integer>(new Callable<Integer>() {
        private  Integer result = 0;
        @Override
        public Integer call() throws Exception {
            for(int j = 0;j < 100;j++){
                result += j;
            }
            Thread.sleep(5000);
            System.out.println("子线程：" + index + ",执行完成！");
            return result;
        }
    });
    futureTaskList.add(ft);
    executorService.submit(ft);
}

System.out.println("子线程提交完毕，主线程继续执行！");

try {
    Thread.sleep(1000 * 10);
} catch (InterruptedException e) {
    e.printStackTrace();
}

System.out.println("主线程执行完毕！");

Integer totalResult = 0;
for(FutureTask<Integer> ft : futureTaskList){
    try {
        totalResult =+ ft.get();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}
System.out.println("子线程计算的结果为：" + totalResult);
```





\2. FutureTask在高并发环境下确保任务只执行一次



在很多高并发的环境下，往往我们只需要某些任务只执行一次。这种使用情景FutureTask的特性恰能胜任。举一个例子，假设有一个带key的连接池，当key存在时，即直接返回key对应的对象；当key不存在时，则创建连接。对于这样的应用场景，通常采用的方法为使用一个Map对象来存储key和连接池对应的对应关系，典型的代码如下面所示：

```
private Map<String, Connection> connectionPool = new HashMap<String, Connection>();
private ReentrantLock lock = new ReentrantLock();

public Connection getConnection(String key){
    try{
        lock.lock();
        if(connectionPool.containsKey(key)){
            return connectionPool.get(key);
        }
        else{
            //创建 Connection
            Connection conn = createConnection();
            connectionPool.put(key, conn);
            return conn;
        }
    }
    finally{
        lock.unlock();
    }
}

//创建Connection
private Connection createConnection(){
    return null;
}
```

在上面的例子中，我们通过加锁确保高并发环境下的线程安全，也确保了connection只创建一次，然而确牺牲了性能。改用ConcurrentHash的情况下，几乎可以避免加锁的操作，性能大大提高，但是在高并发的情况下有可能出现Connection被创建多次的现象。这时最需要解决的问题就是当key不存在时，创建Connection的动作能放在connectionPool之后执行，这正是FutureTask发挥作用的时机，基于ConcurrentHashMap和FutureTask的改造代码如下：



```
private ConcurrentHashMap<String,FutureTask<Connection>>connectionPool = new ConcurrentHashMap<String, FutureTask<Connection>>();

public Connection getConnection(String key) throws Exception{
    FutureTask<Connection>connectionTask=connectionPool.get(key);
    if(connectionTask!=null){
        return connectionTask.get();
    }
    else{
        Callable<Connection> callable = new Callable<Connection>(){
            @Override
            public Connection call() throws Exception {
                // TODO Auto-generated method stub
                return createConnection();
            }
        };
        FutureTask<Connection>newTask = new FutureTask<Connection>(callable);
        connectionTask = connectionPool.putIfAbsent(key, newTask);
        if(connectionTask==null){
            connectionTask = newTask;
            connectionTask.run();
        }
        return connectionTask.get();
    }
}

//创建Connection
private Connection createConnection(){
    return null;
}
```





经过这样的改造，可以避免由于并发带来的多次创建连接及锁的出现。