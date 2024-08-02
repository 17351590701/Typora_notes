#  第一章、Security入门



> [!NOTE]
> 新版本security教程
>
> https://blog.csdn.net/weixin_46073538/article/details/128641746
>
> **遗留问题**：未解决前后端整合（不分离）打包环境下，前端资源放行问题

##  1.0、底层架构

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061740224.png" alt="image-20240429220317094" style="zoom:67%;" />

![image-20240727201333918](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407272013059.png)

## 1.1、SecurityPorperties配置

导入security依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

访问localhost:8080/test  只要没有登录，统一跳转浏览器提供的/login页面，未定义时，登录名为：user，密码在idea控制台

```java
@RestController
public class UserController {
    @GetMapping("/test")
    public String test(){
        return "登陆成功";
    }
}

```

application.yml定义默认初始化用户

```java
spring:  
	security:
    user:
      name: admin
      password: 123456
```

# 第二章、Security自定义配置

## 2.1、基于内存的用户认证

> 程序启动时：
>
> - 创建==InMemoryUserDetailsManager==对象
> - 创建==User==对象，封装用户名，密码
> - 使用 InMemoryUserDetailsManager将==User存入内存==
>
> 校验用户时：
>
> - SpringSecurity自动使用InMemoryUserDetailsManager 的==loadUserByUsername==方法从 ==内存中==获取User对象
> - 在==UsernamePassowrdAuthenticationFilter== 过滤器中==attemptyAuthentication== 方法中将用户输入的用户名密码和从内存中获取到的用户信息进行比较，进行用户认证

在config包下新建WebSecurityConfig.java

InMemoryUserDetailsManager-->实现 UserDetailsManager 继承-->UserDetailsService

#### WebSecurityConfig.java  

```java
@Configuration
//@EnableWebSecurity //springboot项目可省略
public class WebSecurityConfig {
  @Bean
  //用户认证服务
  public UserDetailsService userDetailsService(){
    //创建基于内存的用户信息管理器
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    //manager 管理 UserDetails对象
    manager.createUser(
      //创建还能UserDetails对象，用于管理用户名，密码，角色，权限等内容
 			 User.withDefaultPasswordEncoder() //默认密码加密方案
      .username("zyr")
      .password("123456")
      .roles("USER")
      .build());
    return manager;
  }
}
```

## 2.2、基于数据库的用户认证

> 程序启动时：
>
> - 自定义==DBUserDetailsManager==类,实现接口 UserDetailsManager,UserDetailsPasswordService
> - 在应用程序中初始化此对象
>
> 校验用户时：
>
> - SpringSecurity自动使用DBUserDetailsManager 的==loadUserByUsername==方法从 ==数据库==获取User对象
> - 在UsernamePassowrdAuthenticationFilter 过滤器中==attemptyAuthentication== 方法中将用户输入的用户名密码和从数据库获取到的用户信息进行比较，进行用户认证


DBUserDetailsManager-->实现 UserDetailsManager 继承-->UserDetailsService··

#### DBUserDetailsManager

> 不需要再WebSecurityConfig中再进行以下操作，否则会栈溢出
>
> ```java
>@Bean
> public UserDetailsService userDetailsService() {
>  return new DBDetailsService();
> }
>    ```

```java
// UserDetailManager继承了UserDetailService
@Component
public class DBUserDetailsManager implements UserDetailsManager, UserDetailsPasswordService {
    @Resource
    private UserMapper userMapper;

    .
    .//重写的一些方法
    .
    .
      
    // 获取用户信息
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        QueryWrapper<User> query = new QueryWrapper<>();
        // query.eq("username",username);
        query.lambda().eq(User::getUsername, username);
        User user = userMapper.selectOne(query);
        if (user == null) {
            throw new UsernameNotFoundException("用户不存在");
        } else {
            // 空权限列表
            Collection<GrantedAuthority> authorities = new ArrayList<>();
          	//组装security中的User对象
            return new org.springframework.security.core.userdetails.User(
                    user.getUsername(),
                    user.getPassword(),
                    user.getEnabled(),//需要为true
                    true,// 用户账户是否过期
                    true,// 用户凭证是否过期
                    true,// 用户是否锁定
                    authorities// 权限列表
            );
        }
    }
}
```

#### WebSecurityConfig

需要有实现PasswordEncoder的Bean

```java
@Configuration
//@EnableWebSecurity //springboot项目可省略
public class WebSecurityConfig {
    @Bean
    public UserDetailsService userDetailsService() {
        // 基于数据库的用户信息管理器
        DBUserDetailsManager manager = new DBUserDetailsManager();
        return manager;
    }
   	//密码解析器
    @Bean
    public PasswordEncoder passwordEncoder() {
      	//生成默认代理密码加密方案，带编码器{id}的BCryptPasswordEncoder,如{bcrypt}$2a$10$jXL9y71Fh7W......
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```



## 2.3、过滤器链

> UserDetailsService不直接在过滤器链中，但它是过滤器链中认证过程的关键组成部分，通过AuthenticationProvider间接参与了过滤器链的逻辑

WebSecurityConfig中未配置SecurityFilterChain时会有默认配置

```java
@Configuration
//@EnableWebSecurity //springboot项目可省略
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(authorize -> authorize
                        .anyRequest()      // 对所有请求开启授权保护
                        .authenticated()   // 已认证的请求会被自动授权
                )
                .formLogin(withDefaults()) // 表单授权方式
                .httpBasic(withDefaults());// 基本授权方式
        return http.build();
    }
}
```

### 使用过滤器链完成新增用户

> 使用DBUserDetailsManager中的createUser方法，参数为，封装成UserDetails的User用户

#### UserController

```java
@ResponseBody//当使用@Controller注解时，应使用  
@PostMapping("/add")
    public void add(@RequestBody User user){
        userService.saveUserDetails(user);
    }
```



#### UserServiceImpl

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    @Resource
    private DBUserDetailsManager dbUserDetailsManager;
    @Override
    public void saveUserDetails(User user) {
        UserDetails userDetails = org.springframework.security.core.userdetails.User
                .withDefaultPasswordEncoder()   //使用默认密码加密方案,BCryptPasswordEncoder
                .username(user.getUsername())
                .password(user.getPassword())
                .build();
        dbUserDetailsManager.createUser(userDetails);
    }
}
```



#### DBUserDetailsManager

```java
@Component
 public class DBUserDetailsManager implements UserDetailsManager, UserDetailsPasswordService {
    @Resource
    private UserMapper userMapper;
    //向数据库中插入新用户信息
    @Override
    public void createUser(UserDetails userDetails) {
        User user= new User();
        user.setUsername(userDetails.getUsername());
        user.setPassword(userDetails.getPassword());
        user.setEnabled(true);
        userMapper.insert(user);
    }
   .
   .
   .
   //其他重写方法

}
```

#### WebSecurityConfig

```java
@Configuration
//@EnableWebSecurity //springboot项目可省略
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(authorize -> authorize
                        .anyRequest()      // 对所有请求开启授权保护
                        .authenticated()   // 已认证的请求会被自动授权
                )
                .formLogin(withDefaults()); // 表单授权方式
                // .httpBasic(withDefaults());// 基本授权方式
        // 关闭csrf攻击防御
        http.csrf(csrf -> csrf.disable());
        return http.build();
    }
		//密码解析器
    @Bean
    public PasswordEncoder passwordEncoder() {
      	//生成默认代理密码加密方案，带编码器{id}的BCryptPasswordEncoder,如{bcrypt}$2a$10$jXL9y71Fh7W......
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```



## 2.4、自定义密码解析器

> Security框架定义的接口：PasswordEncoder
>
> ​	Security框架强制要求，必须在spring容器中存在PasswordEncoder类型对象，且对象唯一

#### MyPasswordencoder

```java
public class MyPasswordEncoder implements PasswordEncoder {
    /**
     * 密码加密方法
     * @param rawPassword 明文
     * @return 密文
     */
    @Override
    public String encode(CharSequence rawPassword) {
        //直接返回，不做加密
        return rawPassword.toString();
    }
    
    /**
     * 校验明文，密文是否相同
     * @param rawPassword 明文，登录时客户端传递的
     * @param encodedPassword 密文，服务器中存储的，一般存储在数据库
     * @return 是否相同
     */
    @Override
    public boolean matches(CharSequence rawPassword, String encodedPassword) {
        // 用相同加密策略，将明文加密与密文比较
        return encode(rawPassword).equals(encodedPassword);
    }
    
    /**
     *  是否需要升级，强化密码解析策略
     * @param encodedPassword 
     * @return
     */
    @Override
    public boolean upgradeEncoding(String encodedPassword) {
        return PasswordEncoder.super.upgradeEncoding(encodedPassword);
    }
}

```

#### MySecurityConfig.java

```java
@Configuration
public class WebSecurityConfig {
    //创建类型为PasswordEncoder的Bean对象
    @Bean
    public PasswordEncoder passwordEncoder(){
        return new MyPasswordEncoder();
    }
}
```

==认证安全强化，修改密码解析器的实现对象==

> Security框架提供若干密码解析器实现类型，其中BCryptPasswordEncoder为强散列加密。可以保证相同明文，多次加密后，密文有相同的散列数据，而不是相同结果，匹配时，是基于相同的散列数据做的匹配

```java
@Configuration
public class WebSecurityConfig {
    //创建类型为PasswordEncoder的Bean对象
    @Bean
    public PasswordEncoder passwordEncoder(){
      	//指定BCrypt加密方案
        //参数4~31，密码散列强度，越高，加密越好，加密速度越慢
        return new BCryptPasswordEncoder(4);
    }
}

```

## 2.5、自定义登录页面

```xml
<!--thymeleaf依赖-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

#### Login.html

使用==thymeleaf==在resource--templates下新建login.html，提交表单至==/user/login==

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>
<h1>登录</h1>
<div th:if="${param.error}">
    错误使用用户名和密码
</div>
<!--method必须为post
使用动态参数，表单会自动生成_csrf字段，防御csrf攻击-->
<form th:action="@{/user/login}" method="post">
    <div>
        <label><!--默认必须username-->
            <input type="text" name="username" placeholder="请输入用户名">
        </label>
    </div>
    <div>
        <label><!--默认必须password-->
            <input type="text" name="password" placeholder="请输入密码">
        </label>
    </div>
    <input type="submit" value="登录">
</form>
</body>
</html>
```

#### UserController.java

==需要跳转thymeleaf页面时，要使用@Controller注解，而非统一使用@RestController==

```java
@Controller
@RequestMapping("/user")
public class UserController {
    @GetMapping("/login")
    public String login(){
        return "login";
    }
    
}
```

#### WebSecurityConfig

对访问/user/login 放行

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(authorize -> authorize
                        .anyRequest()      // 对所有请求开启授权保护
                        .authenticated()   // 已认证的请求会被自动授权
                )
                .formLogin(form -> {
                            form.loginPage("/user/login").permitAll();//无需授权访问
                        }
                );
      					 // .formLogin(form -> {
                //             form.loginPage("/user/login").permitAll()// 无需授权访问
                //                     .usernameParameter("myUsername")   //联动指定表单字段
                //                     .passwordParameter("myPassword");
                //         }
                // );
        // 关闭csrf攻击防御
        http.csrf(AbstractHttpConfigurer::disable);
        return http.build();
    }
}
```

# 第三章、前后端分离

```xml
<!--fastJson2-->
  <dependency>
  <groupId>com.alibaba.fastjson2</groupId>
  <artifactId>fastjson2</artifactId>
  <version>2.0.37</version>
  </dependency>
```

## 3.1登录认证成功后响应

#### MyAuthenticationSuccessHandler

> 返回登录用户部分信息json数据给前端

```java
public class MyAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        Object principal = authentication.getPrincipal();//获取用户身份信息
        // Object credentials = authentication.getCredentials();//获取用户凭证信息
        // Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();//获取用户权限信息
        HashMap<String,Object> result = new HashMap<>();
        result.put("code", 200);
        result.put("msg", "登录成功");
        result.put("data", principal);
        //利用fastjson工具，将对象转换成json字符串
        String json = JSON.toJSONString(result);
        //返回json数据到前端
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().println(json);
    }
}
```

#### WebSecurityConfig

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .formLogin(form->{
                    form.loginPage("/user/login").permitAll()
                            .successHandler(new MyAuthenticationSuccessHandler())//认证成功后处理
                            ;
                }); // 表单授权方式

        return http.build();
    }

}
```

## 3.2登录认证失败响应

#### MyAuthenticationFailureHandler

> 返回失败信息json数据给前端

```java
public class MyAuthenticationFailureHandler implements AuthenticationFailureHandler {
    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        String localizedMessage = exception.getLocalizedMessage(); //登录失败的本地信息
        HashMap<String,Object> result = new HashMap<>();
        result.put("code", -1);
        result.put("msg", localizedMessage);
        //利用fastjson工具，将对象转换成json字符串
        String json = JSON.toJSONString(result);
        //返回json数据到前端
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().println(json);
    }
}
```

#### WebSecurityConfig

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .formLogin(form->{
                    form.loginPage("/user/login").permitAll()
                            .successHandler(new MyAuthenticationSuccessHandler())//认证成功后处理
                            .failureHandler(new MyAuthenticationFailureHandler())//认证失败处理
                            ;
                }); // 表单授权方式

        return http.build();
    }

}
```

## 3.3注销后处理

#### MyLogoutSuccessHandler

```java
public class MyLogoutSuccessHandler implements LogoutSuccessHandler {
    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        HashMap<String,Object> result = new HashMap<>();
        result.put("code", 0);
        result.put("msg", "注销成功");
        //利用fastjson工具，将对象转换成json字符串
        String json = JSON.toJSONString(result);
        //返回json数据到前端
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().println(json);
    }
}
```

#### WebSecurityConfig

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        //注销登出后处理
        http.logout(logout -> logout.logoutSuccessHandler(new MyLogoutSuccessHandler()));

        return http.build();
    }

}
```

#### UserController

```java
//主页  
@GetMapping()
    public String index(){
        return "index";
    }
```

#### index.html

```html
<!--提供登出按钮-->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="en">
<head>
    <meta charset="UTF-8">
    <title>主页</title>
</head>
<body>
<h1>Hello Security</h1>
<a th:href="@{/logout}">Logout</a>
</body>
</html>
```

## 3.4未认证返回登录处理

#### MyAuthenticationEntryPoint

```java
public class MyAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        String localizedMessage = "未认证，需要登录";//authException.getLocalizedMessage();
        HashMap<String,Object> result = new HashMap<>();
        result.put("code", -1);
        result.put("msg", localizedMessage);
        //利用fastjson工具，将对象转换成json字符串
        String json = JSON.toJSONString(result);
        //返回json数据到前端
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().println(json);
    }
}
```

#### WebSecurityConfig

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        //未认证返回登录页时处理
        http.exceptionHandling(exception -> exception.authenticationEntryPoint(new MyAuthenticationEntryPoint()));
        return http.build();
    }

}
```



## 3.5跨域处理

#### WebSecurityConfig

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
       //开启全局跨域
        http.cors(withDefaults());
        return http.build();
    }

}
```

# 第四章、用户认证信息

![securitycontextholder](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061740225.png)



> `SecurityContextHolder：` 是 Spring Security 存储已认证用户详细信息
>
> `SecurityContext`：是从 SecurityContextHolder 中获得的，包含当前已认证用户Authentication信息
>
> `Authentication`： 表示用户的身份认证，包含了用户的`Principal`,`Credential`和`Authority`信息
>
> `Principal`：表示用户的身份标识，通常是一个表示用户的实体对象，例如用户名等等
>
> `Credential:`通常是一个密码。在许多情况下，这在用户被认证后被清除，以确保它不会被泄露
>
> `authority:`实例是用户被授予的高级权限。两个例子是角色（role）和作用域（scope）



## 4.1获取已认证用户信息

#### UserController

```java
    @ResponseBody
    @GetMapping()
    public Map index(){
        SecurityContext context = SecurityContextHolder.getContext();
        Authentication authentication = context.getAuthentication();
        String name = authentication.getName();
        //Object principal = authentication.getPrincipal(); //凭证
        //Object credentials = authentication.getCredentials(); //脱敏
      	//权限信息
        Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities(); 
        HashMap<String, Object> result = new HashMap<>();
        result.put("name",name);
        result.put("authorities",authorities);

        return result;
    }
```



## 4.2会话并发处理

> 用户，客户端登录数量限制

#### OnEpiredSessionDetected

```java
public class OnEpiredSessionDetected implements SessionInformationExpiredStrategy {
    @Override
    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
        HashMap<String,Object> result = new HashMap<>();
        result.put("code", -1);
        result.put("msg", "该账号已从其他设备登录");
        //利用fastjson工具，将对象转换成json字符串
        String json = JSON.toJSONString(result);

        HttpServletResponse response = event.getResponse();
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write(json);
        //返回json数据到前端
    }
}

```

#### WebSecurityConfig

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        //会话失效处理，同一个账号，最多有几个客户端登录
        http.sessionManagement(session -> {
            session.maximumSessions(1).expiredSessionStrategy(new OnEpiredSessionDetected());
        });
        return http.build();
    }

}
```

# 第五章、授权

## 5.1基于request的授权

### 5.1.1用户--权限-资源

==为资源分类了权限，再为用户授予权限==

#### WebSecurityconfig

> 访问/user/list 路径时，判断该用户是否拥有 user:list 权限
>
> 访问/user/add 路径时，判断该用户是否拥有 user:add 权限

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(authorize -> authorize
                        .requestMatchers("/user/list").hasAuthority("user:list")
                        .requestMatchers("/user/add").hasAuthority("user:add")

                        .anyRequest()      // 对所有请求开启授权保护
                        .authenticated()   // 已认证的请求会被自动授权
                );

        return http.build();
    }

}
```

> 为简化便于演示，直接在加载该用户时，为其添加了权限 user:list, user:add

#### DBUserDetailsManager

```java
 @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        QueryWrapper<User> query = new QueryWrapper<>();
        // query.eq("username",username);
        query.lambda().eq(User::getUsername, username);
        User user = userMapper.selectOne(query);
        if (user == null) {
            throw new UsernameNotFoundException("用户不存在");
        } else {
            // 权限列表
            Collection<GrantedAuthority> authorities = new ArrayList<>();
            //直接为当前用户分配权限
            authorities.add(()->"user:list");
            authorities.add(()->"user:add");
            return new org.springframework.security.core.userdetails.User(
                    user.getUsername(),
                    user.getPassword(),
                    user.getEnabled(),//需要为true
                    true,// 用户账户是否过期
                    true,// 用户凭证是否过期
                    true,// 用户是否锁定
                    authorities// 权限列表
            );
        }
    }
```

### 5.1.2用户访问无权限处理

#### MyAccessDeniedHandler

```java
public class MyAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        HashMap<String,Object> result = new HashMap<>();
        result.put("code", -1);
        result.put("msg", "用户无权限");
        //利用fastjson工具，将对象转换成json字符串
        String json = JSON.toJSONString(result);
        //返回json数据到前端
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().println(json);
    }
}
```

#### WebSecurityConfig

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.exceptionHandling(
                exception -> {
                  	//未认证返回登录页时处理
                    exception.authenticationEntryPoint(new MyAuthenticationEntryPoint());
                  	//用户访问无权限时处理
                    exception.accessDeniedHandler(new MyAccessDeniedHandler());
                }
        );
        return http.build();
    }

}

```

### 5.1.3用户-角色-资源

> 为/user/**，所有路径下资源分配角色“ADMIN”

#### WebSecurityConfig

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(authorize -> authorize
                        .requestMatchers("/user/**").hasRole("ADMIN")
                        .anyRequest()      // 对所有请求开启授权保护
                        .authenticated()   // 已认证的请求会被自动授权
                );

        return http.build();
    }


}
```

> 为简化，在获取用户时，手动为其添加了“ADMIN”角色并返回

#### DBUserDetailsManager

```java
// 获取用户信息
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
  QueryWrapper<User> query = new QueryWrapper<>();
  // query.eq("username",username);
  query.lambda().eq(User::getUsername, username);
  User user = userMapper.selectOne(query);
  if (user == null) {
    throw new UsernameNotFoundException("用户不存在");
  } else {
    return org.springframework.security.core.userdetails.User
      .withUsername(user.getUsername())
      .password(user.getPassword())
      .password(user.getPassword())
      .disabled(!user.getEnabled())// 是否禁用
      .credentialsExpired(false)// 是否过期
      .accountLocked(false)// 是否锁定
      .roles("ADMIN")
      .build();

  }
}
```

本质上，直接用户--权限--资源，与用户--角色--资源，还是为`authorities`属性字段分类了`authority`

### 5.1.4用户--角色--权限--资源

> RBAC（Role-Based Access Control，基于角色的访问控制），是一种常用的数据库设计方案，它将用户的权限分配和管理与角色相关联。

#### 1.用户表

| 列名       | 数据类型    | 描述   |
| -------- | ------- | ---- |
| user_id  | int     | 用户id |
| username | varchar | 用户名  |
| password | varchar | 密码   |
| ...      |         |      |

#### 2.角色表

| 列名        | 数据类型 | 描述     |
| ----------- | -------- | -------- |
| role_id     | int      | 角色id   |
| role_name   | varchar  | 角色名   |
| description | varchar  | 角色描述 |
| ...         |          |          |

#### 3.权限表

| 列名              | 数据类型    | 描述   |
| --------------- | ------- | ---- |
| permission_id   | int     | 权限id |
| permission_name | varchar | 权限名  |
| description     | varchar | 权限描述 |
| ...             |         |      |

#### 4.用户角色关联表

| 列名         | 数据类型 | 描述   |
| ------------ | -------- | ------ |
| user_role_id | int      | id     |
| user_id      | int      | 用户id |
| role_id      | int      | 角色id |

#### 5.角色权限关联表

| 列名               | 数据类型 | 描述   |
| ------------------ | -------- | ------ |
| role_permission-id | int      | id     |
| role_id            | int      | 角色id |
| permission_id      | int      | 权限id |

## 5.2基于方法的授权

#### WebSecirityConfig

> 不使用requestMatchers 且使用注解`@EnableMethodSecurity`，开启方法授权

```java
@Configuration
@EnableMethodSecurity
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(authorize -> authorize
                        // .requestMatchers("/user/list").hasAuthority("user:list")
                        // .requestMatchers("/user/add").hasAuthority("user:add")
                        // .requestMatchers("/user/**").hasRole("ADMIN")

                        .anyRequest()      // 对所有请求开启授权保护
                        .authenticated()   // 已认证的请求会被自动授权
                );
        return http.build();
    }

}
```

#### DBUserDetailsManager

> ==.role与.authority不能同时使用，不然会被覆盖==
>
> 为获取用户时，添加角色“`ADMIN`”,或者添加权限‘`user:list`’,‘`user:add`’等等

```java
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        QueryWrapper<User> query = new QueryWrapper<>();
        // query.eq("username",username);
        query.lambda().eq(User::getUsername, username);
        User user = userMapper.selectOne(query);
        if (user == null) {
            throw new UsernameNotFoundException("用户不存在");
        } else {
            return org.springframework.security.core.userdetails.User
                    .withUsername(user.getUsername())
                    .password(user.getPassword())
                    .password(user.getPassword())
                    .disabled(!user.getEnabled())// 是否禁用
                    .credentialsExpired(false)// 是否过期
                    .accountLocked(false)// 是否锁定
                    .roles("ADMIN")
              			//.authority("user:list","user:add")
                    .build();

        }
    }
```

#### UserController

> 开启方法授权后，不使用`@PreAuthorize`注解标注的接口，默认都可以访问。
>
> `PreAuthorize`指定角色或者权限等等

```java
@CrossOrigin
@Controller
@RequestMapping("/user")
public class UserController {
    @Resource
    private UserService userService;

    // 主页显示内容
    @ResponseBody
    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping()
    public HashMap<String,Object> index(){
        SecurityContext context = SecurityContextHolder.getContext();
        Authentication authentication = context.getAuthentication();
        String name = authentication.getName();
        Object principal = authentication.getPrincipal(); //凭证
        Object credentials = authentication.getCredentials(); //脱敏
        Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities(); //权限信息
        
        HashMap<String, Object> result = new HashMap<>();
        result.put("name",name);
        result.put("authorities",authorities);

        return result;
    }

    @GetMapping("/login")
    public String login(){
        return "login";
    }
    @ResponseBody
    @PreAuthorize("hasRole('USER')")
    @GetMapping("/list")
    public List<User> getLIst(){
        return userService.list();
    }
    @ResponseBody
  	//@PreAuthorize("hasAuthority('user:add')")
    @PostMapping("/add")
    public void add(@RequestBody User user){
        userService.saveUserDetails(user);
    }
    
}
```

## 总结：WebSecurityConfig

```java
@Configuration
@EnableMethodSecurity
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(auth -> auth
                        // .requestMatchers("/user/list").hasAuthority("user:list")
                        // .requestMatchers("/user/add").hasAuthority("user:add")
                        // .requestMatchers("/user/**").hasRole("ADMIN")

                        .anyRequest()      // 对所有请求开启授权保护
                        .authenticated()   // 已认证的请求会被自动授权
                );
        http
                .formLogin(form -> {
                    form.loginPage("/user/login").permitAll()
                            // 认证成功后处理
                            .successHandler(new MyAuthenticationSuccessHandler())
                            // 认证失败处理
                            .failureHandler(new MyAuthenticationFailureHandler())
                    ;
                });
        // 注销登出后处理
        http.logout(logout -> logout.logoutSuccessHandler(new MyLogoutSuccessHandler()));
        http.exceptionHandling(
                exception -> {
                    exception.authenticationEntryPoint(new MyAuthenticationEntryPoint());// 未认证返回登录页时处理
                    exception.accessDeniedHandler(new MyAccessDeniedHandler());//用户访问无权限时处理
                }
        );
        // 会话失效处理，同一个账号，最多有几个客户端登录
        http.sessionManagement(session -> {
            session.maximumSessions(1).expiredSessionStrategy(new OnEpiredSessionDetected());
        });
  
           //禁用浏览器的同源策略限制
        http
                .headers(headers->headers
                         .frameOptions(HeadersConfigurer.FrameOptionsConfig::disable));
        // 全局跨域
        http.cors(Customizer.withDefaults());
        // 禁用csrf防御
        http.csrf(AbstractHttpConfigurer::disable);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```



# 第六章、整合JWT结合实战

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061740226.png" alt="image-20240501231529699" style="zoom:67%;" />

## 0、访问请求统一添加前缀

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    //为所有 RestController 添加前缀
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.addPathPrefix("/api", HandlerTypePredicate.forAnnotation(RestController.class));
    }
}
```

> 在shopping项目整合springsecurity结合JwtToken认证

引入依赖spring-security框架

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

## 1、SysUser实现

> 由于loadUserByUsername方法需要返回UserDetail类型，所以通过SysUser实现UserDetails接口，便于整合
>
> 增加authorities字段，为其设置权限

```java
@Data
@TableName("sys_user")
public class SysUser implements UserDetails {
    @Serial
    private static final long serialVersionUID = 9215887261717144118L;
    @TableId(type = IdType.AUTO)
    private Long userId;
    private String username;
    private String password;
    private String phone;
    private String email;
    private String sex;
    private String isAdmin;
    //不属于用户表，需要排除
    @TableField(exist = false)
    private String roleId;
    //账户过期 1 未过期 0 过期
    private boolean isAccountNonExpired=true;
    //账户锁定 1 未锁定 0 锁定
    private boolean isAccountNonLocked=true;
    //密码枸杞 1 未过期 0 过期
    private boolean isCredentialsNonExpired=true;
    //账户可用 1 可用  0 喊出用户
    private boolean isEnabled=true;
    private String nickName;
    private Date createTime;
    private Date updateTime;

    //权限字段集合
    @TableField(exist = false)
    Collection<? extends GrantedAuthority> authorities;
}
```

## 2、DBUserDetailsService实现

创建DBDetailsService.java主要实现loadUserByUsername方法

> loadUserByUsername调用sysUserService的loadUser方法(根据username查询SysUser用户)
>
> 判断该用户是否为超级管理员并分配其对应的menuList菜单
>
> 映射出menuList的code权限字段转化为数组再转化格式之后，注入user的authorities字段

```java
@Slf4j
@Component
public class DBDetailsService implements UserDetailsService {
    @Resource
    private SysUserService sysUserService;
    @Resource
    private SysMenuService sysMenuService;

    // 重写使security知道用户从哪里来
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        SysUser user = sysUserService.loadUser(username);
        if (Objects.isNull(user)) {
          	//自定义异常
            throw new CustomerAuthenionException("用户名错误或账户不存在!");
        }
        // 查询菜单权限
        List<SysMenu> menuList = null;
        // 判断是否为超级管理员
        if (StringUtils.isNotEmpty(user.getIsAdmin()) && "1".equals((user.getIsAdmin()))) {
            // 是就全部查询
            menuList = sysMenuService.list();
        } else {
            menuList = sysMenuService.getMenuByUserId(user.getUserId());
        }

        List<String> list = Optional.ofNullable(menuList)
                .orElseGet(new ArrayList<>())
                .stream()
                .filter(item -> item != null && StringUtils.isNotEmpty(item.getCode()))
                .map(SysMenu::getCode)
                .toList();
        //将权限字段转为数组并存入权限Authorities中管理
        String[] array = list.toArray(new String[list.size()]);
        List<GrantedAuthority> authorityList = AuthorityUtils.createAuthorityList(array);
        user.setAuthorities(authorityList);
        // SysUser 已经继承了UserDetails接口
        return user;
    }
}
```

## 3、JWT工具配置

> 使用到的JwtUtils工具，用于登录时，生成token返回给前端以及后端token过滤器的校验
>
> 在application.yml中设置**jwt的参数**秘钥，过期时间等等
>
> `jwt:`
>   `issuer: zyr`
>   `secret: zyr030426`
>   `expiration: 86400`

```xml
<!--jwt依赖-->
<dependency>
  <groupId>com.auth0</groupId>
  <artifactId>java-jwt</artifactId>
  <version>4.4.0</version>
</dependency>
<!--另一个jwt库-->
<!--<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt</artifactId>
  <version>0.9.1</version>
</dependency>-->
```

```java
import com.auth0.jwt.JWT;
@Component
@Data
//获取yml配置文件中jwt前缀的配置
@ConfigurationProperties(prefix = "jwt")
public class JwtUtils {
    //颁发者
    private String issuer;
    //密钥
    private String secret;
    //过期时间，定义为86400s
    private Integer expiration;

    /**
     * 生成JWT令牌。
     *
     * @param map 用于设置JWT令牌中的claim的信息，键值对形式。
     * @return 生成的JWT令牌字符串。
     */
    public String generateToken(Map<String,String> map){
        // 获取当前时间实例，并设置令牌过期时间
        Calendar instance = Calendar.getInstance();
        // 设置令牌失效时间，基于当前时间往后推算expiration秒
        instance.add(Calendar.SECOND,expiration);
        // 创建JWT构建器
        JWTCreator.Builder builder = JWT.create();
        // 设置payload，将map中的所有键值对加入到令牌的claim中
        map.forEach(builder::withClaim);
        String token;
        // 设置令牌的发行者、过期时间，并使用秘钥签名生成最终令牌字符串
        token = builder.withIssuer(issuer).withExpiresAt(instance.getTime())
                .sign(Algorithm.HMAC256(secret));

        return token;
    }

    /**
     * 验证JWT(token)的有效性。
     *
     * @param token 待验证的JWT令牌。
     * @return 如果令牌验证成功，则返回true；如果验证失败，则返回false。
     */
    public boolean verify(String token){
        try{
            // 使用JWT的要求构建验证器，并基于提供的密钥进行验证
            JWT.require(Algorithm.HMAC256(secret)).build().verify(token);
        }catch(JWTVerificationException e){
            // 如果捕获到JWT验证异常，则表示验证失败，返回false
            return false;
        }
        // 无异常捕获，表示验证通过，返回true
        return true;
    }

    /**
     * 解码验证JWT token。
     *
     * @param token 待解码的JWT字符串。
     * @return 解码后的DecodedJWT对象，包含JWT的详细信息。
     * @throws RuntimeException 当遇到签名验证失败、算法不匹配、token过期或token无效的情况时抛出。
     */
    public DecodedJWT jwtDecode(String token){
        try{
            return JWT.require(Algorithm.HMAC256(secret)).build().verify(token);
        }catch(SignatureVerificationException e){
            throw new RuntimeException("token签名验证失败");
        }catch(AlgorithmMismatchException e){
            throw new RuntimeException("token算法不一致");
        }catch(TokenExpiredException e){
            throw new RuntimeException("token过期");
        }catch(Exception e){
            throw new RuntimeException("token无效 ");
        }
    }

}
```

## 4、Token过滤器链

> 创建token验证过滤器CheckTokenFilter，在appliaction.yml配置了token验证url白名单 
>
> `ignore:url: /api/sysUser/login,/api/sysUser/getImage`
>
> 根据token解析，并生成令牌细节添加到安全上下文中

```java
@Data
@Slf4j
@Component
@EqualsAndHashCode(callSuper = true)
public class CheckTokenFilter extends OncePerRequestFilter {
    @Resource
    private JwtUtils jwtUtils;
    @Resource
    private DBUserDetailsService dbUserDetailsService;
    // @Resource
     private LoginFailureHandler loginFailureHandler;
    //token验证url白名单
    @Value("#{'${ignore.url}'.split(',')}")
    public List<String> ignoreUrl = Collections.emptyList();

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            // 获取请求的url
            String url = request.getRequestURI();
            // 判断是否需要验证，如果不在白名单内
            if (!ignoreUrl.contains(url)) {
                validateToken(request);
            }
        } catch (AuthenticationException e) {
           	// 登录失败处理
             loginFailureHandler.commence(request,response,e);
             return ;
        }
        filterChain.doFilter(request, response);
    }

    /**
     * 验证令牌（token）的有效性，并基于该token进行用户认证和设置安全上下文。
     *
     * @param request HttpServletRequest对象，用于从中提取token。
     */
    protected void validateToken(HttpServletRequest request) {
        // 尝试从请求头获取token，如果不存在则尝试从请求参数获取
        String token = request.getHeader("token");
        if (StringUtils.isEmpty(token)) {
            token = request.getParameter("token");
        }
        // 记录token为空的情况
        if (StringUtils.isEmpty(token)) {
            log.info("token为空");
        }
        // 验证token的合法性
        if (!jwtUtils.verify(token)) {
            log.info("非法的token");
        }
        // 解析token以获取用户信息
        DecodedJWT decodedJWT = jwtUtils.jwtDecode(token);
        Map<String, Claim> claims = decodedJWT.getClaims();
        // 通过username获取用户详情
        String username = claims.get("username").asString();
        UserDetails userDetails = dbUserDetailsService.loadUserByUsername(username);
        // 创建认证令牌并设置细节
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
        authenticationToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
        // 将认证令牌设置到安全上下文中
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);
    }
}
```

## 5、配置WebSecurityConfig

> 配置验证请求过滤器链
>
> 添加token过滤器
>
> 开启跨域，禁用csrf防护，禁用headers同源策略，关闭会话策略
>
> 添加密码解析器，添加登录认证管理器
>
> 配置未认证返回登录页处理、未认证访问处理

```java
@Slf4j
@Configuration
@EnableMethodSecurity
public class WebSecurityConfig {
    @Resource
    private CheckTokenFilter checkTokenFilter;
    // 过滤器链
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(
                                "/api/sysUser/login",
                                "/api/sysUser/getImage"
                        ).permitAll()
                        .anyRequest()
                        .authenticated()
                );
        http
                .exceptionHandling(e->{
                    //未认证返回登录页处理
                    e.authenticationEntryPoint(new LoginFailureHandler());
                    //无权限访问处理
                    e.accessDeniedHandler(new CustomerAccessDeineHandler());
                });
        // token过滤器
        http.addFilterBefore(checkTokenFilter, UsernamePasswordAuthenticationFilter.class);
        // 禁用浏览器的同源策略限制
        http.headers(headers -> headers.frameOptions(HeadersConfigurer.FrameOptionsConfig::disable));
        // 关闭全局跨域,已弃用
        //http.cors().disabled();
       	http.cors(AbstractHttpConfigurer::disable);
        // 基于token，不需要 csrf
        http.csrf(AbstractHttpConfigurer::disable);
        // 基于token，不需要 session
        http.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }

    // 获取AuthenticationManager（认证管理器），登录时认证使用
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }

    // 密码解析器
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

}
```

### 5.1、请求未认证返回登录页处理

```java
@Component
public class LoginFailureHandler implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException e) throws IOException, ServletException {
        int code = 500;
        String str= "";
        if (e instanceof AccountExpiredException) {
            str = "账户过期，登录失败";
        } else if (e instanceof BadCredentialsException) {
            str = "用户名或密码错误，登录失败";
        } else if (e instanceof CredentialsExpiredException) {
            str = "密码过期，登录失败";
        } else if (e instanceof DisabledException) {
            str = "账户被禁用，登录失败";
        } else if (e instanceof LockedException) {
            str = "账户锁定，登录失败";
        }else if (e instanceof InternalAuthenticationServiceException) {
            str = "用户名错误或不存在，登录失败";
        }else if (e instanceof CustomerAuthenionException) {
            code=600;
            str = e.getMessage();
        }else if (e instanceof InsufficientAuthenticationException) {
            str = "无权限访问资源";
        } else {
            str = "登录失败!";
        }
        String res = JSONObject.toJSONString(new Result(code,str,null), SerializerFeature.DisableCircularReferenceDetect);
        response.setContentType("application/json;charset=utf-8");
        ServletOutputStream out = response.getOutputStream();
        out.write(res.getBytes(StandardCharsets.UTF_8));
        out.flush();
        out.close();
    }
}

```

### 5.1、请求无权限访问处理

```java
@Component
public class CustomerAccessDeineHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        String res = JSONObject.toJSONString(new Result(700,"无权限访问，请联系管理员",null), SerializerFeature.DisableCircularReferenceDetect);
        response.setContentType("application/json;charset=utf-8");
        ServletOutputStream out = response.getOutputStream();
        out.write(res.getBytes(StandardCharsets.UTF_8));
        out.flush();
        out.close();
    }
}
```

### 5.3、自定义认证异常

```java
public class CustomerAuthenionException extends AuthenticationException {
    @Serial
    private static final long serialVersionUID = -2860772942229373595L;

    public CustomerAuthenionException(String msg) {
        super(msg);
    }
}
```

## 6、UserController方法

### 6.1、Login

```java
    /**
     * 用户登录接口
     * @param param 包含登录所需信息的参数对象，如用户名、密码和验证码
     * @return 返回登录结果，成功则包含用户信息和token，失败则返回错误信息
     */
    @PostMapping("/login")
    public Result login(@RequestBody LoginParam param) {
        // 获取前端传来的验证码
        String Ucode = param.getCode();
        // 从Redis中获取验证码的文本
        String capText = (String) redisTemplate.boundValueOps("capText").get();
        if (StringUtils.isEmpty(capText)) {
            // 验证码不存在，已过期
            return Result.error("验证码过期");
        }
        if (capText.equalsIgnoreCase(Ucode)) {
            // 验证码正确，交给security开始认证过程
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(param.getUsername(), param.getPassword());
            Authentication authenticate = authenticationManager.authenticate(authenticationToken);
            // // 设置认证信息
            SecurityContextHolder.getContext().setAuthentication(authenticate);
            // 获取认证后的用户信息
            SysUser user = (SysUser) authenticate.getPrincipal();

            // SysUser user = sysUserService.loadUser(param.getUsername());
            // 准备登录成功后的用户信息和token
            LoginVo vo = new LoginVo();
            vo.setUserId(user.getUserId());
            vo.setNickName(user.getNickName());
            // 生成并设置token
            Map<String, String> map = new HashMap<>();
            map.put("userId", Long.toString(user.getUserId()));
            map.put("username", user.getUsername());
            String token = jwtUtils.generateToken(map);
            vo.setToken(token);
            return Result.success("登录成功", vo);
        } else {
            // 验证码错误
            return Result.error("验证码错误");
        }
    }
```

### 6.2、getImage

```java
/**
     * 通过GET请求获取一个基于Base64编码的图形验证码。
     *
     * @param response HttpServletResponse对象，用于向客户端发送响应。
     * @return 返回一个表示操作结果的对象，包含成功信息。
     * @throws IOException 如果在输出过程中发生IO异常。
     */
    @GetMapping("/getImage")
    public void imageCodeBase64(HttpServletResponse response) throws IOException {
        // 设置响应类型为图片格式，将验证码图片输出到浏览器
        response.setContentType("image/jpeg");
        response.setHeader("Pragma", "No-cache");

        // 创建一个图形验证码，指定其长度、宽度、字符数和干扰线宽度
        ShearCaptcha captcha = CaptchaUtil.createShearCaptcha(250, 150, 4, 4);
        RandomGenerator randomGenerator = new RandomGenerator("0123456789", 4);
        captcha.setGenerator(randomGenerator);

        String capText = captcha.getCode();

        // redis设置 60s key 过期
        redisTemplate.opsForValue().set("capText", capText, 60, TimeUnit.SECONDS);
        log.info("验证码：{}", capText);

        captcha.write(response.getOutputStream());
        // response.getOutputStream().close();
    }
```

### 6.3 add

> 由于WebSecurityConfig启用了@EnableMethodSecurity方法安全注解
>
> 所以可以在一些按钮上，例如添加，编辑，删除等上添加
>
> `@PreAuthorize`, @PostAuthorize, @PreFilter, 和 @PostFilter 等注解来做安全限制

```java
    // 新增
    @PostMapping
    @PreAuthorize("hasAuthority('sys:user:add')")//判断有无权限sys:user:add
    public Result add(@RequestBody SysUser sysUser) {
        sysUser.setCreateTime(new Date());
        sysUser.setPassword(passwordEncoder.encode(sysUser.getPassword()));
        sysUserService.saveUser(sysUser);
        return Result.success("新增成功");
    }
```





