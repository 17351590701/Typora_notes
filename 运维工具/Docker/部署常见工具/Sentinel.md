## Windows环境下部署Sentinel

1. Docker 配置 Sentinel

```shell
# docker 拉取Sentinel镜像
docker pull docker.io/bladex/sentinel-dashboard
#创建并运行该容器
docker run -d --name sentinel -p 8858:8858 bladex/sentinel-dashboard
```

2. 访问 http://localhost:8858/ 默认用户名密码： sentinel

