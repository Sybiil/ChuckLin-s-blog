# 深入 DelayQueue 内部实现

---

如果要提及使用 java 的程序员最幸福（e xin）的事情的话，其中一定有 java 对于并发全面的支持所带来的便利了（fu za）
Doug Lea  提供的 JUC 包自从 jdk1.7 开始就提供了大量的阻塞队列供我们使用，其中比较特别的就有延迟阻塞队列。
##阻塞队列
阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。下面是 java 常见的阻塞队列。

- ArrayBlockingQueue ：一个由数组结构组成的有界阻塞队列。
- LinkedBlockingQueue ：一个由链表结构组成的有界阻塞队列。
- PriorityBlockingQueue ：一个支持优先级排序的无界阻塞队列。
- DelayQueue：一个使用优先级队列实现的无界阻塞队列。
- SynchronousQueue：一个不存储元素的阻塞队列。
- LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
- LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。


##延迟阻塞队列
延迟阻塞队列就是在阻塞队列的提供商提供了延迟获取任务的功能。先用一个例子来了解延迟阻塞队列的用法。
```java
import java.time.Instant;
import java.time.LocalDateTime;
import java.util.concurrent.DelayQueue;
import java.util.concurrent.Delayed;
import java.util.concurrent.TimeUnit;

/**
 * 延迟队列示例
 */
public class DelayQueueTester {

    private static DelayQueue<DelayTask> delayQueue = new DelayQueue<>();

    static class DelayTask implements Delayed {

        // 延迟时间
        private final long delay;
        // 到期时间
        private final long expire;
        // 数据
        private final String msg;
        // 创建时间
        private final long now;


        /**
         * 初始化 DelayTask 对象
         *
         * @param delay 延迟时间 单位：微妙
         * @param msg   业务信息
         */
        DelayTask(long delay, String msg) {
            this.delay = delay; // 延迟时间
            this.msg = msg; // 业务信息
            this.now = Instant.now().toEpochMilli();
            this.expire = now + delay; // 到期时间 = 当前时间+延迟时间
        }

        /**
         * 获取延迟时间
         *
         * @param unit 单位对象
         * @return
         */
        @Override
        public long getDelay(TimeUnit unit) {
            return unit.convert(expire - Instant.now().toEpochMilli(), TimeUnit.MILLISECONDS);
        }

        /**
         * 比较器
         * 比较规则：延迟时间越长的对象越靠后
         *
         * @param o
         * @return
         */
        @Override
        public int compareTo(Delayed o) {
            if (o == this) // compare zero ONLY if same object
                return 0;
            return (int) (this.getDelay(TimeUnit.MILLISECONDS) - o.getDelay(TimeUnit.MILLISECONDS));
        }

        @Override
        public String toString() {
            return "DelayTask{" +
                    "delay=" + delay +
                    ", expire=" + expire +
                    ", msg='" + msg + '\'' +
                    ", now=" + now +
                    '}';
        }
    }

    /**
     * 生产者线程
     *
     * @param args
     */
    public static void main(String[] args) {
        initConsumer();
        try {
            // 等待消费者初始化完毕
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        delayQueue.add(new DelayTask(1000, "Task1"));
        delayQueue.add(new DelayTask(2000, "Task2"));
        delayQueue.add(new DelayTask(3000, "Task3"));
        delayQueue.add(new DelayTask(4000, "Task4"));
        delayQueue.add(new DelayTask(5000, "Task5"));

    }

    /**
     * 初始化消费者线程
     */
    private static void initConsumer() {
        Runnable task = () -> {
            while (true) {
                try {
                    System.out.println("尝试获取延迟队列中的任务。" + LocalDateTime.now());
                    System.out.println(delayQueue.take());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        Thread consumer = new Thread(task);
        consumer.start();
    }
}

---
尝试获取延迟队列中的任务。2017-04-05T18:28:03.282
DelayTask{delay=1000, expire=1491388087234, msg='Task1', now=1491388086234}
尝试获取延迟队列中的任务。2017-04-05T18:28:07.235
DelayTask{delay=2000, expire=1491388088235, msg='Task2', now=1491388086235}
尝试获取延迟队列中的任务。2017-04-05T18:28:08.237
DelayTask{delay=3000, expire=1491388089235, msg='Task3', now=1491388086235}
尝试获取延迟队列中的任务。2017-04-05T18:28:09.237
DelayTask{delay=4000, expire=1491388090235, msg='Task4', now=1491388086235}
尝试获取延迟队列中的任务。2017-04-05T18:28:10.240
DelayTask{delay=5000, expire=1491388091235, msg='Task5', now=1491388086235}
尝试获取延迟队列中的任务。2017-04-05T18:28:11.240

```
上面的例子中当队列中没有元素时，消费者阻塞在 take 方法上面。直到队列中添加进延迟任务并且满足延时任务时，任务被成功取出。 

##DelayQueue 实现原理
理解 DelayQueue 实现原理最好的办法是从 take方法与 add 方法入手。
###take
```java
/**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element with an expired delay is available on this queue.
     *
     * @return the head of this queue
     * @throws InterruptedException {@inheritDoc}
     */
    public E take() throws InterruptedException {
        // 获取锁。每个延迟队列内聚了一个重入锁。
        final ReentrantLock lock = this.lock;
        // 获取可中断的锁。
        lock.lockInterruptibly();
        try {
            for (;;) {
                // 尝试从优先级队列中获取队列头部元素
                E first = q.peek();
                if (first == null)
                    // 无元素，当前线程节点加入等待队列，并阻塞当前线程
                    available.await();
                else {
                    // 通过延迟任务的 getDelay 方法获取延迟时间
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        // 延迟时间到期，获取并删除头部元素。
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            // 线程节点进入等待队列 x 纳秒。
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            // 若还存在元素的话，则将等待队列头节点中的线程节点移动到同步队列中。
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }

```

###add
```java
/**
     * Inserts the specified element into this delay queue.
     *
     * @param e the element to add
     * @return {@code true} (as specified by {@link Collection#add})
     * @throws NullPointerException if the specified element is null
     */
    public boolean add(E e) {
        return offer(e);
    }

    /**
     * Inserts the specified element into this delay queue.
     *
     * @param e the element to add
     * @return {@code true}
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        // 获取到重入锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            // 添加成功元素
            if (q.peek() == e) {
                leader = null;
                // 将等待队列中的头节点移动到同步队列。
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```

###DelayQueue 的主要成员
```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    // 持有内部重入锁。
    private final transient ReentrantLock lock = new ReentrantLock();
    // 优先级队列，存放工作任务。
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    
    private Thread leader = null;
    // 依赖于重入锁的 condition。
    private final Condition available = lock.newCondition();
}
```

如果你具备 JUC 包中的 Lock 接口以及同步队列器的相关知识，上述代码大部分应该都比较容易理解。DelayQueue 将实现了 Delayed 接口的对象添加到优先级队列中，通过 在依赖内聚重入锁的 Condition 上调用 await(delayTime) 方法，实现了延迟获取阻塞队列中元素的功能。相关的细节请参考代码段中的中文注释。

若你对相关的同步队列器的知识还不够熟悉，请参考我的博客中关于*《AQS 锁》*系列的相关文章。 

DelayQueue 的难点在于*【LF designPattern (跟随者设计模式)】*的运用。由于 DelayQueue 的主要功能实现与 LF 设计模式关系不大，故本章节不涉及这方面的内容。如果你对这方面的知识有兴趣，请参考
《Pattern-Oriented Software Architecture》软件架构设计模式的系列书籍。

##总结

- DelayQueue 是一个内部依靠 AQS 队列同步器所实现的无界延迟阻塞队列。
- 延迟对象需要覆盖 getDelay 与 compareTo 方法，并且要注意 getDelay 的时间单位的统一，以及 compareTo 根据业务逻辑进行合理的比较逻辑重写。
- DelayQueue 中内聚的重入锁是非公平的。
- DelayQueue 是实现定时任务的关键。下一章将通过定时任务的讲解再次回顾 DelayQueue。

---
Copyright 2017/08/15 by Chuck Lin

若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！
![alipay.jpg-17.7kB][99]
![wechat.jpg-16.7kB][98]


[99]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[98]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg
