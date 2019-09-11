## Spring Cloud(一) Eureka组合Feign

### 一、Eureka和Feign

### 1.Eureka简介

&nbsp;&nbsp;&nbsp;&nbsp;Eureka是Netflix开源的**一款提供服务注册和发现的产品**，它提供了完整的Service Registry和Service Discovery实现。也是springcloud体系中最重要最核心的组件之一。

&nbsp;&nbsp;&nbsp;&nbsp;Spring Cloud封装了Netflix公司开发的Eureka模块来实现服务注册和发现。Eureka采用了 C-S的设计架构。Eureka Server作为服务注册功能的服务器，它是服务注册中心。而系统中的其他微服务，使用 Eureka的客户端连接Eureka Server，并维持心跳连接。系统的维护人员可以通过 Eureka Server来监控系统中各个微服务是否正常运行。Spring Cloud的一些其他模块（比如Zuul）就可以通过 Eureka Server 来发现系统中的其他微服务，并执行相关的逻辑。

&nbsp;&nbsp;&nbsp;&nbsp;Eureka由两个组件组成：Eureka服务器和Eureka客户端。Eureka服务器用作服务注册服务器。Eureka客户端是一个java客户端，用来简化与服务器的交互、作为轮询负载均衡器，并提供服务的故障切换支持。

#### 2.Feign简介

&nbsp;&nbsp;&nbsp;&nbsp;Feign是一个声明式Web Service客户端（实际上就是一个http请求调用的轻量级框架）。使用Feign能让编写Web Service客户端更加简单, 它的使用方法是定义一个接口，然后在上面添加注解，同时也支持JAX-RS标准的注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持了Spring MVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

#### 3.Eureka组合Feign

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-cloud/spirng-cloud-eureka/Eureka.png)

图中主要包括三个角色

1. Eureka Server

- 提供服务注册和发现

2. Service Provider

- 服务提供方,使用Feign实现
- 将自身服务注册到Eureka，从而使服务消费方能够找到

3. Service Consumer

- 服务消费方，使用Feign实现
- 从Eureka获取注册服务列表，从而能够消费服务

### 二、Eureka Server

#### 1. pom中添加依赖

```xml
	<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <!--注意：这里的spring boot版本和下面的spring cloud版本是有对应关系的-->
        <version>1.5.9.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
 
    <properties>
        <java.version>1.8</java.version>
         <!--注意：这里的spring cloud版本和上面的spring boot版本是有对应关系的-->
        <spring-cloud.version>Dalston.SR5</spring-cloud.version>
    </properties>

	<!--需要引入的依赖包-->
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

```

**注意:Spring Boot和Spring Cloud是有版本对应关系的,版本不对应的时候启动会报错！！！**

大版本对应:

| Spring Boot   | Spring Cloud             |
| ------------- | ------------------------ |
| 1.2.x         | Angel版本                |
| 1.3.x         | Brixton版本              |
| 1.4.x stripes | Camden版本               |
| 1.5.x         | Dalston版本、Edgware版本 |
| 2.0.x         | Finchley版本             |
| 2.1.x         | Greenwich.SR2            |

详细版本信息可查**https://start.spring.io/actuator/info**

#### 2、启动类

添加启动代码中添加`@EnableEurekaServer`注解

```java
@SpringBootApplication
@EnableEurekaServer
public class SpringCloudEurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEurekaApplication.class, args);
    }

}
```

#### 3. 配置文件(application.properties)

```xml
server.port=10001
spring.application.name=spring-cloud-eureka

<!--Eureka配置-->
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:${server.port}/eureka/
```

- `eureka.client.register-with-eureka` ：表示是否将自己注册到Eureka Server，默认为true。

- `eureka.client.fetch-registry` ：表示是否从Eureka Server获取注册信息，默认为true。

  (在默认设置下，该服务注册中心也会将自己作为客户端来尝试注册它自己，所以我们需要禁用它的客户端注册行为)

- `eureka.client.serviceUrl.defaultZone` ：设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。

启动工程后，访问：http://localhost:10001

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-cloud/spirng-cloud-eureka/eureka_start.jpg)



### 三、Feign Producer

#### 1.pom中添加依赖

同二Eureka中的pom

#### 2.配置文件(application.properties)

```xml
spring.application.name=spring-cloud-producer
server.port=10002

eureka.client.serviceUrl.defaultZone=http://localhost:10001/eureka/
```

#### 3.启动类

启动类中添加`@EnableDiscoveryClient`注解

```java
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudFeignProducerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudFeignProducerApplication.class, args);
    }
}
```

#### 4.controller

提供Person服务

```java
@RestController
public class PersonProducerController {

    @RequestMapping("/person")
    public String get(@RequestParam String name) {
        return new Person(name).toString();
    }
}


public class Person {
    public Person(String name) {
        this.name = name;
    }    
    private String name;
    @Override
    public String toString() {
        return MessageFormat.format("Hello,{0}!", name);
    }
}
```

添加`@EnableDiscoveryClient`注解后，项目就具有了服务注册的功能。依次启动eureka和producer，就可以在注册中心的页面看到**SPRING-CLOUD-PRODUCER**服务。

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-cloud/spirng-cloud-eureka/eureka_feign_producer_start.png)

### 四、Feign Consumer

#### 1.pom中添加依赖

同二Eureka中的pom，另外新增依赖

```
  <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-feign</artifactId>
  </dependency>
```



#### 2.配置文件(application.properties)

```
spring.application.name=spring-cloud-consumer
server.port=10003
eureka.client.serviceUrl.defaultZone=http://localhost:10001/eureka/
```

#### 3、启动类

启动类添加`@EnableDiscoveryClient`和`@EnableFeignClients`注解。

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class SpringCloudFeignConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudFeignConsumerApplication.class, args);
    }
}
```

- `@EnableDiscoveryClient` :启用服务注册与发现
- `@EnableFeignClients`：启用feign进行远程调用

#### 4、调用实现

```java
@FeignClient(name = "spring-cloud-producer")
public interface Person {

    @RequestMapping(value = "/person")
    String get(@RequestParam(value = "name") String name);
}
```

- name:远程服务名，即producer中`spring.application.name`配置的名称

#### 5. controller调用

```java
@RestController
public class PersonConsumerController {

    @Autowired
    private Person person;

    @RequestMapping("/person")
    public String get(@RequestParam String name) {
        return person.get(name);
    }

}
```

依次启动eureka和producer，就可以在注册中心的页面看到**SPRING-CLOUD-PRODUCER**服务和**SPRING-CLOUD-PRODUCER**。

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-cloud/spirng-cloud-eureka/eureka_feign_producer_consumer_start.png)

测试服务，在浏览器键入`localhost:10003/person?name=张三`

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/spring-cloud/spirng-cloud-eureka/eureka_feign_result.png)

### 附:

[**示例地址**](https://github.com/hunter-droid/spring-cloud-examples)