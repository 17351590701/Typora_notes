# 1、基础篇

## 1.1、基本使用

引入依赖

```xml
<!--data-redis-->  
   <dependency>
   	<groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
   </dependency>
<!--对象池-->
   <dependency>
   	<groupId>org.apache.commons</groupId>
   	<artifactId>commons-pool2</artifactId>
   	<version>2.12.0</version>
   </dependency>
```

application.yml配置

```yaml
spring:  
	data:
    redis:
      host: localhost
      port: 6379
      lettuce:
       pool:
        max-active: 8 	 # 最大连接数，负值表示没有限制，默认8
        max-idle: 8 		 # 最大空闲连接，默认8
        min-idle: 0  	 # 最小空闲连接，默认0
        max-wait: 100ms  # 最大阻塞等待时间，负值表示没限制，默认-1

```



基础操作，新增查询

> 默认的RedisTemplate 采用JDK序列化方式存储key，value
>
> 缺点：可读性差，内存占用大

```java
    @Resource
    private RedisTemplate redisTemplate;

    @Test
    void contextLoads() {
        ValueOperations ops = redisTemplate.opsForValue();
        ops.set("name","张雨润");
        log.info(ops.get("name"));
    }
```

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737502.png" alt="image-20240519180020370" style="zoom: 80%;" />

## 1.2、RedisTemplate序列化

### 方式一：

配置序列化方式：`RedisConfig` ，修改序列化方式 `GenericJackson2JsonRedisSerializer`

```java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        // 创建RedisTemplate对象
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        // 设置redis连接工厂对象
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        // 设置key的序列化--字符串
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        // 创建JSON序列化工具
        GenericJackson2JsonRedisSerializer jsonRedisSerializer = new GenericJackson2JsonRedisSerializer();
        // 设置value序列化--json处理
        redisTemplate.setValueSerializer(jsonRedisSerializer);
        return redisTemplate;
    }
}
```

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737503.png" alt="image-20240519180730201" style="zoom: 80%;" />



> 当用redis**存储类**时，json化器会将类的class类型也写入value中，会带来**额外的内存开销**
>

```java
    @Test
    void contextLoads() {
        ValueOperations<String, Object> ops = redisTemplate.opsForValue();
        ops.set("user",new User("张雨润",21));
        log.info(String.valueOf(ops.get("user")));
    }
```

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737504.png" alt="image-20240519182115191" style="zoom:80%;" />

> 为了节省内存空间，并不会使用JSON序列化器来处理value，而是统一通过String序列化器，要求只能存储String类型的key，value。当需要存储Java对象时，手动完成对象的序列化和反序列化。

### 方式二：

使用`StringRedisTemplate` ，默认序列化方式是String ，**操作对象时**，需要手动序列化、反序列化

```java
    @Resource
    private StringRedisTemplate stringRedisTemplate;
    private static final ObjectMapper mapper = new ObjectMapper();
   @Test
    void StringRedisTest() throws JsonProcessingException {
        User user = new User("张雨润", 21);
        //手动将对象转为json
        String json = mapper.writeValueAsString(user);
        stringRedisTemplate.opsForValue().set("user",json);
			//获取将json对象转化为对象
        String jsonUser = stringRedisTemplate.opsForValue().get("user");
        User user1 = mapper.readValue(jsonUser, User.class);
        System.out.println(user1);
    }
```

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737505.png" alt="image-20240519203341177" style="zoom: 80%;" />

#### StringTemplate操作hash

```java
    @Test
    void HashTest(){
        HashOperations<String, Object, Object> ops = stringRedisTemplate.opsForHash();
        ops.put("girl","name","beautiful");
        //需要String类型，不能是Integer
        ops.put("girl","age","18");
        System.out.println(ops.entries("girl"));
    }
```





# 2、实战篇

> 基于B站Redis教程，以 黑马点评 项目为原型

## 2.1、基于Redis实现登录

> ==session共享问题==:多台Tomcat并不共享session存储空间，当请求切换到不同Tomcat服务器时导致数据丢失问题
>
> 使用session，保存验证码，以及用户信息时，在多Tomcat服务器情况下，会出现session不共享的问题，导致刚登陆访问后又说没token等问题
>
> ，而redis 可以实现 ==数据共享==，==内存存储==（读写快） ==key,value结构== 等优点

设计思路：

redis存储验证码，以phone作为key，唯一，且方便通过phone查找value

redis存储用户信息，通过UUID生成token，作为key，唯一，且返回token给前端，方便使用

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737506.png" alt="image-20240520193056442" style="zoom:67%;" />

### 2.1.1、实现验证码发送

`UserServiceImpl`

```java
    @Override
    public Result sendCode(String phone, HttpSession session) {
       //自定义正则校验工具
        if (RegexUtils.isPhoneInvalid(phone)) {
            return Result.fail("手机号格式错误");
        }
        String code = RandomUtil.randomNumbers(6);
        stringRedisTemplate.opsForValue().set(LOGIN_CODE_KEY + phone, code, LOGIN_CODE_TTL, TimeUnit.MINUTES);
       //模拟发送验证码
        log.debug("发送验证码成功，验证码为：{}", code);
        return Result.ok();
    }
```



### 2.1.2、实现登录功能

`UserServiceImpl`

```java
    @Override
    public Result login(LoginFormDTO loginForm, HttpSession session) {
        String phone = loginForm.getPhone();
        if (RegexUtils.isPhoneInvalid(phone)) {
            return Result.fail("手机号格式错误");
        }
        // 获取存储redis的生成的验证码
        String cacheCode = stringRedisTemplate.opsForValue().get(LOGIN_CODE_KEY + phone);
        String code = loginForm.getCode();
        if(cacheCode==null||!cacheCode.equals(code)){
            return Result.fail("验证码错误");
        }
        User user = query().eq("phone", phone).one();
        // 如果没有用户，就注册创建用户
        if(user==null){
             user = createUserWithPhone(phone);
        }
        // hutool工具生成UUID作为token凭证
        String token = UUID.randomUUID().toString(true);
        // hutool工具，复制类属性，过滤用户敏感信息
        UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
        // 将userDTO转成map，且将值类型转换成String,以符合StringRedisTemplate
        Map<String, Object> userMap = BeanUtil.beanToMap(userDTO, new HashMap<>(),
                CopyOptions.create()
                        .setIgnoreNullValue(true)
                        .setFieldValueEditor((fileName, fileValue) -> fileValue.toString())

        );
        String tokenKey = LOGIN_USER_KEY+token;
      	//将token 和 部分用户信息存储在redis
        stringRedisTemplate.opsForHash().putAll(tokenKey, userMap);
        // hash值额外设置过期时间
        stringRedisTemplate.expire(tokenKey,LOGIN_USER_TTL,TimeUnit.MINUTES);
        // 返回token给前端
        return Result.ok(token);
    }
```

### 2.1.3、拦截器配置

#### 2.1.3.1、拦截器配置类

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 登录拦截器
        registry.addInterceptor(loginInterceptor())
                .excludePathPatterns(
                        "/user/code",
                        "/user/login"
                )
                .order(1);
        // order(0)，保证刷新拦截器，先执行
        registry.addInterceptor(refreshInterceptor())
                .addPathPatterns("/**")
                .order(0);
    }

    @Bean
    public LoginInterceptor loginInterceptor() {
        return new LoginInterceptor();
    }
    @Bean
    public RefreshInterceptor refreshInterceptor() {
        return new RefreshInterceptor();
    }
}
```



#### 2.1.3.2、刷新拦截器

> 主要负责用户访问接口时，刷新redis用户信息存储时间。
>
> 同时获取token，以校验redis中存储用户信息

```java
public class RefreshInterceptor implements HandlerInterceptor {
    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String token = request.getHeader("authorization");
        if (StringUtils.isBlank(token)) {
            return true;
        }
        String key = LOGIN_USER_KEY + token;
      	// 根据key获取redis中用户信息
        Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(key);
        if (userMap.isEmpty()) {
            return true;
        }
      	// 转化为 部分用户信息
        UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
      	// 将 部分用户 存储在本线程
        UserHolder.saveUser(userDTO);
				// 刷新 TTL
        stringRedisTemplate.expire(key,LOGIN_USER_TTL, TimeUnit.MINUTES);
        return true;
    }
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        UserHolder.removeUser();
    }
}
```

#### 2.1.3.3、登录拦截器

> 校验局部线程内是否存储用户

```java
public class LoginInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //    判断是否需要拦截
        if (UserHolder.getUser() == null) {
            response.setStatus(401);
            return false;
        }
        // 有用户就放行
        return true;
    }
}
```





## 2.2、商户查询缓存

### 双写一致性

> 缓存：就是数据交换的缓冲区（Cache）,是存储数据的临时地方，一般读写性能较高
>
> 成本：数据一致性成本（和数据库中一致），代码维护成本，运维成本等等

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737507.png" alt="image-20240520201837831" style="zoom: 67%;" />

#### 添加商铺到缓存（Version 0）

> StringRedisTemplate存储String类型，所以存储shop对象时，需要转化为JSON类型，返回时再转化成Bean

```java
@Service
public class ShopServiceImpl extends ServiceImpl<ShopMapper, Shop> implements IShopService {
    @Resource
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public Result queryById(Long id) {
        // 从redis查询店铺信息
        String shopJson = stringRedisTemplate.opsForValue().get(CACHE_SHOP_KEY+id);
        // redis中存在 json数据转化
        if (StringUtils.isNotBlank(shopJson)) {
            Shop shop = JSONUtil.toBean(shopJson, Shop.class);
            return Result.ok(shop);
        }
        // redis不存在，从数据库查询
        Shop shop = getById(id);
        if (shop == null) {
            return Result.fail("店铺不存在");
        }
        // 缓存到redis
        stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY+id, JSONUtil.toJsonStr(shop));
        return Result.ok(shop);
    }
}
```

#### 缓存更新策略（补充）

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737508.png" alt="image-20240520210621758" style="zoom:50%;float:left" />

==案例实现：缓存，数据库双写一致性==

> 给查询商铺的缓存添加超时剔除和主动更新策略
>
> ShopController业务逻辑需求：
>
> ① 根据id查询店铺时，如果缓存未命中，则查询数据库，将数据库结果写入缓存，并设置**过期时间**
>
> ② 根据id修改店铺时，先**修改**数据库，再**删除**缓存

`ShopServiceImpl`

查询店铺 （Version 1 ==过期时间==）

```java
    @Override
    public Result queryById(Long id) {
        String key = CACHE_SHOP_KEY + id;
        // 从redis查询店铺信息
        String shopJson = stringRedisTemplate.opsForValue().get(key);
        // redis中存在，类型转换 JSON-->Bean
        if (StringUtils.isNotBlank(shopJson)) {
            Shop shop = JSONUtil.toBean(shopJson, Shop.class);
            // 返回
            return Result.ok(shop);
        }
        // redis不存在，从数据库查询
        Shop shop = getById(id);
        if (shop == null) {
            return Result.fail("店铺不存在");
        }
        // 缓存到redis,Bean-->JSON
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);
        return Result.ok(shop);
    }
```

修改店铺 （Version 1 `@Transactional`）

```java
    @Override
    @Transactional//如果删除缓存抛异常，更新需要回滚
    public Result updateShop(Shop shop) {
        Long id = shop.getId();
        if(id==null){
            return Result.fail("店铺id不能为空");
        }
        // 更新到数据库
        updateById(shop);
        // 删除缓存
        stringRedisTemplate.delete(CACHE_SHOP_KEY+id);
        return Result.ok();
    }
```

### 缓存穿透

> 指客户端请求的数据在缓存中和数据库中都不存在，这样缓存永远不会生效，这些请求都会打到数据库，影响效率
>

#### 两种解决方案

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737509.png" alt="image-20240523123231695" style="zoom:80%;" />

#### 缓存空对象

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737510.png" alt="image-20240523123943460" style="zoom:80%;" />

查询数据（Version 2）`ShopServiceImpl`

```java
@Override
    public Result queryById(Long id) {
        String key = CACHE_SHOP_KEY + id;
        // 从redis查询店铺信息
        String shopJson = stringRedisTemplate.opsForValue().get(key);
        // redis中存在，类型转换 JSON-->Bean ,shopJson有确切值
        if (StringUtils.isNotBlank(shopJson)) {
            Shop shop = JSONUtil.toBean(shopJson, Shop.class);
            // 返回
            return Result.ok(shop);
        }
        //shopJson为“”（自定义缓存空值）
        if(shopJson!=null){
            return Result.fail("店铺不存在");
        }
        // redis不存在，从数据库查询
        Shop shop = getById(id);
        if (shop == null) {
            // 缓存空对象
            stringRedisTemplate.opsForValue().set(key,"",CACHE_NULL_TTL,TimeUnit.MINUTES);
            return Result.fail("店铺不存在");
        }
        // 缓存到redis,Bean-->JSON
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);
        return Result.ok(shop);
    }
```

### 缓存雪崩

> 指同一时间段大量的缓存Key同时失效或者Redis服务宕机，导致大量请求到达数据库，带来巨大压力

#### 解决方案

- 给不同的Key的TTL添加随机值
- 利用Redis集群提高服务的可用性
- 给缓存业务添加降级限流策略
- 给业务添加多级缓存



### 缓存击穿

> 也叫热点Key问题，就是一个被**高并发访问**并且**缓存重建业务时间较长**的Key突然失效了，无数的请求访问会在瞬间给数据库带来巨大的冲击

#### 两种解决方案

- 互斥锁
- 逻辑过期

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737512.png" alt="image-20240523130510531" style="zoom:80%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737513.png" alt="image-20240523130558190" style="zoom:80%;" />

#### 互斥锁

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737514.png" alt="image-20240523130852963" style="zoom: 80%;" />

查询数据（Version 3）`ShopSeviceImpl.java`

```java
    @Override
    public Result queryById(Long id) {
      	//互斥锁，缓存重建
        Shop shop = queryWithMutex(id);
        if (shop==null){
            return Result.fail("店铺不存在");
        }
        return Result.ok(shop);
    }
```

`queryWithMutex` 方法

```java
private Shop queryWithMutex(Long id) {
        String key = CACHE_SHOP_KEY + id;
        // 从redis查询店铺信息
        String shopJson = stringRedisTemplate.opsForValue().get(key);
        // redis中存在，类型转换 JSON-->Bean
        if (StringUtils.isNotBlank(shopJson)) {
            // 返回
            return JSONUtil.toBean(shopJson, Shop.class);
        }
        // 判断是否命中空值
        if (shopJson != null) {
            return null;
        }
        /* 实现缓存重建*/
        // 获取互斥锁
        String lockKey = "lock:shop:" + id;
        Shop shop = null;
        try {
            boolean isLock = tryLock(lockKey);
            // 判断是否获取 锁 成功
            if (isLock) {
                // 失败，休眠并重试
                Thread.sleep(50);
                queryWithMutex(id);
            }
            // redis不存在，从数据库查询
            shop = getById(id);
            // 模拟重建延时
            Thread.sleep(200);
            if (shop == null) {
                // 缓存空对象
                stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES);
                return null;
            }
            // 缓存到redis,Bean-->JSON
            stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }finally {
        //解锁
        unLock(lockKey);
        }
        return shop;
    }

```

**上锁，解锁**

```java
    private boolean tryLock(String key) {
      	//setIfAbsent 如果不存在 就设置 ， key value time TimeUnit
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);
    }

    private void unLock(String key) {
        stringRedisTemplate.delete(key);
    }
```



#### 逻辑过期

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737515.png" alt="image-20240524130737927" style="zoom: 67%;" />

> 热点key已经==提前==封装成RedisData存入缓存,设置逻辑过期时间，而非通过Redis设置TTL的方式，理论上永久存在

`ShopServiceImpl.java`

```java
    @Override
    public Result queryById(Long id) {
        //逻辑过期查询
        Shop shop = queryWithLogicalExpire(id);
        if (shop == null) {
            return Result.fail("店铺不存在");
        }
        return Result.ok(shop);
    }
```

`queryWithLogicalExpire`方法

```java
  // 线程池
   private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

		// 逻辑过期
    public Shop queryWithLogicalExpire(Long id) {
        String key = CACHE_SHOP_KEY + id;
        // 从redis查询店铺信息
        String shopJson = stringRedisTemplate.opsForValue().get(key);
        // 判断是否存在
        if (StringUtils.isBlank(shopJson)) {
            return null;
        }
        // 命中，将JSON反序列化为对象
        RedisData redisData = JSONUtil.toBean(shopJson, RedisData.class);
        Shop shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
        LocalDateTime expireTime = redisData.getExpireTime();
        // 判断是否过期
        if (expireTime.isAfter(LocalDateTime.now())) {
            // 未过期
            return shop;
        }
        String lockKey = LOCK_SHOP_KEY + id;
        boolean isLock = tryLock(lockKey);

        if (isLock) {
            // 成功，开启独立线程，实现缓存重建
            CACHE_REBUILD_EXECUTOR.submit(() -> {
                try {
                    this.saveShopRedis(id, 30L);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                } finally {
                    // 释放锁
                    unLock(lockKey);
                }
            });
        }
        return shop;
    }
```

`saveRedisData`

```java
    public void saveShopRedis(Long id, Long expireSeconds) throws InterruptedException {
        Shop shop = getById(id);
        // 模拟缓存重建延时
        Thread.sleep(200);
        // 封装逻辑过期时间
        RedisData redisData = new RedisData();
        redisData.setData(shop);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(expireSeconds));
        // 写入redis
        stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, JSONUtil.toJsonStr(redisData));
    }
```

`RedisData`

```java
@Data
public class RedisData {
    private LocalDateTime expireTime;
    private Object data;
}
```



## 2.3、优惠券秒杀

### 全局唯一ID

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737516.png" alt="image-20240524144702772" style="zoom:67%;" />

生成策略：

- UUID
- Redis自增
  - 每天一个key，方便统计订单量
  - ID构造是 时间戳 + 计数器
- snowflake算法
- 数据库自增

`RedisIdWorker`

```java
@Component
public class RedisIdWorker {
    @Resource
    private StringRedisTemplate stringRedisTemplate;

    // 记录2024,1,1的秒数
    private static final long BEGIN_TIMESTAMP = 1704067200L;
    // 序列号位数
    private static final int COUNT_BITS= 32;

    public long nextId(String keyPrefix) {
        // 获取时间戳
        LocalDateTime ldt = LocalDateTime.now();
        long nowSecond = ldt.toEpochSecond(ZoneOffset.UTC);
        long timeStamp = nowSecond - BEGIN_TIMESTAMP;
        // 生成序列号
        String date = ldt.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        Long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
        // 将时间戳和序列号合并为一个长整型数字
        return timeStamp<<COUNT_BITS|count;
    }

}
```

`Test`

```java
    private final ExecutorService es = Executors.newFixedThreadPool(500);
      /**
     * 测试函数：通过创建多个任务并发地从Redis生成ID，以测试ID生成的性能。
     * 该测试方法不接受参数，也不返回任何结果。
     * 主要步骤包括：
     * 1. 创建一个计数器 latch，用于同步所有任务的完成。
     * 2. 定义一个任务，该任务会循环执行一定次数，每次生成一个ID并打印。
     * 3. 记录开始时间，然后提交大量任务到执行服务。
     * 4. 等待所有任务完成。
     * 5. 记录结束时间并打印执行时间。
     *
     * @throws InterruptedException 如果在等待任务完成时被中断。
     */
    @Test
    void test() throws InterruptedException {
        // 创建一个计数器，初始值为300，用于同步300个任务的完成
        CountDownLatch latch = new CountDownLatch(300);
        // 定义一个任务，该任务会循环生成100个ID并打印
        Runnable task = () -> {
            for (int i = 0; i < 100; i++) {
                long id = redisIdWorker.nextId("order"); // 生成订单ID
                System.out.println("id:" + id); // 打印生成的ID
            }
            latch.countDown(); // 任务完成，计数器减一
        };
        // 记录开始时间
        long begin = System.currentTimeMillis();
        // 提交300个相同任务到执行服务
        for (int i = 0; i < 300; i++) {
            es.submit(task);
        }
        // 等待所有任务完成
        latch.await();
        // 记录结束时间
        long end = System.currentTimeMillis();
        // 打印执行时间
        System.out.println("time="+(end-begin));
    }
```

### 实现秒杀下单



<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737517.png" alt="image-20240524190043007" style="zoom:67%;" />

`VoucherOrderServiceImpl.java`

> 秒杀优惠券，将订单id，用户id，优惠券id保存

```java
   @Resource
    private ISeckillVoucherService seckillVoucherService;
    @Resource
    private RedisIdWorker  redisIdWorker;
    @Override
    @Transactional
    public Result seckillVoucher(Long voucherId) {
        //查询优惠券
        SeckillVoucher voucher = seckillVoucherService.getById(voucherId);
        //判断秒杀是否开始
        if (voucher.getBeginTime().isAfter(LocalDateTime.now())) {
            //秒杀尚未开始
            return Result.fail("秒杀尚未开始");
        }
        //判断秒杀是否结束
        if(voucher.getEndTime().isBefore(LocalDateTime.now())){
            return Result.fail("秒杀已经结束");
        }
        if(voucher.getStock()<1){
            return Result.fail("库存不足");
        }
        // 扣减库存
        boolean success = seckillVoucherService.update()
                .setSql("stock=stock-1")
                .eq("voucher_id", voucherId)
                .update();
        if(!success){
            return Result.fail("库存不足");
        }
        // 保存 订单id 用户id 优惠券id
        VoucherOrder voucherOrder = new VoucherOrder();
        // 生成全局唯一订单ID
        long orderId = redisIdWorker.nextId("order");
        voucherOrder.setId(orderId);
        // 用户id
        Long userId = UserHolder.getUser().getId();
        voucherOrder.setUserId(userId);
        // 优惠券id
        voucherOrder.setVoucherId(voucherId);
        // 保存优惠券订单
        save(voucherOrder);
        return Result.ok(orderId);
    }
```



### 超卖问题

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737518.png" alt="image-20240524193628247" style="zoom:67%;" />

#### 常见解决方案

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737519.png" alt="image-20240524194339012" style="zoom:67%;" />



#### 乐观锁

> 判断之前查询得到的数据是否有被修改过
>
> ==缺点==：库存stock被修改了，但库存>0的情况下,能进行秒杀的用户不能进行秒杀，失败率高

##### 版本号法

> 更新时判断数据库中的版本号是否与查询时的一致

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737520.png" alt="image-20240524194915265" style="zoom:67%;" />

##### CAS法

> 更新时判断数据库中的库存是否无查询时的一致

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737521.png" alt="image-20240524195224191" style="zoom:67%;" />

只有数据库中库存与查询时一致才允许秒杀，失败率高

```java
// 扣减库存
boolean success = seckillVoucherService.update()
  .setSql("stock=stock-1")
  .eq("voucher_id", voucherId)
  // CAS法判断
  .eq("stock",voucher.getStock())
  .update();
```



==改进==：只要库存>0就允许当前用户进行秒杀

```java
// 扣减库存
boolean success = seckillVoucherService.update()
  .setSql("stock=stock-1")
  .eq("voucher_id", voucherId)
  // CAS法判断
  .gt("stock",0)
  .update();
```





### 一人一单

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737522.png" alt="image-20240524201707583" style="zoom: 67%;" />

> 先进行判断是否已经购买过，再进行下单确认

```java
        // 一人一单,用户id
        Long userId = UserHolder.getUser().getId();
        Long count = query()
                .eq("user_id", userId)
                .eq("voucher_id", voucherId)
                .count();
        if(count>0){
            return Result.fail("该用户已购买过一次");
        }
        // 扣减库存
        boolean success = seckillVoucherService.update()
                .setSql("stock=stock-1")
                .eq("voucher_id", voucherId)
                // CAS法判断
                .gt("stock",0)
                .update();
        if(!success){
            return Result.fail("库存不足");
        }
```

> 多线程并发还是会导致一人多单，继续通过悲观锁解决
>
> 将一人一单提取为事务方法,==将该方法锁住==，防止事务未提交就释放锁，导致当前用户高并发访问该方法，
>
> 通过==intern()==方法确保，当userId一样时，锁是同一个
>
> 使用AopContext获取当前==代理对象==调用事务方法，避免事务失效

```java
 Long userId = UserHolder.getUser().getId();
				//intern().用户id值一样时，锁是一样的
        synchronized(userId.toString().intern()) {
            // 获取代理对象
            IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);
```

```java
@Transactional
    public Result createVoucherOrder(Long voucherId) {
        // 一人一单,用户id
        Long userId = UserHolder.getUser().getId();
            Long count = query()
                    .eq("user_id", userId)
                    .eq("voucher_id", voucherId)
                    .count();
            if (count > 0) {
                return Result.fail("该用户已购买过一次");
            }
            // 扣减库存
            boolean success = seckillVoucherService.update()
                    .setSql("stock=stock-1")
                    .eq("voucher_id", voucherId)
                    // CAS法判断
                    .gt("stock", 0)
                    .update();
            if (!success) {
                return Result.fail("库存不足");
            }

            // 保存 订单id 用户id 优惠券id
            VoucherOrder voucherOrder = new VoucherOrder();
            // 生成全局唯一订单ID
            long orderId = redisIdWorker.nextId("order");
            voucherOrder.setId(orderId);
            voucherOrder.setUserId(userId);
            // 优惠券id
            voucherOrder.setVoucherId(voucherId);
            // 保存
            save(voucherOrder);
            return Result.ok(orderId);

    }
```

> 在非单体项目，集群的情况下，有多个JVM存在，每个JVM都有自己的锁，依然不能保证一人一单
>
> 解决：需要多个JVM使用同一个锁

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737523.png" alt="image-20240525133418362" style="zoom: 67%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737524.png" alt="image-20240525133715204" style="zoom: 67%;" />

## 2.4、分布式锁

> 满足在分布式系统或集群模式下，多进程可见，并且互斥的锁

|        |           MySQL           |          Redis           |            Zookeeper             |
| :----: | :-----------------------: | :----------------------: | :------------------------------: |
|  互斥  | 利用mysql本身的互斥锁机制 | 利用setnx这样的互斥命令  | 利用节点的唯一性和有序性实现互斥 |
| 高可用 |            好             |            好            |                好                |
| 高性能 |           一般            |            好            |               一般               |
| 安全性 |   断开连接，自动释放锁    | 利用锁超时时间，到期释放 |    临时节点，断开连接自动释放    |

### Redis分布式锁

- 获取锁：

  - 互斥：确保只有一个线程获取锁

  - 非阻塞式：尝试一次，成功返回true，失败返回false

    同时添加锁和过期时间 `SET  lock  thread  NX EX 10`  

- 释放锁：

  - 手动释放

  - 超时释放:获取锁时添加超时时间

    `DEL  key`

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737525.png" alt="image-20240525135323823" style="zoom:67%;" />

#### Version 1

`SimpleRedisLock.java`

```java
public class SimpleRedisLock implements ILock {

    private final String name;
    private final StringRedisTemplate stringRedisTemplate;
    private static final String KEY_PREFIX = "lock:";

    public SimpleRedisLock(String name, StringRedisTemplate stringRedisTemplate) {
        this.name = name;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public boolean tryLock(long timeoutSec) {
        // 获取线程标识
        long threadId = Thread.currentThread().getId();

        Boolean success = stringRedisTemplate
                .opsForValue()
                // setnx 互斥
                .setIfAbsent(KEY_PREFIX + name, threadId + "", timeoutSec, TimeUnit.SECONDS);
        // 避免拆箱 NULL 风险
        return Boolean.TRUE.equals(success);
    }

    @Override
    public void unlock() {
        stringRedisTemplate.delete(KEY_PREFIX + name);
    }
}
```

`VoucherOrderServiceImpl.java`

```java
//...判断秒杀是否开始，库存是否充足...//
				Long userId = UserHolder.getUser().getId();
        // 创建锁对象
        SimpleRedisLock lock = new SimpleRedisLock("order:" + userId, stringRedisTemplate);
        // 获取锁
        boolean isLock = lock.tryLock(1200);
        if (!isLock) {
            // 获取锁失败
            return Result.fail("不允许重复下单");
        }
        // 获取锁成功
        try {
            // 获取代理对象
            IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
          	//调用创建一人一单方法
            return proxy.createVoucherOrder(voucherId);
        } catch (IllegalStateException e) {
            throw new RuntimeException(e);
        }finally {
            lock.unlock();
        }
```



> 问题，极端情况，线程一业务阻塞超过了锁的过期时间，线程一释放的是别人的锁

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737526.png" alt="image-20240525143025304" style="zoom: 67%;" />

解决方案：判断锁标识

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737527.png" alt="image-20240525143103929" style="zoom:67%;" />

#### Version 2

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737528.png" alt="image-20240525144143705" style="zoom:67%;" />

`SimpleRedisLock.java`

```java
public class SimpleRedisLock implements ILock {
    private final String name;
    private final StringRedisTemplate stringRedisTemplate;
    private static final String KEY_PREFIX = "lock:";
    private static final String ID_PREFIX = UUID.randomUUID().toString(true)+"-";

    public SimpleRedisLock(String name, StringRedisTemplate stringRedisTemplate) {
        this.name = name;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public boolean tryLock(long timeoutSec) {
        // 获取线程标识
        String  threadId = ID_PREFIX+Thread.currentThread().getId();

        Boolean success = stringRedisTemplate
                .opsForValue()
                // setnx 互斥
                .setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
        // 避免拆箱 NULL 风险
        return Boolean.TRUE.equals(success);
    }

    @Override
    public void unlock() {
        // 获取线程标识
        String threadId = ID_PREFIX+Thread.currentThread().getId();
        // 获取锁标识
        String id = stringRedisTemplate.opsForValue().get(KEY_PREFIX + name);
        if(threadId.equals(id)){
        stringRedisTemplate.delete(KEY_PREFIX + name);
        }
    }
}
```

`VoucherOrderServiceImpl.java`

```java
        Long userId = UserHolder.getUser().getId();
        // 创建锁对象
        SimpleRedisLock lock = new SimpleRedisLock("order:" + userId, stringRedisTemplate);
        // 获取锁
        boolean isLock = lock.tryLock(1200);
        if (!isLock) {
            // 获取锁失败
            return Result.fail("不允许重复下单");
        }
        // 获取锁成功
        try {
            // 获取代理对象
            IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);
        } catch (IllegalStateException e) {
            throw new RuntimeException(e);
        }finally {
            lock.unlock();
        }
```

> 问题：线程一判断锁和释放锁之间发生了阻塞，锁超时释放,线程二获取到了锁,线程一释放了线程二的锁
>
> 解决：保证 判断锁和释放锁的原子性

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737529.png" alt="image-20240525145419467" style="zoom:67%;" />

#### Version 3

通过Redis Lua脚本，保证原子性，一致性

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737530.png" alt="image-20240525152529162" style="zoom:67%;" />

`unlock.lua`

```lua
-- 比较线程标识与锁中的标识是否一致
if(redis.call('get',KEYS[1])==ARGV[1]) then
    -- 释放锁 del key
    return redis.call('del',KEYS[1])
end
return 0
```

`SimpleRedisLock.java`

```java
    private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static{
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }
    @Override
    public void unlock(){
    //     调用lua脚本
        stringRedisTemplate.execute(
                UNLOCK_SCRIPT,
                Collections.singletonList(KEY_PREFIX + name),
                ID_PREFIX+Thread.currentThread().getId());
    }
```

### Redisson分布式锁

#### 基本使用

> Redisson是一个在Redis的基础上实现的Java驻内数据网络（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务，其中就包括了各种分布式锁的实现。

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737531.png" alt="image-20240525162822094" style="zoom:67%;" />

1.引入依赖

```xml
        <!--Redisson-->
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson</artifactId>
            <version>3.30.0</version>
        </dependency>
```

2.Redisson 配置类

```java
@Configuration
public class RedissonConfig {
    @Bean
    public RedissonClient redissonClient() {
        // 配置
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1");//.setPassword("")
        return Redisson.create(config);
    }
}
```

3.创建锁对象

```java
// SimpleRedisLock lock = new SimpleRedisLock("order:" + userId, stringRedisTemplate);
// 创建分布式锁对象
RLock lock = redissonClient.getLock("order:" + userId);
// 获取锁
boolean isLock = lock.tryLock();
```

#### 可重入锁原理

> 锁的加减，为0时表示业务结束，释放锁

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737532.png" alt="image-20240525170515489" style="zoom: 67%;" />

#### 锁重试和WatchDog机制

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737533.png" alt="image-20240525171845473" style="zoom:67%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737534.png" alt="image-20240525171920617" style="zoom: 50%;" />



#### 主从一致性问题

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737535.png" alt="image-20240525173116295" style="zoom:67%;" />



<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737536.png" alt="image-20240525172925770" style="zoom: 50%;" />

## 2.5、Redis优化秒杀

> Redis和Tomcat异步

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737537.png" alt="image-20240525173959310" style="zoom: 67%;" />



<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737538.png" alt="image-20240525174334997" style="zoom:67%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737539.png" alt="image-20240525175324024" style="zoom:67%;" />



①.`VoucherServiceImpl.java`

```java
    @Override
    @Transactional
    public void addSeckillVoucher(Voucher voucher) {
        // 保存优惠券
        save(voucher);
        // 保存秒杀信息
        SeckillVoucher seckillVoucher = new SeckillVoucher();
        seckillVoucher.setVoucherId(voucher.getId());
        seckillVoucher.setStock(voucher.getStock());
        seckillVoucher.setBeginTime(voucher.getBeginTime());
        seckillVoucher.setEndTime(voucher.getEndTime());
        seckillVoucherService.save(seckillVoucher);
        // 保存秒杀库存到Redis
        stringRedisTemplate
                .opsForValue()
                .set(SECKILL_STOCK_KEY+voucher.getId(),voucher.getStock().toString());
    }
```

② `seckill.lua`

```lua
local voucherId = ARGV[1]
local userId = ARGV[2]
-- 库存key
local stockKey = 'seckill:stock:'..voucherId
-- 订单key
local orderKey = 'seckill:order:'..voucherId
-- 判断库存是否冲粗
if(tonumber(redis.call('get,stockKey'))<=0) then
    return 1
end
-- 判断用户是否重复下单
if(redis.call('sismember',orderKey,userId)==1) then
    return 2
end
-- 扣减库存
redis.call('incrby',stockKey,-1)
-- 下单（保存用户）
redis.call('sadd',orderKey,userId)
return 0
```

`VoucherOrderServiceImpl.java`

```java
   private static final DefaultRedisScript<Long> SECKILL_SCRIPT;
    static{
        SECKILL_SCRIPT = new DefaultRedisScript<>();
        SECKILL_SCRIPT.setLocation(new ClassPathResource("seckill.lua"));
        SECKILL_SCRIPT.setResultType(Long.class);
    }
    @Override
    public Result seckillVoucher(Long voucherId) {
        Long userId = UserHolder.getUser().getId();
        // 执行lua脚本
        Long result = stringRedisTemplate.execute(
                SECKILL_SCRIPT,
                Collections.emptyList(),
                voucherId.toString(),
                userId.toString()
        );
        int r = result.intValue();
        if(r!=0){
            return Result.fail(r==1?"库存不足":"不能重复下单");
        }
        // 为0，有购买资格，把下单信息保存到阻塞队列
        long orderId = redisIdWorker.nextId("order");

        return Result.ok(0);
    }
```

③

## 2.6、Redis消息队列

> 消息队列：存储和管理消息，也被称为消息代理
>
> 生产者：发送消息到消息队列
>
> 消费者从消息队列获取消息并处理消息

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737540.png" alt="image-20240525190915099" style="zoom:67%;" />



<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737541.png" alt="image-20240525191151265" style="zoom:67%;" />

![image-20240525191308808](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737542.png)

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737543.png" alt="image-20240525191417766" style="zoom: 50%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737544.png" alt="image-20240525191700330" style="zoom:67%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737545.png" alt="image-20240525191714266" style="zoom:50%;" />

### 三种对比

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737546.png" alt="image-20240525191754099" style="zoom:67%;" />

### 基于Stream结构，异步秒杀

![image-20240525192404171](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737547.png)



## 2.7、达人探店











------



## 工具类

### 正则验证工具包

#### 正则验证工具

`RegexUtils` 

```java
public class RegexUtils {
    /**
     * 是否是无效手机格式
     * @param phone 要校验的手机号
     * @return true:符合，false：不符合
     */
    public static boolean isPhoneInvalid(String phone){
        return mismatch(phone, RegexPatterns.PHONE_REGEX);
    }
    /**
     * 是否是无效邮箱格式
     * @param email 要校验的邮箱
     * @return true:符合，false：不符合
     */
    public static boolean isEmailInvalid(String email){
        return mismatch(email, RegexPatterns.EMAIL_REGEX);
    }

    /**
     * 是否是无效验证码格式
     * @param code 要校验的验证码
     * @return true:符合，false：不符合
     */
    public static boolean isCodeInvalid(String code){
        return mismatch(code, RegexPatterns.VERIFY_CODE_REGEX);
    }

    // 校验是否不符合正则格式
    private static boolean mismatch(String str, String regex){
        if (StrUtil.isBlank(str)) {
            return true;
        }
        return !str.matches(regex);
    }
}
```

#### 正则验证规则

`Regexpatterns` 

```java
public abstract class RegexPatterns {
    /**
     * 手机号正则
     */
    public static final String PHONE_REGEX = "^1([38][0-9]|4[579]|5[0-3,5-9]|6[6]|7[0135678]|9[89])\\d{8}$";
    /**
     * 邮箱正则
     */
    public static final String EMAIL_REGEX = "^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\\.[a-zA-Z0-9_-]+)+$";
    /**
     * 密码正则。4~32位的字母、数字、下划线
     */
    public static final String PASSWORD_REGEX = "^\\w{4,32}$";
    /**
     * 验证码正则, 6位数字或字母
     */
    public static final String VERIFY_CODE_REGEX = "^[a-zA-Z\\d]{6}$";

}
```



### 缓存工具封装

![image-20240524135837123](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061737548.png)

```java
@Slf4j
@Component
public class CacheClient {
    private final StringRedisTemplate stringRedisTemplate;

    public CacheClient(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }
		//设置值
    public void set(String key, Object value, Long time, TimeUnit unit) {
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value), time, unit);
    }
		//设置逻辑过期值
    public void setWithLogicalExpire(String key, Object value, Long time, TimeUnit unit) {
        RedisData redisData = new RedisData();
        redisData.setData(value);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
        // 写入redis
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value));
    }
		//缓存穿透查询
    public <R, ID> R queryWithPassThrough(String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        // 从redis查询店铺信息
        String json = stringRedisTemplate.opsForValue().get(key);
        // redis中存在，类型转换 JSON-->Bean
        if (StringUtils.isNotBlank(json)) {
            return JSONUtil.toBean(json, type);
        }
        // 判断是否命中空值
        if (json != null) {
            return null;
        }
        // redis不存在，从数据库查询
        R r = dbFallback.apply(id);
        if (r == null) {
            // 缓存空对象
            stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES);
            return null;
        }
        // 缓存到redis,Bean-->JSON
        this.set(key, r, time, unit);
        return r;
    }

    private boolean tryLock(String key) {
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);
    }

    private void unLock(String key) {
        stringRedisTemplate.delete(key);
    }

    // 线程池
    private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

    // 逻辑过期查询
    public <R, ID> R queryWithLogicalExpire(String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        // 从redis查询店铺信息
        String json = stringRedisTemplate.opsForValue().get(key);
        // 判断是否存在
        if (StringUtils.isBlank(json)) {
            return null;
        }
        // 命中，将JSON反序列化为对象
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        R r = JSONUtil.toBean((JSONObject) redisData.getData(), type);
        LocalDateTime expireTime = redisData.getExpireTime();
        // 判断是否过期
        if (expireTime.isAfter(LocalDateTime.now())) {
            // 未过期
            return r;
        }
        String lockKey = LOCK_SHOP_KEY + id;
        boolean isLock = tryLock(lockKey);

        if (isLock) {
            // 成功，开启独立线程，实现缓存重建
            CACHE_REBUILD_EXECUTOR.submit(() -> {
                try {
                    R r1 = dbFallback.apply(id);
                    this.setWithLogicalExpire(key, r1, time, unit);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                } finally {
                    // 释放锁
                    unLock(lockKey);
                }
            });

        }
        return r;
    }

}
```











