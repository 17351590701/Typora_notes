## Windows环境下部署RabbitMq

1. 拉取镜像 `docker pull rabbitmq:management` ,该镜像默认开启了web插件

2. 创建并运行容器,设置用户名和密码     (==默认用户==  guest:guest)

   ```shell
   docker run -d --name rabbitmq `
    -p 15673:15672 `
    -p 5673:5672 `
    -e RABBITMQ_DEFAULT_USER=admin `
    -e RABBITMQ_DEFAULT_PASS=admin `
    rabbitmq:management
   ```

3. (如果拉取的镜像时 `rabbitmq`)开启web可视化插件

   ```shell
   # 方法一：进入rabbitmq容器后开启
   docker exec -it rabbitmq bash
   rabbitmq-plugins enable rabbitmq_management
   
   # 方法二：直接开启
   docker exec -it rabbitmq rabbitmq-plugins enable rabbitmq_management
   ```

4. 访问网址 http://localhost:15673

> 端口介绍：
>
> - 15672 管理页面通信的端口
> - 5672 Client端通信端口
> - 25672 Server间通信端口
> - 4369 EPMD默认端口号
> - 61613 Stomp消息传输端口