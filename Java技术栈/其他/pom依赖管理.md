## dependency的区别

==< dependencies >==
**直接引入依赖**：在<dependencies>标签下声明的依赖会直接被当前项目所使用。这意味着这些依赖会被包含进项目的类路径(classpath)中。
**自动继承**：如果项目是一个多模块项目中的一个子模块，子模块会自动继承并获得父模块<dependencies>中声明的所有依赖，除非子模块明确排除了某些依赖或指定了不同的版本。
**适用场景**：适用于单模块项目或那些不需要集中管理依赖版本的简单多模块项目。

==< dependencyManagement >==
**依赖声明而非引入**：在<dependencyManagement>中声明的依赖并不会直接导致依赖被引入到当前项目中。它更像是一个版本控制中心，只声明依赖的坐标和版本，但不实际引入这些依赖。
**版本控制**：主要目的是为了解决多模块项目中依赖版本不一致的问题。子模块可以在自己的<dependencies>标签下引用<dependencyManagement>中声明的依赖，而无需指定版本号，从而确保整个项目中所有模块使用的依赖版本一致。
**灵活性与控制**：子模块可以选择性地引入<dependencyManagement>中声明的依赖，并且可以覆盖版本号，但通常推荐保持一致以避免版本冲突。
**适用场景**：特别适合大型多模块项目，或者需要严格控制依赖版本一致性的项目。

总结来说，<dependencies>直接决定了项目实际使用的依赖，而<dependencyManagement>则是一种集中管理依赖版本的机制，提供了一种声明依赖而不直接使用的手段，便于版本控制和维护。

## 父模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
    </parent>

    <groupId>org.example</groupId>
    <artifactId>pomFather</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>pomFather</name>
    <description>pomFather</description>
    <!--父模块的打包方式需要为pom-->
    <packaging>pom</packaging>

    <!--子模块-->
    <modules>
        <module>test1</module>
    </modules>

    <!--属性版本-->
    <properties>
        <java.version>17</java.version>
        <mysql.version>8.3.0</mysql.version>
        <hutool.version>5.8.27</hutool.version>
        <mybatis-plus.version>3.5.5</mybatis-plus.version>
        <spring-cloud.version>2023.0.1</spring-cloud.version>
        <spring-cloud-alibaba.version>2023.0.1.0</spring-cloud-alibaba.version>
    </properties>

    <!--子模块继承的是版本-->
    <dependencyManagement>
        <dependencies>
            <!--spring-cloud-->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!--spring-cloud-alibaba-->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!--hutool工具包-->
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
                <version>${hutool.version}</version>
            </dependency>
            <!--mybatis-plus-->
            <dependency>
                <groupId>com.baomidou</groupId>
                <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
                <version>${mybatis-plus.version}</version>
            </dependency>
            <!--mysql连接-->
            <dependency>
                <groupId>com.mysql</groupId>
                <artifactId>mysql-connector-j</artifactId>
                <version>${mysql.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!--子模块直接继承并使用-->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--spring热部署-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <!--lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--整合swagger-->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.5.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>

```

## 子模块1

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.example</groupId>
        <artifactId>pomFather</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>test1</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>test1</name>
    <description>test1</description>

    <properties>

    </properties>
    <dependencies>
        <!--nacos注册发现-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <!--mysql连接-->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
        </dependency>
        <!--RabbitMQ客户端等-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <!--redis-data-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
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



