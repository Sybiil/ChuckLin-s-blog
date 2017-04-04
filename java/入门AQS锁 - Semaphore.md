# 入门AQS锁 - Semaphore


---

终于来到AQS锁系列的最后一篇文章了。虽然这是AQS系列的最后一篇文章，但并不代表semaphore理解起来会很困难。Semaphore的用法与实现原理和[CountDownLatch](https://www.zybuluo.com/mikumikulch/note/273645)类似。对于掌握了CountDownLatch的读者来说，Semaphore理解起来应该是游刃有余。
Semaphore直译为信号。实际上Semaphore可以看做是一个信号的集合。不同的线程能够从Semaphore中获取若干个信号量。当Semaphore对象持有的信号量不足时，尝试从Semaphore中获取信号的线程将会阻塞。直到其他线程将信号量释放以后，阻塞的线程会被唤醒，重新尝试获取信号量。

##使用Semaphore
下面这个例子可以很好地解释Semaphore的作用。
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

public class SemaphoreDemo {

    public static void main(String[] args) throws InterruptedException {

        SemaphoreDemo sd = new SemaphoreDemo();
        // 创建一个信号量集合.信号量总数为10
        Semaphore sem = new Semaphore(10);
        
        ExecutorService es = Executors.newCachedThreadPool();
        // 获取6个信号量
        es.execute(sd.new Thread1(sem, 6));
        // 获取3个信号量
        es.execute(sd.new Thread2(sem, 3));
        // 获取2个信号量
        es.execute(sd.new Thread3(sem, 2));

        es.shutdown();
    }

    public class Thread1 implements Runnable {

        private Semaphore sem;

        private int countDown;

        public Thread1(Semaphore sem, int countDown) {
            this.sem = sem;
            this.countDown = countDown;
        }

        @Override
        public void run() {
            try {
                // 尝试从信号量集合中获取countDown个信号量
                sem.acquire(countDown);
                System.out.printf("%s获取到%d个信号量\n", Thread1.class.getSimpleName(), countDown);
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.printf("%s释放%d个信号量\n", Thread1.class.getSimpleName(), countDown);
                sem.release(countDown);
            }
        }

    }

    public class Thread2 implements Runnable {

        private Semaphore sem;

        private int countDown;

        public Thread2(Semaphore sem, int countDown) {
            this.sem = sem;
            this.countDown = countDown;
        }

        @Override
        public void run() {
            try {
                // 尝试从信号量集合中获取countDown个信号量
                sem.acquire(countDown);
                System.out.printf("%s获取到%d个信号量\n", Thread2.class.getSimpleName(), countDown);
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.printf("%s释放%d个信号量\n", Thread2.class.getSimpleName(), countDown);
                sem.release(countDown);
            }
        }
    }

    public class Thread3 implements Runnable {

        private Semaphore sem;

        private int countDown;

        public Thread3(Semaphore sem, int countDown) {
            this.sem = sem;
            this.countDown = countDown;
        }

        @Override
        public void run() {
            try {
                // 尝试从信号量集合中获取countDown个信号量
                sem.acquire(countDown);
                System.out.printf("%s获取到%d个信号量\n", Thread3.class.getSimpleName(), countDown);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.printf("%s释放%d个信号量\n", Thread3.class.getSimpleName(), countDown);
                sem.release(countDown);
            }
        }
    }

}
---------------------
Thread1获取到6个信号量
Thread2获取到3个信号量
Thread1释放6个信号量
Thread2释放3个信号量
Thread3获取到2个信号量
Thread3释放2个信号量

```
在main函数中，启动Thread1，Thread2，Thtread3三个线程任务。
Thread1与Thread2首先从总数为10的Semaphore信号对象中获取到9个信号量。Thread3由于Semaphore中的信号量不足，线程阻塞。直到Thread1与Thread2释放掉持有的信号量，Thread3才得以被唤醒并执行。


##Semaphore原理
让我们还是同往常一样，先从类的数据结构入手来剖析Semaphore的原理。在接下来的内容里，就如开头所说的那样你会发现Semaphore在设计理念上与CountDownLatch有许多相似之处。
```java
public class Semaphore implements java.io.Serializable {
    private static final long serialVersionUID = -3222578661600680210L;
    // Semaphore中持有sync的引用。sync继承自AQS锁。
    private final Sync sync;

    /**
     * Synchronization implementation for semaphore.  Uses AQS state
     * to represent permits. Subclassed into fair and nonfair
     * versions.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }
    ..........
    ........
    ......
    ....
    // Semaphore的默认构造函数。默认创建的是非公平锁。设置锁状态为permits
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
    // 通过fair创建公平/非公平锁。
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
    ...........
    ........
    .....
    ..
    // 尝试获取permits个信号量。
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }
    // 尝试释放permits个信号量。
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }
    
```
分析该类大致的结构以后，可见创建Semaphore对象时传入的信号量参数，实际就是用来设置该共享锁的state。根据前面的知识，我们很自然的能够联想到，acquire()获取信号量，实际是否就是在减少state值呢？
```java 

    // 尝试获取permits个信号量。
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        // 调用AQS抽象类中的共通方法来实现“获取信号量”
        sync.acquireSharedInterruptibly(permits);
    }
    
```

```java 
package java.util.concurrent.locks;

public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    private static final long serialVersionUID = 7373984972572414691L;
    ....
    ...
    ..
    // 定义在AQS抽象类中的acquireSharedInterruptibly共通方法。
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
       // 尝试获取定义在Semaphore中的共享锁。
        if (tryAcquireShared(arg) < 0)
            // 共享锁获取失败，则通过doAcquireSharedInerruptibly()获取共享锁。
            doAcquireSharedInterruptibly(arg);
    }
    
    
     private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        // 将当前线程包装为节点加入到CLH队列末尾
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                // 获取当前线程节点的前继节点
                final Node p = node.predecessor();
                // 若前继节点为head，则尝试获取共享锁
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                // 判断是否需要阻塞当前线程。若需要则阻塞当前线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
}
```
Semaphore基于公平锁和非公平锁的不同实现，重写了tryAcquireShared方法。
```java 
 // 公平锁
 static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;
        ......
        ....
        ...
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                // 判断是否有比当前线程等的更久的线程存在。有则获取共享锁失败
                if (hasQueuedPredecessors())
                    return -1;
                // 获取到信号量集合中的集合数（锁状态）
                int available = getState();
                // 集合数-尝试获取的信号量
                int remaining = available - acquires;
                // 重设信号量。
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
        .....
        ...
        ..
 }
```
果然，Semaphore在公平锁的状态下调用acquire尝试获取信号量，实际就是就是对锁的state进行操作。
首先判断当前队列中是否有比尝试获取锁的线程等待的更久的线程，若没有，则获取锁。线程继续执行。若当前线程不是队列中的第一个线程，则通过AQS抽象类的doAcquireSharedInterruptibly方法，将当前线程加入到队列中，然后线程阻塞。

非公平锁的Semaphore的acquire方法与公平锁大同小异。唯一区别就是首次尝试获取信号量时不判断线程是否处于CLH队列中。但是需要注意的是，若获取信号量失败，线程依然会进入CLH排队并阻塞。
```java 
    // 非公平锁
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }
        
        protected int tryAcquireShared(int acquires) {
            // 调用非公平所的共享锁获取。
            return nonfairTryAcquireShared(acquires);
        }
    }

```

```java 

 abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }
        // 尝试在非公平锁的情况下获取信号量。
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                // 直接开始对锁状态进行操作。不判断是否处于CLH队列队首
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

```

信号量能够被获取，自然也能够被释放。在第一小节中我们也看到了Thread3在另外两个线程释放掉自己的信号量后，Thread3获取到了请求的信号量，继续了线程的工作。释放信号量的操作，实际就是释放锁，重设锁状态的操作。
```java
    // 释放信号量
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        // 调用定义在AQS抽象类中的releaseShared()共通方法实现信号量释放
        sync.releaseShared(permits);
    }
```
```java

public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    private static final long serialVersionUID = 7373984972572414691L;

    public final boolean releaseShared(int arg) {
        // 调用Semaphore重写了tryReleaseShared方法
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
    
    // 唤醒CLH队列中所有等待的线程。
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
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

    
}
```
```java

 abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }
        // 释放信号量
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                // 取得当前的锁状态
                int current = getState();
                // 对锁状态进行计算。
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded"); 
                // 重设锁状态
                if (compareAndSetState(current, next))
                    return true;
            }
        }
        
}
```



##总结
AQS锁系列的博文到此就告一段落了。本系列是以入门为目的的文章。若你真的想精通AQS锁，在阅读了本系列的文章后我认为还需要长时间深入源码一探究竟。
AQS架构庞大，设计巧妙。就像文章所介绍的那样，要整体理解AQS的运作原理，运用AQS锁解决问题不是很难。难点在于AQS架构中有很多细节上的处理比如node，各种锁状态，线程中断后的处理甚至是写代码的风格，各种设计模式的运用，都是需要花时间，也值得花时间进行钻研的。
随着java版本的提升，AQS的内容还在增多。concurrent包里的内容也变得越来越丰富。使用这些高级工具，不仅能解决更多更复杂的需求，提高代码的阅读性的同事，这些高级工具的设计思想也值得我们借鉴。