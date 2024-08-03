门面使用lombok 的@Slf4j

```xml
       <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.32</version>
        </dependency>
```

导入logback日志实现

```xml
        <!-- https://mvnrepository.com/artifact/ch.qos.logback/logback-classic -->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.5.6</version>
        </dependency>
```

配置logback.xml文件（**可选**）

不需要生成**日志文件**时，把`<appender>`标签中的`<rollingPolicy>`删除

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="LOG_PATH" value="logs" />
    <property name="LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] %-40.40logger{39} : %msg%n" />

    <!-- 控制台输出 -->
    <appender name="consoleLog" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 彩色日志 -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss} [%thread] %magenta(%-5level) %green([%-50.50class]) >>> %cyan(%msg) %n
            </pattern>
        </layout>
    </appender>

    <!-- 按照每天生成日志文件 -->
    <appender name="fileLog"  class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--日志文件输出的文件名-->
            <FileNamePattern>${LOG_PATH}/cms.%d{yyyy-MM-dd}.%i.log</FileNamePattern>
            <!--日志文件最大的大小-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- 日志输出级别 -->
    <root level="info">
        <appender-ref ref="consoleLog" />
        <appender-ref ref="fileLog" />
    </root>

</configuration>

```

