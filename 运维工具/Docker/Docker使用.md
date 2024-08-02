## 镜像和容器

当我们利用Docker安装应用时，Docker会自动搜索并下载应用**镜像**（image）。镜像不仅包含应用运行所需要的环境、配置、系统函数库。Docker会在运行镜像时创建一个**隔离**环境，称为**容器**（container）

## 快速入门

### 部署mysql

> **docker run** :创建并运行一个容器
>
> **-d**：容器后台运行
>
> **--name**：容器名
>
> **-p**：设置端口映射（宿主机端口(用于访问）)：容器内端口）
>
> **-e**：环境变量（key=value）
>
> **repository:tag**：指定运行的镜像的名字和版本,不写tag，默认latest最新

```shell
docker run -d `
  --name mysql `
  -p 3306:3306 `
  -e TZ=Asia/Shanghai `
  -e MYSQL_ROOT_PASSOWRD=123456 `
  mysql
```

### 复制sql到docker

```shell
# 进入mysql容器内查看有哪些目录以便复制路劲
docker exec -it mysql bash
ls -l
# 复制本地文件heima.sql到mysql容器内
doker cp F:/heima.sql mysql:/tmpc/heima.sql
# 登录mysql用户
docker exec -it mysql bash
mysql -u root -p
# 创建数据库以便复制
create database heima;
use heima;
# 运行文件 /容器路劲/.sql
source /tmp/heima.sql
exit
```



## docker命令

| 命令                           | 说明                               |
| ------------------------------ | ---------------------------------- |
| docker pull xxx                | 拉取                               |
| docker push xxx                | 推送                               |
| docker images                  | 显示所有镜像                       |
| docker rmi xxx                 | 删除镜像                           |
| docker rm xxx                  | 删除容器    -f 强制删除（运行）    |
| docker run xxx                 | 创建并运行容器                     |
| docker start xxx               | 运行容器                           |
| docker stop xxx                | 停止容器                           |
| docker ps                      | 查看所有运行的容器     -a 所有容器 |
| docker save -o xxx.tar xxx     | 打包xxx镜像为tar包                 |
| docker load -i xxx.tar         | 加载tar包为镜像                    |
| docker logs xx                 | 查看日志 -f 跟踪                   |
| docker network connect xxx zzz | 添加xxx网络到zzz容器               |

遇到问题 使用 管理员cmd

```shell
net stop winnat
net start winnat
```



### 案例

> 查看DockerHub，拉取Nginx镜像，创建并运行镜像的名字

```shell
docker pull nginx  # 默认拉取最新版本镜像
docker images # 查看本地所有镜像

docker save -o nginx.tar nginx:latest # 保存镜像压缩包到本地
docker rmi nginx:latest # 删除镜像
docker load -i nginx.tar # 加载本地tar包为镜像

docker run -d --name nginx -p 80:80 nginx #创建并运行容器nginx镜像为nginx
# 访问网址 localhost:80 查看nginx
docker ps # 查看本地运行的容器
docker stop nginx # 停止容器运行
docker start nginx # 启动容器
docker exec -it mysql bash #进入运行的容器内终端

mysql -u root -p # 访问容器内数据库
show databases;

exit # 退出容器内终端

```

## 数据卷

> **数据卷**（volume）是一个虚拟目录，是**容器内目录**与**宿主机目录**之间映射的桥梁
>
> 数据卷中的更改时同时**双向绑定**在宿主机和容器内的

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733374.png" alt="image-20240604162451734" style="zoom:67%;" />

### 常见命令

| 命令                      | 说明                 |
| ------------------------- | -------------------- |
| docker volume create xxx  | 创建数据卷           |
| docker volume ls          | 查看所有数据卷       |
| docker volume rm xxx      | 删除指定数据卷       |
| docker volume inspect xxx | 查看某个数据卷的详情 |
| docker volume prune       | 清除数据卷           |



### 案例

> ==提示==
>
> - 在执行docker run 命令时，使用 `-v 数据卷:容器内目录` 可以完成数据卷挂载
> - 当创建容器时，如果挂载了数据卷且数据卷不存在，会自动创建数据卷

#### 案例一（数据卷）

利用Nginx容器部署静态资源

需求：

- 创建Nginx容器，修改nginx容器内的html目录下的index.html文件，查看变化
- 将静态资源部署到nginx的html目录

```shell
docker run -d --name nginx -p 80:80 -v html:/usr/share/nginx/html nginx #创建容器，数据卷html，并启动 
docker volume ls # 查看所有数据卷
docker volume inspect html #查看数据卷html 信息
```

> ==注意==
>
> 对于windows来说
>
> 宿主机目录为 :
>
> - 访问资源管理器路径`\\wsl$\docker-desktop-data\data\docker\volumes`
> - 访问资源管理器 Linux->docker-destop-data->data->docker->volumes

```shell
docker volume inspect html # 查看html数据卷详情
[
    {
        "CreatedAt": "2024-06-04T08:35:31Z",
        "Driver": "local",
        "Labels": null,
      	# 宿主机目录 linux
        "Mountpoint": "/var/lib/docker/volumes/html/_data",
        "Name": "html",
        "Options": null,
        "Scope": "local"
    }
]
```

#### 案例二（本地目录）

mysql容器的数据挂载

需求:

- 查看mysql容器，判断是否有数据卷挂载
- 基于宿主机目录实现Mysql数据目录、配置文件、初始化脚本的挂载(查阅官方镜像文档)
  - ==注意==在 “/”跟的是==绝对目录==,"/"是用于区别数据卷和本地目录
  - 挂载F/Docker/volumes/mysql/data到容器内的/var/lib/mysql目录
  - 挂载F/Docker/volumes/mysql/init 到容器内的/docker-entrypoint-initdb.d目录
  - 挂载F/Docker/volumes/msyql/cont 到容器内的/etc/mysql/conf.d目录

heima资料：https://pan.baidu.com/s/1inZog6YV0f3_Qqx2WDNJag?pwd=9988#list/path=%2F

`docker inspect mysql` # 查看mysql容器详情

发现已经有匿名数据卷,名称太乱目录太深且不易管理<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733375.png" alt="image-20240604170822356" style="zoom:67%;" />

> ==提示==
>
> - 在执行docker run 命令时，使用 **-v 本地目录:容器内目录** 可以完成本地目录挂载
> - 本地目录必须以**“/”**或**"./"**开头，如果直接以名称开头，会被识别为数据卷而非本地目录
>   - -v mysql:/var/lib/mysq 会被识别为一个数据卷叫mysql
>   - -v ./mysql:/var/lib/mysq 会被识别为当前目录下的mysql目录

##### 本地目录结构

本地创建mysql目录用于作为数据卷为：

F:\Docker\volumes\mysql

- conf

  - hm.cnf文件

    ```shell
    [client]
    default_character_set=utf8mb4
    [mysql]
    default_character_set=utf8mb4
    [mysqld]
    character_set_server=utf8mb4
    collation_server=utf8mb4_unicode_ci
    init_connect='SET NAMES utf8mb4'
    ```

    

- data： 初始化后数据保存的地方

- init ：`hmall.sql` 文件

##### 创建容器

映射本地目录

```shell
# 不能使用 -v/F:/ 要使用 -v /F/
docker run -d --name mysql -p 3307:3306 `
  -e MYSQL_ROOT_PASSWORD=123456 `
  -v /F/Docker/volumes/mysql/data:/var/lib/mysql `
  -v /F/Docker/volumes/mysql/init:/docker-entrypoint-initdb.d `
  -v /F/Docker/volumes/mysql/conf:/etc/mysql/conf.d `
 mysql:8.0.36
```

docker中的mysql容器和本地的mysql是相互隔离的 

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733376.png" alt="image-20240604175502102" style="zoom:67%;" />



## 自定义镜像

> 镜像就是包含了应用程序、程序运行的系统函数库、运行配置等文件的文件包。构建镜像的过程其实就是把上述文件打包的过程

### Dockerfile

Dockerfile就是一个文本文件，其中包含一个个的**指令（Instruction）**，用指令来说明要执行什么操作来构建镜像。Docker可以根据Dockerfile帮我们构建镜像

#### 常见命令

| 指令       | 说明                                     | 示例                                                         |
| ---------- | ---------------------------------------- | ------------------------------------------------------------ |
| FROM       | 指定基础镜像                             | FROM centos:                                                 |
| ENV        | 设置环境变量，可在后面指令使用           | ENV key value                                                |
| COPY       | 拷贝本地文件到镜像目录                   | COPY  ./jrell.tar.gz  /tmp                                   |
| RUN        | 执行linux的shell命令                     | RUN tar -zxvf  /tmp/jrell.tar.gz&&EXPORTS path=/tmp/jrell:$path |
| EXPOSE     | 指定容器运行时监听的端口，给镜像使用者看 | EXPOSE 8080                                                  |
| ENTRYPOINT | 镜像中应用的启动命令，容器运行时调用     | ENTRYPOINT  java  -jar  xx.jar                               |

### Build

> 编写好Dockerfile后，可以使用以下命令来构建镜像:
>
> -**t**:  是给镜像起名，格式是**repository:tag**，不指定tag，默认为latest
>
> **.**  :是指定Dockerfile所在目录，如果在当前目录则指定为"."

```shell
docker build -t myImage:1.0 .
```

准备一个openjdk:21镜像作为基础镜像用于打包 `docker pull openjdk:21`

**==Dockerfile==**

```shell
#基础镜像
FROM openjdk:21
#设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
#拷贝jar包
COPY jardemo.jar /app.jar
#入口
ENTRYPOINT ["java","-jar","/app.jar"]
```

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733377.png" alt="image-20240604211344817" style="zoom: 80%;" />

```shell
 docker build -t docker-demo . #在本目录下cmd运行打包为镜像
 docker run -d --name dd -p 8080:8080 docker-demo # 创建镜像容器并运行
 docker logs dd # 查看日志 -f强制
```

## 网络

默认情况下，所有所有容器都是以bridge方式连接到Docker的一个虚拟网桥上：

创建容器时，会有创建默认的network,有默认的网桥

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733378.png" alt="image-20240604212611390" style="zoom: 80%;" />

==注意==：

加入自定义网络的容器才可以通过容器名互相访问，Docker的网络操作命令如下：

| 命令                      | 说明                     |
| :------------------------ | :----------------------- |
| docker network create     | 创建一个网络             |
| docker network ls         | 查看所有网络             |
| docker network rm         | 删除指定网络             |
| docker network pruse      | 清除未使用的网络         |
| docker network connect    | 使指定容器连接加入某网络 |
| docker network disconnect | 是指定容器连接离开某网络 |
| docker network inspect    | 查看网络详细信息         |

```shell
docker network ls # 查看所有网络
docker network create heima #创建新网络
docker network inspect heima #查看网络详情
# 连接mysql到该网络 connect 网络名 容器名
docker network connect heima mysql 
docker inspect mysql #查看mysql容器的networks
# 创建容器时可以指定默认网络
docker run -d --name dd -p 8080:8080 --network heima docker-demo
```



## 部署应用

### 后端

同一网络下，springboot可以通过访问mysql容器名（而非ip地址）的方式来访问数据库

```shell
docekr netword create heima
docker netword connect heima mysql
docker run -d --name dd -p 8080:8080 --network heima docker-demo
```

`application.yml`

```yml
spring:
	datasource:
		url: jdbc:mysql//${hm.db.host}:3306/hmall?....
		drive-class-name: com.mysql.cj.jdbc.Driver
		username: root
		passowrd: ${hm.db.password}
```

`application-dev.yml`生产环境 访问容器名 数据库

```yml
hm:
	db:
		host: mysql #mysql为容器名
		password: 123456
```

`application-local.yml`

```yml
hm:
	db:
		host: localhost #本地ip地址
		passowrd: 123456
```

### 前端

...

## DockerCompose

### dockercompose下载

> 最新版本的DockerDeskTop(4.31.1) 已经整合了Docker和DockerCompose

Docker Compose 通过一个单独的**docker-compose.yml**模版文件来定义一组相关联的应用容器，帮助我们实现**多个相互关联的Docker容器的快速部署**

## 常见问题解决

```shell
net stop winnat
net start winnat
```

