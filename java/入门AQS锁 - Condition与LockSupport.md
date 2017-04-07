# 入门AQS锁 - Condition与LockSupport

---

在[第一章节](https://www.zybuluo.com/mikumikulch/note/268244)中，我们已经初步接触了ReentrantLock独占锁与Condition接口，并且学习了ReentrantLock与Synchronized关键字的联系与区别，以及Condition接口中3个比较重要的方法的含义与用法。
在本章节中，我们将对第一章节介绍的Condition接口进行更加深入的学习，从而理解由LockSupport提供的更为“先进”的线程间通信是如何在AQS锁中进行运用的。

## 运用Condition
经过前面的学习，我们已经了解了通过Condition对象能够实现唤醒“特定”阻塞线程的工作。下面是一个ReentrantLock与Condition构建的并发示例。通过这个示例来加深和巩固我们对唤醒“特定”阻塞线程的认识。
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionDemo {

    // 获取独占锁
    private ReentrantLock lock = new ReentrantLock();
    // 获取条件1
    private Condition cd1 = lock.newCondition();
    // 获取条件2
    private Condition cd2 = lock.newCondition();
    // 获取条件3
    private Condition cd3 = lock.newCondition();

    public static void main(String[] args) throws InterruptedException {
        ConditionDemo demo = new ConditionDemo();
        ExecutorService es = Executors.newCachedThreadPool();
        Thread1 tr1 = demo.new Thread1();
        Thread2 tr2 = demo.new Thread2();
        Thread3 tr3 = demo.new Thread3();
        // 启动线程任务123.
        es.execute(tr1);
        es.execute(tr2);
        es.execute(tr3);
        Thread.sleep(2000);
        // 依次唤醒各线程任务.
        signalTest(demo);
        es.shutdown();
    }

    // 依次唤醒各线程任务,以观察condition的作用
    public static void signalTest(ConditionDemo demo) throws InterruptedException {
        // 获取独占锁 唤醒cd1的线程
        demo.lock.lock();
        demo.cd1.signal();
        // 释放独占锁 等待thread1执行完毕.
        demo.lock.unlock();
        Thread.sleep(2000);

        // 获取独占锁 唤醒cd2的线程
        demo.lock.lock();
        demo.cd2.signal();
        // 释放独占锁 等待thread2执行完毕.
        demo.lock.unlock();
        Thread.sleep(2000);

        // 获取独占锁 唤醒cd3的线程
        demo.lock.lock();
        demo.cd3.signal();
        // 释放独占锁 等待thread2执行完毕.
        demo.lock.unlock();
        Thread.sleep(2000);
    }

    // 线程任务定义类
    public class Thread1 implements Runnable {

        public Thread1() {
            
        }

        @Override
        public void run() {
            try {
                // 设置线程名称
                Thread.currentThread().setName(Thread1.class.getSimpleName());
                System.out.printf("%s线程启动\n", Thread.currentThread().getName());
                lock.lock();
                // 在cd1上阻塞，并且释放独占锁lock.
                cd1.await();
                System.out.printf("%s线程被唤醒\n", Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }

    public class Thread2 implements Runnable {
        public Thread2() {
            
        }

        @Override
        public void run() {
            try {
                Thread.currentThread().setName(Thread2.class.getSimpleName());
                System.out.printf("%s线程启动\n", Thread.currentThread().getName());
                lock.lock();
                // 在cd2上阻塞，并且释放独占锁lock.
                cd2.await();
                System.out.printf("%s线程被唤醒\n", Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }

        }

    }

    public class Thread3 implements Runnable {
        public Thread3() {
            
        }

        @Override
        public void run() {
            try {
                Thread.currentThread().setName(Thread3.class.getSimpleName());
                System.out.printf("%s线程启动\n", Thread.currentThread().getName());
                lock.lock();
                // 在cd3上阻塞，并且释放独占锁lock.
                cd3.await();
                System.out.printf("%s线程被唤醒\n", Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }

}

------------
Thread2线程启动
Thread1线程启动
Thread3线程启动
Thread1线程被唤醒
Thread2线程被唤醒
Thread3线程被唤醒

```
上例中，3个不同的线程在3个不同的Condition对象上阻塞。然后逐一被唤醒。线程之间的通信互相独立，互不干扰。
可见，使用Condition对象，线程的同步操作，是以“线程”为单位的。而Sychronized，Object.wait()，Object.notify()则是以监视器（锁）为单位对线程进行同步操作的。

那么，Condition是如何做到以线程为单位，对线程进行同步操作的呢？要弄明白这个问题，就需要引入下面的LockSupport的相关知识了。
## Condition与LockSupport
在上面的例子中，我们利用Condition.await()对线程进行了阻塞操作。接下来我们通过源码来分析await()到底做了哪些事情。

```java
// AQS锁共通父类。本类是公平锁与非公平锁的共通父类。
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    private static final long serialVersionUID = 7373984972572414691L;
    .....
    ...
    ..
    .
    // 定义在AQS锁中的内部类ConditionObject。
    public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;

        /**
         * Creates a new <tt>ConditionObject</tt> instance.
         */
        public ConditionObject() { }

        // Internal methods
        
        ....
        ...
        ..
        .
        // 调用Condition对象的await()方法
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            // 将该线程包装成node加入condition等待队列
            Node node = addConditionWaiter();
            // 释放当前线程锁。并设置锁拥有者为空，并且唤醒AQS队列中的head节点线程的下一个线程
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            // 判断当前线程是否在AQSCLH队列。若不在，则线程阻塞
            while (!isOnSyncQueue(node)) {
                // 调用LockSupport类的park方法对线程进行阻塞。park方法为native实现的非开源方法。本篇文章不作深究
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 被唤醒后，重新开始正式竞争锁，同样，如果竞争不到还是会将自己沉睡，等待唤醒重新开始竞争。
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

}
        /**
         * Adds a new waiter to wait queue.
         * @return its new wait node
         */
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            // 线程包装为node对象
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            // 将当前线程包装为node，添加到等待队列并且设置为firstWaiter
            if (t == null)
                firstWaiter = node;
            else
            // 将node添加到等待队列并且设置为lastWaiter
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }

```
经过分析与整理，我们将await()方法实现的处理归纳为以下4条。
**1. ConditionObject对象也维护了一个条件队列。
2. 每当在某个ConditionObject对象上调用await()方法时，会将当前线程封装为node，添加到ConditionObject维护的条件队列中。
3. 当前线程释放掉持有的线程锁，并且唤醒AQS维护的CLH队列中的有效的head线程节点的下一个节点。(若AQS存在的话)注意，若该线程之前由于尝试获取锁失败，而进入了CLH队列的话，那么此时调用await()，释放掉锁后该线程节点就失去了它在CLH队列中的位置。唤醒线程（unparkSuccessor方法）将会重置队列head。
4. 遍历AQS队列，判断当前线程节点是否处于AQS队列中。若不处于队列中，则调用LockSupport.park()阻塞当前线程。**

现在，可能你会对第四条处理感到疑惑。不用着急，当分析完singnal方法的具体实现细节以后，你的疑惑将得到解决。
定义在ConditionObject中的signal方法的源码如下：

```java

public final void signal() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
            // 获取节点中的firstWaiter
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
        
private void doSignal(Node first) {
    do {
        // 修改Condition条件等待队列中的头结点，移除旧节点
        if ((firstWaiter = first.nextWaiter) == null) 
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) && // 将唤醒的Condition头节点，加入到AQS队列中
             (first = firstWaiter) != null);
}
 
    // 将条件等待队列中的线程节点，加入到AQSCLH队列中排队。
     final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
            
         // 将被唤醒的Condition节点中的线程加入到AQSCLH队列中。若没有AQS队列，则新建队列再加入。
        Node p = enq(node);
        int ws = p.waitStatus;
        // 如果该结点的状态为cancel 或者修改waitStatus失败，则直接唤醒。
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }

```
经过分析与整理，我们将signal()方法实现的处理归纳为以下2条。
**1. 将条件等待队列中的头节点移除，队列firstWaiter指向下一位节点对象。
2. 将取得的头节点对象(firstWaiter)，加入到AQS锁的CLH队列尾部。此操作的理由为：“保持公平锁，非公平锁的CLH队列的FIFO原则性”。**

无论是公平或非公平锁，一旦线程主动调用await()阻塞后，它就失去了在队列中的位置。当线程被唤醒时，该线程必须按照FIFO原则进行重新排队。也就是说，即使是非公平锁的线程，在被唤醒后，也需按照FIFO原则，等到前面的节点都处理完成后，自己才能被唤醒从而进入就绪状态。

现在，你是否理解了之前await()方法中的第四条处理的原因了？
当第一次调用await()对线程进行阻塞时，当前线程会首先调用park()进入阻塞，并且加入到条件等待队列中。当某个线程调用此Condition对象的signal时，等待队列中的firstWaiter(第一个阻塞对象)会被加入到AQS锁的CLH队列中。
此时，若外部有一个竞争到锁的线程此时调用了 await，那么 会执行!isOnSyncQueue(node)判断，由于不在同步队列中，所以该线程会正常的执行阻塞方法。
假设有一个被调用 signal ，加入到了 CLH 队列中的线程节点被唤醒。 由于被唤醒的阻塞线程会在 loop 中继续执行 !isOnSyncQueue(node) 方法，而被唤醒的线程不需要进行阻塞，而应该尝试执行竞争锁的逻辑。

所以，!isOnSyncQueue(node)就是用来判断当前线程是否在CLH队列，来确定当前线程是被唤醒的线程，还是获取到锁的外部线程的。

## 总结

1. Condition接口中的await与signal默认是通过AQS类中的内部类ConditionObject实现的。每一个ConditionObject对象都维护了一个单独的等待队列，注意这个队列跟AQS所维护的CLH队列是两个完全不同的队列。
2. AQS维护的CLH队列是用来对竞争锁的线程进行管理的队列。而CondtionObject维护的队列，则是用来对调用了await()方法进入阻塞状态的线程进行管理的队列。
3. 调用了await()方法的线程，会被加入到conditionObject等待队列中，并且唤醒当前AQS的CLH队列中head节点(也就是当前执行中的线程节点)的下一个节点。而线程在某个ConditionObject对象上调用了singnal()方法后，等待队列中的firstWaiter会被加入到AQS的CLH队列中，等待被唤醒。当线程调用unLock()方法释放锁时，CLH队列中的head节点的下一个节点(在本例中是firtWaiter)，会被唤醒。
4. 综上所述，在ConditionObject()上调用await()进入阻塞的线程，其被唤醒的顺序一定和进入阻塞的顺序相同。(非公平锁的情况下，需满足没有外部线程和CLH队列中的线程同时竞争锁这个条件)
**(你可以尝试对本章节的第一个java示例进行修改，让各线程都在cd1上await()，然后调用signal()方法以验证这一点)**
