# 认识与拥抱协程

---

##并发与并行
计算机自诞生以来，有两方面的性能在持续改进。一是尝试让计算机运行的更快，我们设计了并发使系统具备处理多个任务的能力。二是人类在尝试让计算机做的更多，我们设计了并行使系统可以同时运行多个任务。

> 我们利用术语并发（concurrency）来指一个同时具有多个活动的系统。用并行（parallelism）指代同时运行几个任务使一个系统运行的更快。

为使并发与并行在计算机中得到广泛的应用，操作系统屏蔽了底层硬件细节，提供了进程与线程两种抽象供开发人员使用，这就是线程的由来。
线程的概念想必你已经很熟悉了。同一个进程由多个称为线程的执行单元组成，线程们共享一个进程上下文。cpu 在多个线程之间由操作系统调度器进行切换，并保存当前线程状态到寄存器或主存。

不过除了线程以外，协程也能使计算机具备多任务处理能力。换句话说，使用协程也能实现并发程序。

##为什么使用协程
协程是一种已经存在了几十年的轻量级线程，这种抽象技术具备线程的大部分特点。利用协程也能设计出支持并发的程序。与线程的不同之处在于，协程不是抢占式的，你在代码中必须标注出什么时候可以安全的暂停执行当前任务，让位给其他任务。

学术定义总是枯燥难懂，还是代码示例更加的平易近人。

传统的生产者-消费者模型是一个线程写消息，一个线程取消息，通过锁机制控制队列和等待，但一不小心就可能死锁。下面是一个用 python 的协程实现的一个生产者消费者模型。给自己20分钟，近距离观察这个模型，快速构建协程在你大脑中的基本面貌，分析新模型的优缺点。


```python
# 生成器函数（消费者）
def consumer():
    r = ''
    while True:
        # 切换任务到生产者并 yield 结果
        n = yield r
        if not n:
            return
        print('[CONSUMER] Consuming %s...' % n)
        r = '200 OK'

# 生产者
def produce(c):
    # 开启生成器
    c.send(None)
    n = 0
    while n < 5:
        n = n + 1
        print('[PRODUCER] Producing %s...' % n)
        # 切换任务到消费者
        r = c.send(n)
        print('[PRODUCER] Consumer return: %s' % r)
    c.close()

c = consumer()
produce(c)


```
执行结果
```
[PRODUCER] Producing 1...
[CONSUMER] Consuming 1...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 2...
[CONSUMER] Consuming 2...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 3...
[CONSUMER] Consuming 3...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 4...
[CONSUMER] Consuming 4...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 5...
[CONSUMER] Consuming 5...
[PRODUCER] Consumer return: 200 OK

```

模型改用协程，生产者生产消息后直接通过 yield 跳转到消费者开始执行，待消费者执行完毕后，切换回生产者继续生产，效率极高。

可见，相比线程，协程实现的并发逻辑具备无锁、轻量级（无上下文切换）、逻辑简单开发快速等特点。在业务场景合适的情况下，没有理由不选择协程。

##更多的业务场景
到此协程的基本概念应该已经在你的大脑中留下了一些痕迹。接着请允许我再介绍一个协程的经典示例，帮助你让协程在脑海里变得更加清晰牢固。

谈到协程的运用，就必须提到 lua。或许是因为 lua 不支持线程的缘故。在这门纯粹的面向原型的编程语言中，协程得到了大量运用。来看一个通过协程与生成器相结合，使用几十行代码开发的一个定时任务调度器。

```lua
-- 进攻函数
function punch( ... )
	for i=1,5 do
		print('punch' .. i)
		-- yelid 执行秒数 下次执行位置
		scheduler.wait(1.0)
	end
end

-- 防守函数
function block( ... )
	for i=1,3 do
		print('block' .. i)
		-- yelid 执行秒数 下次执行位置
		scheduler.wait(2.0)
	end

end


-- 创建协程任务
scheduler.schedule(0.0,coroutine.create(punch))
scheduler.schedule(0.0,coroutine.create(block))


-- 待处理任务队列
local pending = {}

-- 入队函数
local function schedule(time,action)
	pending[#pending + 1] = {
	time = time,
	action = action
}

-- 按照未来执行时间排序
sort_by_time(pending)

end

-- 任务执行出让函数
local function wait(seconds)
	coroutine.yield(seconds)
end

-- 主函数
function run()
	-- 从队列中获取处理任务
	while #pending > 0 do

		while os.clock() < pending[1].time do end -- busy-wait

		-- 移除任务
		local item = remove_first(pending)
		-- 执行生成器
		local _, seconds = coroutine.resume(item.action)
		-- 将任务加上生成器 yelid 回来的执行时间，放回待处理队列中
		if seconds then 
			later = os.clock() + seconds
			sschedule(later, item.action)
		end
	end

```



##结语
协程让你从并发锁、上下文切换、线程通信、共享内存等桎梏中解脱出来，专心于业务逻辑的思考，极大提高了开发效率。清晰明了的结构让代码具备极高的维护性。单线程的特点可节省上下文切换的开销。
所以，当今后再次遇到并发程序的开发时，试试把你的逻辑交给协程。现在，想想你的代码中哪些地方可以使用到协程？赶快重构吧。

---

*参考书籍*
*《深入理解计算机系统》*
*《七周七语言-卷2》*
*参考内容*
*[廖雪峰的官网](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432090171191d05dae6e129940518d1d6cf6eeaaa969000)*

