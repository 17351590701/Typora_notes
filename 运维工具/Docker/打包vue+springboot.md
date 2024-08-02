## 1、打包vue

### 1.构建vue项目

`npm run build`构建项目到dist目录

### 2.配置Dockerfile文件

在dist同目录上新建Dockerfile文件

```shell
From nginx #默认latest
LABEL Author zhangyurun # 标注镜像作者
COPY dist /usr/share/nginx/html
```

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733947.png" alt="image-20240505192126769" style="zoom: 80%" />

### 3.制作镜像

启动docker desktop，在根目录上cmd ==镜像名字后的“.”不能省略==

```shell
docker build -t 镜像名字 .
#比如
docker build -t vue3project .
```

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733948.png" alt="image-20240505192516336" style="zoom:80%" />

制作完成后，在docker destop的 images镜像中

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733949.png" alt="image-20240505192724041" style="float: left; zoom: 67%;"/>

### 4.创建容器

点击运行镜像，会创建contaners容器(镜像运行需要容器),输入容器名和端口号

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733950.png" alt="image-20240505193053903" style="zoom: 67%" />

点击==启动容器==的port端口或者浏览器访问localhost:8990浏览

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733951.png" alt="image-20240505193335009" style="zoom: 67%" />



## 2、打包springboot

1.用maven打包,clean,package,打包成jar目标文件

配置Dockerfile

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733952.png" alt="image-20240505213028767" style="zoom:67%;" />

```shell
FROM openjdk:21
#监听端口8080
EXPOSE 8080
#ADD 后面的参数是项目名字 / 后面的参数是自定义的别名
ADD shopping.jar shopping.jar
#这里的最后一个变量需要和前面起的别名相同
ENTRYPOINT [“java","-jar","/shopping.jar"]

```

2.制作镜像，在此目录上cmd

`docker build -t springbooot .`

3.生成对应容器，启动mysql容器和springboot容器



## 3、发布与拉取

> 科学上网，网络可能不稳定，会出现EOF状况，尝试更换节点

1.通过github或google注册登录docker-destop

2.在cmd中输入`docker login`查看是否登录成功

3.为避免与他人镜像重名，可修改名字,以用户名作为前缀，并且打上`tag`标签（:zyr）

```cmd
docker tag 原镜像名 新镜像名
#例如
docker tag vue3project zhangyurun/vue3project:zyr
```

==改名后，会有原镜像和新镜像，可将原来的删除==

4.在docker Hub的Repositories中新建仓库，仓库名要与镜像名相同

![image-20240505204000428](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733953.png)

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733954.png" alt="image-20240505202241755" style="zoom:67%;" />

5.发布镜像,在docker-destop 中`push to Hub`镜像或者通过cmd命令

```cmd
docker push 镜像名
#例如
docker push zhangyurun/vue3project:zyr
```

6.拉取镜像

在docker-destop搜索镜像pull拉取,或者cmd中命令`docker pull zhangyurun/vue3project:zyr`

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061733955.png" alt="image-20240505203455387" style="zoom:67%;" />

> 运行镜像,创建容器-->启动容器，完成拉取
