# Java中内部类到底有什么用？    

标签（空格分隔）： java

---

java中内部类种类较多，语法比较复杂，用法也不尽相同。
概括下来，可以分类为以下五种内部类。

1. *内部类*
2. *嵌套内部类*
3. *局部内部类*
4. *接口内部类*
5. *匿名内部类*

本篇文章只对实际项目开发中用的较多的，普通内部类与匿名内部类做一定介绍。其他三种若有兴趣请自行通过谷歌或书籍进行了解。

##内部类
首先通过一个简单的小示例，来看看内部类的语法吧。
```java
import java.util.HashMap;

public class Parcell {

    private HashMap<String, String> testMap = new HashMap<String, String>();

    class Contents {
        // 返回一个外部类的引用.
        public Parcell ParcellRef = Parcell.this;
    }

    class Destination {
        public void putSomethingInMap() {
            testMap.put("hello", "world");
            System.out.println(testMap.get("hello"));
        }
        
    }

    public Destination to() {
        return new Destination();
    }

    public Contents contents() {
        return new Contents();
    }

    public void ship(String dest) {
        Contents c = new Contents();
        Destination d = new Destination();
    }

    public static void main(String[] args) {
        Parcell p = new Parcell();
        Parcell.Contents c = p.contents();
        Parcell.Destination d = p.to();
        d.putSomethingInMap();
        Parcell.Contents c1 = p.new Contents();
    }

}

```
内部类的语法和相信大家都很熟悉。
在这里我再自作主张的为大家概括一下

1. 普通内部类持有一个指向外部类的引用。要创建普通内部类，一定要先创建外部类。
2. 普通内部类就像人体的心脏一样，能够随意访问外部类的任意成员变量。
3. 在内部类中可以通过“外部类类名.this”的方式返回一个指向外部类实例的引用.如Parcell.this
4. 在外部类的static方法中若要创建内部类对象，则需要通过“外部类类名.new XXX()”的方式来创建。
5. 普通内部类中不能拥有静态成员变量。静态内部类中可以拥有静态成员变量。也可以拥有非静态成员变量。但是静态内部类不能访问外部类中非静态的成员变量。而普通内部类可以访问外部类的静态成员变量。


>为什么static方法中需要p.new XXX()的方式而非static方法中我们直接new 内部类名 就可以创建一个对象了呢？
如果你有这样的疑问请再看看第一条，一定可以想明白的。

普通内部类的语法大致就是这样了。
那么，回到我们主题。

内部类到底有什么用？我们在实际项目中，应该如何使用内部类呢？
内部类的用处或者说用法，归根结底，主要就是一点：
:   “内部类，主要是设计出来用来解决java中所‘缺少’的，多重继承的概念的。”

>什么？难道java不是靠接口来实现多重继承的吗？我认为，这种说法对，也不对。下面我们来看这样一个例子。

**你正在参与一项代号X的星际飞船项目。
宇航局的人希望飞船上的综合机器人能够完成两个工作。一是作为向导，能够供人们查阅信息。二是作为修理机器人，完成一些简单的飞船日常维护工作。
作为软件工程师，你被要求编写一段代码来实现这两个功能。**

**好消息是，向导部分的功能与飞船修理维护的功能，已经由你的同事们完成了！太好了，我只需要调用他们提供给我们的接口就大功告成了！
坏消息是，同事们编写的向导功能与飞船修理功能的接口，竟然都叫做work!
这可伤脑筋了，应该怎么办呢？**


```java 
public class Guider {

    public void work(String name) {
        System.out.println("欢迎光临" + name + ",请查阅飞船信息");
    }

}

-------------------------------------------------------------------

public class Repairer {
    
    public void work (String name) {
        System.out.println("你好" + name + ",开始准备对飞船进行维护.");
    }

}

-------------------------------------------------------------------

public class SpacecraftRobot extends Guider {
    
    
    public void doGuidWork(String name) { 
        // 调用guider的work方法
        work(name);
    }
    
    public void doRepairWork(String name) {
        // 返回内部类引用，调用内部类实例的work方法。
        new repairerRobot().doRepairWork(name);
    }
    
    public class repairerRobot extends Repairer {
        
        public void doRepairWork(String name) {
            work(name);
        }
    }
}

```

**太棒了。通过使用内部类与“多重继承”，我们实现了这个功能。现在这个综合机器人能够正常工作了！**

对于用户来说，只需要走到机器人面前，告诉机器人你想要doGuidWork还是doRepairWork，它就能够帮你干活儿了。内部类的代码对用户，对外界彻底隐藏了起来，用户唯一能够获得的信息就是这两个方法而已。



##匿名内部类
**综合机器人原型机试做成功后，新的工作来了！
我们需要对原型机进行量产。以满足每艘星际飞船的需要。
现在我们要编写一间生产综合机器人的工厂。每当我们访问一次工厂，就能够从工厂中提取出一台崭新的综合机器人。聪明的你想到了用工厂设计模式来解决这个问题！但是由于有了内部类，所以我们的工厂，稍稍显得有点不同**


```java 
// 机器人工厂接口。通过getSpaceCraftRobot方法对外提供机器人
public interface SpaceCraftRobotFactory {
    SpacecraftRobot getSpaceCraftRobot();
}

-------------------------------------------------------------------


public class ProduceSpaceCraftRobot {
    // 再也不用显示的创建工厂类的对象了！
    private ProduceSpaceCraftRobot() {

    }
    // 通过匿名内部类，创建工厂对象！将工厂封装到了内部。不对外界暴露
    public static SpaceCraftRobotFactory produceRobot = new SpaceCraftRobotFactory () {
        public SpacecraftRobot getSpaceCraftRobot() {
            return new SpacecraftRobot();
        }
    };
    
}

-------------------------------------------------------------------

// 客户
public class Consumer {

    public static void main(String[] args) {
        // 客户来提取机器人了.
        SpacecraftRobot x1 = ProduceSpaceCraftRobot.produceRobot.getSpaceCraftRobot();
        x1.doGuidWork("lch");
        x1.doRepairWork("lch");
    }

}

```

**通过创建匿名内部类，我们使传统的工厂设计模式优雅了许多！再也不用在外部编写new xxxFactory()这样丑陋，多余的代码了。
现在的工厂被匿名内部类隐藏了起来。客户只需要关心有没有拿到称心如意的机器人。不应该，不需要关心工厂的名字，也不需要知道工厂是干嘛的。**


真棒，你顺利完成了宇航局交给你的任务。
恭喜你，你是这个星球的英雄。



##Java中的闭包，闭包与内部类的关系
作为一个程序员，即使你从来没有使用过，你也应该听说过闭包与回调。
要从java，特别是j2ee的方向入手去讲解闭包与回调，会比较困难。
所以我们首先从python来入手，来了解闭包与回调到底是什么。

>**python是一门优秀的解释性语言。你应该掌握他。**

下面我们来看一看，标准的回调，在Python中是什么样子的。
```python
#定义一个返回sum函数的名叫base_sum函数的函数
def base_sum(a,b):
    #在base_sum中定义一个sum()函数用来计算base_sum(a,b)的形参之合
    def sum():
        #返回a+b的值
        return a + b
    #返回定义的sum函数
    return sum
#调用base_sum返回函数sum()，可以理解为返回了一个函数指针
return_method = base_sum(1,2)
#打印出返回的函数对象
print(return_method)
#通过指针回调函数对象，返回a与b的合
print(return_method())

----------
<function base_sum.<locals>.sum at 0x1018f3c80>
3
```
对于java程序员来说，在一个函数中定义另外一个函数也许会比较烧脑。
你可以试着这样去理解他：
**“首先你需要了解的是，函数也需要占据内存空间，所以函数在内存中也是有地址的。在c语言中，函数名就代表这个函数的地址。
如果你有过c语言的编程经验，你就应该知道在一个函数中，返回一个指针是一件很容易的事情。
所以，对于以上这段python代码，你可以尝试把它理解为：
base_sum()函数中定义了一个指向sum()函数的指针，并且这个指针作为base_sum()的返回值。”**

好了，现在我们根据上面的例子，来“定义一下闭包”。
:   调用外部函数，返回一个持有外部函数变量，参数引用的内部函数对象的程序结构，我们就称它为“闭包”。

遗憾的是，java中没有为我们显示的提供指针供我们操作，也没有提供类似python，javascrpit中的函数定义的语法，那么我们应该如何实现闭包呢？
不妨还是通过综合机器人来解答这个疑问吧。这一次，让我们稍稍修改一下综合机器人的代码如下：

```java 
public class SpacecraftRobot extends Guider {
    // 外部类的成员变量
    private String name;
    
    public SpacecraftRobot(String name) {
        this.name = name;
    }
    
    public class repairerRobot extends Repairer {
        
        public void doRepairWork() {
            // 内部类持有外部类的引用，访问外部类的成员变量name。
            work(name);
        }
    }
    
    public void doGuidWork() { 
        // 调用guider的work方法
        work(name);
    }
    
    public void doRepairWork() {
        // 返回一个持有外部类变量引用的内部类的对象,然后调用这个对象,实现具体的业务逻辑.
        new repairerRobot().doRepairWork();
    }
}

```
通过对java内部类的合理利用，我们“模拟”出了一个闭包的程序结构。
该程序通过调用外部类对象，从而返回了一个持有外部类对象变量引用的内部类对象。当我们再次调用内部类对象的某个方法时，我们实现了具体的业务逻辑。

##总结
1. 内部类通常用来解决“多重继承”的问题。
2. 当你希望隐藏一个类的实现，减少工程中.java文件数量，或者这个类不想被扩展时，你可以通过匿名内部类来创建一个类的对象。
3. java虽然无法直接在语法层面上支持闭包，但是可以通过内部类来模拟一个闭包的程序结构。


























