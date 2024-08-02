UserServiceImpl.class--->推断构造方法--->普通对象--->依赖注入(属性填充)--->初始化(afterPropertiesSet)--->初始化后（AOP）--->(代理对象)--->放入Map<beanName,Bean对象>

Bean实例化-->依赖注入-->Bean初始化



==AOP逻辑==

> 通过CGLIB利用继承实现代理对象
>
> 代理对象执行test（）方法，实际上是**调用**之前已经进行**依赖注入后的普通对象**test（）方法

```java
class UserServiceProxy extends UserService{
  //普通对象
	UserService target;
  
  public void test(){
    //切面逻辑...
    target.test();//UserService普通对象.test(),打印orderService属性
  }
}
```



==事务及传播机制==

> `@Transactional` 注解，会生成代理对象
> 利用事务管理器新建一个数据库连接conn
> conn.autocommit=false
>
> 通过ThreadLocal获取关闭自动提交的conn.  ThreadLocal<Map<DataSource对象，conn>>
>
> 异常就conn.rollback() , conn.commit()

```java
class UserServiceProxy extends UserService{
  //普通对象
	UserService target;
  
  public void test(){
    //Spring事务切面逻辑...

    target.test();//UserService普通对象.test(), 执行sql 金钱交易
  }
}
```

`UserService.java`

```java
@Transactional
public void test(){
  //autocommit=true
  jdbcTemplate.execute("insert into t1 values(1,1,1,1,'1')");
  a();
}
```

==事务失效==

> 实际是AOP中的普通对象执行了a()方法,而非代理对象有额外的切面逻辑

```java
//不在任何事务上下文中运行
@Transactional(propagation = Propagation.NEVER)
public void a(){
  jdbcTemplate.execute("insert into t1 values(2,2,2,2,'2')")
}
```

解决：

```java
@Autowired
private UserService userService;
@Transactional
public void test(){
  jdbcTemplate.execute("insert into t1 values(1,1,1,1,'1')");
  userService.a();//通过代理对象调用
}
```

