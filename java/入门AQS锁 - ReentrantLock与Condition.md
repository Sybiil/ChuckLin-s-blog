# 入门AQS锁 - ReentrantLock与Condition

标签（空格分隔）： java

---
网络上关于AQS锁系列的说明介绍文章已经挺多了。撰写本文时我也参考过其他博文。但是我发现，用简单明了的举例说明，整洁易读的排版来把这件事情讲明白，真是一件不容易的事情。
所以希望你喜欢本系列的文章。也欢迎提出建议与指摘。

## ReentrantLock
#### java.util.concurrent.locks.ReentrantLock

ReentrantLock是一个默认非公平的，可重入的，互斥（独占）锁。天哪，光这一句话概念就很多！
不过不用怕，我们下面会这些概念挨个进行详细的介绍。由于理解这些概念，是理解jdk1.5加入的juc包中各种并发类与接口的关键所在。所以不要觉得枯燥，一定要仔细阅读！



1. 独占锁
:   独占锁是指，该锁只能被一个线程获得（持有），其他线程想要获得本锁，必须要等锁的持有者释放该锁后，才能有机会获得。sychronized关键字获取的锁，就是一个独占锁。

2. 非公平锁
:   非公平锁是指，所有处于就绪状态的线程，在独占锁被当前持有锁的线程释放后，都有机会，（依靠jvm，cpu来判断）竞争被释放后的锁。sychronized关键字获取的锁，就是一个非公平锁。
注意，在非公平独占锁被释放前，尝试竞争非公平独占锁失败的线程，会进入CLH队列排队。持有锁的线程释放锁的同时会唤醒队列中的最先一个阻塞线程使其进入就绪状态。此时，被唤醒的线程与其他就绪线程都有机会竞争到锁。

3. 可重入锁
:   可重入锁是指，能够被同一个线程，反复多次获取的锁。sychronized关键字获取的锁，就是一个可重入锁。

4. 公平锁
:   公平锁是指，所有竞争同一个独占公平锁的线程，都必须通过CLH队列(FIFO)的规则来获取线程锁。尝试获取公平锁的线程，会首先判断“独占锁是否被占用”，与“当前线程是否处于CLH队列head处”这两个条件，若不满足任何一个，则获取失败。然后以node的形式将当前线程加入CLH队列的末尾。当公平锁被释放时，会唤醒队列中处于阻塞状态，且位于CLH队列head处的线程(node)。继而得以实现“公平”获得锁的功能。



## Condition
#### java.util.concurrent.locks.Condition
Condition接口主要是用来对持有锁的线程进行同步控制。Condition接口中有以下3个比较重要的方法如下：

        Condition.await() == Object.wait()
        Condition.signal() == Object.notify()
        Condition.signalAll() == Object.notifyAll()
除了这些方法以外，Condition接口中还有很多方法能对线程锁进行更精确的操作。
在后面的章节中，我们还会对Condition的内部原理做更加深入的分析与说明。请关注AQS系列的博文更新。

在下面的内容中，我们通过一个生产者消费者模型，来说明Condition的运用方式以及以上提到的三个方法与Object类中的三个方法在运用上的区别。

**假设现在需要你设计一个单生产者，单消费者模型。
一个线程，从缓存中读取（消费）对象。另一个线程，往缓存里写入对象。读取线程消费对象后，唤醒写入线程。若缓存中没有可读取对象，该读取线程就阻塞。写入线程写入完对象后，唤醒读取线程。若缓存超出容量大小，就阻塞写入线程。**

**生产者**
```flow
st=>start: 生产者
prepare_write_file=>operation: 准备写入缓存
cond=>condition: size < max
write_file=>operation: 写入对象
notify_reader=>operation: 唤醒消费者
e=>end
w=>end: wait

st(right)->prepare_write_file(right)->cond(yes)->write_file(right)->notify_reader(right)->e
cond(no)->w
```

**消费者**
```flow
st=>start: 消费者
prepare_read_file=>operation: 准备读取缓存
cond=>condition: size == 0
read_file=>operation: 消费对象
notify_writer=>operation: 唤醒生产者
e=>end
w=>end: wait

st(right)->prepare_read_file(right)->cond(no)->read_file(right)->notify_writer(right)->e
cond(yes,right)-->w
```



在接触AQS锁之前，我们通常会选择synchronized关键字与Object.wati，Object.notify来解决这个问题。
```java
import java.util.LinkedList;
import java.util.List;

//生产者(写入线程)
public class SynchronizedProviderDemo implements Runnable {

    public static List<String> cache = new LinkedList<String>();

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                Thread.sleep(10);
                synchronized (cache) {
                    if (cache.size() < 1) {
                        System.out.println("生产bread");
                        cache.add(new String("bread"));
                        cache.notifyAll();
                    } else {
                        System.out.println("bread生产超过总量");
                        cache.wait();
                    }
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}



-------------------------------------------------------------------


//消费者(读取线程)
public class SynchronizedConsumerDemo implements Runnable {

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                Thread.sleep(10);
                synchronized (SynchronizedProviderDemo.cache) {
                    if (SynchronizedProviderDemo.cache.size() == 0) {
                        System.out.println("bread全部消费完毕");
                        SynchronizedProviderDemo.cache.wait();
                    } else {
                        System.out.println("消费bread");
                        SynchronizedProviderDemo.cache.get(0);
                        SynchronizedProviderDemo.cache.remove(0);
                        SynchronizedProviderDemo.cache.notifyAll();
                        
                    }
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

}


-------------------------------------------------------------------


public class ExecuteConsumerProviderDemoClass {

    public static void main(String[] args) {
        Thread d1 = new Thread(new SynchronizedConsumerDemo());
        Thread d2 = new Thread(new SynchronizedProviderDemo());

        d1.start();
        d2.start();
    }

}


----------
生产bread
消费bread
生产bread
bread生产超过总量
消费bread
bread全部消费完毕
生产bread
消费bread
生产bread
消费bread
生产bread
.
.
.
```
从上面的Demo的执行结果上来看，问题解决了。代码按照我们预想的同步规律执行成功。
接下来让我们来尝试一下在多生产者单消费者模型中，是否还能得到如愿的结果
```Java

public class ExecuteConsumerProviderDemoClass {

    public static void main(String[] args) {
        Thread d1 = new Thread(new SynchronizedConsumerDemo());
        Thread d2 = new Thread(new SynchronizedProviderDemo());
        // 产生10个生产消费者线程
        for (int i = 0; i < 10; i++) {
            new Thread(new SynchronizedProviderDemo()).start();
            new Thread(new SynchronizedProviderDemo()).start();
        }
        d1.start();
        d2.start();
    }

}


----------
生产bread
bread生产超过总量
消费bread
bread全部消费完毕
生产bread
bread生产超过总量
消费bread
生产bread
消费bread
生产bread
```
代码依然按照预想的结果执行了。

但是需要注意，每当线程在某个锁上调用notifyAll()时，所有持有这个锁的处于阻塞状态的线程都会被唤醒。
虽然在上面的例子中，我们通过逻辑避免了“生产者调用notfiyAll()时连同生产者一起唤醒”这样的情况出现，
但是，这样的处理方式实在是十分“低级”，且不易阅读。

试想，我们是否能够显示的在代码中只唤醒“对方”处于阻塞状态的线程呢？
换句话说，在持有同一个锁的情况下，生产者只唤醒消费者，消费者只唤醒生产者，这样的需求是否能实现？
答案是可以的。使用Condition配合Lock，就能解决这样的需求了。

让我们来看看Condition，如何在多生产者消费者模型中实现交替唤醒功能。
下面是一个实际项目(web application)中比较常见的，多生产者单消费者模型示例。


```Java
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockConditionDemo {

    private Lock theLock = new ReentrantLock();
    // 消费者用判断条件
    private Condition full = theLock.newCondition();
    // 生产者用判断条件
    private Condition empty = theLock.newCondition();

    private static List<String> cache = new LinkedList<String>();

    // 生产者线程任务
    public void put(String str) throws InterruptedException {
        Thread.sleep(200);
        // 获取线程锁
        theLock.lock();
        try {
            while (cache.size() != 0) {
                System.out.println("超出缓存容量.暂停写入.");
                // 生产者线程阻塞
                full.await();
                System.out.println("生产者线程被唤醒");
            }
            System.out.println("写入数据");
            cache.add(str);
            // 唤醒消费者
            empty.signal();
        } finally {
            // 锁使用完毕后不要忘记释放
            theLock.unlock();
        }
    }

    // 消费者线程任务
    public void get() throws InterruptedException {
        try {
            while (!Thread.interrupted()) {
                // 获取锁
                theLock.lock();
                while (cache.size() == 0) {
                    System.out.println("缓存数据读取完毕.暂停读取");
                    // 消费者线程阻塞
                    empty.await();
                }
                System.out.println("读取数据");
                cache.remove(0);
                // 唤醒生产者线程
                full.signal();
            }

        } finally {
            // 锁使用完毕后不要忘记释放
            theLock.unlock();
        }
    }

}




-------------------------------------------------------------------

public class ExecuteReentrantLockConditionDemo {

    public static void main(String[] args) {
        final ReentrantLockConditionDemo rd = new ReentrantLockConditionDemo();
        // 创建1个消费者线程
        for (int i = 0; i < 1; i++) {
            
            new Thread(new Runnable() {
                public void run() {
                    try {
                        rd.get();
                    } catch (InterruptedException e) {

                        e.printStackTrace();
                    }
                }
            }).start();
        }

        // 创建10个生产者线程
        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                public void run() {
                    try {
                        rd.put("bread");
                    } catch (InterruptedException e) {

                        e.printStackTrace();
                    }
                }
            }).start();
        }

    }

}
-------------
缓存数据读取完毕.暂停读取
写入数据
超出缓存容量.暂停写入.
超出缓存容量.暂停写入.
超出缓存容量.暂停写入.
超出缓存容量.暂停写入.
超出缓存容量.暂停写入.
读取数据
缓存数据读取完毕.暂停读取
写入数据
超出缓存容量.暂停写入.
超出缓存容量.暂停写入.
超出缓存容量.暂停写入.
生产者线程被唤醒
超出缓存容量.暂停写入.
读取数据
缓存数据读取完毕.暂停读取
生产者线程被唤醒
写入数据

```
通过Condition与Lock结合运用写出来的代码，相对notifyAll()来说更加易读。阻塞，唤醒，同步读写的过程，体现的更加清楚了。

## 总结
Condition与ReentrantLock搭配起来，能够实现比Synchronized关键字与Object.wait/notify的组合更精确地对线程的控制。
Object.wait/notify是以“锁”为单位对线程进行阻塞唤醒，而Condition则是以“线程”为单位，对线程进行阻塞与唤醒的。所以Condition能够唤醒某个特定的阻塞线程。而Object.notify则不可以。




