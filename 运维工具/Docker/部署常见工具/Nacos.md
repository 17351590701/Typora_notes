## Windows下部署Nacos

1. 建立网桥 `docker network create my-net`

2. 创建Mysql容器并加入该网桥 `--network my-net`

3. 下载nacos.sql文件，导入到Docker容器的MySql中（配置映射后，可直接在navicat中导入）https://pan.baidu.com/s/1i6sZbO57dS4IkV3HyXDe9A?pwd=6zbd

4. docker拉取镜像 `docker pull nacos/nacos-server`

5. 创建运行容器，并连接到与MySql==同一网桥==

   > 当连接到同一网桥时，MYSQL_SERVICE_HOST可以使用通过访问mysql容器名连接
   >
   > 其中 MYSQL_SERVICE_DB_NAME=nacos 指定其为mysql的nacos数据库

   ```shell
   docker run -d `
   --name nacos `
   -p 8848:8848 `
   -p 9848:9848 `
   -p 9849:9849 `
   --network hm-net `
   -e PREFER_HOST_MODE=hostname `
   -e MODE=standalone `
   -e SPRING_DATASOURCE_PLATFORM=mysql `
   -e MYSQL_SERVICE_HOST=mysql `
   -e MYSQL_SERVICE_PORT=3306 `
   -e MYSQL_SERVICE_USER=root `
   -e MYSQL_SERVICE_PASSWORD=123456 `
   -e MYSQL_SERVICE_DB_NAME=nacos `
   -e MYSQL_SERVICE_DB_PARAM=characterEncoding=utf8"&"connectTimeout=1000"&"socketTimeout=3000"&"autoReconnect=true"&"useSSL=false"&"allowPublicKeyRetrieval=true"&"serverTimezone=Asia/Shanghai `
   nacos/nacos-server
   ```

6. 访问 http://localhost:8848/nacos ==默认用户==：nacos   nacos