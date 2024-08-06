# JDK 8 之前传统 时间 API

## Date

> 日期和时间 

```java
Date date = new Date();
//Wed May 22 21:19:44 CST 2024
System.out.println(date);
long time = date.getTime();
//获取该时间毫秒值 1716383984188 
System.out.println(time);
//time + 2s 毫秒值+2s
time+=2*1000;
//将修改后的time再转化为日期对象 Wed May 22 21:19:46 CST 2024 
System.out.println(new Date(time)); //date.setTime(time)
```

## SimpleDateFormat

> 简单日期格式化，把日期对象、时间毫秒值格式化成指定格式

```java
Date date = new Date();
long time = date.getTime();
//格式化日期对象 和 毫秒值
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd日 HH:mm:ss EEE a");
//2024-05-22日 21:29:40 周三 下午
System.out.println(sdf.format(date));
System.out.println(sdf.format(time));
```

==解析== 字符串时间 为 日期对象

```java
String str = "2022-12-12 12:12:12";
// 格式应与被解析字符串格式一致
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
//解析字符串
Date date = sdf.parse(str);
//Mon Dec 12 12:12:12 CST 2022
System.out.println(date);
SimpleDateFormat sdf2 = new SimpleDateFormat("yyyy-MM-dd日 HH:mm:ss EEE a");
//格式转化 2022-12-12日 12:12:12 周一 下午
System.out.println(sdf2.format(date));
```



## Calendar

> 代表系统此时时间对应的日历，通过它可以单独获取、修改时间中的年、月、日、时、分、秒等


```java
// 系统此时日历对象
Calendar now = Calendar.getInstance();
System.out.println(now);
// 获取月份要+1
System.out.println(now.get(Calendar.MONTH)+1);
// 日期对象
System.out.println(now.getTime());
// 一年中的第325天
now.set(Calendar.DAY_OF_YEAR,325);
now.set(Calendar.YEAR,2025);
now.add(Calendar.YEAR,1);
```



------



# JDK 8 新增 时间 API

## LocalDate

> 年 月 日

```java
//2024-5-22
LocalDate ld = LocalDate.of(2024,5,22);//构造方法初始化， 年 月 日
//2024-5-22
LocalDate ld = LocalDate.now();//获取当前时间
```

## LocalTime

> 小时 分钟 秒 纳秒

```java
//08:00
LocalTime lt = LocalTime.of(8,0,0);//构造方法初始化，小时 分钟 秒
//20:16:17.041375400
LocalTime lt = LocalTime.now();
```

## LocalDateTime

> 表示从 Unix 纪元（1970年1月1日 00:00:00 UTC）到当前时刻的日期和时间，但不包含任何时区信息。这意味着它反映了系统==默认时区==的当前日期和时间
>
> **七个参数**： 年 月 日 时 分 秒 纳秒（毫秒-微秒-纳秒）

```java
//2024-05-22T08:00
LocalDateTime ldt = LocalDateTime.of(2024,5,22,8,0,0);
//2024-05-22T20:16:17.041375400 其中的“041375400”代表从这秒开始的 纳秒数
LocalDateTime ldt = LocalDateTime.now();
```

## Instant

> 自 Unix 纪元（1970年1月1日 00:00:00 UTC）以来的偏移量，`T` 分隔日期和时间，`Z` 表示 UTC 时间区（协调世界时）
>

```java
//2024-05-22T12:19:12.800523600Z
Instant.now()
//1970-01-01T00:00:10.000001Z  从1970后开始的秒数和纳秒数
Instant.ofEpochSecond(10, 1000);
```

## Duration

> 用途：Duration 类用于处理时间间隔，重点关注时、分、秒以及纳秒级别的精确时间度量。它适用于计算两个瞬时时间点之间的时间差，比如计算两个 Instant 对象或 LocalTime 对象之间的时间差。

```java
//PT10S  P 前缀表示这是一个时间段，T 分割了日期部分（如果有的话）和时间部分
Duration.between(LocalTime.of(20, 10, 10), LocalTime.of(20, 10, 20))
//PT10S
Duration.of(10, ChronoUnit.SECONDS)
```

## Period

> 用途：Period 类则用于处理日期间隔，关注年、月、日这三个日期单位的差距。它适用于计算两个日期（如 LocalDate 实例）之间的差异，而不考虑具体的时间。

```java
//P1D
Period.between(LocalDate.of(2024, 5, 22), LocalDate.of(2024, 5, 23));
//P1Y1M1D  P 表示这是一个日期间隔，Y 年，M 月，D 天
Period.of(1, 1, 1)
```

## 日期操作

```java
//2024-5-22
LocalDate today = LocalDate.now();
//2023-5-22 调整年份2023
today.withYear(2023);
//2025-05-22 年份+1
today.PlusYears(1);
//2024-6-22  月份+1
today.plus(1,ChronoUtil.MONTHS)
   
//2025-5-25  本月最后一个星期六
today.with(TemporalAdjusters.lastInMonth(DayOfWeek.SATURDAY));
```

## 日期格式化

> DateTimeFormatter 用于时间的格式化和解析

```java
//2024-05-22T20:42:24.825027900
LocalDateTime now = LocalDateTime.now();
//2024-05-22 ISO日期格式
now.format(DateTimeFormatter.ISO_DATE)
//20240522  BASIC_ISO日期格式
now.format(DateTimeFormatter.BASIC_ISO_DATE)
//格式化到指定格式
String str = "2024-03-22";
LocalDate.parse(str, DateTimeFormatter.ofPattern("yyyy-MM-dd"));
```

## ZoneDateTime

> 表示一个日期和时间，同时包含了时区信息
>
> ZoneId 时区

```java
//2024-05-22T20:53:57.853325800+08:00[Asia/Shanghai]
ZonedDateTime.now();

// 获取指定时区的时间信息
//2024-05-22T14:53:57.853325800+02:00[Europe/Paris]
ZonedDateTime.now(ZoneId.of("Europe/Paris"));
```

## Instant转化LocalDateTime

```java
// 创建一个 Instant 对象，表示当前时间
Instant instant = Instant.now();

// 指定一个时区，例如 Asia/Shanghai
ZoneId zoneId = ZoneId.of("Asia/Shanghai");

// 将 Instant 转换为 ZonedDateTime
ZonedDateTime zonedDateTime = instant.atZone(zoneId);

// 从 ZonedDateTime 转换为 LocalDateTime
LocalDateTime localDateTime = zonedDateTime.toLocalDateTime();

// 输出转换后的 LocalDateTime
System.out.println(localDateTime);
```

