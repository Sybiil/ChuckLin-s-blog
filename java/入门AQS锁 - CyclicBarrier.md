# 入门AQS锁 - CyclicBarrier

标签（空格分隔）： java

---

在本章节内容开始之前，先让我们来回忆一下[CountDownLatch](https://www.zybuluo.com/mikumikulch/note/273442 ) 的定义，再与CyclicBarrier的定义进行比较，明确他们之间的区别。

CountDownLatch:
:    CountDownLatch是一个不可重复利用的，允许一个线程或多个线程，等待另外一组线程执行完后再运行的，用来对并发进行同步的计数锁。
CountDownLatch是通过可重入共享锁所实现的计数锁。

CyclicBarrier:
:    本章节介绍的CyclicBarrier，是一个可以循环利用的，允许多个线程互相等待，当等待条件被满足后，等待的多个线程被唤醒继续执行的循环障碍锁。CyclicBarrier是通过可重入独占锁所实现的同步用障碍锁。


同样，本章节先对CyclicBarrier的使用方式进行介绍，再分析源码，来说明其运行原理。

## 使用CyclicBarrier

**现在，让我们通过设计一个非常简单的赛马游戏仿真程序，来对CyclicBarrier进行介绍。
该游戏程序要求每场赛马比赛都必须有五匹参赛马，每匹赛马跑完跑道所花费的时间不同。当所有的马匹到达终点后，就自动开始进行下一场比赛。**

```flow
st=>start: 赛马比赛开始
house_start=>operation: 赛马出发
horse_wait=>operation: 等待所有赛马到达终点
cond=>condition: comp end?
e=>end: 赛马比赛结束

st->house_start->horse_wait->cond(no)->house_start
cond(yes)->e

```
代码实现：
```java
// 当所有马匹到达终点后，开始进行下一场比赛
public class Competition implements Runnable {
    // 比赛进行次数记录
    private int index = 1;

    @Override
    public void run() {
        System.out.printf("所有马匹到达终点。准备开始第%d场比赛\n", index);
        index++;
    }

}

-------------------------------------------------------------------

import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

// 赛马马匹
public class Horse implements Runnable {
    // 赛马编号
    private String id;
    // 障碍锁
    private CyclicBarrier cb;

    public Horse(CyclicBarrier cb, String id) {
        this.cb = cb;
        this.id = id;
    }

    @Override
    public void run() {
        try {
            // 赛马出发
            System.out.printf("%s号出发\n", id);
            // 比赛中
            Thread.sleep(new Random().nextInt(1000));
            // 等待其他赛马到达终点
            cb.await();
            // 所有赛马到达终点，开始统一进行下一场比赛的准备工作
            System.out.printf("%s号准备进行下一场比赛\n", id);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }

    }

}

-------------------------------------------------------------------

import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CyclicBarrierDemo {
    
    public static void main(String[] args) throws InterruptedException {
        // 开始比赛
        Competition c = new Competition();
        // 建立一个线程池.
        ExecutorService exec = Executors.newCachedThreadPool();
        // 创建一个循环障碍锁对象。总数是5次，且当障碍锁条件满足时，运行比赛实例c的run方法.
        CyclicBarrier cb = new CyclicBarrier(5, c);
        int j = 0;
        // 总共进行二场比赛
        while (j < 2) {
            for (int i = 0; i < 5; i++)
                exec.execute(new Horse(cb, String.valueOf(i)));
            j++;
            // 循环利用障碍锁。等待第一场比赛结束后，继续进行下一场比赛
            Thread.sleep(5000);
            System.out.println("开始下一场比赛");
        }

        // 线程全部执行完毕后结束掉线程。
        exec.shutdown();
    }

}

-------------------
0号出发
4号出发
3号出发
2号出发
1号出发
所有马匹到达终点。准备开始第1场比赛
2号准备进行下一场比赛
1号准备进行下一场比赛
4号准备进行下一场比赛
0号准备进行下一场比赛
3号准备进行下一场比赛
开始下一场比赛
0号出发
2号出发
4号出发
1号出发
3号出发
所有马匹到达终点。准备开始第2场比赛
4号准备进行下一场比赛
0号准备进行下一场比赛
2号准备进行下一场比赛
1号准备进行下一场比赛
3号准备进行下一场比赛
开始下一场比赛


```
在上面的赛马游戏仿真中，每匹赛马都必须等待其他赛马到达终点后，才能准备开始下一场比赛。而下一场比赛想要准备开始，也必须等待所有赛马到达终点。并且，每次比赛结束后，通过重复利用CyclicBarrier，实现了继续自动开始下一场比赛的功能。


## CyclicBarrier原理
掌握了CyclicBarrier的基本使用方法，CyclicBarrier的运行原理就变得更容易理解了。由于CyclicBarrier不考虑公平锁与非公平锁的不同实现，没有判断CLH队列位置等比较复杂的逻辑，所以源代码十分简洁易懂。
现在，与前一章节一样，先从分析它的代码结构开始入手分析它的运行原理

```java
public class CyclicBarrier {
    
    private static class Generation {
        boolean broken = false;
    }

    // 独占锁成员
    private final ReentrantLock lock = new ReentrantLock();
    // 条件成员
    private final Condition trip = lock.newCondition();
    // 必须满足障碍条件的线程个数
    private final int parties;
    // 当障碍条件满足会被自动执行的任务
    private final Runnable barrierCommand;
    // 当前世代(cyclicBarrier可重复利用，每一次利用是一个世代)
    private Generation generation = new Generation();
    // 满足障碍锁条件还需要的阻塞线程个数
    private int count;
    
    ...
    ..
    .
    // 构造函数
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        // 初始化障碍条件总数
        this.parties = parties;
        // 初始化阻塞线程个数
        this.count = parties;
        // 初始化条件满足时自动执行任务
        this.barrierCommand = barrierAction;
    }
    
```

在上例中，House线程会调用await()方法对当前线程进行阻塞。
await()方法的内部实现如下所示：
```java

public int await() throws InterruptedException, BrokenBarrierException {
        try {
            // 调用dowait来实现。
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen;
        }
    }
    
    
 private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        // 获取独占锁。下面的操作都是同步的。
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 设置当前世代
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            // 由于本线程调用await()，则需要的阻塞线程个数-1。
           int index = --count;
           // 若为0则代表障碍条件满足
           if (index == 0) {  // tripped
               boolean ranAction = false;
               try {
                   // 执行注册的自动执行任务
                   final Runnable command = barrierCommand;
                   if (command != null)
                       command.run();
                   ranAction = true;
                   // 设置世代为下一世代，以方便障碍锁的二次利用。
                   nextGeneration();
                   return 0;
               } finally {
                   if (!ranAction)
                       breakBarrier();
               }
           }

            // 通过循环
            for (;;) {
                try {
                    // 若没有阻塞时间限制，则阻塞
                    if (!timed)
                        trip.await();
                    // 若有阻塞时间限制，则阻塞相应的时间
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        Thread.currentThread().interrupt();
                    }
                }
                
                if (g.broken)
                    throw new BrokenBarrierException();
                // 阻塞线程被唤醒后，若generation已被更新则代表障碍条件达成，线程继续执行。
                if (g != generation)
                    return index;
                // 阻塞线程若超过了阻塞时间，被唤醒后，则抛出异常
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            // 释放同步锁。
            lock.unlock();
        }
    }
    
    
    .....
    ...
    ..
    // 更新当前障碍锁世代。
    private void nextGeneration() {
        // 唤醒所有在当前障碍锁上阻塞的线程
        trip.signalAll();
        // reset障碍锁条件
        count = parties;
        // 初始化新世代
        generation = new Generation();
    }
```

可见，通过分析源代码，我们终于弄明白CyclicBarrier的奇妙之处了。
当任意线程调用await()时，会通过其所持有的独占锁，对操作进行同步。在方法调用的过程中，若当前障碍锁条件不满足，则阻塞当前线程。若障碍锁条件满足，则唤醒所有在该锁上等待的线程，接着重设障碍锁状态。得以实现循环利用。

## 总结
学习CyclicBarrier除了要搞清楚上面提到的定义与用法以外，还应该明确CyclicBarrier与CountdownLatch在运行原理上的区别。
CyclicBarrier是由独占锁实现，而CountdownLatch是由共享锁实现。前者在障碍锁条件被满足时，所有阻塞线程被唤醒，并且一个锁对象能够被重复利用。而后者，每调用一次countdown()，就会唤醒CLH队列中全部的阻塞的线程，这些线程会对锁状态进行判断，继而决定是获取锁，还是继续阻塞。且一个锁对象，不可被重复利用。
