MPConfig配置类

```java
@Configuration
public class MPConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
      	//添加分页拦截器
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        return interceptor;
    }
}
```

Test测试

> getRecords():当前页总数据集合
>
> getCurrent():当前页
>
> getSize(): 页大小
>
> getPages():总页数
>
> getTotal():总数据量

```java
//当前页: 1，页大小: 10
IPage<User> page = new Page<>(1, 10);
//userMapper.selectPage(page,null);
IPage list = userService.page(page);
//当前页所有数据
list.getRecords().forEach(System.out::println);

```



```java
String username = "yurun";
LambdaQueryWrapper<User> query = new LambdaQueryWrapper<>();
//like前后模糊 %yurun%
//eq = ,lt <, gt >, le< =, ge >=, ne!=
query.like(User::getUsername, username);
User one = userService.getOne(query);
System.out.println(one);
```

