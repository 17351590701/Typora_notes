## Hutool实现

1. 引入hutool依赖

   ```xml
   <dependency>
      <groupId>cn.hutool</groupId>
      <artifactId>hutool-all</artifactId>
      <version>5.8.27</version>
   </dependency>
   ```

2. 编写excel导入数据库

   ```java
   @PostMapping("/import")
   public void imp(@RequestBody MultipartFile file) throws Exceprion{
      //1. 使用InputStream读取文件
      InputStraem is = file.getInputStream();
      //2. 读取流中的数据
      ExcelReader reader = ExcelUtil.getReader(is);
      //3. 将读取到的数据填充为List<User>
      List<User> users = reader.readAll(User.class);
      //4. 插入数据库
      userMapper.batchInsert(users);
   }
   ```

3. 编写excel导出excel表

   ```java
   @GetMapping("/export")
   public void export(HttpServletResponse response) throws Exception{
      //1. 查询所有的user数据
      ArrayList<User> users = userMapper.selectAll();
      //2. 工具类创建write写出到浏览器
      ExcelWriter writer = ExcelUtil.getWriter(true);
      //  自定义标题别名
      // writer.addHeaderAlias("username","用户名");
      //3. 一次写出到excel，使用默认方式强制输出标题
      writer.write(users,true);
      //4. 设置浏览器响应格式，直接复制
      response.setContentType("application/vnd.openxmlformats-officedocument."+"spreadsheetml.sheet;charset=utf-8");
      String fileName = URLEncoder.encode("用户信息","UTF-8");
      response.setHeader("Content-Disposition","attachment;filename="+fileName+".xlsx");
      //5. 将writer中的数据刷新到流中
      ServletOutputStream outputStream = response.getOutputStream();
      writer.flush(outputStream,true);
      //6. 关闭流
      outputStream.close();
      writer.close();
   }
   ```


## Easy-excel实现

[官方文档](https://easyexcel.opensource.alibaba.com/docs/current/)

### 写Excel

1. 引入**依赖**

   ```xml
   <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>easyexcel</artifactId>
      <version>3.3.2</version>
   </dependency>
   ```

2. **实体**中指定要导出的字段和其列名

   ```java
   @ExcelProperty("创建时间") // excel工作部中的列名
   private LocadDateTime createTime;
      
   @ExcelIgnore // 导出时忽略的字段
   private LocadDateTime updateTime;
   ```

3. 接口实现

   ```java
   @GetMapping("/export")
   public String export(){
      // 数据库查询的集合
      ArrayList<xxx> list = new ArrayList<>();
      // 文件名为：存储路径+时间戳+后缀名
      String fileName = "D:\\"+ Instant.now().toEpochMilli()+".xlsx";
      // xxx.class 为集合内每个实体类型  list为要写入的实体集合
      EasyExcel.write(fileName,xxx.class).sheet("工作部名").doWrite(list);
      return null;
   }
   ```

   

### 读Excel

```java
fileName = "存储路径" + "demo" + File.separator + "demo.xlsx";
// 这里 需要指定读用哪个class去读，然后读取第一个sheet 文件流会自动关闭
EasyExcel.read(fileName, DemoData.class, new DemoDataListener()).sheet().doRead();
```

