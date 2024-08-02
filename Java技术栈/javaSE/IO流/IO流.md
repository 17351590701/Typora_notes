## 转换流

> JDK11之前

用转换流指定字节流编码读取

```java
//JDK11之前
InputStreamReader isr = new InputStreamReader(new FileInputStream("src\\main\\java\\file\\test2.txt"), StandardCharsets.UTF_8);
int len;
while((len=isr.read())!=-1){
  System.out.print((char)len);
}
isr.close();

```

指定编码写入

```java
//指定字符编码读取写出
FileReader fr = new FileReader("src\\main\\java\\file\\test2.txt", StandardCharsets.UTF_8);
int len;
while((len=fr.read())!=-1){
  System.out.print((char)len);
}
fr.close();

```

> JDK11之后，指定编码

```java
FileWriter fw = new FileWriter("src\\main\\java\\file\\test2.txt", StandardCharsets.UTF_8,true);
fw.write("你好");
fw.close();
```

> 练习

```java
/* 字节流读取中文会乱码，可以用转换流，再用缓冲流读取整行 */
BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("src\\main\\java\\file\\test2.txt")));
String line;
while((line= br.readLine())!=null){
  System.out.println(line);
}
```

## 序列化流，反序列化流

> JavaBean实现Serializable接口。设置版本号，使修改javabean后，序列化，反序列化依然一致

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Student implements Serializable {
    //根据javabean中成员变量等等，计算生成版本号
    @Serial
    private static final long serialVersionUID = 1540597275057374078L;
    private String name;
    private int age;
  	//transient 标记使该属性不被序列化
    private transient String address;
}
```

```java
public class FileDemo05 {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("src\\main\\java\\file\\test2.txt"));
        oos.writeObject(new Student("张三", 18, "北京"));
        oos.close();

        ObjectInputStream ois  =new ObjectInputStream(new FileInputStream("src\\main\\java\\file\\test2.txt"));
        Student student = (Student) ois.readObject();
        System.out.println(student);
        ois.close();
    }
}
```

==结果==：Student(name=张三, age=18, address=null)

## 爬取

### 爬取百家姓

```java
public class FileDemo06 {
    public static void main(String[] args) throws IOException, URISyntaxException {
        String familyNameNet = "https://hanyu.baidu.com/shici/detail?pid=0b2f26d4c0ddb3ee693fdb1137ee1b0d&from=kg0";
        //获取该网址所有数据,拼接成字符串返回
        String familyName = webCrawler(familyNameNet);
        //通过正则表达式筛选
        ArrayList<String> list= getData(familyName,"([\\u4E00-\\u9FA5]{4})(，|。)",1);
        System.out.println(list);

    }

    private static ArrayList<String> getData(String str, String regex,int index) {
        ArrayList<String> list = new ArrayList<>();
        //创建正则表达式对象
        Pattern pattern =Pattern.compile(regex);
        //创建匹配器
        Matcher matcher = pattern.matcher(str);
        while (matcher.find()){
            list.add(matcher.group(index));
        }
        return list;
    }

    private static String webCrawler(String net) throws IOException, URISyntaxException {
        StringBuilder sb = new StringBuilder();
        //创建url对象
        URL url = new URI(net).toURL();
        //连接网址
        URLConnection conn = url.openConnection();
        //转换流读取数据
        InputStreamReader isr = new InputStreamReader(conn.getInputStream());
        int ch ;
        while((ch=isr.read())!=-1){
            sb.append((char)ch);
        }
        return sb.toString();
    }
}
```

==结果==：[赵钱孙李, 周吴郑王, 冯陈褚卫,......]

> 利用huTool工具包爬取

```java
public class FileDemo07 {
    public static void main(String[] args) {
        String familyNameNet = "https://hanyu.baidu.com/shici/detail?pid=0b2f26d4c0ddb3ee693fdb1137ee1b0d&from=kg0";
        // 利用huTool工具包,爬取所有数据返回字符串
        String familyNameStr = HttpUtil.get(familyNameNet);
        // 利用正则表达式匹配任意四个中文+“，。”
        List<String> familyNameTempList = ReUtil.findAll("([\\u4E00-\\u9FA5]{4})(，|。)", familyNameStr, 1);
        System.out.println(familyNameTempList);
    }
}
```

==结果==：[赵钱孙李, 周吴郑王, 冯陈褚卫,......]
