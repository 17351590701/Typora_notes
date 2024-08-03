==适用于多查询，少修改的场景==

1. 引入data-redis依赖，包含了注解式缓存依赖

   ```xml
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
   </dependency>
   ```

   

2. **启动类**(配置类)开启注解式缓存 （默认使用缓存中间件 redis ）

   ```java
   @EnableCache
   ```

3. 在需要添加缓存的**类**上添加缓存**命名空间**，保证唯一性，通常使用 类的全限定名

   ```java
   @Service
   @CacheConfig(cacheName = "com.zyr.service.impl.UserServiceImpl")
   public class UserServiceImpl extends ServiceImpl<UserMapper,User> implements UserService {
      
   }
   ```

4. 在**方法**上添加注解

   ```java
   @Override
   @Cahceable(key = "#longUserId")
   public Set<SysMenu> queryUserMenuListByUserId(Long loginUserId){
      // 根据用户标识查询菜单权限集合
      Set<SysMenu> menus = sysMenuMapper.selectUserListByUserId(loginUserId);
   	// 将菜单权限集合转化为树结构，一级菜单的父节点为0
      return transformTree(menus,0L);
   }
   ```

   > @**Cacheable** (key = "'user:name'")：添加缓存
   >
   > @**CacheEvict**(key = "'user:name'")：删除缓存
   >
   > @**Caching**(evict={
   >
   > ​	@CacheEvict(key = "...")
   >
   > ​	@CacheEvict(key = "...")
   >
   > })：批量删除缓存

5. `CacheConfig`配置类，设置序列化，过期时间（**不设置，默认TTL为 2 minutes**）

   ```java
   @Configuration
   @EnableCaching
   public class CacheConfig {
   
       @Resource
       private RedisConnectionFactory redisConnectionFactory;
   
       @Bean
       public CacheManager cacheManager() {
           // 设置键值序列化方式
           RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                  // .entryTtl(Duration.ofSeconds(-1)) 过期时间
                   .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                   .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()));
   
           return RedisCacheManager.builder(redisConnectionFactory)
                                   .cacheDefaults(config)
                                   .build();
       }
   }
   ```