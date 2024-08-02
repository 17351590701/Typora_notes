1.在环境变量PATH中添加Redis安装路径

2.在Redis文件夹cmd运行 

```cmd
redis-server.exe --service-install redis.windows.conf --loglevel verbose
```

3.配置后，默认为自动开启，应情况设置为手动

```powershell
net start redis
net stop redis
```

