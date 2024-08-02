# ThreadLocal

## 基本使用

### ThreadLcoal基本使用

> 线程隔离
>
> 将content变量与当前线程绑定

```java
public class Demo01 {
    ThreadLocal<String> tl = new ThreadLocal<>();
    private String content;
    public String getContent() {
        return tl.get();
    }
    public void setContent(String content) {
        tl.set(content);
    }
    public static void main(String[] args) {
        Demo01 demo = new Demo01();
        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(() -> {
                demo.setContent(Thread.currentThread().getName() + "的数据");
                System.out.println("--------------------------");// 等待时间
                System.out.println(Thread.currentThread().getName() + "的数据:" + demo.getContent());
            });
            thread.setName("线程" + i);
            thread.start();
        }
    }
}
```



### synchronized同步

`Demo02.java`

```java
public class Demo02 {
    private String content;
    public String getContent() {
        return this.content;
    }
    public void setContent(String content) {
        this.content = content;
    }
    public static void main(String[] args) {
        Demo02 demo = new Demo02();
        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(() -> {
                synchronized (Demo02.class) {
                    demo.setContent(Thread.currentThread().getName() + "的数据");
                    System.out.println("--------------------------");// 等待时间
                    System.out.println(Thread.currentThread().getName() + "的数据:" + demo.getContent());
                }
            });
            thread.setName("线程" + i);
            thread.start();
        }
    }
}
```



## 对比

|        | synchronized                                                 | ThreadLocal                                                  |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 原理   | 同步机制采用以”时间换空间“的方式，只提供了一份变量，让不同的线程排队访问 | ThreadLocal采用以”空间换时间“的方式，为每个线程都提供了一份变量的副本，从而实现同时访问而相不干扰 |
| 侧重点 | 多个线程之间访问资源的同步                                   | 多线程中让每个线程之间的数据相互隔离                         |

> 虽然在本案例中，使用ThreadLocal和synchronized都能解决问题，但使用ThreadLocal更为合适，因为这样可以使程序拥有更高的并发性



## TreadLocal设计方案

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407132059367.png" alt="image-20240522151713851" style="zoom:67%;" />

## 存在问题

`ThreadLocal`存在线程继承问题，子线程无法获取到父线程设置过的变量，可以通过`InheritableThreadLocal`解决

```java
// 问题
public static void main(String[] args) {
   ThreadLocal<Integer> t1 = ThreadLocal.withInitial(() -> null);
   t1.set(1);
   // 子线程
   new Thread(() -> {
      // 输出 t1:null
      System.out.println("t1:" + t1.get());
   }).start();
}
```

`InHeritableThreadLocal` 存在线程池变量刷新问题，通过阿里的 `TransmittableThreadLocal`解决，需要导包

```java
//问题   
static ExecutorService threadPool = Executors.newSingleThreadExecutor();

public static void main(String[] args) {
   
   try {
      threadPool.submit(() -> {
         // 预期输出 null，实际 null
         System.out.println(t1.get());
      });
      // 主线程 main 改变 t1  
      t1.set("123");
      threadPool.submit(() -> {
         // 预期输出 123，实际 null
         System.out.println(t1.get());
      });
   } catch (Exception e) {
      log.error(e.getMessage());
   } finally {
      threadPool.shutdown();
   }
}
```

`TransmittableThreadLocal`

```xml
<dependency>
<groupId>com.alibaba</groupId>
	<artifactId>transmittable-thread-local</artifactId>
<version>2.14.3</version>
</dependency>
```

