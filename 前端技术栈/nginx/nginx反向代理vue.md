1. vue项目打包 `npm run build` 为 /dist 文件夹

2. [下载 Nginx](https://nginx.org/en/download.html)

    <img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407211443133.png" alt="image-20240721144332079" style="zoom:67%;" />

3. 修改 Nginx 目录下的 /conf/nginx.conf文件（将dist移动到Nginx目录下，方便打包）

   ```powershell
   worker_processes  1;
   events {
       worker_connections  1024;
   }
   http {
       include       mime.types;
       default_type  application/octet-stream;
       sendfile        on;
       keepalive_timeout  65;
       server {
           listen       80;
           server_name  localhost;
   ############   root 为 dist所在目录（路径需要"/"） ##########
           location / {
               root   F:/nginx-1.26.1/dist;
               index  index.html index.htm;
           }
           error_page   500 502 503 504  /50x.html;
           location = /50x.html {
               root   html;
           }
       }
   }
   ```

4. 双击 **Nginx.exe** 启动服务 访问端口 http://localhost/

5. 可在任务管理器中 关闭 Nginx 服务

> ==注意==：启动多次同一个nginx.exe服务,访问网页可能报错
>
> **解决办法**：资源管理器，关闭所有nginx服务，重新启动一个

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407211516762.png" alt="image-20240721151657708" style="zoom:67%;" />

