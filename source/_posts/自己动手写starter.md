---
title: 自己动手写spring-boot-starter
date: 2019-11-02 09:21:08
tags: [java,springboot]
categories: spring
---

#### 写在前面的废话

经常有人问我编程是一件很难的事情吗，我想说写程序其实不是一件很难的事情。你只不过是按照你自己的逻辑去写一些if-else、for-while循环，而且好多逻辑产品都会帮你梳理（有需求），你也不是凭空去写一些没有实际应用的功能。但是对于写程序，程序能跑起来有一个很重要的东西，就是跑程序的环境。

这个门槛一下子就提高了，Spring中有很多配置文件，就会让很多人折腾好久。最后发现可能就是一个配置项没有配置（某个配置项写的不对），这个可能这个坑有人踩过（你可以去Google一下），因为你要使用的好多组件都要接入Spring（组件中的对象变成Spring管理的Bean），但是要配置那些Bean是要你很熟悉这个组件，而且你要很了解Spring，知道为什么报错如何解决。为了解决这个spring-boot引入自动配置（这个之前有讲spring-factories中讲过）。这样Spring就可以通过别人之前配置好的Bean直接引入。省去你对环境的配置让你更专注业务的开发，而不是被环境搞的焦头烂额，而且也让Spring更容易使用，引入插件也更方便。（你可以搜SpringMVC和spring-boot接入mybatis的差距）。

#### 什么是spring-boot-starter

官方给的解释

```
Starter POMs are a set of convenient dependency descriptors that you can include in your application. You get a one-stop-shop for all the Spring and related technology that you need, without having to hunt through sample code and copy paste loads of dependency descriptors. For example, if you want to get started using Spring and JPA for database access, just include the spring-boot-starter-data-jpa dependency in your project, and you are good to go.
鄙人的英语水平很烂，求轻喷
翻译：
Starter的POMs是一系列方便的依赖集合（pom.xml），在你的应用里（定语从句）。你可以一站式获取Spring和你需要的技术的依赖集合，而不用通过示例代码去复制依赖集合。例如你想通过JPA访问数据库，只需要在你的工程中添加spring-boot-starter-data-jpa这个依赖就可以。
```

大概意思就是说starter是一种对依赖的synthesize（合成）。可以说stater是一个包装好默认你需要组件的配置信息（默认的配置等于约定），同时你可以自己自定义新的配置来定制化。这样的好处，就是我们不用每次创建一个工程都要把之前实例代码的配置在copy一遍，而是直接在你的工厂加入starter就好了。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/spring-starter.png)

上面的starter依赖的jar和我们自己手动配置的时候依赖的jar并没有什么不同，所以我们可以认为starter其实是把这一些繁琐的配置操作交给了自己，而把简单交给了用户。除了帮助用户去除了繁琐的构建操作，在“约定大于配置”的理念下，`ConfigurationProperties`还帮助用户减少了无谓的配置操作。并且因为 `application.properties` 文件的存在，即使需要自定义配置，所有的配置也只需要在一个文件中进行，使用起来非常方便。

starter其实就是帮助用户简化了配置的操作之后，要理解starter和被配置了starter的组件之间并不是竞争关系，而是辅助关系，即我们可以给一个组件创建一个starter来让最终用户在使用这个组件的时候更加的简单方便。基于这种理念，我们可以给任意一个现有的组件创建一个starter来让别人在使用这个组件的时候更加的简单方便，事实上Spring Boot团队已经帮助现有大部分的流行的组件创建好了它们的starter

#### 约定大于配置（convention over configuration）

我们这里简称`COC`。本质是说，开发人员仅需规定应用中不符约定的部分。旨在减少软件开发人员需做决定的数量，获得简单的好处，而又不失灵活性。如果您所用工具的约定与你的期待相符，便可省去配置；反之，你可以配置来达到你所期待的方式。（了解Ruby on Rails的朋友一定知道）。

##### spring boot中的默认配置

1、maven的目录文件结构，默认有resources文件夹,存放资源配置文件。`src-main-resources`,`src-main-java`

2、默认有target文件夹，将生成class文件盒编程生成的jar存放在target文件夹下。

3、spring boot默认只会去`src-main-resources`文件夹下去找application配置文件。

#### 自己动手写一个spring-boot-starter

如果想自己写一个starter重要是把你这个Jar里面的Bean放到Spring的容器里。我们可以参考一个Spring中自动注入的简单的例子。这个是JdbcTemplate的自动注入过程。这个是JdbcTemplate的自动注入类，这个自动注入的一个实例类。对应spring.fatories中会有一条`org.springframework.boot.autoconfigure.EnableAutoConfiguration=\org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,`的配置

```java
@Configuration//用于声明Bean配置类
@ConditionalOnClass({ DataSource.class, JdbcTemplate.class })//当存在DataSource类和JdbcTemplate类才会生效配置
@ConditionalOnSingleCandidate(DataSource.class)//当Spring容器中只有一个DataSource才生效配置
@AutoConfigureAfter(DataSourceAutoConfiguration.class)//指定加载顺序在DataSourceAutoConfiguration类后面
@EnableConfigurationProperties(JdbcProperties.class) //指定自动配置类JdbcProperties.class
public class JdbcTemplateAutoConfiguration {//jdbc模板类的自动配置类

	@Configuration
	static class JdbcTemplateConfiguration {  //嵌套的依赖类要使用static，内置的配置类

		private final DataSource dataSource;//数据源对象

		private final JdbcProperties properties;//Jdbc属性配置信息

		JdbcTemplateConfiguration(DataSource dataSource, JdbcProperties properties) {
			this.dataSource = dataSource;
			this.properties = properties;
		}//JdbcTemplateConfiguration类构造方法

		@Bean
		@Primary//主要类，当多个实例，选择这个为主要的
		@ConditionalOnMissingBean(JdbcOperations.class)//不存在JdbcOperations.class生效
		public JdbcTemplate jdbcTemplate() {
            //配置jdbcTemplate信息（丰富jdbc模板类的属性信息）
			JdbcTemplate jdbcTemplate = new JdbcTemplate(this.dataSource);//设置数据源
			JdbcProperties.Template template = this.properties.getTemplate();//获取模板信息
			jdbcTemplate.setFetchSize(template.getFetchSize());
			jdbcTemplate.setMaxRows(template.getMaxRows());//设置最大行数
			if (template.getQueryTimeout() != null) {//查询超时时间
				jdbcTemplate
						.setQueryTimeout((int) template.getQueryTimeout().getSeconds());
			}
			return jdbcTemplate;
		}

	}

	@Configuration
	@Import(JdbcTemplateConfiguration.class)
	static class NamedParameterJdbcTemplateConfiguration {

		@Bean
		@Primary
		@ConditionalOnSingleCandidate(JdbcTemplate.class)
		@ConditionalOnMissingBean(NamedParameterJdbcOperations.class)
		public NamedParameterJdbcTemplate namedParameterJdbcTemplate(
				JdbcTemplate jdbcTemplate) {
			return new NamedParameterJdbcTemplate(jdbcTemplate);
		}

	}

}
```

Jdbc属性配置类通过`@ConfigurationProperties`注解指定配置类前缀

```java
@ConfigurationProperties(prefix = "spring.jdbc")
public class JdbcProperties {

	private final Template template = new Template();

	public Template getTemplate() {
		return this.template;
	}

	/**
	 * {@code JdbcTemplate} settings.
	 */
	public static class Template {


		private int fetchSize = -1;
		
		private int maxRows = -1;

		@DurationUnit(ChronoUnit.SECONDS)
		private Duration queryTimeout;

		public int getFetchSize() {
			return this.fetchSize;
		}

		public void setFetchSize(int fetchSize) {
			this.fetchSize = fetchSize;
		}

		public int getMaxRows() {
			return this.maxRows;
		}

		public void setMaxRows(int maxRows) {
			this.maxRows = maxRows;
		}

		public Duration getQueryTimeout() {
			return this.queryTimeout;
		}

		public void setQueryTimeout(Duration queryTimeout) {
			this.queryTimeout = queryTimeout;
		}

```

在application.properties文件时这样指定的。

```
spring.jdbc.template.fetch-size=1
spring.jdbc.template.query-timeout=1
spring.jdbc.template.max-rows=1
```

这里面`spring.jdbc`是前缀名，`template`变量名字，`fetch-size`变量的属性名字，有了`@ConfigurationProperties`这个注解，可以通过类直接去指定配置文件属性对应的变量，而不是在xml或者`@Bean`配置方法去设置。如果使用自动配置还要使用一个注解`@EnableConfigurationProperties`。

Starter中代码看上去不难，只不过多了好多注解，我这边解释一下这几个注解的使用。我们在自己创建Starter的时候也会用到这些注解，所以我们要对这些注解有很好的理解，这里也有Spring中通用注解。

| 注解                             | 解释                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| `@AutoConfigureAfter`            | 指定自动配置类加载顺序，在某个特定的配置类后面               |
| `@AutoConfigureBefore`           | 指定自动配置类加载顺序，在某个特定的配置类前面               |
| `@AutoConfigureOrder`            | z指定初始化顺序数越小越先初始化                              |
| `@Configuration`                 | 配置到类上，表明这个类声明了多个`@Bean`注释的方法，`@Bean`注释的方法用于`Spring`容器运行中`Bean`的定义，生成的处理方法。（一般`@Bean`都是有返回对象的）。主要用于Bean声明和处理。配置类必须作为类提供（即不作为实例返回），从而允许通过生成的子类增强运行时。配置类必须是非最终的。  配置类必须是非本地的（即，不能在方法中声明）。**任何嵌套的配置类都必须声明为 static**。  `@Bean`方法可能不会再创建其他配置类任何此类实例都将被视为具有常规Bean的配置 |
| `@ConditionalOnClass`            | 指定路径`classpath`中存在该类时起效。基于Spring的`@Conditional` |
| `@ConditionalOnSingleCandidate`  | Spring容器中该类型Bean只有一个或@Primary的只有一个时起效     |
| `@ConditionalOnBean `            | Spring容器中存在该类型Bean时起效                             |
| `@ConditionalOnMissingBean`      | 指定路径`classpath`中不存在该类时起效。                      |
| `@ConditionalOnExpression `      | SpEL表达式结果为true时                                       |
| `@ConditionalOnProperty `        | 参数设置或者值一致时起效                                     |
| `@ConditionalOnWebApplication `  | Web应用环境下起效                                            |
| `@EnableConfigurationProperties` | 通过这个注解，可以把外部对应配置的属性值，设置到`@Configuration`声明的类中`@Bean`声明的方法中的对象的属性。通过`EnableConfigurationPropertiesImportSelector.class`实现的 |
| `@Import`                        | 导入一个或者多个Bean配置类(用`@configration`注解解释的类)    |
| `@Bean`                          | 一般使用在方法上面，是这个方法的返回对象，注册到Bean容器里面 |
| `@ConfigurationProperties`       | 用于外部绑定配置文件，通过配置`（prefix=“spring.jdbc”）`前缀配置文件指定外部头。然后通过变量去绑定（`spring.jdbc.hello`）就是配置对象中hello属性。 |
| `@Primary`                       | 当一个类有多个实例在bean容器里，这个是指定主要的使用的的那个Bean（声明Bean标识） |

简单介绍完大致需要的注解和大致的玩法，下面我说一下怎么玩。

##### 创建工程

首先要建一个maven工程，然后给指定一些依赖

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.qjq</groupId>
    <artifactId>just-spring-boot-starter</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>just-spring-boot-starter</name>
    <description>自定义starter</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <!-- @ConfigurationProperties annotation processing (metadata for IDEs)
                     生成spring-configuration-metadata.json类，需要引入此类-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
        </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

注意其中 `spring-boot-configuration-processor` 的作用是编译时生成`spring-configuration-metadata.json`， 此文件主要给IDE使用，用于提示使用。如在intellij idea中，当配置此jar相关配置属性在`application.yml`， 你可以用ctlr+鼠标左键，IDE会跳转到你配置此属性的类中。

这里说下artifactId的命名问题，Spring 官方 Starter通常命名为`spring-boot-starter-{name}` 如 `spring-boot-starter-web`。

Spring官方建议非官方Starter命名应遵循`{name}-spring-boot-starter`的格式

##### 创建配置属性类

```java
package com.qjq.justspringbootstarter;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "spring.qjq.just")
public class SimpleBeanProperties {

    private String  name;

    private String add;

    private String  filed;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAdd() {
        return add;
    }

    public void setAdd(String add) {
        this.add = add;
    }

    public String getFiled() {
        return filed;
    }

    public void setFiled(String filed) {
        this.filed = filed;
    }
}

```

这里在properties文件中可以配置一些属性。`@SimpleBeanProperties`指定前缀名称

##### 编写注入的Bean

```java
package com.qjq.justspringbootstarter;

public class SimpleService {

    private String name;

    private String filed;

    private String add;

    public SimpleService(String name, String filed, String add) {
        this.name = name;
        this.filed = filed;
        this.add = add;
    }

    /**
     *  处理业务
     * @return
     */
    public String dealSimple() {
        return "hello" + name + "world" + filed + "simple" + add;
    }
}

```

##### 编写自动注入的类

```java
package com.qjq.justspringbootstarter;


import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@EnableConfigurationProperties(SimpleBeanProperties.class)
@Configuration
@ConditionalOnClass({SimpleBeanProperties.class,SimpleService.class})
public class SimpleBeanAutoConfigure {


    private final SimpleBeanProperties properties;

    public SimpleBeanAutoConfigure(SimpleBeanProperties properties) {
        this.properties = properties;
    }
    @Bean
    @Primary
    @ConditionalOnProperty(prefix = "spring.qjq.just", value = "enabled",havingValue = "true")
    public SimpleService simpleService(){
       return new SimpleService(properties.getName(),properties.getFiled(),properties.getAdd());
    }
}

```

##### 编写spring.factories文件

`resources/META-INF/`下创建`spring.factories`文件

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.qjq.justspringbootstarter.SimpleBeanAutoConfigure
```

使用Maven install命令打包。然后就可以使用了

#### 测试

##### 创建一个spring-boot工程，导入依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.10.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>Demo project for Spring Boot</description>

    <properties>
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
            <groupId>com.qjq</groupId>
            <artifactId>just-spring-boot-starter</artifactId>
            <version>0.0.2-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>5.1.11.RELEASE</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test</artifactId>
            <version>2.1.10.RELEASE</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

##### 在配置文件中加入配置

```
spring.qjq.just.name=demo
spring.qjq.just.add=1234567890
spring.qjq.just.filed=application
spring.qjq.enable=true

```

##### 测试代码

```java
package com.example.demo.templates;
import com.just.share.spring.boot.autoconfigure.SimpleService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import javax.annotation.Resource;

@RunWith(SpringRunner.class)
@SpringBootTest
public class DemoApplicationTests {
    @Resource
    private SimpleService simpleService;
    @Test
    public void contextLoads() {
        simpleService.dealSimple();
    }

}

```

##### 输出结果

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/result.png)

##### 总结

Spring Boot应用中这些第三方库几乎可以零配置的开箱即用（out-of-the-box），大部分的Spring Boot应用都只需要非常少量的配置代码，开发者能够更加专注于业务逻辑。

#### 参考

[Spring Boot Starters是什么？](https://www.cnblogs.com/tjudzj/p/8758391.html)

[SpringBoot系列 - 自己写starter](https://www.xncoding.com/2017/07/22/spring/sb-starter.html)