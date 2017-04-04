# 入门AQS锁 - CountDownLatch    

---

如果你还记得前一章[可重入共享锁](https://www.zybuluo.com/mikumikulch/note/273442 ) 的知识，并且掌握的还算不错的话，那么CountDownLatch会非常容易理解。
如果你觉得共享锁的知识有所遗忘，建议最好再翻回去巩固一下，会有助于你理解CountDownLatch的运行原理。

本文会先对CountDownLatch的用法进行介绍。在了解了用法以后，再对CountDownLatch的运行原理进行说明。
## 使用CountDownLatch
简单来讲，这个类是用“计数锁”来使线程任务按照顺序执行的一个计数器工具类。
你可以向countDownLatch计数器设置一个初始计数值。当你尝试调用计时器的await()方法时候，线程将被阻塞(可阻塞复数个线程)。当其他线程调用计数器的countDown()方法时，可以让初始计数值-1。当初始计数值为0时，之前调用await()方法阻塞的线程任务能够“被通知”，进入“就绪”状态。

**试想，项目需要你设计一个数据处理模块。
当用户发起某个业务请求时，该业务在执行过程中，需要启动4个线程做4张表的更新处理。当数据库处理全部完成后，将执行结果提交到另外的业务处理机能。若4张表中任何一张表更新失败，则所有更新全部回滚。**
(请暂时忘记分布式事务处理，把重点放到业务流程的解决方式上来)

```flow
st=>start: 用户发起业务请求
execute_sql_update_dabase=>operation: 启动4个线程对数据库进行操作
cond=>condition: all successful?
result_commit=>operation: 将结果提交到A系统
e=>end
r=>end: rollback

st->execute_sql_update_dabase->cond(yes)->result_commit->e
cond(no)->r
```

```java

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

// 更新dataBase任务类
public class TaskPortion implements Runnable {
    // 计数器
    private final CountDownLatch latch;

    TaskPortion(CountDownLatch latch) {
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            // 更新数据库
            updateDatabase();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

    // 更新数据库
    public void updateDatabase() throws InterruptedException {
        try {
            // 花费一些时间更新数据库
            TimeUnit.MILLISECONDS.sleep(1000);
            // 数据库更新完毕,将结果写入缓存
            String result = String.format("result from %s", Thread.currentThread().getName());
            WaitingTask.cache.add(result);
            System.out.printf("%s线程更新数据库完毕。\n", Thread.currentThread().getName());
            latch.countDown();
        } catch (Exception e) {
            System.out.println("数据库更新失败");
            latch.countDown();
        }
    }

}

-------------------------------------------------------------------

import java.util.Collections;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.CountDownLatch;

// 将数据更新结果提交到A系统
public class WaitingTask implements Runnable {

    // 数据库执行结果存储缓存
    public static List<String> cache = Collections.synchronizedList(new LinkedList<String>());
    // 计数器对象
    private CountDownLatch latch;

    WaitingTask(CountDownLatch latch) {
        this.latch = latch;
    }

    @Override
    public void run() {
        System.out.println("等待数据库更新完成");
        try {
            // 该处将阻塞，直到计数器位0,所有任务解决后才会开始调用.
            latch.await();
            commitResults();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 将结果提交到A系统
    public void commitResults() {
        if (cache.size() != 4) {
            System.out.println("业务操作失败，数据库更新回滚");

        } else {
            for (String result : cache) {
                System.out.printf("将%s结果提交到A系统。\n", result);
            }
        }
    }
}

-------------------------------------------------------------------

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CountDownLatchDemo {

    public static void main(String[] args) {
        // 建立一个线程池.
        ExecutorService exec = Executors.newCachedThreadPool();
        // 创建一个计数器对象.次数是5次.
        CountDownLatch latch = new CountDownLatch(4);
        // 启动业务执行结果提交线程
        exec.execute(new WaitingTask(latch));
        for(int i = 0;i<4;i++)
            // 启动数据库更新线程
            exec.execute(new TaskPortion(latch));
        // 线程全部执行完毕后结束掉线程。
        exec.shutdown();
    }

}

-------------------------------------------------------------------
等待数据库更新完成
pool-1-thread-2线程更新数据库完毕。
pool-1-thread-3线程更新数据库完毕。
pool-1-thread-4线程更新数据库完毕。
pool-1-thread-5线程更新数据库完毕。
将result from pool-1-thread-2结果提交到A系统。
将result from pool-1-thread-5结果提交到A系统。
将result from pool-1-thread-4结果提交到A系统。
将result from pool-1-thread-3结果提交到A系统。

```
通过计数器，得以确保了结果提交处理线程一定会在表更新线程全部结束之后，才会对执行结果进行提交的同步操作。


## CountDownLatch运作原理
CountDownLatch本质上是通过可重入的共享锁所实现的计数器工具。
要理解它的运作原理，首先需要大致对CountDownLatch的代码结构做一个了解，然后再通过剖析await()与countDown()方法的实现，来明确它是如何阻塞与唤醒线程的。

java.util.concurrent.CountDownLatch
数据结构
```java
public class CountDownLatch {
    /**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.
     */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
        // 初始化锁状态（计数器计数）
        Sync(int count) {
            setState(count);
        }
        // 获取当前锁状态（计数器计数）
        int getCount() {
            return getState();
        }
        // 尝试获取共享锁。
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
        // 尝试释放共享锁
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
    // CountDownLatch中持有AQS锁的成员变量
    private final Sync sync;
    ........
    ......
    .....
    
    // 通过构造器创建了AQS锁sync构造器，并且初始化了计数器计数(锁状态)
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```

对代码结构有了大致的了解后，接下来对await()方法进行剖析。
**上例中，执行结果提交到系统A的线程调用await()时，计数器实际通过下面的方法对线程与锁进行了操作。**
java.util.concurrent.CountDownLatch
await()
```java
    public void await() throws InterruptedException {
        // 调用AQS类中的acquireSharedInterruptibly方法
        sync.acquireSharedInterruptibly(1);
    }
```

java.util.concurrent.locks.AbstractQueuedSynchronizer
acquireSharedInterruptibly()
```java
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        // 尝试获取共享锁。方法定义在Sync类中。
        if (tryAcquireShared(arg) < 0)
            // 若锁获取失败则调用该方法
            doAcquireSharedInterruptibly(arg);
    }
```

```java
public class CountDownLatch {
    /**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.
     */
    private static final class Sync extends AbstractQueuedSynchronizer {
    .....
    ....
    ...
    // 尝试获取共享锁。
    protected int tryAcquireShared(int acquires) {
            // 当前计数(锁状态)为0，则获取成功。否则失败返回-1。上例中计数器的初始状态为4。
            return (getState() == 0) ? 1 : -1;
    }
    
    .....
    ....
    ...
}
```

由于上例中计数器初始值为4，所以共享锁获取失败。开始调用doAcquireSharedInterruptibly方法。
java.util.concurrent.locks.AbstractQueuedSynchronizer
doAcquireSharedInterruptibly()
```java
/**
     * Acquires in shared interruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        // 通过addWaiter将当前共享锁加入等待队列
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                // 获取队列中当前线程的前一个线程节点
                final Node p = node.predecessor();
                // 若前一个节点为当前队列head，则尝试获取锁
                if (p == head) {
                    // 尝试获取锁
                    int r = tryAcquireShared(arg);
                    // 获取成功
                    if (r >= 0) {
                        // 设置队列head位为当前线程节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                // 判断当前线程是否需要阻塞。若前一个线程节点不是head，则需要阻塞。
                if (shouldParkAfterFailedAcquire(p, node) &&
                    // 阻塞线程
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

可见，调用await()阻塞的线程想要获取到锁，有两个条件。
1. *共享锁被持有他的所有线程释放，锁状态(计数)还原为0。
2. 当前线程被唤醒后，CLH队列中当前线程的前一个线程处于队列的head处。或者队列为空。*

要还原计数器计数，需要持有共享锁的线程release掉锁，改变锁状态。
```java
    public void countDown() {
        sync.releaseShared(1);
    }
    
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            // 释放CLH队列中head节点对象的下一个对象
            doReleaseShared();
            return true;
        }
        return false;
    }
    
    protected boolean tryReleaseShared(int releases) {
        for (;;) {
            // 获取当前“计数”（锁状态）
            int c = getState();
            if (c == 0)
                return false;
            // 计数-1
            int nextc = c-1;
            // 更新锁状态
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
        
    }
        
    private void doReleaseShared() {
        for (;;) {
            // 获取当前CLH队列中的head线程
            Node h = head;
            // 取得的线程对象不为空切不为队列中最后一个
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                         // loop to recheck cases               
                        continue;           
                    // 释放取得的线程对象的下一个线程对象
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
    
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        // head线程对象的下一个线程对象(next)
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            // 唤醒head线程的下一个线程对象，并重新设置head节点所属对象
            LockSupport.unpark(s.thread);
    }


```
在上例中，更新数据库的线程在任务结束后，就是通过调用countdown方法来还原计数器计数的。每次调用countdown方法释放锁时，都会通过doReleaseShared方法将CLH队列中所有的节点，也就是线程对象唤醒。当锁状态还原为0时，则代表共享锁现在不被任何线程持有，被唤醒的节点对象获取到线程锁，开始执行任务。

## 总结：
CountdownLatch工具通过可重入的共享锁，实现了一个或多个线程将等待其他线程执行完后，才执行任务的同步操作。
在项目中若有类似的并发需求，可以尽量使用更高级的类似CountdownLatch这样的并发工具来解决。而不是选择synchronized。