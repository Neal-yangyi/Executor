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
&emsp;&emsp;corePoolSize：线程池的核心池大小，在创建线程池之后，线程池默认没有任何线程。这个参数最直白的意思就是允许线程池中允许同时运行的最大线程数。关于这个参数的值应该设置多大网上有很多说法，有一种就是如果是IO密集型corePoolSize的大小设置为两倍的CPU处理器个数，如果是CPU密集型就设置为CPU处理器个数+/-1。  
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
