# Jdk8 supportted DateUtil


---


jdk8带来了一些列的改变。新的日期/时间api是最受欢迎的改变之一。
为了方便理解新日期/时间的特性，有必要让我们回顾一下“时间相关的基本概念”。


##格林威/尼治标准时间的来历
时间是物质的运动、变化的持续性、顺序性的表现。
最初，世界各地都按照自己的时间为标准在进行各种人类活动。在1884年召开的华盛顿国际经度会议上，为了定义大家都统一遵守的时间标准，便规定了国际标准时间的概念。

它要求全球范围内都以零经度线上的时间作为国际上统一采用的标准时间。 因为零经度线通过英国格林尼治天文台，所以国际标准时间也称为格林尼治时间，又称世界时/标准时。

随着时代进步，现在格林尼治时间已经不再被作为标准时间使用。新的标准时间，是由原子钟报时的协调世界时，也称为UTC。

##时区与时差
为了克服时间上的混乱，1884年的华盛顿国际经度会议上，又将全球划分为了24个时区。分别为：
被称作“零时区”的英国时区，以及以零时区为基准的东1-12区，西1-12区。
每个相邻时区之间的时差为1小时。由于中国被划分到的时区为东-8区，所以中国与英国的时差为8小时。
故，中国的时间应该为 UTC 时间再加上8小时。( UTC+8 )


##ISO 8601
既然定义了时间概念上的标准，自然也就需要定义时间的表示标准。
国际标准化组织定义了《数据存储和交换形式·信息交换·日期和时间的表示方法》，简称 ISO8601，以规范时间的表示标准。
在标准中，定义了日期，时间，日期+时间等表示方法。

其中，最常用的日期和时间的组合表示法为：
在表示北京时间2004年5月3日下午5点30分8秒，可以写成
```python
2004-05-03T17:30:08+08:00
```
或
```python
20040503T173008+08
```
中间的T用以区分日期与时间。

##计算机中的时间[^1]
在计算机中，时间是通过数字来表示的。
我们把1970年1月1日 00:00:00 UTC+00:00时区（零时区）的时刻称为 epoch time，记为0（1970年以前的时间 timestamp 为负数），当前时间就是相对于 epoch time 的秒数，称为 timestamp。
你可以认为：
```python
timestamp = 0 = 1970-1-1 00:00:00 UTC+0:00
```
对应的北京时间则为：
```python
timestamp = 0 = 1970-1-1 08:00:00 UTC+8:00
```
可见 timestamp 的值与时区毫无关系，因为 timestamp 一旦确定，其 UTC 时间就确定了，转换到任意时区的时间也是完全确定的，这就是为什么计算机存储的当前时间是以 timestamp 表示的，因为全球各地的计算机在任意时刻的 timestamp 都是完全相同的。

##JDK8所带来的时间变革[^2]
如果你对 Java 混乱的日期操作烦恼不已，那么恭喜你！
新版 API 拥有以下这些激动人心的特性，让你有充分的理由彻底告别 Calendar，Date，TimeZone 等对象。
> * **不变性**
新的日期/时间API中，所有的类都是不可变的，这对多线程环境有好处。
关注点分离：新的API将人可读的日期时间和机器时间（unix timestamp）明确分离，它为日期（Date）、时间（Time）、日期时间（DateTime）、时间戳（unix timestamp）以及时区定义了不同的类
。
* **清晰**
在所有的类中，方法都被明确定义用以完成相同的行为。举个例子，要拿到当前实例我们可以使用now()方法，在所有的类中都定义了format()和parse()方法，而不是像以前那样专门有一个独立的类。为了更好的处理问题，所有的类都使用了工厂模式和策略模式，一旦你使用了其中某个类的方法，与其他类协同工作并不困难。
* **实用操作**
所有新的日期/时间API类都实现了一系列方法用以完成通用的任务，如：加、减、格式化、解析、从日期/时间中提取单独部分，等等。
可扩展性：新的日期/时间API是工作在 ISO-8601 日历系统上的，但我们也可以将其应用在非IOS的日历上。

##DateUtil in JDK8


**JDK8日期/时间API常用类简介：**

> 
* LocalDateTime：常用日期时间类。
* Instant：时间戳。
* Period：顾名思义，一段时间！你知道该怎么用了吧？
* DayOfWeek：与周相关的信息，选他就没错了！注意这是一个枚举。还记得枚举中每个类型都拥有自己的value这个概念吧？
* TemporalAdjusters：时间调节器。通过调解器能够获取到以指定日期为基础的某个特定日期。，比如说，可以找到某月的第一天或最后一天。
* DateTimeFomatter：格式化类，解析日期对象的类。现在他终于是线程安全的了。

```java
/**
 * 日期工具
 *
 * @author lincanhan
 */
public class DateUtil {

    public static final ZoneId chinaZone = ZoneId.systemDefault();


    /**
     * 日期时间对象转换为日期对象
     *
     * @param localDateTime 日期时间对象
     * @return 日期对象
     */
    public static LocalDate dateTime2Date(LocalDateTime localDateTime) {
        return localDateTime.toLocalDate();

    }

    /**
     * 日期对象转换为日期对象
     *
     * @param localDate 日期对象
     * @return 日期时间对象
     */
    public static LocalDateTime date2DateTIme(LocalDate localDate) {
        return LocalDateTime.of(localDate, LocalTime.NOON);
    }


    /**
     * 字符串转换为日期
     *
     * @param strDate 字符串日期
     * @return 日期对象 yyyy-mm-dd
     */
    public static LocalDate str2Date(String strDate) {
        return LocalDate.parse(strDate, DateTimeFormatter.ISO_DATE);
    }


    /**
     * 日期对象转换为字符串
     *
     * @param localDate 日期对象
     * @return 日期字符串 yyyy-mm-dd
     */
    public static String date2Str(LocalDate localDate) {
        return localDate.format(DateTimeFormatter.ISO_DATE);
    }


    /**
     * 日期时间对象转换为字符串
     *
     * @param localDateTime     日期时间对象
     * @param dateTimeFormatter 格式化字符串
     * @return 日期字符串
     */
    public static String dateTime2Str(LocalDateTime localDateTime, String dateTimeFormatter) {
        return localDateTime.format(DateTimeFormatter.ofPattern(dateTimeFormatter));
    }

    /**
     * 日期时间转字符串函数
     * 返回ISO标准的日期字符串
     *
     * @param localDateTime 日期时间对象
     * @return 日期字符串
     */
    public static String dateTime2Str(LocalDateTime localDateTime) {
        return localDateTime.format(DateTimeFormatter.ISO_DATE_TIME);
    }

    /**
     * 计算两个日期之间相差的天数
     *
     * @param date1 起始日期
     * @param date2 结束日期
     * @return
     */
    public static int daysBetween(LocalDate date1, LocalDate date2) {
        Period period = Period.between(date1, date2);
        return period.getDays();
    }

    /**
     * 计算两个日期之间相差的月数
     *
     * @param date1 起始日期
     * @param date2 结束日期
     * @return
     */
    public static int monthsBetween(LocalDate date1, LocalDate date2) {
        Period period = Period.between(date1, date2);
        return period.getMonths();
    }

    /**
     * 计算两个日期之间相差的年数
     *
     * @param date1 起始日期
     * @param date2 结束日期
     * @return
     */
    public static int yearsBetween(LocalDate date1, LocalDate date2) {
        Period period = Period.between(date1, date2);
        return period.getYears();
    }

    /**
     * 计算两个日期之间相差的天数
     *
     * @param date1 起始日期
     * @param date2 结束日期
     * @return
     */
    public static int daysBetween(Date date1, Date date2) {
        Instant instantDate1 = date1.toInstant();
        Instant instantDate2 = date2.toInstant();
        LocalDate localDate1 = instantDate1.atZone(chinaZone).toLocalDate();
        LocalDate localDate2 = instantDate2.atZone(chinaZone).toLocalDate();
        instantDate1.atZone(chinaZone);
        Period period = Period.between(localDate1, localDate2);
        return period.getDays();
    }

    /**
     * 计算两个日期之间相差的月数
     *
     * @param date1 起始日期
     * @param date2 结束日期
     * @return
     */
    public static int monthsBetween(Date date1, Date date2) {
        Instant instantDate1 = date1.toInstant();
        Instant instantDate2 = date2.toInstant();
        LocalDate localDate1 = instantDate1.atZone(chinaZone).toLocalDate();
        LocalDate localDate2 = instantDate2.atZone(chinaZone).toLocalDate();
        instantDate1.atZone(chinaZone);
        Period period = Period.between(localDate1, localDate2);
        return period.getMonths();
    }

    /**
     * 计算两个日期之间相差的年数
     *
     * @param date1 起始日期
     * @param date2 结束日期
     * @return
     */
    public static int yearsBetween(Date date1, Date date2) {
        Instant instantDate1 = date1.toInstant();
        Instant instantDate2 = date2.toInstant();
        LocalDate localDate1 = instantDate1.atZone(chinaZone).toLocalDate();
        LocalDate localDate2 = instantDate2.atZone(chinaZone).toLocalDate();
        instantDate1.atZone(chinaZone);
        Period period = Period.between(localDate1, localDate2);
        return period.getYears();
    }

    /**
     * 获取指定日期对象当前月的起始日
     *
     * @param localDate 指定日期
     * @return
     */
    public static int getFirstDayInMonth(LocalDate localDate) {
        LocalDate result = localDate.with(TemporalAdjusters.firstDayOfMonth());
        return result.getDayOfMonth();

    }

    /**
     * 获取指定日期对象的当前月的结束日
     *
     * @param localDate 指定日期
     * @return
     */
    public static int getLastDayInMonth(LocalDate localDate) {
        LocalDate result = localDate.with(TemporalAdjusters.lastDayOfMonth());
        return result.getDayOfMonth();
    }


    /**
     * 获取指定日期对象本月的某周某天的日期
     *
     * @param localDate  日期对象
     * @param weekNumber 周
     * @param dayNumber  日
     * @return
     */
    public static LocalDate getLocalDateBydayAndWeek(LocalDate localDate, int weekNumber, int dayNumber) {
        return localDate.with(TemporalAdjusters.dayOfWeekInMonth(weekNumber, DayOfWeek.of(dayNumber)));
    }


}
```
JDK8的日期API优雅简洁，我认为大多数情况即使不使用DateUtil，也不会让业务代码混乱不堪。


##总结
1. JDK8日期/时间API设计优雅，使用方便，命名清晰。
2. 除了优雅的设计以外，新API保持了完整的向下兼容能力。
3. 如果你不是我这样的强迫症患者，在新的项目中完全不使用 DateUtil 也没有问题。无论什么需求，你都能通过几句代码轻松应对。
4. 你还有什么理由不在JDK8的项目中使用新API？
5. 除了这个 DateUtil 以外，笔者提供了一个更详细的新日期API使用示例[JDK8DateAPI.java](https://github.com/mikumikulch/model-application/tree/dev/src/date)，供参考。


---
Copyright 2017/08/15 by Chuck Lin

若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！

![alipay.jpg-17.7kB][99]![wechat.jpg-16.7kB][98]


[99]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[98]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg

[^1]: 本小节参照了[廖雪峰的官方网站：python教程-datatime](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001431937554888869fb52b812243dda6103214cd61d0c2000)中的相关段落，有兴趣的读者请前往廖老师的官网了解详情。

[^2]: 本小节参照了[importNew-Java8 日期/时间（Date Time）API指南](http://www.importnew.com/14140.html)中的相关段落，有兴趣的读者请前往importNew了解详情。
原文链接： journaldev 翻译： ImportNew.com - Justin Wu
译文链接： http://www.importnew.com/14140.html
