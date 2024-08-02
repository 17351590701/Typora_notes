SpringDataRedis依赖

```xml
<!--redis依赖-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```



通过application.yml配置数据源

```yaml
spring:   
		data:
   	 redis:
     	host: localhost
      port: 6379
      database: 0
```



Redis配置类

```java
@Configuration
@Slf4j
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        // 设置redis连接工厂对象
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        // 设置redis key的序列化
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        return redisTemplate;
    }
}
```



操作字符串数据

```java
// 设置字符串值
redisTemplate.opsForValue().set("name","Tom");
redisTemplate.opsForValue().set("name","Jim");
String name = (String) redisTemplate.opsForValue().get("name");
System.out.println(name); //Jim
// 设置过期时间
redisTemplate.opsForValue().set("code",1234,30, TimeUnit.SECONDS);
//ifAbsent 不会覆盖相同key数据
redisTemplate.opsForValue().setIfAbsent("code",4321,30, TimeUnit.SECONDS);
```



操作hash数据

```java
// key hashKey value 
HashOperations<String, Object, Object> hashOperations = redisTemplate.opsForHash();
hashOperations.put("user","name","Tom");
hashOperations.put("user","age","18");
String name = (String) hashOperations.get("user", "name");
System.out.println(name);
String age = (String) hashOperations.get("user", "age");
System.out.println(age);
// 获取所有key [name,age]
Set<Object> keys = hashOperations.keys("user");
System.out.println(keys);
// 获取所以values [Tom,18]
List<Object> values = hashOperations.values("user");
System.out.println(values);
```

