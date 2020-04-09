### newFixedThreadPool 方法

```java
//创建一个固定线程数的线程池
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
//创建一个固定线程数的线程池，传入线程工厂
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

* 核心线程池等于最大线程池
* 保活时间为0，线程一空闲就会被销毁。不能复用，浪费资源
* 队列没有边界，可能会产生OOM

### newSingleThreadExecutor 方法

```java
//创建只有一个工作线程的线程池
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

//创建只有一个工作线程的线程池，指定线程工厂
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

* 只有一个线程
* 队列没有边界，可能会产生OOM

### newCachedThreadPool 方法

```java
//创建有缓存线程的线程池
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

//创建有缓存线程的线程池，指定线程工厂
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

* 核心线程数为0，最大线程数是Integer.MAX_VALUE，可能会产生OOM
* 保活时间60s
* 不缓存任务，意味着来一个任务创建一个线程，增大了OOM的风险

### newScheduledThreadPool 方法

```java
//创建一个调度线程池，指定核心线程数
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

//创建一个调度线程池，指定核心线程数和线程工厂
public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }


 public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```

* 指定核心线程数，最大线程数是Integer.MAX_VALUE，可能会产生OOM
* 保活时间为0，线程一空闲就会被销毁。不能复用，浪费资源
* 使用延时队列存储任务



### 小结

从上面可以看出，不指定边界有出现OOM的漏洞。阿里的规范文档建议自己手动创建线程池，不仅可以避开这些漏洞还能够更清楚的了解线程池运行的流程便于解决问题