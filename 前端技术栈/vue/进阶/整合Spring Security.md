## 配置

### 1.添加SpringSecurity启动依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```



### 2.在config包下创建WebSecurityConfig.java文件

> springboot项目可以省略@EnableWebSecurity

```java
@Configuration//配置类
@EnableWebSecurity//开启SpringSecurity自定义配置类
public class WebSecurityConfig {
    
}
```

## 使用

### 1.基于内存的用户认证流程

> 创建manager管理user，代替默认的用户user，密码*********（控制台）

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {
  @Bean
  public UserDetailsService userDetailsService() {
    // 创建基于内存的用户信息管理器
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    // 使用manager对象，管理userDetails对象
    manager.createUser(
      // 创建UserDetails对象，用于管理用户名用户密码，用户权限等内容
      User
      .withDefaultPasswordEncoder()
      .username("zyr")
      .password("123456")
      .roles("USER")
      .build()
    );
    return manager;
  }
}
```

### 2.基于数据库的用户认证流程

- 程序启动时：
  - 创建==DBUserDetailsManager==对象，实现接口UserDetailsManager，UserDetailsPasswordService
- 校验用户时：
  - 在SpringSecurity自动使用==DBUserDetailsManager==的==loadUserByUsername==方法从数据库中获取User对象
  - 在==UsernamePasswordAuthenticationFilter==过滤器中的==attempAuthentication==方法中将用户输入的用户名密码和从数据库中获取的用户信息进行比较进行用户认证
