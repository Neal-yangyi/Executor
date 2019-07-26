# Executor
### 前言
&emsp;&emsp;写这篇文章的原因是因为公司的运营要一些软件使用方的登录情况和运营情况的数据统计。因为数据量非常大，并且部份数据涉及大量的计算和复杂的关联查询，直接查询数据库需要的时间需要90秒左右，并且有时候公司的数据库服务器网络不好有查询时连接不上的异常。  
&emsp;&emsp;随后将这个需求做成了每晚23：55分去数据库中统计生成当天相关数据的统计存到数据库中，在运营需要这些统计数据时再去数据库中查询。   
&emsp;&emsp;仔细考虑一下肯定有可以优化的地方：  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;1.优化查询。无非是加上索引和优化sql语句本身。   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;2.在统计的时候使用线程池来并行统计。   
### 正文
#### Executor
&emsp;&emsp;不知道大家有没有了解过Executor这个东西，它本身有一些实现类，本文挑选了个别进行讨论。  
1. **ThreadPoolExecutor**  
&emsp;&emsp;ThreadPoolExecutor的构造方法中有一些参数：   
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
```
&emsp;&emsp;corePoolSize：线程池的核心池大小，在创建线程池之后，线程池默认没有任何线程。这个参数最直白的意思就是允许线程池中允许同时运行的最大线程数。关于这个参数的值应该设置多大网上有很多说法，有一种就是如果是IO密集型corePoolSize的大小设置为两倍的CPU处理器个数，如果是CPU密集型就设置为CPU处理器个数+/-1。不同的线程数量会影响处理时间，并不是数量越大速度越快，因为涉及到线程的上下文切换、创建、销毁等等。  
&emsp;&emsp;maximumPoolSize：线程池中线程的最大数。把大量任务提交给线程池处理后一部分会由空闲线程执行，剩下的进入等待队列等待，如果任务数量超过了队列的长度，会创建新的线程来服务，线程的数量就是由maximumPoolSize来限制的。  
&emsp;&emsp;unit：时间单位，可以设置为秒。  
&emsp;&emsp;workQueue：当任务数量超过corePoolSize时，任务会进入等待队列。队列分为有界队列和无界队列。  
&emsp;&emsp;无界任务队列LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE。   
&emsp;&emsp;有界任务队列ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小。   
这里并不推荐使用无界队列，如果提交了大量的任务，无界队列会变的非常大占用大量的内存。maximumPoolSize这个参数也没有用武之地了。  
&emsp;&emsp;threadFactory：这个参数是拒绝策略。当有界的等待队列并且线程数到达maximumPoolSize时，再向线程池中提交任务线程池必须要有策略来处理这种情况。  
##### 创建一个线程池
```
ExecutorService executor = new ThreadPoolExecutor(3, 3, 0L, TimeUnit.SECONDS, new LinkedBlockingQueue());
```
我们假设有50家使用方需要我们统计  
```
List<Object> resultList = new ArrayList<>();
List<Future<Object>> futureList = new ArrayList<>();
for (int i = 0; i < 50; i++) {
    //将任务提交给线程池，如果不需要返回值可以使用Runnable，并用executor.execute(Runnable run)来执行
    Future<Object> future = executor.submit(new Callable<Object>() {
        @Override
        public Object call() throws Exception {
            //TODO 关于业务的一些操作
            return null;
        }
    });
    futureList.add(future);
}
```
这样就将所有的任务提交给了线程池来处理。由于自身业务需求我需要拿到每家使用方的统计结果。
```
for (Future<ReportRunVo> future : futureList) {
    try {
        //按顺序获取执行的结果。future.get()会一直阻塞直到拿到结果，
        这个地方如果觉得会影响性能可以用ExecutorCompletionService来解决这个问题，接下来会将
        resultList.add(future.get());
    } catch (ExecutionException | InterruptedException e) {
        e.printStackTrace();
    }
}
//关闭线程池
executor.shutdown();
```
以上是最简单的使用方法。
