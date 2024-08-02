### 常见报错

`java.lang.IllegalArgumentException: Invalid value type for attribute 'factoryBeanObjectType': java.lang.String`

### 原因

mybatis-plus导入的mybatis-spring2.1.2版本太低，需要3.x以上版本

### 解决

#### 方式一：

1.导入最新的mybatis-plus3.5.5依赖，并且排除自带的mybaits-spring（可不排除）

```xml
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-boot-starter</artifactId>
  <version>3.5.5</version>
   <!--排除mybatis-spring-->
  <exclusions>
    <exclusion>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis-spring</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

2.重新手动添加mybaits-spring依赖

```xml
  <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>3.0.3</version>
</dependency>
```

#### 方式二：

不排除，直接导入mybatis-spring

```xml
<dependency>
	<groupId>com.baomidou</groupId>
	<artifactId>mybatis-plus-spring-boot3-starter</artifactId>
	<version>3.5.5</version>
</dependency>
<dependency>
	<groupId>org.mybatis</groupId>
	<artifactId>mybatis-spring</artifactId>
	<version>3.0.3</version>
</dependency>
```

​	