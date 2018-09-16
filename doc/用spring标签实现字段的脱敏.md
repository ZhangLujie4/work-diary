## 问题描述

在springboot中实现数据库脱敏，可以选择在对象序列化之前进行处理脱敏，接下来就是具体的操作了。（参考了网上的资料进行了整理）

## 解决方案

### 第一步

构建枚举类，用来识别每个脱敏字段

```java
/**
 * @author tori
 * 2018/7/16 上午11:19
 */
public enum SensitiveType {
    /** 手机号 */
    MOBILE,

    /** 电子邮箱 */
    EMAIL,

    /** openId */
    OPEN_ID
}
```

### 第二步

创建处理脱敏字段的工具类（这里只展示了mobile，其他类推）

```java
/**
 * @author tori
 * 2018/7/16 下午1:57
 */
public class SensitiveUtil {

    public static String mobile(String mobile) {
        if (StringUtils.isBlank(mobile)) {
            return "";
        }

        return StringUtils.left(mobile, 2).concat(
                StringUtils.removeStart(
                        StringUtils.leftPad(
                            StringUtils.right(mobile, 4), StringUtils.length(mobile), "*"),
                        "***"));
    }
}
```

### 第三步

创建标签类（构建标签）

```java
/**
 * @author tori
 * 2018/7/16 上午11:33
 */

//@Target({ElementType.TYPE, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@JacksonAnnotationsInside
@JsonSerialize(using = SensitiveInfoSerialize.class)
public @interface SensitiveInfo {

    SensitiveType value();
}
```

有时在写标签类时会碰到@Target，@Retention还有@Order的用法，这里简单的概括一下：

```
@Target

这个注解就是表明该注解类能够作用的范围，也就是能够注解在哪，比如 类、方法、参数等。 
下面是他的一些参数： 
@Target(ElementType.TYPE) //接口、类、枚举、注解 
@Target(ElementType.FIELD) //字段、枚举的常量 
@Target(ElementType.METHOD) //方法 
@Target(ElementType.PARAMETER) //方法参数 
@Target(ElementType.CONSTRUCTOR) //构造函数 
@Target(ElementType.LOCAL_VARIABLE)//局部变量 
@Target(ElementType.ANNOTATION_TYPE)//注解 
@Target(ElementType.PACKAGE) ///包 
里面的参数是可以多选的，使用方法比如@Target({ElementType.METHOD,ElementType.TYPE})。

@Retention

这个注解是保留说明，也就是表明这个注解所注解的类能在哪里保留，他有三个属性值： 
RetentionPolicy.SOURCE —— 这种类型的Annotations只在源代码级别保留,编译时就会被忽略 
RetentionPolicy.CLASS —— 这种类型的Annotations编译时被保留,在class文件中存在,但JVM将会忽略 
RetentionPolicy.RUNTIME —— 这种类型的Annotations将被JVM保留,所以他们能在运行时被JVM或其他使用反射机制的代码所读取和使用。 
从上面可以看出一般使用的事第三个属性，其余两个属性，说实话 我也不清楚什么情况下使用这两种。

@Order

@Order标记定义了组件的加载顺序，这个标记包含一个value属性。属性接受整形值。如：1,2 等等。值越小拥有越高的优先级。Ordered.HIGHEST_PRECEDENCE这个属性值是最高优先级的属性，它的值是-2147483648，对应的最低属性值是Ordered.LOWEST_PRECEDENCE，它的值是2147483647。
```

### 第四步

最重要的标签序列化处理类：

```java
/**
 * @author tori
 * 2018/7/16 下午1:33
 */
public class SensitiveInfoSerialize extends JsonSerializer<String> implements ContextualSerializer {

    private SensitiveType type;

    public SensitiveInfoSerialize(SensitiveType type) {
        this.type = type;
    }

    public SensitiveInfoSerialize() {}

    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider serializers) throws IOException, JsonProcessingException {
        switch (this.type) {
            case MOBILE:
                gen.writeString(SensitiveUtil.mobile(value));
                break;
            case EMAIL:
                gen.writeString(SensitiveUtil.email(value));
                break;
            case OPEN_ID:
                gen.writeString(SensitiveUtil.openId(value));
                break;
        }
    }

    @Override
    public JsonSerializer<?> createContextual(SerializerProvider prov, BeanProperty property) throws JsonMappingException {
        if (property != null) { // 为空直接跳过
            if (Objects.equals(property.getType().getRawClass(), String.class)) { // 非 String 类直接跳过
                SensitiveInfo sensitiveInfo = property.getAnnotation(SensitiveInfo.class);
                if (sensitiveInfo == null) {
                    sensitiveInfo = property.getContextAnnotation(SensitiveInfo.class);
                }
                if (sensitiveInfo != null) { // 如果能得到注解，就将注解的 value 传入 SensitiveInfoSerialize

                    return new SensitiveInfoSerialize(sensitiveInfo.value());
                }
            }
            return prov.findValueSerializer(property.getType(), property);
        }
        return prov.findNullValueSerializer(property);
    }
}
```

### 最后一步

就是使用和测试啦：

```java
 /**
  * 手机号码
  */
@SensitiveInfo(SensitiveType.MOBILE)
private String mobile;

```

这样一来获得的json中mobile，email，openId就可以隐藏部分内容，实现脱敏了。



以上是对spring标签实现脱敏的流程描述。☺
