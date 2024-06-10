- # 多线程 +1操作的几种实现方式，及效率对比

  ![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

  [朱清震](https://me.csdn.net/zqz_zqz) 2017-02-28 16:40:06 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 6684 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 6

  分类专栏： [java](https://blog.csdn.net/zqz_zqz/category_6371643.html)
  
  版权
  

比较LongAdder ,Atomic，synchronized 以及使用Unsafe类中实现的cas 和模拟Atomic，在多线程下的效率 ，见代码，放开对应注释，运行即可看到结果，通过更改线程数，可以查看不同并发情况下性能对比，通过更改循环执行次数，可以查看长时间或短时间持续并发情况下性能对比;

测试服务器cpu 为i3-4170， 4核 3.7GHz





```java
  import java.lang.reflect.Field;

  

  import java.util.concurrent.atomic.AtomicInteger;

  

  import java.util.concurrent.atomic.LongAdder;

  

   

  

  import sun.misc.Unsafe;
  

  
 
  

  
 
  

  
/**
  
  
  
   * 线程安全的+1操作实现种类
  
  
  
 * @author Administrator
  

  
 *
  

  
 */
  
  

  public class Test extends Thread {

  

      

  

      //整体表现最好,短时间的低并发下比AtomicInteger性能差一点，高并发下性能最高；
      //实现原理可以搜索“longadder原理” 参见 https://www.jianshu.com/p/b3c5b05055de

  

      private static LongAdder longAdder = new LongAdder();

  

      

  

      //短时间低并发下，效率比synchronized高，甚至比LongAdder还高出一点，但是高并发下，性能还不如synchronized；不同情况下性能表现很不稳定；可见atomic只适合锁争用不激烈的场景//底层也是用的cas

  

      private static AtomicInteger atomInteger = new AtomicInteger(0);
  
  
  
      
  

  
    //单线程情况性能最好，随着线程数增加，性能越来越差，但是比cas高
  
  

      private static  int $synchronized = 0;

  

      

  

      //高并发下，cas性能最差
  
  

      public static volatile int cas = 0;

  

      private static long casOffset;

  

      
  
  
  
      public static  Unsafe UNSAFE;
  

  
    
  

  
    static {
  

  
            try {
  
  
  
                @SuppressWarnings("ALL")
  

  
                Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
  

  
                  theUnsafe.setAccessible(true);
  
  
  
                UNSAFE = (Unsafe) theUnsafe.get(null);
  

  
                casOffset = UNSAFE.staticFieldOffset(Test.class.getDeclaredField("cas"));
  

  
            } catch (Exception e) {
  
  

                  e.printStackTrace();

  

              }
  
  
  
    }
  
  
  
      

  

      //乐观锁   调用unsafe类实现cas
  
  
  
      public void cas(){
  

  
         boolean bl = false;
  
  
  
           int tmp;
  

  
         while(!bl){
  

  
               tmp = cas;

  

               bl = UNSAFE.compareAndSwapInt(Test.class, casOffset, tmp,tmp+1);
  

  
           }
  
  
  
      }
  

  
    
  

  
    //模拟AtomicInteger的实现
  

  
    public void atomicInteger(){
  

  
         UNSAFE.getAndAddInt(this, casOffset, 1);
  

  
    }
  

  
    
  

  
    
  

  
    //对a执行+1操作，执行10000次
  

  
    public void run(){
  

  
        int i =1;
  

  
        while(i<=10000000){
  

  
            //测试AtomicInteger
  
  
  
            atomInteger.incrementAndGet();
  

  
            
  

  
            //atomicInteger实现;
  
  

  //            atomicInteger();

  

              

  

              //测试LongAdder
  
  

  //            longAdder.increment();

  

              

  
  
              
  
  
  
              //测试volatile和cas  乐观锁  
  

  
  //            cas();

  

              

  
  
              //测试锁
  
  
  
  //            synchronized(lock){
  
  
  
  //                ++$synchronized;
  
  
  
  //            }
  
  
  
              
  
  
  
              i++;
  
  
  
          }
  
  
  
      }
  
  
  
      public static void main(String[] args){
  
  
  
          long start = System.currentTimeMillis();
  
  
  
          //100个线程
  
  
  
          for(int i =1 ; i<=60;i++ ){
  
  
  
              new Test().start();
  
  
  
          }
  
  
  
          while(Thread.activeCount()>1){
  
  
  
              Thread.yield();
  
  
  
          }
  
  
  
   
  
  
  
          System.out.println(System.currentTimeMillis() - start);
  
  
  
          System.out.println($synchronized);
  
  
  
          System.out.println(atomInteger);
  
  
  
          System.out.println(longAdder);
  
  
  
          System.out.println(cas);
  
  
  
      }
  
  
  
      
  
  
  
  }
```