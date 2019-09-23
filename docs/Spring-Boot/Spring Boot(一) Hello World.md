## Spring Boot(一) Hello World

### 一、Spring Boot之我见

&nbsp;&nbsp;&nbsp;&nbsp;*Spring Boot*是由Pivotal团队提供的全新框架，其设计目的是用来简化新*Spring*应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。

&nbsp;&nbsp;&nbsp;&nbsp;Spring Boot 是所有基于 Spring 开发的项目的起点。Spring Boot 的设计是为了让你尽可能快的跑起来 Spring 应用程序并且尽可能减少你的配置文件

&nbsp;&nbsp;&nbsp;&nbsp;这是百科上对Spring Boot的说明。实际上在我看来Spring Boot就是Spring等一系列的java技术栈框架的组合加上约定大于配置的思想。它并不是一项新的技术也不是一个新的框架，而是各种技术的一个组合，它默认配置了很多框架的使用方式。它的出现，极大的提高了java应用的开发效率。其特性就是轻量级、可插拔、微服务。同时idea和spring boot的出现也颠覆了我一个.net程序员曾经对java的认知（我是.net出身的程序员，对于java最大的印象就是“配置”、“配置”、“配置”和难用的一笔的IDE）。原来Java项目开发也可以变得优雅起来！

### 二、项目搭建

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在还是SSM(SSH)的时代搭建一个Web项目大概需要以下几步

 1. 新建项目

 2. 配置Web.xml，加载Spring和Spring Mvc

 3. 配置数据库连接、配置Spring事务，配置mybatis

 4. 配置加载配置文件的读取，开启注解

 5. 配置日志文件

 6. 配置完成后部署Tomcat调试

    ...

    一堆的配置，看得人头大，也让很多初学者望而却步。

    而有了Spring Boot，一切将会简单而优雅。

    本文将从Hello World做起，搭建一个简单的Spring Boot Web项目。

1. File——New——Project（Moudle）

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/HelloWorld/1.png)

*注意*: 图中的https://start.spring.io/，也可以直接打开，在网站上直接生成项目下载，用idea打开也是一样的。

2. Next将看到以下界面

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/HelloWorld/2.png)

3. 继续下一步，选择Web->Spring Boot，在这里我们可以选择Spring Boot版本

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/HelloWorld/3.png)

4. 继续下一步

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/HelloWorld/4.png)

5. 点击完成将看到如下目录结构

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/HelloWorld/5.png)

至此，一个Spring Boot项目就创建完成。

### 三、项目结构说明

1. 项目结构中主要目录说明大致如下

```
->spring-boot-web						 (项目或模块)
---> src
----->main
------->java							 (Java代码目录)
--------->spring.boot.web				  (包spring.boot.web)
----------->SpringBootWebApplication	   (Spring Boot启动类)
------->resources
--------->static						 (静态资源目录)
--------->templates						 (视图模板目录)
--------->application.properties		  (项目配置文件)
------->test							(测试)
--------->java						
----------->spring.boot.web
----------->SpringBootWebApplicationTests  (Spring Boot测试启动类)
--->pom.xml								(maven pom配置文件)
```

2. pom文件说明

*&nbsp;&nbsp;&nbsp;&nbsp;pom.xml*，它是maven来管理各种jar包依赖的一个配置文件，maven相当于.net中的nuget，是一个包管理工具。我们可以看到项目搭建好之后，就默认为我们加上了下面这些配置。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <!--父节点配置了Spring Boot的一些信息，父节点代表子节点可以继承父节点的一些配置，如版本号，在这里配置了就不用在子节点配置了-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.8.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <!--项目信息-->
    <groupId>spring.boot.examples</groupId>
    <artifactId>spring-boot-web</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-boot-web</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <!--依赖的包配置在下面这个节点-->
    <dependencies>
        <!--引入Spring Boot Web模块-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
		<!--引入Spring Boot 测试模块-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!--Spring Boot Maven 插件-->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

3. Spring Boot启动类

```java
package spring.boot.web;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringBootWebApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootWebApplication.class, args);
    }

}

```

启动类是Spring Boot程序的入口。

### 四、Hello World

1. 新建控制器

   我们在spring.boot.web下建一个controller包，包内新建HelloWorld类

   ```java
   package spring.boot.web.controller;
   
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   public class HelloWorld {
   
       @RequestMapping("/helloWorld")
       public String Hello() {
           return "Hello World!";
       }
   }
   
   ```

   这里有两个注解：

   `@RestController`的意思就是controller里面的方法都以json格式输出，不用再写什么jackjson配置的了！

   `@RequestMapping`配置Url映射

2. 启动Spring Boot项目

   由于Spring Boot项目内置Tomcat服务器，我们不需要在部署到Tomcat。只需要要在配置文件application.properties里配置一下端口号，然后idea->run->run。

```java
  server.port=8888
```

如下图，代表启动成功！

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/HelloWorld/6.png)

打开浏览器输入：http://localhost:8888/helloWorld

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-boot/HelloWorld/7.png)

**至此，Hello World项目完成。于Java上，再次体验到了简洁优雅的开发！**



**[示例代码](https://github.com/hunter-droid/spring-boot-examples)**

