# 有效的单元测试 - 单元测试的设计心理学

---

##审美之前先审丑
单元测试的设计不是主观臆断凭直觉的产物。好的设计和工业设计一样，存在着公认的行为准则。
在学习优秀的单元测试设计之前，了解一下糟糕的设计是快速掌握设计诀窍的“捷径”。

##单元测试的坏味道

###不可读的单元测试
单元测试的可读性应该体现在程序员阅读测试代码之后，就该理解代码应当做什么。程序员运行那些测试时，就该了解代码实际上在做些什么。
避免基本断言。基本断言的含义隐藏在了复杂的代码背后。使用更加自然化的描述语言来表达你的断言。
```java
      @Test
    public void outputHasLineNumbers() {
        String content = "1st match on 1";
        assertTrue(content.indexOf("1") != -1);
    }

    /**
     * 利用好 Hamcrest 匹配器，使用更加自然化的描述语言来编写测试
     */
    @Test
    public void outputHasLineNumbers2() {
        String content = "1st match on 1";
        assertThat(content, containsString("1"));
    }

```

###拥有条件逻辑的单元测试
在测试中存在条件逻辑，一般都不是什么好事儿。你很难弄清楚究竟在测试什么。

```java
 @Test
    public void conditionTestDemo() {
        Map<String, String> strMap = new HashMap<>();
        strMap.put("a", "1");
        strMap.put("b", "2");
        
        for (Map.Entry<String, String> strEntry : strMap.entrySet()) {
            if ("a".equals(strEntry.getKey())) {
                assertEquals("1", strEntry.getValue());
            }
            if ("b".equals(strEntry.getKey())) {
                assertEquals("1", strEntry.getValue());
            }
        }
    }

```


条件逻辑会让本来需要被运行的断言操作因为轻微的改动，导致断言无法执行，让人误以为断言执行成功！
想想办法，对上面的测试加以改动使他更加的清晰和准确。

```java
 @Test
    public void conditionTestDemo2() {
        Map<String, String> strMap = new HashMap<>();
        strMap.put("a", "1");
        strMap.put("b", "2");
        assertContains(strMap, "c", "3");
    }


    /**
     * 判断 stringMap 是否包含期望的 key 与 value
     *
     * @param stringMap strmap
     * @param key       期望的 key
     * @param value     期望的 value
     */
    private void assertContains(Map<String, String> stringMap, String key, String value) {
        for (Map.Entry<String, String> entry : stringMap.entrySet()) {
            if (key.equals(entry.getKey())) {
                assertEquals(value, entry.getValue());
                return;
            }
        }
        fail(String.format("stringMap 未包含期望的 key -> %s ", key));
    }


```

若所有的条件逻辑都没有执行时，记得一定要有一行 fail 使断言失败。

###脆弱的单元测试
脆弱的测试往往都涉及了线程和静态条件或者依赖时间和平台的代码。或者访问一切 IO 的代码。避免代码涉及到日期等一切可变因素的影响。
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

用于获取地图最佳距离算法的 Route 对象依赖与时间以及当前某条线路的交通状况。它很可能会返回你预料之外的结果。

###运行缓慢的烦人测试

####使用 sleep
在测试代码在并发状态下的执行状况时，由于对测试结果的期待，或许会需要某些 Reduce 线程在 Map 线程之后才运行。最简单的方式是使用 Thread.sleep();
```java
@Test
    public void threadTestDemo() throws InterruptedException {
        Runnable mapRunable = new Runnable() {
            @Override
            public void run() {
                System.out.println("执行计算方法");
            }
        };


        Runnable reduceRunable = new Runnable() {
            @Override
            public void run() {
                System.out.println("收集计算结果");
            }
        };

        Thread mapThread = new Thread(mapRunable);
        mapThread.start();

        // 等待 map 函数结束
        Thread.sleep(3000);

        Thread reduceThread = new Thread(reduceRunable);
        reduceThread.start();

        // 等待 reduce 函数结束
        Thread.sleep(3000);

    }


```
随着单元测试代码的日积月累，大量的 Thread.sleep 使测试速度越来越慢，单个本地测试也许还可以忍受，但全面自动化测试时乌龟一般的运行速度能让你抓狂。我猜你一定不想看到那一天。所以，现在开始想想办法，免去无意义的多余等待，尝试使用 CountDownLatch 来测试并发逻辑。
```java

    @Test
    public void threadCountDownLautchDemo() throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(1);
        Runnable mapRunable = new Runnable() {
            @Override
            public void run() {
                System.out.println("执行计算方法");
                latch.countDown();
            }
        };
        Runnable reduceRunable = new Runnable() {
            @Override
            public void run() {
                System.out.println("收集计算结果");
                try {
                    latch.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        Thread mapThread = new Thread(mapRunable);
        mapThread.start();
        Thread reduceThread = new Thread(reduceRunable);
        reduceThread.start();
    }

```

####在测试中使用 IO 以及打印日志
IO 与日志也是 cpu 资源消耗大户。对于 IO 测试请使用替身来加快执行速度。而日志系统。。。。在测试运行环境中请关闭它吧！

###永不失败的快乐测试
促成快乐测试的原因往往有两点：

####有关于抛出异常的测试。


```java
@Test
    public void exceptionTest() {
        try {
            System.out.println("准备开始执行非法的业务处理逻辑");
            new Environment().include("不存在的环境变量");
        } catch (Exception e) {
            assertThat(e.getMessage(), containsString("不存在的环境变量"));
        }

    }

```
我们期望在传入不存在的环境变量时运行报错，出现异常并且断言异常。但是若没有出现异常时，此测试依然能顺利结束，让我们误以为测试通过。
增加 fail() 方法调用，使上面的测试正确的达成我们的期望。

```java
@Test
    public void exceptionFailTest() {
        try {
            System.out.println("准备开始执行非法的业务处理逻辑");
            new Environment().include("不存在的环境变量");
            fail("运行失败，未抛出期望的异常");
        } catch (Exception e) {
            assertThat(e.getMessage(), containsString("不存在的环境变量"));
        }
    }
```

####没有断言的测试

```java
 /**
     * 测试获取还款计划的方法
     *
     * @throws ServiceException exp
     */
    @Test
    public void testGetRepayPlan() throws ServiceException {
        String loanNo = "201710240000021213";
        // 查询提款信息
        DrawMoneyInfoVO drawMoneyInfoVO = commCustBaseInfoServiceImpl.queryDrawMoneyInfoByLoanNo(loanNo, null);
        if (drawMoneyInfoVO == null) {
            logger.warn("由于测试获取还款计划的方法 testGetRepayPlan 未查询到提款合同，测试未运行");
        }
        CommCustBaseInfoVO commCustBaseInfoVO = new CommCustBaseInfoVO();
        commCustBaseInfoVO.setLoanNo(drawMoneyInfoVO.getLoanNo());
        commCustBaseInfoVO.setMainLoanNo(drawMoneyInfoVO.getMainLoanNo());
        commCustBaseInfoVO.setUserName(drawMoneyInfoVO.getName());
        commCustBaseInfoVO.setIdCard(drawMoneyInfoVO.getIdCard());
        commCustBaseInfoVO.setLoanPurpose(drawMoneyInfoVO.getLoanPurposeCN());
        commCustBaseInfoVO.setAmount(drawMoneyInfoVO.getAmount());
        commCustBaseInfoVO.setRepaymentMethod(drawMoneyInfoVO.getRepaymentMethod());
        commCustBaseInfoVO.setProductNo(drawMoneyInfoVO.getProductCode());
        commCustBaseInfoVO.setProductVersion(drawMoneyInfoVO.getProductVersion());
        commCustBaseInfoVO.setRepaymentIssue(drawMoneyInfoVO.getRepaymentIssue());
        commCustBaseInfoVO.setSignTime(new Date());
        commCustBaseInfoVO.setLoanMode(drawMoneyInfoVO.getLoanMode());
        commCustBaseInfoVO.setPhone(drawMoneyInfoVO.getPhone());
        commCustBaseInfoVO.setChannel(drawMoneyInfoVO.getChannel());
        commCustBaseInfoVO.setPartnerOrderId(drawMoneyInfoVO.getPartnerOrderId());
        List<RepayPlanDetail> list = simulatedCalculateService.getRepayPlan(commCustBaseInfoVO);
    }

```

我们将修复这个快乐测试的任务留给各位亲爱的读者朋友们。给自己一些时间，想想应该从哪些点入手进行修改。

1. 删除多余的查询。
2. 删除条件逻辑。
3. 增加断言。
4. 断言应该断言更加具体的内容。
5. 搞清楚自己测试的范围是什么。将需要测试的数据和协作对象隔离开来，使测试保证单一职责原则。
6. 最大的需要解决的问题，使还款计划的设计更加的可测。使用伪造对象替代 dao 层，保证利率信息在各环境都能获取到值并且计算出期望的结果。

##优秀单元测试设计指南

学习完了上述测试的坏味道再加上适当的实践，就可以编写出合格的单元测试的。
但是，要设计出优秀的单元测试只是避开坏味道是不够的。下面这些设计指南能够帮助你更上一层楼，写出优秀的测试代码！

###复杂逻辑的方法不应该成为私有方法。
private 的私有方法无法被外部访问，这对单元测试带来了困难。并不是说所有方法都不应该使用 private 方法。而是 private 方法应该尽可能短小，简单，其存在的目的是为了使 public 方法更易读。

###避免 final 方法。

无法被覆盖以及定义为 static 的方法无法被打桩。不要去纠结 final 关键字的性能，但是当你出于维护目的编写 final 方法的时候，请考虑这个方法将来是否可能需要被打桩。

###警惕使用 new 关键字。
因为在 new 关键字中已经指定了具体实现。今后会很难替换了。调用对象的方法时尽可能的使用注入的对象，利用组合解决问题。

###隔离与测试无关的协作者。

每个方法都完成一件事情，每个测试都测试一件事情。保持测试方法的单一职责。

###不要使用测试基类。使用 BeforeClass 代替 Before。

before 注解在每个测试类的每个测试方法运行之前，都会运行一次。所以当你的初始化工作确定只需要一次时，请使用 BeforeClass 代替 Before。
另外请不要使用测试基类。junit 在运行测试时会扫描 Before 注解遍历到最高父级。所以你的每个测试子类都可能运行 Before 注解很多次。


##结语
要想写出优雅的单元测试除了理论知识以外，大量实践是必不可少的。写测试的思路不同于单纯的业务逻辑开发。如果你觉得你的代码写的还行，那么现在就开始尝试 TDD 编程吧，一段时间过后你会和我一样，惊讶于之前的代码中竟然充满了糟糕透顶的设计，并庆幸自己使用 TDD 的时间还不算晚。


---

《有效的单元测试》 Lasse Koskela








