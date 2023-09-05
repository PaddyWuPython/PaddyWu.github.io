---
layout: post
title: 【spring boot 系列】Logback 配置样例
categories: Spring
description: Logback 配置样例
keywords: SpringBoot, MyBatis, Generator
---

这篇文章主要提供一个 xml格式的 Logback 配置文件参考，详细技术细节参考

[日志实现框架 Logback](2020-03-06-spring-boot-logback.md)

## 配置说明

以下所有的配置只依赖 Logback 官方的 jar 包，不使用任何第三方的扩展包


## 配置文件注意事项

Logback 会按照以下顺序查找配置文件
- 首先在 classpath 中查找 logback-test.xml，如果找不到
- Logback 会在 classpath 中查找 logback.groovy，如果找不到
- Logback 会在 classpath 中查找 logback.xml，如果找不到
- Logback 将进行自动配置


## xml 配置样例

```
<?xml version="1.0" encoding="UTF-8"?>

<!-- scan ：开启"热更新" scanPeriod："热更新"扫描周期，默认 60 seconds(60秒)-->
<configuration scan="true" scanPeriod="300 seconds">

    <!-- 自定义变量，用于配置日志输出格式，这个格式是尽量偏向 spring boot 默认的输出风格
    %date：日期，默认格式 yyyy-MM-dd hhh:mm:ss,SSS 默认使用本机时区，通过 %d{yyyy-MM-dd hhh:mm:ss,SSS} 来自定义
    %-5level：5个占位符的日志级别，例如" info"、"error"
    %thread : 输出日志的线程
    %class : 输出日志的类的完全限定名，效率低
    %method : 输出日志的方法名
    %line : 输出日志的行号，效率低
    %msg : 日志消息内容
    %n : 换行
    -->
    <property name="LOG_PATTERN" value="%date %-5level --- [%thread] %class.%method/%line : %msg%n"/>
    <!-- 彩色日志格式 -->
    <property name="LOG_PATTERN_COLOURS" value="%date %green(%-5level) --- [%thread] %cyan(%class.%method/%line) : %msg%n"/>

    <!--日志输出器. ch.qos.logback.core.ConsoleAppender : 输出到控制台-->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <!-- 配置日志输出格式 -->
            <pattern>${LOG_PATTERN_COLOURS}</pattern>
            <!-- 使用的字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 日志输出器。ch.qos.logback.core.rolling.RollingFileAppender : 滚动输出到文件 -->
    <appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 活动中的日志文件名(支持绝对和相对路径) -->
        <file>logs/app1/app1.log</file>
        <!-- 滚动策略. ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy : 按照大小和时间滚动-->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 何时触发滚动，如何滚动，以及滚动文件的命名格式
            %d : 日期，默认格式 yyyy-MM-dd，通过 %d{yyyy-MM-dd hhh:mm:ss} 来自定义格式。logback 就是通过 %d 知道了触发滚动的时机
            %i : 单个滚动周期内的日志文件的序列号
            .zip : 将日志文件压缩成zip。不想压缩，可以使用.log 结尾
            如下每天0点以后的第一日志请求触发滚动，将前一天的日志打成 zip 压缩包存放在 logs/app1/backup 下，并命名为 app1_%d_%i.zip
            -->
            <fileNamePattern>logs/app1/backup/app1_%d{yyyy-MM-dd}_%i.zip</fileNamePattern>

            <!--单个日志文件的最大大小-->
            <maxFileSize>10MB</maxFileSize>

            <!--删除n个滚动周期之前的日志文件(最多保留前n个滚动周期的历史记录)-->
            <maxHistory>30</maxHistory>
            <!-- 在有 maxHistory 的限制下，进一步限制所有日志文件大小之和的上限，超过则从最旧的日志开始删除-->
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <!-- 日志输出格式 -->
            <pattern>${LOG_PATTERN}</pattern>
            <!-- 使用的字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 记录器 name : 包名或类名， level : 要记录的日志的起始级别， additivity : 是否追加父类的 appender -->
    <logger name="com.wqlm.boot" level="debug" additivity="false" >
        <appender-ref ref="STDOUT" />
    </logger>

    <!-- 根记录器 -->
    <root level="info">
        <!-- 使用 STDOUT、ROLLING 输出记录的日志-->
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="ROLLING"/>
    </root>
</configuration>
```

## 简单配置方案

如果只是进行简单的配置，使用 application.properties 配置文件就够了
```
# 当前活动的日志文件名
# logging.file.name=logs/app1/app.log
# 最多保留多少天的日志
# logging.file.max-history=30
# 单个日志文件最大容量
# logging.file.max-size=10MB
# dao 层设置成 debug 级别以显示sql
logging.level.com.wqlm.boot.user.dao=debug
```

## 样例代码

[springboot-logback-demo](https://github.com/piterjia/springboot-logback-demo)