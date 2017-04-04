# 入门AQS锁 - ReadLock与WriteLock

---

1. WriteLock(写入锁)。
:   写入锁是一个可重入的，默认非公平的独占锁。
关于独占锁的概念介绍请参考[ReentrantLock](https://www.zybuluo.com/mikumikulch/note/268244)

2. ReadLock(读取锁)。
:   读取锁是一个可重入的，默认非公平的共享锁。
共享锁就是指这个锁能够被多个线程同时获得。每次线程尝试获取一个共享锁时，共享锁会判断当前锁的状态。若尝试获取的锁为非公平共享锁，并且内部的共享最大计数小于MAX_COUNT的话，则将共享锁的获取计数+1。然后获取该锁。若尝试获取的锁为公平共享锁，则判断当前线程是否需要阻塞(是否处在CLH队列首位)，若需要，则将自己加入到CLH队列，等待被唤醒后获取线程锁。若不需要，则更新共享锁获取状态，获取线程锁。
若尝试获取的锁为非公平共享锁，则只要满足条件就无视队列，直接获取锁。

在学习ReadLock之前，首先有必要了解一下它们与顶级类**ReentrantReadWriteLock**的关系，以方便我们掌握读写锁。
*(jdk1.7version)*
**java.util.concurrent.locks.ReentrantReadWriteLock**
```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    /** 持有内部类读取锁成员变量引用 */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** 持有内部类写入锁成员变量引用 */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;
    ....
    ....
    ....
    // 通过writeLock()返回独占写入锁
    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    // 通过readLock()返回共享读取锁
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```
可见，由于ReentrantReadWriteLock中持有ReadLock和WriteLock的成员变量，而若要取得ReadLock，WriteLock，则必须创建ReentrantReadWriteLock的对象，然后通过readLock(),writeLock()返回读写锁对象。

接下来再进一步分析ReentrantReadWriteLock，ReadLock，WriteLock的构造函数。
```java
    // 读写锁顶级类ReentrantReadWriteLock构造函数
    public ReentrantReadWriteLock(boolean fair) {
    // 通过多态持有公平锁或非公平锁对象
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
    
-------------------------------------------------------------------
    // 读取锁构造函数
    protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
    }
    
-------------------------------------------------------------------     
    // 写入锁构造函数
    protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
    }
```
通过分析3个构造函数可得出，ReadLock，WriteLock中持有的sync变量，就是他们的顶级类ReentrantReadWriteLock中持有的sync变量。换句话说，读写锁中持有相同的sync对象。公平锁与非公平锁是sync的子类，而sync类又是AQS锁的子类。

原来，ReadLock，与WriteLock就是通过持有同一个AQS锁来保证互相影响，读写互斥的功能的。

现在，你是否觉得有点头晕？。。。不用担心。。读写锁概念性的东西比较多，如果并发相关的知识掌握的不够好，初次接触就去深究读写锁的实现原理，会比较难以理解。

好消息是，读写锁使用起来其实相当方便。
我们先来通过一个例子了解一下读写锁的大致用法。当我们熟悉了使用方法过后，再来深究底层原理，就变的容易多了。

**用读写锁实现对某个缓存文件进行同步读写处理。允许多个线程同时对缓存文件进行读取处理，并禁止在读取文件的时候，对文件进行写入修改等操作。禁止多个线程同时对缓存文件进行写入处理，并禁止在写入文件的时候，对文件进行读取操作。**
```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLockDemo {
    // 读写锁顶级类
    private ReadWriteLock lk = new ReentrantReadWriteLock();
    // 缓存文件
    private int cacheData = 0;

    
    // 读取缓存文件
    public void cacheRead() throws InterruptedException{
        // 缓存能够供多个用户同时读取，获取读取锁
        try {
            // 通过顶级类的method获取到读取锁
            lk.readLock().lock();
            System.out.printf("准备读取数据,获取读取锁。缓存数据内容为：%d\n" , cacheData);
            // 模拟读取线程花费一定时间读取数据。
            Thread.sleep(1000);
            System.out.printf("数据读取完成。缓存数据内容为：%d\n" , cacheData);
        } finally {
            lk.readLock().unlock();
        }

    }
    
    // 写入缓存文件
    public void cacheWrite() throws InterruptedException{
        // 缓存只能同时供1个用户写入，获取写入锁
        try {
            // 通过顶级类的method获取到写入锁
            lk.writeLock().lock();
            System.out.printf("准备写入数据,获取写入锁。缓存数据内容为：%d\n" , cacheData);
            // 模拟写入线程花费一定时间写入数据。
            Thread.sleep(1000);
            cacheData++;
            System.out.printf("写入完毕。缓存内容为：%d\n" , cacheData);
        } finally {
            lk.writeLock().unlock();
        }
       
    }

}


-------------------------------------------------------------------


import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecuteReadWriteLockDemo {

    public static void main(String[] args) {
        ExecutorService es = Executors.newCachedThreadPool();
        final ReadWriteLockDemo rwd = new ReadWriteLockDemo();
        int i = 0, j = 0;
        // 建立3个写入线程进行写入工作.
        while (i < 3) {
            es.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        rwd.cacheRead();
                    } catch (InterruptedException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }

            });
            i++;
        }
        
        // 建立3个读取线程进行读取工作.
        while (j < 3) {
            es.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        rwd.cacheWrite();
                    } catch (InterruptedException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }

            });
            j++;
        }
        es.shutdown();
    }

}

--------------------------------------
准备读取数据,获取读取锁。缓存数据内容为：0
准备读取数据,获取读取锁。缓存数据内容为：0
准备读取数据,获取读取锁。缓存数据内容为：0
数据读取完成。缓存数据内容为：0
数据读取完成。缓存数据内容为：0
数据读取完成。缓存数据内容为：0
准备写入数据,获取写入锁。缓存数据内容为：0
写入完毕。缓存内容为：1
准备写入数据,获取写入锁。缓存数据内容为：1
写入完毕。缓存内容为：2
准备写入数据,获取写入锁。缓存数据内容为：2
写入完毕。缓存内容为：3
```
从执行结果来看，由于读取数据的线程获得的线程锁为读取锁，所以有多个线程同时获得了读取锁，对数据进行共同读取。
写入线程获取了写入锁，写入锁是独占锁，所以写入线程总是完成一个完整的写入动作后，才会有新的线程获取到写入锁。

现在，你是否对读写锁有了基本的认识了？你可以使用读写锁自己写一些Demo，这会帮助你更深的理解读写锁。

## 总结
读写锁其实就是一种读写分离思想的体现。
在Java中，还有其他体现读写分离思想的类。你可以尝试通过谷歌找找他们，再看看他们是如何实现的。
最后，如果项目中真的有需要，你最好使用读写锁来实现相关需求。
对于java语言，尽量采用高级的处理方式来解决问题，总是比较好的选择。





