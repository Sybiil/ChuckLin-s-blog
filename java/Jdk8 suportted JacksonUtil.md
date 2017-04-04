# Jdk8 suportted JacksonUtil

---
##XML、JSON 处理的烦恼
java中处理json与xml数据时，有各种各样的第三方库供用户选择。其中具有代表性的有 Gson、FastJson、Jackson、Json-lib、xstream、jaxb 等等。
正因为 java 中可供的选择实在太多，若随意的选用库去处理 json 与 xml 的话，不仅会造成代码混乱难以维护，还会让你的 jar 或 war 的体积膨胀，增加 release 成本。

##使用Jackson

为了避免这样的灾难发生，我们要想办法统一程序员们处理 json 与 xml 数据时使用的库。恰恰 jackson 在易用性、效率、社区活跃程度上都是上述第三方库中的佼佼者，能够满足我们的需求。

优秀的 common-Utils 往往都能支持最新的 API，而最新的 jackson 库已经 全面支持了 jdk8 中的新日期 API。
使用支持 jdk8 日期处理 API 的 jackson 需要导入下面这些版本，或以上版本的库。

![屏幕快照 2016-11-19 下午2.36.33.png-21.4kB][1]

##Jackson中的注解支持##
使用 jackson 处理 xml 与 json 的一大好处在于，序列化时使用的注解是通用的。同样的注解能够生效与 json 与 xml 的序列化与反序列化。

 - **@JsonIgnoreProperties({"extra"})**
类注解。在反序列化 json 数据时能够忽略注解参数中对应的 json         keys。而使用@JsonIgnoreProperties(ignoreUnknown=true)         能够使忽略所有未知元素 。
 
 - **@JsonSerialize(using = JacksonUtil.JsonLocalDateSerializer.class)**
属性注解。需要使用自定义序列化与反序列化解析器时，使用该注解修饰类成员变量。

 - **@JsonIgnore**
属性注解。序列化与反序列化时忽略被该注解修饰的成员。

 - **@JsonProperty("firstName")**
属性注解。在序列化与反序列化时使用别名。


##自定义序列化与反序列化解析器##

通过继承 JsonSerializer 与 JsonDeserializer，我们能够实现自己的解析器。通常自己的解析器用来处理将一个 json 字符串映射为 pojo 的 DateTime 类型属性的问题。
###序列化###
```java
    /**
     * 自定义日期序列化处理类
     * LocalDateTime
     * jdk8 support
     */
    public static class JsonLocalDateTimeSerializer extends JsonSerializer<LocalDateTime> {


        @Override
        public void serialize(LocalDateTime localDateTime, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
            String localdateTimeStr = localDateTime.format(DateTimeFormatter.ISO_DATE_TIME);
            jsonGenerator.writeString(localdateTimeStr);
        }
    }
```
###反序列化###
```java
/**
     * 自定义日期序列化处理类
     * LocalDateTime
     * jdk8 support
     */
    public static class JsonLocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {


        @Override
        public LocalDateTime deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
            String str = jsonParser.getText().trim();
            return LocalDateTime.parse(str, DateTimeFormatter.ISO_DATE_TIME);
        }
    }

```

##Class TypeFactory##
当尝试把一个 jsonArray 转换为为 pojoList 时你会遭遇到一些小麻烦。
你必须首先使用工厂方法创建出一个 JavaType 对象，然后根据你创建出的 JavaType 对象来告诉 jackson 你希望的反序列化对象是什么样子。
```java
 /**
     * json数据转PojoList
     *
     * @param jsonStr json数据
     * @param cls     类型
     * @param <T>     推导类型
     * @return pojoList
     */
    public static <T> List<T> jsonArray2PojoList(String jsonStr, Class<T> cls) {
        List<T> pojoList = null;
        try {
            CollectionType listType = objectMapper.getTypeFactory().constructCollectionType(List.class, cls);
            pojoList = objectMapper.readValue(jsonStr, listType);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return pojoList;
    }

```




##Class TypeReference<T>##
jackson 在反序列化对象时需要获取到对象的字节码对象。通常使用 Object.class 就能解决这个问题。不过当你期望的反序列化对象是一个泛型对象时，由于泛型的补偿与擦除问题，你是无法使用 List<>.class 的方式获取到字节码对象的。如何是好呢？
查阅文档，看看是否能获取到帮助。
>This generic abstract class is used for obtaining full generics type information by sub-classing; it must be converted to ResolvedType implementation (implemented by JavaType from "databind" bundle) to be used. Class is based on ideas from http://gafter.blogspot.com/2006/12/super-type-tokens.html, Additional idea (from a suggestion made in comments of the article) is to require bogus implementation of Comparable (any such generic interface would do, as long as it forces a method with generic type to be implemented). to ensure that a Type argument is indeed given.

根据文档描述，通过TypeReference我们就能获取到泛型类的信息。

```java
/**
     * json转listMap
     *
     * @param jsonArray jsonArray字符串
     * @return Listmap对象
     */
    public static List<Map<String, Object>> jsonArray2ListMap(String jsonArray) {
        List<Map<String, Object>> convertedListMap = null;
        try {
            convertedListMap = objectMapper.readValue(jsonArray, new TypeReference<List<Map<String, Object>>>() {
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
        return convertedListMap;
    }
```
##一个完整的JacksonUtil##

```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.type.CollectionType;
import com.fasterxml.jackson.dataformat.xml.XmlMapper;

import java.io.IOException;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.Map;


/**
 * XML,JSON处理工具类
 * 依靠jackson提供pojo2json/xml转换
 *
 * @author lincanhan
 */

final public class JacksonUtil {


    private static ObjectMapper objectMapper = new ObjectMapper();

    private static ObjectMapper xmlMapper = new XmlMapper();


    /**
     * 防止反射调用构造器创建对象
     */
    private JacksonUtil() {
        throw new AssertionError();
    }

    /**
     * 自定义日期序列化处理类
     * LocalDateTime
     * jdk8 support
     */
    public static class JsonLocalDateTimeSerializer extends JsonSerializer<LocalDateTime> {


        @Override
        public void serialize(LocalDateTime localDateTime, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
            String localdateTimeStr = localDateTime.format(DateTimeFormatter.ISO_DATE_TIME);
            jsonGenerator.writeString(localdateTimeStr);
        }
    }

    /**
     * 自定义日期序列化处理类
     * LocalDateTime
     * jdk8 support
     */
    public static class JsonLocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {


        @Override
        public LocalDateTime deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
            String str = jsonParser.getText().trim();
            return LocalDateTime.parse(str, DateTimeFormatter.ISO_DATE_TIME);
        }
    }

    /**
     * 自定义日期反序列化处理类
     * LocalDate
     * jdk8 support
     */
    public static class JsonLocalDateDeserializer extends JsonDeserializer<LocalDate> {

        @Override
        public LocalDate deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException {
            String str = jsonParser.getText().trim();
            return LocalDate.parse(str, DateTimeFormatter.ISO_DATE);
        }
    }

    /**
     * 自定义日期序列化类
     * LocalDate
     * jdk8 support
     */
    public static class JsonLocalDateSerializer extends JsonSerializer<LocalDate> {


        @Override
        public void serialize(LocalDate localDate, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
            String localdateStr = localDate.format(DateTimeFormatter.ISO_DATE);
            jsonGenerator.writeString(localdateStr);
        }
    }

    /**
     * json数据转pojo
     *
     * @param jsonStr json字符串
     * @param cls     映射类型
     * @param <T>     推导类型
     * @return 推导类型json对象
     */
    public static <T> T json2Pojo(String jsonStr, Class<T> cls) {
        T object = null;
        try {
            object = objectMapper.readValue(jsonStr, cls);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return object;
    }


    /**
     * json数据转PojoList
     *
     * @param jsonStr json数据
     * @param cls     类型
     * @param <T>     推导类型
     * @return pojoList
     */
    public static <T> List<T> jsonArray2PojoList(String jsonStr, Class<T> cls) {
        List<T> pojoList = null;
        try {
            CollectionType listType = objectMapper.getTypeFactory().constructCollectionType(List.class, cls);
            pojoList = objectMapper.readValue(jsonStr, listType);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return pojoList;
    }


    /**
     * pojo转json
     *
     * @param obj pojo
     * @return json字符串
     */
    public static String pojo2Json(Object obj) {
        String jsonStr = "";
        try {
            jsonStr = objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        return jsonStr;
    }

    /**
     * json转listMap
     *
     * @param jsonArray jsonArray字符串
     * @return Listmap对象
     */
    public static List<Map<String, Object>> jsonArray2ListMap(String jsonArray) {
        List<Map<String, Object>> convertedListMap = null;
        try {
            convertedListMap = objectMapper.readValue(jsonArray, new TypeReference<List<Map<String, Object>>>() {
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
        return convertedListMap;
    }

    /**
     * json转map
     *
     * @param json json字符串
     * @return map对象
     */
    public static Map<String, Object> json2Map(String json) {
        Map<String, Object> convertedMap = null;
        try {
            convertedMap = objectMapper.readValue(json, new TypeReference<Map<String, Object>>() {
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
        return convertedMap;
    }


    /**
     * listMap转json
     *
     * @param listMap listMap
     * @return
     */
    public static String listMap2JsonArray(List<Map<String, Object>> listMap) {
        String jsonStr = "";
        try {
            jsonStr = objectMapper.writeValueAsString(listMap);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        return jsonStr;
    }


    /**
     * xml转pojo
     *
     * @param xmlStr xml字符串
     * @param cls    映射对象
     * @param <T>    推导类型
     * @return pojo
     */
    public static <T> T xml2Pojo(String xmlStr, Class<T> cls) {
        T pojo = null;
        try {
            pojo = xmlMapper.readValue(xmlStr, cls);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return pojo;
    }

    /**
     * xml转pojoList
     *
     * @param xmlStr xml字符串
     * @param cls    映射对象
     * @param <T>    推导类型
     * @return pojo
     */
    public static <T> List<T> xml2PojoList(String xmlStr, Class<T> cls) {
        CollectionType listType = objectMapper.getTypeFactory().constructCollectionType(List.class, cls);
        List<T> pojoList = null;
        try {
            pojoList = xmlMapper.readValue(xmlStr, listType);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return pojoList;
    }


    /**
     * pojo转xml
     *
     * @param object
     */
    public static String pojo2Xml(Object object) {
        String xml = "";
        try {
            xml = xmlMapper.writeValueAsString(object);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        return xml;
    }


    /**
     * xml转map
     *
     * @param xmlStr xml字符串
     * @return map对象
     */
    public static Map<String, Object> xml2Map(String xmlStr) {
        Map<String, Object> map = null;
        try {
            map = xmlMapper.readValue(xmlStr, new TypeReference<Map<String, Object>>() {
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
        return map;
    }

    /**
     * xml转ListMap
     *
     * @param xmlStr xml字符串
     * @return map对象
     */
    public static List<Map<String, Object>> xml2ListMap(String xmlStr) {
        List<Map<String, Object>> listMap = null;
        try {
            listMap = xmlMapper.readValue(xmlStr, new TypeReference<List<Map<String, Object>>>() {
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
        return listMap;
    }

}

```
###Module###

```java
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import jackson.JacksonUtil;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.Arrays;

/**
 * 用户模型
 */
//@JsonIgnoreProperties(ignoreUnknown=true)忽略所有未知元素
@JsonIgnoreProperties({"extra"})
public class User {

    // 性别
    private Gender gender;
    // 姓名
    private Name name;

    private boolean isVerified;
    // 用户头像
    private byte[] userImage;

    // 用户生日
    @JsonSerialize(using = JacksonUtil.JsonLocalDateSerializer.class)
    @JsonDeserialize(using = JacksonUtil.JsonLocalDateDeserializer.class)
    private LocalDate birthday;

    // 用户登录时间
    @JsonSerialize(using = JacksonUtil.JsonLocalDateTimeSerializer.class)
    @JsonDeserialize(using = JacksonUtil.JsonLocalDateTimeDeserializer.class)
    private LocalDateTime loginTime;


    // 不想输出此属性,使用此注解。
    @JsonIgnore
    private String note;

    // 性别
    public enum Gender {
        MALE, FEMALE
    }

    // 姓名
    public static class Name {

        private String first;

        // 别名
        //@JsonProperty("LAST")
        private String last;


        public String getFirst() {
            return first;
        }

        public void setFirst(String first) {
            this.first = first;
        }

        public String getLast() {
            return last;
        }

        public void setLast(String last) {
            this.last = last;
        }

        @Override
        public String toString() {
            return "Name{" +
                    "first='" + first + '\'' +
                    ", last='" + last + '\'' +
                    '}';
        }
    }


    public LocalDate getBirthday() {
        return birthday;
    }

    public void setBirthday(LocalDate birthday) {
        this.birthday = birthday;
    }

    public LocalDateTime getLoginTime() {
        return loginTime;
    }

    public void setLoginTime(LocalDateTime loginTime) {
        this.loginTime = loginTime;
    }

    public String getNote() {
        return note;
    }

    public void setNote(String note) {
        this.note = note;
    }

    public Gender getGender() {
        return gender;
    }

    public void setGender(Gender gender) {
        this.gender = gender;
    }

    public Name getName() {
        return name;
    }

    public void setName(Name name) {
        this.name = name;
    }

    public boolean isVerified() {
        return isVerified;
    }

    public void setVerified(boolean verified) {
        this.isVerified = verified;
    }

    public byte[] getUserImage() {
        return userImage;
    }

    public void setUserImage(byte[] userImage) {
        this.userImage = userImage;
    }

    @Override
    public String toString() {
        return "User{" +
                "gender=" + gender +
                ", name=" + name +
                ", isVerified=" + isVerified +
                ", userImage=" + Arrays.toString(userImage) +
                '}';
    }


}


```

##最后##
不仅仅是 jacksonUtil，在项目中规范化使用 dateUtil，HttpClientUtil 等等 common Utils，能够极大的增强项目的可维护性，降低bug 发生几率，缩小项目体积，减少 release 成本。
基于这样的考虑，我已经将 common-Utils 创建为了一个[开源项目](https://github.com/mikumikulch/common-utils)，如果你也有安全，高效，注释齐全维护性强的 Util 欢迎与大家分享！




  [1]: http://static.zybuluo.com/mikumikulch/uc75ln83inw2zwknaqzi96d4/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-11-19%20%E4%B8%8B%E5%8D%882.36.33.png
  [2]: http://static.zybuluo.com/mikumikulch/tkp3avlh7ao4b4b2509rycqk/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-11-19%20%E4%B8%8B%E5%8D%883.20.34.png