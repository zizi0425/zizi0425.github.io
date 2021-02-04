---
title: logback的使用
date: 2021-01-23 22:57:47
tags: logback
categories: 日志
toc: true
---

  logback日志打印; 以及控制日志大小/保留天数的控制

<!--more-->

  ### 将error日志和普通打印打印分开

  - 在appender中增加过滤器

  ```xml
      <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
          <level>WARN</level>
      </filter>
  ```

  ### 设置日志文件大小和时效

  - TimeBasedRollingPolicy 升级为 SizeAndTimeBasedRollingPolicy
  - fileNamePattern需要增加.%i用于文件分割
  - maxHistory为最多保留天数
  - totalSizeCap为日志容量;防止日志占用服务器太多资源

  ### 区分不同的环境

  - 如果使用<root>标签则代表所有环境统一,也可以根据不同的环境设置不同的打印方式
    如:

  ```xml
   <springProfile name = "dev">
          <root level="info" additivity="false">
              <appender-ref ref="console-local" />
              <appender-ref ref="ALL" />
              <appender-ref ref="ERROR_FILE" />
          </root>
      </springProfile>
  ```

  - 注意自定义后不可在使用<root>标签会造成日志会打印两份

  ### 链路追踪

  - @see org.slf4j.MDC
  - MDC在logback中用%X可以获取；假设有代码：

  ```java
  MDC.put("tid", UUIDUtils.getUUID());
  ```

  则可以在logback中用%X{tid} 获取到该值 ， 这样便于链路追踪的日志打印



### logback的一个demo

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <contextName>log</contextName>
    <property name="projectName" value="logTest"/>
    <property name="logPath" value="/data/logs" />
    <property name="defaultEncoderPattern" value="%d{HH:mm:ss.SSS} %contextName [%X{tid}] %-5level %logger{36} - %msg%n" />


    <!--输出到控制台-->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <charset>UTF-8</charset>
            <pattern>${defaultEncoderPattern}</pattern>
        </encoder>
    </appender>

    <!--输出到控制台-->
    <appender name="console-local" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <charset>UTF-8</charset>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %boldYellow(%X{ThreadID}) %highlight(%-5level) %boldGreen(%logger{36}.%method:%line)  - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 输出全部日志到文件中 -->
    <appender name="ALL" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${logPath}/${projectName}/${projectName}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${logPath}/${projectName}/${projectName}-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>500MB</maxFileSize>
            <maxHistory>7</maxHistory>
            <totalSizeCap>8GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${defaultEncoderPattern}</pattern>
        </encoder>
    </appender>

    <!-- 错误日志：用于将错误日志输出到独立文件 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${logPath}/${projectName}/error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>500MB</maxFileSize>
            <maxHistory>7</maxHistory>
            <totalSizeCap>8GB</totalSizeCap>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${defaultEncoderPattern}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>WARN</level>
        </filter>
    </appender>

    <root level="info" additivity="false">
        <appender-ref ref="console" />
        <appender-ref ref="ALL" />
        <appender-ref ref="ERROR_FILE" />
    </root>

    <!--
    <springProfile name = "dev">
        <root level="info" additivity="false">
            <appender-ref ref="console-local" />
            <appender-ref ref="ALL" />
            <appender-ref ref="ERROR_FILE" />
        </root>
    </springProfile>-->

</configuration>
```

