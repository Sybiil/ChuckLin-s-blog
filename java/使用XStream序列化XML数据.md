# 使用XStream序列化XML数据

标签（空格分隔）： java

---

##Xstream是什么？它有什么用？
在项目开发中，大家经常会遇到一些处理XML格式数据的需求。
比如公司系统与机票，酒店等第三方接口使用XML格式的数据进行对接。
或者大型ERP系统，几个不同业务模块之间通过webService等手段，使用XML格式数据进行交互等。
面临庞大复杂的xml数据，若单纯的依靠dom4j来解析的话会非常辛苦。笔者曾经使用dom4j处理过一个中等规模的酒店第三方接口，整个接口完成后，代码行数几乎达到了2000行，并且代码中包含大量的"hardCode"使接口面临巨大的安全/可维护性隐患。

对于程序员来讲，重复性的工作不仅讨厌，而且效率特别低下。试想，我们利用反射技术配合dom4j，是不是能够实现自动将xml节点的value映射到JavaBean中去，从而节省大量人工解析的工作呢？
答案是正确的。并且在java的世界中中已经有了这样的第三方库通过上述手段来帮我们解决自动解析xml的问题。
目前较主流的库分别有两个，分别是jdk自带的**jaxb**与来自于thoughtWorks的开源库->**XStream**。
本篇文章只对XStream进行讲解。如果你想要了解jaxb的话请查阅其他相关资料。

##XStream怎么用
###环境准备
首先，你需要将XStream库导入你的项目。
如果你使用maven，请在pom.xml中配置XStream的依赖：
```xml
<dependency>
    <groupId>com.thoughtworks.xstream</groupId>
    <artifactId>xstream</artifactId>
    <version>1.4.9</version>
</dependency>
```

如果你使用传统方式构建你的java项目，可以通过访问[托管于github的源代码](https://github.com/x-stream/xstream)，自己编译成jar包使用。
但需要注意的是，XStream库本身就是在maven的基础上开发而成的，所以建议还是安装一个maven环境为好。


###代码示例
现在，准备工作已经完成，让我们开始正式的coding工作吧。
假设我们需要解析的xml文件如下：
```xml
<HotelOrder>
    <Group>totalextrip</Group>
    <OrderDate>2016-09-09</OrderDate>
    <OrderDates>2016-09-09</OrderDates>
    <Hotel bedType="3">
        <name>丽思卡尔顿大酒店</name>
    </Hotel>
    <GuestList>
        <Guest>
            <name>lin</name>
            <age>23</age>
            <Child>
                <Name>lucy</Name>
            </Child>
        </Guest>
        <Guest>
            <name>li</name>
            <age>21</age>
        </Guest>
    </GuestList>
</HotelOrder>
```
上面的xml数据看似简单，实际上已经涵盖了大多数情况下所面临的情况了。

接下来根据xml文件我们需要建立与其互相映射的JavaBean。
```java
/**
 * 酒店Bean
 */
@XStreamAlias("Hotel")
public class Hotel {

    @XStreamAlias("Name")
    private String name;

    @XStreamAsAttribute
    private String bedType;

    public String getBedType() {
        return bedType;
    }

    public void setBedType(String bedType) {
        this.bedType = bedType;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

```java
/**
 * 入住客户Bean
 */
@XStreamAlias("Guest")
public class Guest {

    @XStreamAlias("Name")
    private String name;
    @XStreamAlias("Age")
    private String age;
    @XStreamAlias("Child")
    private Child child;

    public Child getChild() {
        return child;
    }

    public void setChild(Child child) {
        this.child = child;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }

    public static class Child {
        @XStreamAlias("Name")
        private String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }
}

```

```java
/**
 * 酒店订单Bean
 */
@XStreamAlias("HotelOrder")
public class HotelOrder {

    @XStreamAlias("Group")
    private String group;
    @XStreamAlias("OrderDate")
    @XStreamConverter(value=XStreamDateConverter.class)
    private Date orderDate;
    @XStreamAlias("Hotel")
    private Hotel hotel;
    @XStreamAlias("GuestList")
    private List<Guest> guestList;

    public String getGroup() {
        return group;
    }

    public void setGroup(String group) {
        this.group = group;
    }

    public List<Guest> getGuestList() {
        return guestList;
    }

    public void setGuestList(List<Guest> guestList) {
        this.guestList = guestList;
    }

    public Date getOrderDate() {
        return orderDate;
    }

    public void setOrderDate(Date orderDate) {
        this.orderDate = orderDate;
    }

    public Hotel getHotel() {
        return hotel;
    }

    public void setHotel(Hotel hotel) {
        this.hotel = hotel;
    }
}

```
建立好的JavaBean中，含有若干注解。这些注解都是是较常用的注解，需要熟练掌握。
@XStreamAlias("aliasName")
:   别名注解。可以在类和类的成员变量上使用。使用了该注解的类，或者成员变量，在被解析为xml时，会忽略到本身的名字，而采用别名生成标签。相应的序列化xml时，也会将相应的标签名的value映射到变量中。

@XStreamAsAttribute
:   属性注解。可以在类的成员变量上使用。使用了该注解的成员变量，在映射为xml时会被映射为该类所属标签的属性。相应的序列化xml时，属性的value也会被映射到变量中。

@XStreamConverter
:   自定义转换器注解。XStream中提供了各种各样的转换器。其中比较常用的转换器是单值转换器AbstractSingleValueConverter。该转换器可以在成员变量上使用。使用了该注解的成员变量，在被映射为xml时，会先通过转换器对变量值进行转换，最后再映射为xml。相应的序列化xml时，相应的标签的value，也会通过转换器转换后，赋值给变量。

```java
/**
 * 自定义XStream单值类
 * Date -> String
 * String -> Date
 */
public class XStreamDateConverter extends AbstractSingleValueConverter {


    private final DateFormat DATEFORMAT = new SimpleDateFormat("yyyy-MM-dd");

    /**
     * 告诉xstream,该转换器用来转换Date类型的对象.
     *
     * @param type
     * @return
     */
    @Override
    public boolean canConvert(Class type) {
        return type.equals(Date.class);
    }

    /**
     * object->xml用
     * 将变量值转换为String,赋值为标签值.
     *
     * @param str
     * @return
     */
    @Override
    public Object fromString(String str) {
        // 这里将字符串转换成日期
        try {

            return DATEFORMAT.parseObject(str);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        throw new ConversionException("Cannot parse date " + str);
    }

    /**
     * xml->object用
     * 将xml的标签值,包装为对象,映射到成员变量中
     *
     * @param obj
     * @return
     */
    @Override
    public String toString(Object obj) {
        // 将日期转换成字符串
        return DATEFORMAT.format((Date) obj);
    }
}

```
最后，通过简单的代码来完成xml序列化的工作。
```java
/**
 * Xstream测试
 * 利用xstream对xml序列化进行测试
 * pojoToXml - 对象toXml
 * XmlToPojo - Xmlto对象
 */
public class XStreamTester {

    public static void main(String[] args) {
        // 酒店订单对象
        HotelOrder hotelOrder = new HotelOrder();
        hotelOrder.setGroup("totalextrip");
        hotelOrder.setOrderDate(new Date());

        // 酒店对象
        Hotel hotel = new Hotel();
        hotel.setName("丽思卡尔顿大酒店");
        hotel.setBedType("3");

        // 入住客户
        Guest user = new Guest();
        // 没有值,也想生成节点的话,请将值设置为"";
        user.setName("");
        user.setAge("15");
        Guest user2 = new Guest();
        user2.setName("lily");
        user2.setAge("18");
        List<Guest> guestList = new LinkedList<>();
        guestList.add(user);
        guestList.add(user2);

        // 儿童姓名
        Guest.Child child = new Guest.Child();
        child.setName("lucy");
        user.setChild(child);

        // 赋值
        hotelOrder.setGuestList(guestList);
        hotelOrder.setHotel(hotel);

        // 调用函数
        pojoToXml(hotelOrder);
        xmlToPojo();
    }


    /**
     * 将Pojo转换为xml
     *
     * @param hotelOrder
     */
    public static void pojoToXml(HotelOrder hotelOrder) {
        XStream xstream = new XStream(new XppDriver(new NoNameCoder()));
        xstream.aliasSystemAttribute(null, "class"); // 去掉class 属性
        xstream.autodetectAnnotations(true);
        String xmlStr = xstream.toXML(hotelOrder);
        System.out.println(xmlStr);
    }
    
    
    /**
     * 将xml文档转换为pojo
     */
    public static void xmlToPojo() {
        XStream xstream = new XStream(new DomDriver("utf-8"));
        File xmlFile = new File("file.xml");
        //xstream.autodetectAnnotations(true);
        // 若使用注解,则一定要调用此方法.
        xstream.processAnnotations(HotelOrder.class);
        // 忽略掉xml中的未知元素.
        xstream.ignoreUnknownElements();
        HotelOrder obj = (HotelOrder) xstream.fromXML(xmlFile);
        System.out.println(obj.getGuestList().get(0).getChild().getName());
        System.out.println(obj.getOrderDate());
    }
}

```

###注意事项
1. *xstream.ignoreUnknownElements()*用于忽略掉xml中的未知元素，换句话说bean中若不存在和xml文件中对应的元素的话，该节点则会被忽略。但是，低版本的库也许不包含此api。
2. *xstream.processAnnotations(HotelOrder.class)*使用注解序列化xml文件时必须调用此方法。否则映射失败。
3. *xstream.autodetectAnnotations(true)*使用注解映射bean到xml文件时，必须调用此方法开启注解扫描功能。


##总结
XStream的作用在于xml格式的数据与Bean的相互映射。使用XStream能够帮助我们处理异构系统之间数据交互，带来的xml解析问题。并且XStream还能通过自定义解析器，解决某些项目中比较特殊的需求。
综上所述，笔者实在找不出在解决上述问题时不使用XStream代替dom4j的理由。


























