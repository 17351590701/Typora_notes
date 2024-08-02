## Windows环境下部署Mysql

1. 拉取最新mysql `docker pull mysql`

2. 创建本地目录和文件

   - F:/Docker/volumes/mysql/conf

     - mysql.cnf (可选，用于配置字符编码等)

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

       

   - F:/Docker/volumes/mysql/data

   - F:/Docker/volumes/init

     - sql文件，用于初始化数据

3. 创建运行容器

   ```shell
   docker run -d `
   --name mysql `
   -p 3306:3306 `
   -e MYSQL_ROOT_PASSWORD=123456 `
   -v /F/Docker/volumes/mysql/data:/var/lib/mysql `
   -v /F/Docker/volumes/mysql/conf:/etc/mysql/conf.d `
   -v /F/Docker/volumes/mysql/init:/docker-entrypoint-initdb.d `
   mysql #镜像名 无tag，默认latest 
   ```

   