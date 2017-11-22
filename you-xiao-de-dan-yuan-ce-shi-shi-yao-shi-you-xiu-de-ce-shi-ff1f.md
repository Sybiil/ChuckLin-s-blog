# 有效的单元测试 - 什么是优秀的测试？

##单元测试的意义
合理编写单元测试，可使团队工程师告别牛仔式编程，产出易维护的高质量代码。随着单元测试覆盖率的上升，项目会更加的健壮，团队的信心满满，充满斗志。无论是瀑布团队还是敏捷团队，单元测试及自动化单元测试作为重要的质量保证手段的价值已经被大家接纳与认可。

对于大多数团队来讲，当测试覆盖率提升一定阶段后，收益会迎来瓶颈。

![image_1bvct211t71ccjb1stq18u91nuom.png-479.9kB][1]

而单元测试除了作为质量保证手段以外，更应该作为设计手段。单元测试成为设计手段会使你在测试中学习，以调用者的思维方式来写代码。这种方式的收益会比作为保护手段更高。
利用测试驱动开发，可以让你的设计更加优良、编写出可测试的代码、还能避免在不切实际的假设上过度设计系统。


##优秀测试的共性
1. 良好的测试代码的可读性和可维护性。
2. 测试代码在项目中有结构化的组织方式。
3. 测试具备可靠性和可重复性。
4. 合理的使用测试替身。
5. 准备、执行、断言。当你编写测试代码时，有一个大多数程序员认为合理的、确定的实践、叫做准备、执行、断言。先准备用于测试的对象、触发执行、然后输出断言。

##测试替身（test double）
为了方便的测试代码以及绕开各种环境限制，应该使用替身来代替真实对象。

```java
public class Car {

    // 对外封闭的引擎对象。你无法知道内在信息。
    private Engine engine;

    public Car(Engine engine) {
        this.engine = engine;
    }

    // 需要测试的启动功能
    public void start() {
        engine.start();
    }

    // 需要测试汽车的驾驶功能
    public void drive(Route route) {
        // 根据路线状态获取提供给 car 使用的各种方向 (十分复杂的算法，初始化时需要涉及 gis 算法。最终获取的对对象与当前的时间有关）
        for (Directions directions : route.getDirections()) {
            directions.follow();
        }
    }

}


```

为了验证一段代码的行为符合你的期望，最好的选择是替换其周围的代码，使你获得对环境的完整控制。使用了替身，能让执行速度变得更快，并且可以随意模拟特殊情况。

```java

public class TestRoute extends Route {
        // 伪造的替身实现了快速返回路线以及返回固定路线的方法，使测试具备了可靠与可重复性
        @Override
        public List<Directions> getDirections() {
            System.out.println("快速的返回固定的路线");
            return new ArrayList<Directions>();
        }
    }
    
    public class TestEngine extends Engine {

        private boolean engineFlg = false;

        @Override
        public void start() {
            engineFlg = true;
        }

        @Override
        public void stop() {

            engineFlg = false;
        }

        public boolean isEngineFlg() {
            return engineFlg;
        }

        public void setEngineFlg(boolean engineFlg) {
            this.engineFlg = engineFlg;
        }
    }


```

使用替身替换协作者，隔离被测代码意味着将需要测试的代码与其他代码隔离开来。

```java
@Test
    public void testCarStart() {
        TestEngine engine = new TestEngine();
        new Car(engine).start();
        assertTrue(engine.isEngineFlg());
    }

    @Test
    public void testCarDrive() {
        TestRoute testRoute = new TestRoute();
        TestEngine engine = new TestEngine();
        new Car(engine).drive(testRoute);
    }

```


如何区分协作者被测代码的关系呢？
上面的代码中，与对象 Car 一起工作的对象为 Engine 和 Route、以及 directions。Car 直接的使用了 Engine 和 Route ，间接的使用了 Directions。所以 Car 的协作者只有 Engine 和 Route。


###测试替身的类型
替身拥有多种类型。不同的测试场景选择合理的测试替身是优秀测试的必要条件之一。测试替身的类型也决定了测试类的命名方法，尤其特别注意。

####桩（stub）
用最简单的可能实现来代替真实实现。桩总是短小的。最好的例子就是一个对象的所有方法都只有一行，并且返回一个默认值。
```
public class LoggerStub implements Logger {
    
    @Override
    public void log() {
        // Nope
    }
}

```

####伪造对象 （fake）
有时，我们至少需要填充一些行为，而有时候你需要测试替身根据收到的消息种类来表现出不同的行为。这些情况下，你会借助伪造对象。Fake 就像是真实事物的简单版本，优化滴伪造真实事物的行为，但是没有副作用或者使用真实事物的其他后果。

```java
public class TestRoute extends Route {
        // 伪造的替身实现了快速返回路线以及返回固定路线的方法，使测试具备了可靠与可重复性
        @Override
        public List<Directions> getDirections() {
            System.out.println("快速的返回固定的路线");
            return new ArrayList<Directions>();
        }
    }

```
伪造对象与测试桩十分常用，你可以在测试时用它们替换掉缓慢的真实事物，以及鞭长莫及的依赖。

####测试间谍 (spy)
测试间谍用于记录你和某个不对你开放的对象之间的交互。当对象与协作者之间交互时，无法获取协作者交互结果的情况下，会使用测试间谍。

```java
// 测试间谍
public class TestEngine extends Engine {
        // 对外暴露信息
        private boolean engineFlg = false;
        
        // 比真实对象的 start 方法更具有交互性的方法。
        @Override
        public void start() {
            engineFlg = true;
        }

        @Override
        public void stop() {

            engineFlg = false;
        }
        // 对外提供用于获取交互结果的接口，用于断言。
        public boolean isEngineFlg() {
            return engineFlg;
        }

        public void setEngineFlg(boolean engineFlg) {
            this.engineFlg = engineFlg;
        }
    }

```
####模拟对象 mock 
模拟对象是在一个特定情境下可配置行为的对象。模拟对象时一种更加高级与特殊的测试间谍。
下面是一段关于 Mock 的伪代码。

```java
public void mockTest(){
    final Internet internet = context.mock(Internet.class);
    context.checking(new Expectations() {{
        one(internet).get(with(containsString("langpair=en")));
        will(returnValue("ok"))
    }};
    Translator t = new Translator(internet);
    String translation = t.translate("flower",ENGLISH);
    assertEquals("ok",translation)
    
}

```
通过使用模拟对象的预测返回值行为，我们可以更加精确的对交互行为进行验证。

##如何选择合适的替身类型

- 如果你测试的重点是交互，即两个对象之前的调用，你可能需要一个模拟对象 mock
- 如果你决定使用 Mock ，但测试代码最终看起来不像你想象的那么漂亮了，则考虑使用 Spy。
- 如果你只关心协作对象向被测对象输送的响应，用 stub 解决问题。
- 如果你运行的是一个复杂场景，其中它所依赖的服务无法供测试使用，而你所有的交互打桩快速尝试却戛然而止，或者产出了难以维护的糟糕代码，那就考虑使用 Fake。









  [1]: http://static.zybuluo.com/mikumikulch/njz109ixdoxpnosgi7jn11nh/image_1bvct211t71ccjb1stq18u91nuom.png