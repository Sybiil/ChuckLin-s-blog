#  ScheduledExecutorService 定时任务运行原理

---

技术行业每一分钟都在发生变革。2017年 java web 技术栈中 spring 已经占领了方方面面成为了事实上的标准。使用 java 的程序员大部分用过 spring。若要在 spring 中实现一个定时任务怎么做？

**spring.xml 中增加以下配置信息。**
```xml
<task:annotation-driven executor="myExecutor" scheduler="myScheduler"/>
    <task:executor id="myExecutor" pool-size="5"/>
    <task:scheduler id="myScheduler" pool-size="10"/>
```
**在 Spring 管理的 Bean 中的方法上使用 Scheduled 注解。**
```java
@Scheduled(fixedDelay = 1000 * 60)
    public void checkAndUpdateRegisterZset() {
        //Do something
    }

```
一个每分钟定时执行的方法就完成了。编程正在向着越来越简单的方向发展。开源框架带来了巨大的生产力效率提升，这是一件好事。

不过，像这样简单的配置就能实现的定时任务到底是如何运行的呢？实际上，现阶段 spring 的定时任务与 JUC 包中的周期性线程池密不可分。

##Executor
JUC 包中的 Executor 架构带来了线程的创建与执行的分离。Executor 的继承者 ExecutorService 下面衍生出了两个重要的实现类，他们分别是

- ThreadPoolExecutor 线程池
- ScheduledThreadPoolExecutor 支持周期性任务的线程池

通过 ThreadPoolExecutor 可以实现各式各样的自定义线程池，而 ScheduledThreadPoolExecutor 类则在自定义线程池的基础上增加了周期性执行任务的功能。

###使用示例
通过使用示例来了解 ScheduledThreadPoolExecutor 的用法
```java
import java.time.LocalDateTime;

/**
 * 工作任务
 */
public class WorkerThread implements Runnable {

    private String command;

    public WorkerThread(String s) {
        this.command = s;
    }

    @Override
    public void run() {
        
        System.out.println(Thread.currentThread().getName() + " Start. Time = " + LocalDateTime.now());
        processCommand();
        System.out.println(Thread.currentThread().getName() + " End. Time = " + LocalDateTime.now());
    }

    private void processCommand() {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public String toString() {
        return this.command;
    }
}



/**
 * 线程池测试类
 *
 * @author lincanhan
 */
public class ScheduledExecutorServiceTest {


    public static void main(String[] args) {
        scheduleWithDelay();
        //scheduleAtRate();
    }


    /**
     * scheduleWithFixedDelay 中的 delayTime
     * 代表每次线程任务执行完毕后，直到下一次开始执行开始之前的时间间隔。
     */
    public static void scheduleWithDelay() {
        ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(3);
        WorkerThread workerThread = new WorkerThread("do some thing");
        scheduledExecutorService.scheduleWithFixedDelay(workerThread, 3000, 3000, TimeUnit.MILLISECONDS);
    }

    /**
     * scheduleAtFixedRate 中的 delayTime/period 表示从线程池中首先开始执行的线程算起，假设period为1s，
     * 若线程执行了5s，那么下一个线程在第一个线程运行完后会很快被执行。
     */
    public static void scheduleAtRate() {
        ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(3);
        WorkerThread workerThread = new WorkerThread("do some thing");
        scheduledExecutorService.scheduleAtFixedRate(workerThread, 3000, 3000, TimeUnit.MILLISECONDS);
    }

}


---

pool-1-thread-1 Start. Time = 2017-04-06T23:07:36.575
pool-1-thread-1 End. Time = 2017-04-06T23:07:41.579
pool-1-thread-1 Start. Time = 2017-04-06T23:07:44.582
pool-1-thread-1 End. Time = 2017-04-06T23:07:49.586
pool-1-thread-2 Start. Time = 2017-04-06T23:07:52.592
pool-1-thread-2 End. Time = 2017-04-06T23:07:57.597
pool-1-thread-2 Start. Time = 2017-04-06T23:08:00.603
pool-1-thread-2 End. Time = 2017-04-06T23:08:05.608
pool-1-thread-2 Start. Time = 2017-04-06T23:08:08.612
pool-1-thread-2 End. Time = 2017-04-06T23:08:13.616
pool-1-thread-2 Start. Time = 2017-04-06T23:08:16.619
```
通过使用 ScheduledExecutorService，很方便的实现了每3秒执行一次任务的需求。

下面通过几个关键性的 api 入手，逐步分析 ScheduledExecutorService 是如何实现定时任务的功能的。

### newScheduledThreadPool 方法
```java
/**
     * Creates a thread pool that can schedule commands to run after a
     * given delay, or to execute periodically.
     * @param corePoolSize the number of threads to keep in the pool,
     * even if they are idle
     * @return a newly created scheduled thread pool
     * @throws IllegalArgumentException if {@code corePoolSize < 0}
     */
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }


 /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     * @throws IllegalArgumentException   {@inheritDoc}
     */
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
            // 将线程任务结合参数，封装为 DelayedTask
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
    
    
    
```
将线程任务结合构造参数，封装为一个 DelayedFutureTask。 如果你还不了解什么是 DelayedTask，请参考我之前的一篇文章：[深入 DelayQueue 内部实现](https://www.zybuluo.com/mikumikulch/note/712598)
另外请注意，DelayedFutureTask 是默认实现了 Compare 接口的。
```java
public int compareTo(Delayed other) {
            if (other == this) // compare zero if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }


```

### delayedExecute 方法
```java
 private void delayedExecute(RunnableScheduledFuture<?> task) {
        // 判断线程池是否已经关闭。
        if (isShutdown())
            // 执行拒绝策略。
            reject(task);
        else {
            // 添加封装好的延迟任务到阻塞队列中。
            super.getQueue().add(task);
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }

```
只要线程池正常运行，则将 DelayedTask 添加到 workQueue 中。注意，workQueue 是定义在 ThreadPoolExecutor 当中的用来保存工作任务的阻塞队列。

###ensurePrestart 方法
```java

  /**
     * Same as prestartCoreThread except arranges that at least one
     * thread is started even if corePoolSize is 0.
     */
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        // 当前工作的线程是否超过核心线程数。
        if (wc < corePoolSize)
            // 调用 addWorker 方法，创建新的线程执行任务。
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }

```

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 创建工作线程。
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //将工作线程加入到工作者队列中。工作者队列是保存工作者线程对象的集合。与工作任务队列是不一样的概念。
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 启动创建的工作者线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

```

```java
/**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

```
虽然 addWorker 的代码稍显复杂，不过你完全可以无视那一大块儿 CAS 处理，把注意力放到仅仅与定时任务主要逻辑紧密相关的地方上来。
ensurePrestart 在 addWorker 中创建了工作者线程，添加到工作者队列以后启动线程。注意工作者线程不是工作任务线程。工作者线程是线程池启动的用来执行任务的线程。


### Worker Run 方法
```java
/** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

```

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {  
            // 尝试获取、或从延迟阻塞队列中获取线程任务对象。
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                    // 执行任务的 run() 方法。此处直接调用 run 而不调用 start 方法的原因是因为本身已经处在 worker 工作者线程中了。
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

```
工作者通过循环从工作者任务队列中反复获取任务对象，获取成功后调用对象的 run 方法执行线程任务。当执行结束时，工作者继续尝试从延迟阻塞工作者任务队列中后去新的任务对象进行执行。

无论有几个工作者线程，无论某个工作者线程处理了几个任务，任务的间隔总是由定义在每个任务对象中的间隔时间来决定的。
 
 
### ScheduledFutureTask run 方法
```java
 /**
         * Overrides FutureTask version so as to reset/requeue if periodic.
         */
        public void run() {
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
                // 若无间隔时间，则直接调用 futureTask 的 run 方法。
            else if (!periodic)
                ScheduledFutureTask.super.run();
                // 调用 runAndReset 方法执行任务。
            else if (ScheduledFutureTask.super.runAndReset()) {
                // 设置下次执行时间
                setNextRunTime();
                // 将任务重新放回工作任务队列。
                reExecutePeriodic(outerTask);
            }
        }

```
工作任务队列中的任务对象一旦被工作线程获取成功后，就会被从队列中移出。而其他之前阻塞在队列上，此时竞争到锁的工作者线程将会尝试获取任务队列中的下一个任务。

调用成功获取到的 ScheduledFutureTask 的 run 方法，执行业务逻辑以后 将重新计算对象的 delay 时间，再通过 runAndReset 方法将重新计算的后的对象重置回工作任务阻塞队列中。由于默认实现的 compareTo 方法，
这样，就实现了线程周期性的执行任务的功能。


##总结

ScheduledThreadPoolExecutor 的实现并不复杂，主要是理解有序队列的操作，以及对 FutureTask 的灵活运用，明白这些后，再看 ScheduledThreadPoolExecutor 就不是难事了。

另外，我们常用的 quartz 就是借用了 ScheduledThreadPoolExecutor 来实现定时任务的执行与调度，只不过提供了一种更友好的方式去表达定时任务的配置方式，为 ScheduledThreadPoolExecutor 需要的数据做了封装。真正的功能还是围绕在 ScheduledThreadPoolExecutor上。



