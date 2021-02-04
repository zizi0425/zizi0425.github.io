---
title: Spring Boot 与 Spring Cloud版本不一致(一)
date: 2020-12-13 22:11:56
tags:
	- 版本不兼容
categories: spring
toc: true
---



Spring Boot 与 Spring Cloud版本不一致时引入Prometheus进行服务监控时服务启动报错追踪

<!--more-->

-  Spring Boot 和Spring Cloud 版本关系对应表:  https://spring.io/projects/spring-cloud 

## 项目报错

- 今天测试告知服务启动失败,服务pull代码后正常启动
- 执行maven clean重新build后启动报错,报错日志如下:

```
***************************
APPLICATION FAILED TO START
***************************

Description:

An attempt was made to call a method that does not exist. The attempt was made from the following location:

    org.springframework.cloud.client.discovery.health.DiscoveryCompositeHealthIndicator.<init>(DiscoveryCompositeHealthIndicator.java:42)

The following method did not exist:

    org.springframework.boot.actuate.health.CompositeHealthIndicator.<init>(Lorg/springframework/boot/actuate/health/HealthAggregator;)V

The method's class, org.springframework.boot.actuate.health.CompositeHealthIndicator, is available from the following locations:

    jar:file:/D:/framework/develop/idea/repository/org/springframework/boot/spring-boot-actuator/2.2.2.RELEASE/spring-boot-actuator-2.2.2.RELEASE.jar!/org/springframework/boot/actuate/health/CompositeHealthIndicator.class

It was loaded from the following location:

    file:/D:/framework/develop/idea/repository/org/springframework/boot/spring-boot-actuator/2.2.2.RELEASE/spring-boot-actuator-2.2.2.RELEASE.jar


Action:

Correct the classpath of your application so that it contains a single, compatible version of org.springframework.boot.actuate.health.CompositeHealthIndicator

Disconnected from the target VM, address: '127.0.0.1:0', transport: 'socket'

Process finished with exit code 1

```

## 代码/错误日志 跟踪

根据git log的commit查到有人在服务中接入Prometheus进行服务监控

```java
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>

```

和报错已经特别吻合了; 当时脑海里立马想到的就是是不是springcloud和springboot的版本不一致,这里先把项目原先的版本列出

```
Spring Boot - 2.2.2.RELEASE
Spring Cloud - Greenwich.SR3
```

错误日志提示的比较明显,直接点击到类中

- `DiscoveryCompositeHealthIndicator`对应的jar包`spring-cloud-common:2.1.3.RELEASE`<font color = 'red'>**注**:down下来直接是报错的</font>

```
	@Autowired
	public DiscoveryCompositeHealthIndicator(HealthAggregator healthAggregator,
			List<DiscoveryHealthIndicator> indicators) {
		-- 这里在报错
		super(healthAggregator);
		for (DiscoveryHealthIndicator indicator : indicators) {
			Holder holder = new Holder(indicator);
			addHealthIndicator(indicator.getName(), holder);
			this.healthIndicators.add(holder);
		}
	}
```

- 打开父类方法 (`spring-boot-actuator-2.2.2.RELEASE`) :

```
/**
 * {@link HealthIndicator} that returns health indications from all registered delegates.
 *
 * @author Tyler J. Frederick
 * @author Phillip Webb
 * @author Christian Dupuis
 * @since 1.1.0
 * @deprecated since 2.2.0 in favor of a {@link CompositeHealthContributor}
 */
@Deprecated
public class CompositeHealthIndicator implements HealthIndicator {
   ...
}
```

明显被启用了; 对应的构造器也不存在了

## 升级spring Cloud版本

按照官网版本关系,在项目的pom文件中(因为父pom是设计其他服务伙伴使用; 因此抛出改问题后先修改自己的pom文件)

```
       <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.SR8</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

## -- END AND NOTE

- Spring Cloud 和Spring Boot版本不一致的问题时有发生,有时候可能正常运行,但是一旦接入其他jar包就会由此产生问题,因此Spring Boot和Spring Cloud的版本升级尽量保持同步

