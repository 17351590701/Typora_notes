

## Test.java

创建应用程序长下文，有参构造传入AppConfig.class字节码文件

尝试获取代理后的userService对象(bean)，调用方法，打印orderService(bean)

```java
public class Test {
    public static void main(String[] args) {
        ZYRApplicationContext ctx = new ZYRApplicationContext(AppConfig.class);
        //返回的是代理对象
        UserService userService = (UserService) ctx.getBean("userService");
        userService.printOrder();
    }
}
```

## AppConfig.java

扫描包"com.zyr.service"

```java
@ComponentScan("com.zyr.service")
public class AppConfig {
}
```

## ZYRApplicationContext.java

> 1.扫描`ComponentScan()`指定包名，将该包下的(.class)结尾的文件封装为==BeanDefinition==(Class,Scope...),并存储到`BeanDefinitiomMap`中，同时，如果有实现了`BeanPostProcessor`的类实例化后存储到 `BeanPostProcessorsList`中，用于对象后处理
>
> 2.根据`BeanDefinitionMap`，创建`bean`对象存入`sigletonObjectsMap`(单例池)
>
> 3.==创建bean==时，通过反射创建实例化，进行==依赖注入==（将bean注入到对应属性中）
>
> 判断`bean`是否实现了Aware接口进行Aware回调，循环`BeanPostProcessorsList`
>
> 调用`BeanPostProcessor`的==初始化前==方法，判断`bean`是否实现了`InitializingBean`接口进行==初始化==方法，循环`BeanPostProcessorsList`
>
> 调用`BeanPostProcessor`的==初始化后==方法(Spring与AOP的桥梁)

```java
public class ZYRApplicationContext {
    private static final Logger log = Logger.getLogger("ZYRApplicationContext");
    // 单例 池
    private final ConcurrentHashMap<String, Object> singletonObjectsMap = new ConcurrentHashMap<>();
    // BeanDefinition 池
    private final ConcurrentHashMap<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();
    // 后置处理 BeanPostProcessor
    private final List<BeanPostProcessor> beanPostProcessorsList= new ArrayList<>();

    public ZYRApplicationContext(Class<?> configClass) {
        // 封装bean-->BeanDefinition-->BeanDefinitionMap
        scan(configClass);

        // 根据BeanDefinitionMap，创建bean对象存入单例池
        for (Map.Entry<String, BeanDefinition> entry : beanDefinitionMap.entrySet()) {
            String beanName = entry.getKey();
            BeanDefinition beanDefinition = entry.getValue();
            if (beanDefinition.getScope().equals("singleton")) {
                // 创建单例bean对象，放入单例池
                Object bean = createBean(beanDefinition,beanName);
                singletonObjectsMap.put(beanName, bean);
            }
        }
    }

    /**
     * 扫描指定配置类中@Component注解标识的组件，加载这些组件并注册到BeanDefinitionMap中。
     * @param configClass 配置类，用于获取@ComponentScan注解信息，指定扫描包路径。
     */
    private void scan(Class<?> configClass) {
        // 解析配置类
        // 反射获取注解 ComponentScan
        ComponentScan componentScanAnnotation = configClass.getDeclaredAnnotation(ComponentScan.class);
        // 获取需要扫描路径 value="com.zyr.service"
        String path = componentScanAnnotation.value();
        // 转化格式 com.zyr.service -> com/zyr/service
        path = path.replace(".", "/");

        // 使用类加载器并加载指定路径下的资源
        ClassLoader classLoader = ZYRApplicationContext.class.getClassLoader();
        // 查找资源路径 file:/F:/IDEA/SpringWrite/target/classes/com/zyr/service
        URL resource = classLoader.getResource(path);

        // 扫描目录遍历文件，寻找所有的.class文件
        File file = new File(resource.getFile());
        if (file.isDirectory()) {
            File[] files = file.listFiles();
            if (files != null) {
                for (File f : files) {
                    String fileName = f.getAbsolutePath();
                    // 判断是否是class文件
                    if (fileName.endsWith(".class")) {
                        // 获取类全限定名（包名+类名）
                        String className = fileName.substring(fileName.indexOf("com"), fileName.indexOf(".class"));
                        className = className.replace("\\", ".");
                        try {
                            // 加载类并检查是否标注了@Component注解
                            Class<?> clazz = classLoader.loadClass(className);
                            if (clazz.isAnnotationPresent(Component.class)) {
                                //如果类实现了BeanPostProcessor接口
                                if (BeanPostProcessor.class.isAssignableFrom(clazz)){
                                    BeanPostProcessor instance = (BeanPostProcessor) clazz.getDeclaredConstructor().newInstance();
                                    beanPostProcessorsList.add(instance);
                                }
                                // 判断bean是单例还是原型
                                Component compAnnotation = clazz.getDeclaredAnnotation(Component.class);
                                String beanName = compAnnotation.value();
                                // 封装bean定义 BeanDefinition
                                BeanDefinition beanDefinition = getBeanDefinition(clazz);
                                // 存入BeanDefinition 池
                                beanDefinitionMap.put(beanName, beanDefinition);

                            }
                        } catch (Exception e) {
                            log.info("出现错误" + e.getMessage());
                        }
                    }
                }
            }
        }
    }

    /**
     * 获取指定类的Bean定义。
     *
     * @param clazz 需要创建Bean定义的类对象。
     * @return 返回根据给定类配置的Bean定义对象。
     */
    private static BeanDefinition getBeanDefinition(Class<?> clazz) {
        BeanDefinition beanDefinition = new BeanDefinition();
        // 设置Bean的类类型
        beanDefinition.setClazz(clazz);

        // 处理Bean的作用域，默认为单例
        if (clazz.isAnnotationPresent(Scope.class)) {
            Scope scopeAnnotation = clazz.getDeclaredAnnotation(Scope.class);
            // 如果类上有Scope注解，则使用注解指定的作用域
            beanDefinition.setScope(scopeAnnotation.value());
        } else {
            // 若没有Scope注解，则默认设置为单例模式
            beanDefinition.setScope("singleton");
        }
        return beanDefinition;
    }

    /**
     * 根据beanName获取对应的bean实例。
     * 如果bean实例是单例，则从单例池中获取；如果不是单例，则重新创建实例。
     *
     * @param beanName 要获取的bean的名称
     * @return 返回bean的实例
     * @throws NullPointerException 如果beanName不存在于bean定义映射中，则抛出异常
     */
    public Object getBean(String beanName) {
        // 检查beanName是否存在于beanDefinitionMap中
        if (beanDefinitionMap.containsKey(beanName)) {
            BeanDefinition beanDefinition = beanDefinitionMap.get(beanName);
            // 判断bean定义的范围是否为单例
            if ("singleton".equals(beanDefinition.getScope())) {
                // 如果是单例，则从单例池中获取bean实例
                return singletonObjectsMap.get(beanName);
            } else { // 如果不是单例，则创建一个新的bean实例
                return createBean(beanDefinition,beanName);
            }
        } else {
            // 如果beanName不存在于映beanDefinitionMap中
            throw new NullPointerException("beanName不存在");
        }
    }

    /**
     * 根据BeanDefinition创建一个bean实例。
     *
     * @param beanDefinition 包含bean配置信息的对象。
     * @param beanName bean的名称。
     * @return 创建的bean实例，如果发生异常返回null。
     */
    private Object createBean(BeanDefinition beanDefinition,String beanName) {
        Class<?> clazz = beanDefinition.getClazz();
        Object instance = null;
        try {
            // 通过反射机制创建bean实例
            instance = clazz.getDeclaredConstructor().newInstance();

            // 依赖注入
            for (Field filed : clazz.getDeclaredFields()) {
                if (filed.isAnnotationPresent(Autowired.class)) {
                    // 获取要注入的bean
                    Object bean = getBean(filed.getName());
                    filed.setAccessible(true);
                    // 设置属性值为要注入的bean
                    filed.set(instance, bean);
                }
            }
            //Aware回调，给实例对象属性赋值beanName
            if(instance instanceof BeanNameAware){
            ((BeanNameAware) instance).setBeanName(beanName);
            }
            //调用初始化前方法,加工
            for (BeanPostProcessor beanPostProcessor : beanPostProcessorsList) {
                 instance = beanPostProcessor.postProcessBeforeInitialization(instance, beanName);
            }

            // 初始化方法,设置属性后方法
            if(instance instanceof InitializingBean){
                ((InitializingBean) instance).afterPropertiesSet();
            }
            //初始化后方法
            for (BeanPostProcessor beanPostProcessor : beanPostProcessorsList) {
                instance = beanPostProcessor.postProcessAfterInitialization(instance, beanName);
            }
            // 返回bean实例或者代理对象
            return instance;
        } catch (InstantiationException e) {
            // 记录实例化异常
            log.info("创建bean对象失败" + e.getMessage());
        } catch (IllegalAccessException | InvocationTargetException | NoSuchMethodException e) {
            // 抛出运行时异常处理其他可能的异常
            throw new RuntimeException(e);
        }
        return null;
    }

}
```

## UserService.java

> 在==创建bean时==，实现BeanNameAware回调，设置beanName属性，实现InitializingBean，在设置属性后调用方法

```java
@Component("userService")
// @Scope("prototype")
public class UserServiceImpl implements BeanNameAware, InitializingBean, UserService {
    @Autowired
    private OrderService orderService;

    private String beanName;

    private String name;

    @Override
    public void printOrder() {
        System.out.println(orderService);
    }

    // Aware 回调设置beanName
    @Override
    public void setBeanName(String beanName) {
        this.beanName = beanName;
    }

    // 初始化方法,设置属性之后执行的方法
    @Override
    public void afterPropertiesSet() {
     	 System.out.println(orderService);
    }

    // 初始化前后置处理
    public void setName(String name) {
        this.name = name;
    }
}
```

## BeanNameAware.java

```java
public interface BeanNameAware {
    void setBeanName(String beanName);
}
```

## InitializingBean.java

> 初始化方法

```java
public interface InitializingBean {
  	//设置属性后执行的方法
    void afterPropertiesSet();
}
```

## BeanDefinition.java

> 存储一些bean定义信息

```java
@Data
public class BeanDefinition {
    private Class clazz;
    private String scope;
}
```



## @Autowired

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.FIELD})
public @interface Autowired {
}
```

## @Component

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Component {
    String value();
}
```

## @Scope

> singleton：单例， prototype：原型（多例）

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Scope {
    String value();
}
```

## BeanPostProcessor.java

```java
public interface BeanPostProcessor {
    //初始化前
    Object postProcessBeforeInitialization(Object bean, String beanName);
    //初始化后
    Object postProcessAfterInitialization(Object bean, String beanName);
}

```

## ZyrBeanPostProcessor.java

```java
@Component("zyrBeanPostProcessor")
public class ZyrBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("初始化前");
        // 调用UserService setName方法
        if (beanName.equals("userService")) {
            ((UserServiceImpl) bean).setName("张雨润");
        }
        return bean;
    }

    // AOP 是基于 postProcessAfterInitialization 初始化后实现的
  	// Spring底层默认使用JDK，如果没有接口，就是用CGLIB
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("初始化后");
        // JDK生成代理对象
        if ("userService".equals(beanName)) {
             return Proxy.newProxyInstance(
                    UserServiceImpl.class.getClassLoader(),
                    UserServiceImpl.class.getInterfaces(),
                    (proxy, method, args) -> {
                        System.out.println("寻找切点,执行代理逻辑，返回代理对象");
                        return method.invoke(bean, args);
                    });
        }
        return bean;
    }
}
```

