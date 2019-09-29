## Spring Boot(三) 使用Lombok

&nbsp;&nbsp;&nbsp;&nbsp;C#写的多了用习惯了众多的语法糖，再写起来Java总会有一些非常不舒服的地方。比如用惯了C#的属性在用起来Java的属性，写起来就会感觉不够优雅。如:定义一个`Person`类

```c#
    public class Person
    {
        public string Name { get; set; }

        public int Age { get; set; }  

        public string Describe { get; set; }

    }
```

而同样的代码，java写起来就...

```java

public class Person {

    private String name;

    private Integer age;

    private String escribe;

    public Person() {
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public String getEscribe() {
        return escribe;
    }

    public void setEscribe(String escribe) {
        this.escribe = escribe;
    }

}
```

为了能够更优雅的编码，我们可以使用Lombok

### 一、Lombok介绍

&nbsp;&nbsp;&nbsp;&nbsp;Lombok是一个Java库，能自动插入编辑器并构建工具，简化Java开发。通过添加注解的方式，不需要为类编写getter或equals方法，同时可以自动化日志变量...[官网链接](https://www.projectlombok.org/)

​	简单来说，这玩意能以简单注解的方式简化Java代码，以提高开发效率

### 二、 项目中引入Lombok

1. 添加maven依赖

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.10</version>
    <scope>provided</scope>
</dependency>
```

2. 创建一个`Person`类，并为其加上`@Data`注解

```java
package spring.boot.lombok.model;

import lombok.Data;

@Data
public class Person_Lombok {

    private String name;

    private Integer age;

    private String describe;
}
```

3. 安装idea插件

使用Lombok还需要插件的配合，打开idea的设置，点击**Plugins**，点击**Browse repositories**，在弹出的窗口中搜索**lombok**，然后安装即可。

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/Spring%20Boot%28%E4%B8%89%29%20%E4%BD%BF%E7%94%A8Lombok/1.png)

编译时出错，可能是没有enable注解处理器。`Annotation Processors > Enable annotation processing`。设置完成之后程序正常运行。

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/Spring%20Boot%28%E4%B8%89%29%20%E4%BD%BF%E7%94%A8Lombok/2.png)

4. 测试运行

```java
package spring.boot.lombok;

import org.junit.Assert;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import spring.boot.lombok.model.Person_Lombok;

@RunWith(SpringRunner.class)
@SpringBootTest
public class Person_LombokTests {

    @Autowired
    public Person_Lombok person;

    @Test
    public void Test() {
        person.setName("王小二");
        person.setAge(19);
        person.setDescribe("豆蔻年华，正青春！");
        Assert.assertTrue(person.getAge() == 19);
    }
}

```



### 三、Lombok常用注解

- `@Setter` 注解在类或字段，注解在类时为所有字段生成setter方法，注解在字段上时只为该字段生成setter方法。
- `@Getter` 使用方法同上，区别在于生成的是getter方法。
- `@ToString` 注解在类，添加toString方法。
- `@EqualsAndHashCode` 注解在类，生成hashCode和equals方法。
- `@NoArgsConstructor` 注解在类，生成无参的构造方法。
- `@RequiredArgsConstructor` 注解在类，为类中需要特殊处理的字段生成构造方法，比如final和被@NonNull注解的字段。
- `@AllArgsConstructor` 注解在类，生成包含类中所有字段的构造方法。
- `@Data` 注解在类，生成setter/getter、equals、canEqual、hashCode、toString方法，如为final属性，则不会为该属性生成setter方法。
- `@Slf4j` 注解在类，生成log变量，严格意义来说是常量。private static final Logger log = LoggerFactory.getLogger(UserController.class);
- `@NonNull`给方法参数增加这个注解会自动在方法内对该参数进行是否为空的校验，如果为空，则抛出NPE（NullPointerException）
- `@Cleanup`自动管理资源，用在局部变量之前，在当前变量范围内即将执行完毕退出之前会自动清理资源，自动生成try-finally这样的代码来关闭流
- `@Synchronized`用在方法上，将方法声明为同步的，并自动加锁，而锁对象是一个私有的属性$lock或$LOCK，而java中的synchronized关键字锁对象是this，锁在this或者自己的类对象上存在副作用，就是你不能阻止非受控代码去锁this或者类对象，这可能会导致竞争条件或者其它线程错误
- `@SneakyThrows`自动抛受检异常，而无需显式在方法上使用throws语句

### 四、Lombok原理解析

&nbsp;&nbsp;&nbsp;&nbsp;我们可以反编译一下，idea为我们生成的.class文件，

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/Spring%20Boot%28%E4%B8%89%29%20%E4%BD%BF%E7%94%A8Lombok/3.png)

可以看到虽然使用了lombok后我们没有写get,set等方法，但是是因为idea安装了lombok的插件，在编译的时候帮我们生成了这些方法。

在Lombok使用的过程中，只需要添加相应的注解，无需再为此写任何代码。核心之处就是对于注解的解析上。JDK5引入了注解的同时，也提供了两种解析方式。

- 运行时解析

运行时能够解析的注解，必须将`@Retention`设置为`RUNTIME`，这样就可以通过反射拿到该注解。`java.lang.reflect`反射包中提供了一个接口`AnnotatedElement`，该接口定义了获取注解信息的几个方法，Class、Constructor、Field、Method、Package等都实现了该接口。

- 编译时解析

编译时解析有两种机制，分别简单描述下：

1）Annotation Processing Tool

apt自JDK5产生，JDK7已标记为过期，不推荐使用，JDK8中已彻底删除，自JDK6开始，可以使用Pluggable Annotation Processing API来替换它，apt被替换主要有2点原因：

- api都在com.sun.mirror非标准包下
- 没有集成到javac中，需要额外运行

2）Pluggable Annotation Processing API

`JSR 269`自JDK6加入，作为apt的替代方案，它解决了apt的两个问题，javac在执行的时候会调用实现了该API的程序，这样我们就可以对编译器做一些增强，javac执行的过程如下：



![img](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/Spring%20Boot%28%E4%B8%89%29%20%E4%BD%BF%E7%94%A8Lombok/4.png)

`Lombok`本质上就是一个实现了`JSR 269 API`的程序。在使用javac的过程中，它产生作用的具体流程如下：

1. `javac`对源代码进行分析，生成了一棵抽象语法树（AST）
2. 运行过程中调用实现了`JSR 269 API`的`Lombok`程序
3. 此时`Lombok`就对第一步骤得到的AST进行处理，找到@Data注解所在类对应的语法树AST，然后修改该语法树AST，增加getter和setter方法定义的相应树节点
4. `javac`使用修改后的抽象语法树AST生成字节码文件，即给class增加新的节点（代码块）

### 五、Lombok优缺点

**优点：**

* 能通过注解的形式自动生成构造器、getter/setter、equals、hashcode、toString等方法，提高了一定的开发效率

* 让代码变得简洁，不用过多的去关注相应的方法

* 属性做修改时，也简化了维护为这些属性所生成的getter/setter方法等

**缺点：**

* 不支持多种参数构造器的重载

* 虽然省去了手动创建getter/setter方法的麻烦，但大大降低了源代码的可读性和完整性。



**[示例代码](https://github.com/hunter-droid/spring-boot-examples)**

