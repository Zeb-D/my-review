# 自定义自动配置springboot-starter

<br>

在springboot的开发中，starter是一个核心的配置，只需要引入对应模块的starter，然后在application.properties中引入对应的配置项，就可以开发业务逻辑了。这一切都归功于springboot的自动配置的能力。

<br>

接下来让让我们基于Jedis客户端封装一个我们自己的简易版starter： spring-boot-redis-starter

###开发方案：

1. 基于spring.factories
2. 基于注解式启动

这两种方式前提都有**共同的代码**，只是启动方式不一样，使用方式不一样，效果一致

<br>

###新建工程

新建maven工程，引入springboot核心依赖：`spring-boot-starter`

具体pom.xml如下：

```java
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.9.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.yd</groupId>
    <artifactId>spring-boot-redis-starter</artifactId>
    <packaging>jar</packaging>
    <version>1.0.0</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>
    </dependencies>

</project>
```

加入了Jedis依赖。

### 开发共同代码

1. ### 配置参数映射RedisProperties

   新建一个POJO类，以配置参数映射，具体作用，映射配置文件application.properties到实体的字段

   ```java
   /**
    * 映射配置文件application.properties到实体的字段
    * @author Yd on 2018-06-30
    */
   @ConfigurationProperties(prefix = "redis")//取配置文件前缀为redis配置
   public class RedisProperties {

       private String host;
       private int port;

       public String getHost() {
           return host;
       }

       public void setHost(String host) {
           this.host = host;
       }

       public int getPort() {
           return port;
       }

       public void setPort(int port) {
           this.port = port;
       }
   }
   ```

   ​

2. ### 实现自动化配置RedisAutoConfiguration.java

    自动化配置的核心就是提供实体bean的组装和初始化，对于本starter而言，我们主要组装的核心bean就是 **Jedis** 这个实体。

   ```java
   /**
    * 
    * @author Yd on 2018-06-30
    */
   @Configuration
   @ConditionalOnClass(Jedis.class)    // 存在Jedis这个类才装配当前类
   @ConditionalOnProperty(name = "redis.enabled", havingValue = "true", matchIfMissing = true) //配置文件存在这个redis.enabled=true才启动，允许不存在该配置
   @EnableConfigurationProperties(RedisProperties.class)
   //@ConditionalOnBean：当SpringIoc容器内存在指定Bean的条件
   //@ConditionalOnExpression：基于SpEL表达式作为判断条件
   //@ConditionalOnJava：基于JVM版本作为判断条件
   //@ConditionalOnJndi：在JNDI存在时查找指定的位置
   //@ConditionalOnMissingBean：当SpringIoc容器内不存在指定Bean的条件
   //@ConditionalOnMissingClass：当SpringIoc容器内不存在指定Class的条件
   //@ConditionalOnNotWebApplication：当前项目不是Web项目的条件
   //@ConditionalOnResource：类路径是否有指定的值
   //@ConditionalOnSingleCandidate：当指定Bean在SpringIoc容器内只有一个，或者虽然有多个但是指定首选的Bean
   //@ConditionalOnWebApplication：当前项目是Web项目的条件
   public class RedisAutoConfiguration {


       @Bean
       @ConditionalOnMissingBean   // 没有Jedis这个类才进行装配
       public Jedis jedis(RedisProperties redisProperties) {
           return new Jedis(redisProperties.getHost(), redisProperties.getPort());
       }
   }
   ```

   在这里，我们使用了Spring的@Configuration及@Bean来装配这Jedis Bean，只不过我们在这里加入了一些spring-boot相关注解：

   1. @Configuration: 声明当前类为一个配置类
   2. @ConditionalOnClass(Jedis.class)：当SpringIoc容器内存在Jedis这个Bean时才装配当前配置类
   3. @EnableConfigurationProperties(RedisProperties.class)：这是一个开启使用配置参数的注解，value值就是我们配置实体参数映射的ClassType，将配置实体作为配置来源。
   4. @ConditionalOnMissingBean：当SpringIoc容器内不存在Jedis这个Bean的时候才进行装配，否则不装配。
   5. @ConditionalOnProperty(name = "redis.enabled", havingValue = "true", matchIfMissing = true)  这里需要配置该参数才能加载Bean

   看到这些spring-boot注解，大部分是作为条件来装配Bean，这些注解都是元注解@Conditional演变而来的，根据不用的条件对应创建以上的具体条件注解。这些注解是我们实现自动配置的关键，体现了spring的哲学: **约定优于配置**

   ​

   那这为什么spring-boot能自动化运行呢，让我们深入理解一下

   <br>

   ## Starter自动化运作原理

   我们开发springboot应用的时候都知道要在启动类加注解：**@SpringBootApplication**

   在注解@SpringBootApplication上存在一个开启自动化配置的注解@EnableAutoConfiguration来完成自动化配置，注解源码如下所示：

   ```java
   @Target({ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Inherited
   @AutoConfigurationPackage
   @Import({EnableAutoConfigurationImportSelector.class})
   public @interface EnableAutoConfiguration {
       String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

       Class<?>[] exclude() default {};

       String[] excludeName() default {};
   }
   ```

   在@EnableAutoConfiguration注解内使用到了spring 的 @import注解来完成导入配置的功能

   重点是在于**EnableAutoConfigurationImportSelector**内部则是使用了SpringFactoriesLoader.loadFactoryNames方法进行扫描具有META-INF/spring.factories文件的jar包。

   我们可以看下spring-boot-autoconfigure包内的spring.factories文件内容：

   ```java
   # Initializers
   org.springframework.context.ApplicationContextInitializer=\
   org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
   org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer

   # Application Listeners
   org.springframework.context.ApplicationListener=\
   org.springframework.boot.autoconfigure.BackgroundPreinitializer

   # Auto Configuration Import Listeners
   org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
   org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

   # Auto Configuration Import Filters
   org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
   org.springframework.boot.autoconfigure.condition.OnClassCondition

   # Auto Configure
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
   org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
   org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
   .....省略
   ```

   可以看到配置的结构形式是Key/Value形式，如果存在多个Value时则使用,隔开，那我们在自定义starter内也可以使用这种形式来完成，因为我们的目的是为了完成自动化配置，所以我们这里Key则是需要使用**org.springframework.boot.autoconfigure.EnableAutoConfiguration**

   <br>

   ## 继续开发

   ### 基于spring.factorie

   在src/main/resource目录下创建META-INF目录,并在该目录下添加文件spring.factories，并配置：

   ```
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.yd.redis.starter.config.RedisAutoConfiguration
   ```

### 创建其他maven模块测试：

####并引入该依赖：

```java
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.9.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.yd</groupId>
    <artifactId>spring-boot-hello</artifactId>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.yd</groupId>
            <artifactId>spring-boot-redis-starter</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>

    <build>
        <finalName>${project.artifactId}</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

####在该测试模块中配置文件application.properties中引入下面的配置：

```
redis.enabled=true
redis.host=192.168.1.47
redis.port=7936
```

#### 运行测试

```java
@SpringBootApplication
public class HelloDemoApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext context =
                SpringApplication.run(HelloDemoApplication.class, args);
        Jedis jedis = context.getBean(Jedis.class);
        jedis.set("name", "Zeb");
        jedis.setex("Zeb", 10, "Zeb");
        System.out.println(jedis.get("name"));
        context.close();
    }
}
```

直接运行，可以看到控制台打印：Zeb

这种基于spring.factories，都会去尝试加载，属于主动加载。

<br>

###基于注解

接下来，让我们看另外一种实现方式（注解式启动加载，使用注解了就激活加载）：

```java
@Inherited
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(RedisAutoConfiguration.class)
public @interface EnableRedis { //相当于使用定义spring.factories完成Bean的自动装配
    //spring.factories 在这里配置，只要引用jar就加载到
    //@Import(RedisAutoConfiguration.class) 需要在调用者的Main 类加上该注解就能等效于spring.factories 文件配置
}
```

代码就是这么简洁，核心在于 **@Import(RedisAutoConfiguration.class)** 这个注解，其实它的作用和spring.factories一样。

使用也很容易，只需要在启动类添加注解即可，代码如下：

```java
@SpringBootApplication
@EnableRedis
public class HelloDemoApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext context =
                SpringApplication.run(HelloDemoApplication.class, args);
        Jedis jedis = context.getBean(Jedis.class);
        jedis.set("name", "Zeb");
        jedis.setex("Zeb", 10, "Zeb");
        System.out.println(jedis.get("name"));
        context.close();
    }
}
```

执行的效果和定义spring.factories一致，但是实现方式不一致，可以多深入理解spring-boot 运行原理。

<br><br>

### 代码：

spring-boot-redis-starter ：https://github.com/Zeb-D/spring-cloud/tree/master/spring-boot-redis-starter

测试代码：https://github.com/Zeb-D/spring-cloud/tree/master/spring-boot-hello

<br>

本篇文章已同步到：https://github.com/Zeb-D/my-review



