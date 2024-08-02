

# Redis主从集群

## Redis主从复制

### 搭建主从复制

单节点Redis的并发能力是有上限的，要进一步提高Redis的并发能力，就需要搭建主从集群，实现读写分离。

 

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406231711129.png" alt="image-20240623171125061" style="zoom: 67%;" />

#### 启动多个Redis实例

使用DockeCompose 定义和运行多容器

`docker-compose.yml`

> network_mode:"host"暴露给本机，直接作为本机进程
>
> 访问时需要 docke exec -it r1 redis-cli -p 7001 添加 -p 端口

```yaml
vservices:
  r1:
    image: redis
    container_name: r1
    network_mode: "host"
    entrypoint: [ "redis-server","--port","7001" ]
  r2:
    image: redis
    container_name: r2
    network_mode: "host"
    entrypoint: [ "redis-server","--port","7002" ]
  r3:
    image: redis
    container_name: r3
    network_mode: "host"
    entrypoint: [ "redis-server","--port","7003" ]
```

在该文件夹下进入**命令行**

运行 `docker-compose up -d`

#### 建立集群

> 虽然我们启动了3个Redis实例，但它们并没有形成主从关系，我们需要通过命令来配置主从关系。
>
> 读写分离，数据同步
>
> - 主： 可读写
> - 从： 只可读

```powershell
# redis5.0 以前（新老版本都能使用）
slaveof <masterip> <masterport>
# redis5.0 以后
replicaof <masterip> <masterport>
```

**进入r2和r3，将其设置为r1的从节点**

```powershell
docker exec -it r2 redis-cli -p 7002
# 查看节点情况
info replication
# 建立主从关系 主节点ip 主节点端口
slaveof localhost 7001
```

> ==测试==：在r1中写入数据，r1和r2都可读取，但r2不可写入

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406232343749.png" alt="image-20240623234349682" style="zoom:67%;" />

### 主从同步原理

![image-20240623181936038](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406231819121.png)

![image-20240623182706813](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406231827903.png)

#### 主从集群优化

![image-20240623183503097](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406231835200.png)

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406231835015.png" alt="image-20240623183542938" style="zoom: 50%;" />

### 哨兵原理

![image-20240623191745942](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406231917043.png)

![image-20240623192014235](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406231920341.png)

#### 选举新的master

![image-20240623192219840](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406231922893.png)

#### 如何实现故障转移

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406232053427.png" alt="image-20240623205314306" style="zoom:67%;" />

#### 总结

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406232053147.png" alt="image-20240623205359079" style="zoom:67%;" />



## Redis Sentinel （哨兵集群）



> 1. **监控与故障检测**：哨兵是一个监控和==故障恢复==系统，用于监控Redis主从架构中的主节点状态。当主节点不可达时，哨兵会选举一个从节点作为新的主节点，实现故障转移。
> 2. **主从复制**：哨兵是建立在==主从复制==基础上的，不提供数据分片功能，所有数据依然集中在主节点上，从节点仅作为数据备份和读取扩展。
> 3. **中心化管理**：哨兵系统包含多个哨兵实例，它们协同工作来监控Redis主节点，构成了一个中心化的监控系统，但数据访问仍然是去中心化的（客户端直接与主/从节点通信）。
> 4. **配置相对简单**：相对于Redis Cluster，哨兵的配置和管理较为简单，适合需要高可用但对数据分片需求不高的场景。
> 5. **不支持数据自动分片**：哨兵不提供数据自动分片功能，所有数据都在主节点上，然后复制到从节点。

哨兵集群结构：

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406251713811.png" alt="image-20240625164536984" style="zoom: 80%;" />

目录内容

- r1
  - redis.windows.conf
- s1
  - sentinel.conf

其中 `redis.windows.conf` 内容为

```bash
# r1 
# 从节点监听的端口
port 7001
# 绑定到本地IPv4地址
bind 127.0.0.1
# 开启AOF持久化
appendonly yes

# r2
port 7002
bind 127.0.0.1
appendonly yes
# 设置该从节点复制主节点，主节点地址为127.0.0.1，端口为7001
slaveof 127.0.0.1 7001


# r3
port 7003
bind 127.0.0.1
appendonly yes
slaveof 127.0.0.1 7001
```

其中 `sentinel.conf` 内容为：

```bash
# 当前Sentinel服务监听的端口， s2s3只改变端口 27002/27003
port 27001
# 监控名为mymaster的主节点，位于127.0.0.1:7001，至少需要2个Sentinel认为主节点失联才能发起故障转移
sentinel monitor mymaster 127.0.0.1 7001 2
# 设置主节点失联判断的时间阈值为5000毫秒
sentinel down-after-milliseconds mymaster 5000
# 设置故障转移操作的超时时间为60000毫秒
sentinel failover-timeout mymaster 60000
```

> 说明：
>
> `sentinel monitor mymaster 127.0.0.1 7001 2`：指定集群的主节点信息
>
> - hmaster： 主节点名称，自定义
> - 127.0.0.1  7001：主节点的ip和端口
> - 2：认定master下线时的quorum值（当有quorum个sentinel认为主观下线时，就确定下线）
>
> `sentinel down-after-milliseconds mymaster 5000`：声明master节点超时多久后被标记下线
>
> `sentinel failover-timeout hmaster 60000`：在第一次故障转移失败后多久再次重试

==关键：docker compose.yml==

在该目录下进入cmd/powershell：`docker compose up -d` 运行

```yaml
services:
  r1:
    image: redis
    container_name: r1
    network_mode: "host"
    entrypoint: [ "redis-server", "--port", "7001" ]
    volumes:
      - /f/redis/r1/redis.windows.conf:/etc/redis/redis.conf
  r2:
    image: redis
    container_name: r2
    network_mode: "host"
    entrypoint:
      [
        "redis-server",
        "--port",
        "7002",
        "--slaveof",
        "localhost",
        "7001"
      ]
    volumes:
      - /f/redis/r2/redis.windows.conf:/etc/redis/redis.conf
  r3:
    image: redis
    container_name: r3
    network_mode: "host"
    entrypoint:
      [
        "redis-server",
        "--port",
        "7003",
        "--slaveof",
        "localhost",
        "7001"
      ]
    volumes:
      - /f/redis/r3/redis.windows.conf:/etc/redis/redis.conf
  s1:
    image: redis
    container_name: s1
    volumes:
      - /f/redis/r1/sentinel.conf:/etc/redis/sentinel.conf
    network_mode: "host"
    entrypoint:
      [
        "redis-sentinel",
        "/etc/redis/sentinel.conf",
        "--port",
        "27001"
      ]
  s2:
    image: redis
    container_name: s2
    volumes:
      - /f/redis/r2/sentinel.fonf:/etc/redis/sentinel.conf
    network_mode: "host"
    entrypoint:
      [
        "redis-sentinel",
        "/etc/redis/sentinel.conf",
        "--port",
        "27002"
      ]
  s3:
    image: redis
    container_name: s3
    volumes:
      - /f/redis/r3/sentinel.conf:/etc/redis/sentinel.conf
    network_mode: "host"
    entrypoint:
      [
        "redis-sentinel",
        "/etc/redis/sentinel.conf",
        "--port",
        "27003"
      ]

```

==进入容器内查看信息==

> 配置了`network_mode: "host"`，这意味着容器直接使用宿主机的网络栈，不需要指定IP地址，因为默认就是本机
>

```powershell
#进入r1（主）查看信息，并写入数据
docker exec -it r1 redis-cli -p 7001
#查看状态信息
info replication 

#测试数据
set age 18
get age

#进入r2,r3 尝试获取数据
docker exec -it r2 redis-cli -p 7002
get age
#停止容器r1 ，查看主从切换
docker stop r1
```

> 因为**集群**中的每个节点都是相等的，并没有明确的主从关系

```powershell
# 在r1容器停止后，sentinel会尝试进行主从切换，可以在 r2,r3 的redis-cli 中多次尝试写入数据 ，
docker exec -it r2 redis-cli -p 7002
set name zyr
# 127.0.0.1:7002>Error: Server closed the connection 请多次尝试（可能会有错误）
set name zyr
# 127.0.0.1:7002>OK
```

> **遗留问题**（不影响运行）
>
> sentinel频繁在ipv4和ipv6之间切换
>
> 1:X 25 Jun 2024 14:05:46.801 * +sentinel-address-switch master mymaster ::1 7002 ip ::1 port 27002 for ae0f41d45bb522e1cd2359c75994871cad2b1c43
> 1:X 25 Jun 2024 14:05:46.807 * +sentinel-address-switch master mymaster ::1 7002 ip 127.0.0.1 port 27002 for ae0f41d45bb522e1cd2359c75994871cad2b1c43
> 1:X 25 Jun 2024 14:05:46.812 * +sentinel-address-switch master mymaster ::1 7002 ip ::1 port 27002 for ae0f41d45bb522e1cd2359c75994871cad2b1c43



------



# Redis分片集群

> 主从和哨兵可以解决高可用、高并发读的问题。但是依然有两个问题没有解决：
>
> - 海量数据存储问题
> - 高并发写的问题
>
> 使用分片集群可以解决上述问题，分片集群特征：
>
> - 集群中有多个master，每个master保存不同数据
> - 每个master都可以有多个slave节点
> - master之间通过ping监测彼此健康状态

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406302118682.png" alt="image-20240630211835543" style="zoom: 80%;" />

## Redis Cluster (集群)

> 1. **数据分片**：Redis Cluster支持==数据分片==（sharding），即将数据自动分布在多个节点上。每个节点负责一部分数据，可以显著提升Redis的存储容量和处理能力。
> 2. **自动故障恢复**：当某个节点发生故障时，Redis Cluster能够==自动检测==并重新分配其上的slot（数据分片单位）到其他健康的节点上，从而实现==故障转移==，无需人工干预。
> 3. **无中心架构**：Redis Cluster是一个无中心的分布式系统，每个节点既是服务器也是客户端，可以直接响应客户端请求或转发请求到正确的节点。
> 4. **扩展性**：容易水平扩展，可以通过添加更多的节点来增加存储容量和处理能力。
> 5. **配置复杂度**：相较于哨兵，配置和管理Redis Cluster较为复杂，需要对集群拓扑有深入了解。

### 搭建集群

==至少需要三个master节点，所以至少6个节点==

目录结构

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406261457387.png" alt="image-20240626144237456" style="zoom:67%;" />

- r1
  - conf
    - redis.windows.conf

1. `redis.windows.conf`

```bash
# 其余 的只改变节点 7002,7003....
port 7001
# AOF持久化 默认no
appendonly yes
# 开启集群 默认no
cluster-enabled yes
#指定配置文件名(会自动生成)
cluster-config-file nodes.conf
cluster-node-timeout 5000
```

2. `docker-compose.yml`

```yml
services:
  r1:
    image: redis
    container_name: r1
    ports:
      - "7001:7001"
    command: redis-server /etc/redis/redis.conf --cluster-enabled yes --cluster-node-timeout 5000 --appendonly yes
    volumes:
      - /f/redis/r1/conf/redis.windows.conf:/etc/redis/redis.conf
  r2:
    image: redis
    container_name: r2
    ports:
      - "7002:7002"
    command: redis-server /etc/redis/redis.conf --cluster-enabled yes  --cluster-node-timeout 5000 --appendonly yes
    volumes:
      - /f/redis/r2/conf/redis.windows.conf:/etc/redis/redis.conf
  r3:
    image: redis
    container_name: r3
    ports:
      - "7003:7003"
    command: redis-server /etc/redis/redis.conf --cluster-enabled yes --cluster-node-timeout 5000 --appendonly yes
    volumes:
      - /f/redis/r3/conf/redis.windows.conf:/etc/redis/redis.conf
  r4:
    image: redis
    container_name: r4
    ports:
      - "7004:7004"
    command: redis-server /etc/redis/redis.conf --cluster-enabled yes --cluster-node-timeout 5000 --appendonly yes
    volumes:
      - /f/redis/r4/conf/redis.windows.conf:/etc/redis/redis.conf
  r5:
    image: redis
    container_name: r5
    ports:
      - "7005:7005"
    command: redis-server /etc/redis/redis.conf --cluster-enabled yes --cluster-node-timeout 5000 --appendonly yes
    volumes:
      - /f/redis/r5/conf/redis.windows.conf:/etc/redis/redis.conf
  r6:
    image: redis
    container_name: r6
    ports:
      - "7006:7006"
    command: redis-server /etc/redis/redis.conf --cluster-enabled yes --cluster-node-timeout 5000 --appendonly yes
    volumes:
      - /f/redis/r6/conf/redis.windows.conf:/etc/redis/redis.conf


```

==或者==

`docker-compose.yml` 直接使用宿主主机的网络信息，包括 IP 地址、端口等

```yml
services:
	r1:
	image: redis
	container_name: r1
	network-mode: "host"
	entrypoint: ["redis-server","--port","7001","--cluster-enable","yes","--cluster-config-flie","node.conf"]
	r2:
	...
```



3. 在终端执行 `docker-compose up -d`

4. 进入任意一个redis节点执行创建集群命令

   ```bash
   # 解析docker中各容器的ip
   docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' r1
   # 获取到 172.22.0.2
   ```

   ```bash
    # 合并创建集群
   docker exec -it r1 redis-cli --cluster create 172.22.0.2:7001 172.22.0.7:7002 172.22.0.5:7003 172.22.0.6:7004 172.22.0.4:7005 172.22.0.3:7006 --cluster-replicas 1
    
    #分段
   docker exec -it r1 bash
    #创建
   redis-cli --cluster create --cluster-replicas 1 ...
   ```

   命令说明：

   - `redis-cli --cluster`:代表集群操作命令
   - `create`:代表创建集群
   - `--cluster-replicas 1`:指定集群中每个`master`的副本个数为 1
     - 此时`节点总数/（replicas+1）`得到的就是`master`的数量`n`。因此节点列表中的前`n`个节点就是`master`，其他都是`slave`节点，随机分配到不同的`master`

是否 设置了3个主节点，并各自分配了一个从节点 **==yes==**

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406261457854.png" alt="image-20240626144820779" style="zoom:67%;" />

![image-20240626145647003](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406261457672.png)

### 测试

> 原主节点为 r1,r2,r3
>
> 在从节点r4中尝试写入数据时，实际是发生了==重定向指令==，起始是主节点在写入数据

```powershell
# 进入r1节点，进行 读写测试
docker exec -it r1 redis-cli -c -h 172.22.0.2 -p 7001
set age 18
get age
# 查看该集群状态
cluster info
# 查看集群主从关系
cluster nodes

# 进入从节点 r4 进行读写测试
 set name zyr
#-> Redirected to slot [5798] located at 172.22.0.3:7006


# 故障转移，尝试关闭主节点r2，集群将提升原主节点的从节点r6，为新的主节点
docker exec -it r2 redis-cli -c -h 172.22.0.7 -p 7002
#停止当前redis节点
shutdown
cluster nodes
```

## 散列插槽

> 在Redis集群中，共有16384个hash slots，集群中的每一个master节点都会分配到一定数量的hash slots，Redis数据不是与节点绑定，而是与插槽slot绑定。当我们读写数据时，Redis基于CRC16算法对key做hash运算，计算出key的slot值。然后得到slot所在的Redis节点执行读写操作
>
>  任意一个节点都可以进行读写

redis在计算key的hash值时不一定根据整个key计算，分两种情况：

- 当key中包含{}是，根据{}之间的字符串计算hash slot
- 当key中不包含{}时，根据整个key字符串计算hash slot

例如：key 是num，那么就根据num计算，如果是{itcast}num,则根据itcast计算



------



# Redis数据结构

## RedisObject

Redis中任意数据类型的键和值都会被封装为一个RedisObject，也叫做Redis对象，源码如下：

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406302151516.png" alt="image-20240630215116343" style="zoom:80%;" />

## SkipList

SkipList（跳表）首先是链表，但与传统链表相比有几点差异：

- 元素按照升序排列存储
- 节点可能包含多个指针，指针跨度不同

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406302158632.png" alt="image-20240630215833453" style="zoom:80%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406302159795.png" alt="image-20240630215906649" style="zoom:80%;" />

## SortedSet

SortedSet数据结构的特点是：

- 每组数据都包含score和member
- member唯一
- 可根据score排序

![image-20240630220235645](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406302202796.png)

![image-20240630220248465](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406302202610.png)



------



# Redis内存回收

## 过期key处理

Redis的本身是键值型数据库，其所有数据都存在一个redisDB的结构体中，其中包含==两个哈希表==：

- dict：保存Redis中所有的==键值对==
- expires：保存Redis中所有的设置了过期时间的==Key==及其==到期时间==（**写入时间+TTL**）

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407012129744.png" alt="image-20240701212921573" style="zoom:80%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407012132479.png" alt="image-20240701213256307" style="zoom: 67%;" />

## 内存淘汰策略

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407012136263.png" alt="image-20240701213625073" style="zoom:80%;" />

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407012142464.png" alt="image-20240701214204258" style="zoom:80%;" />

# Redis缓存

## 缓存一致性

先操纵数据库，后删redis

![image-20240702134650864](https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407021346041.png)

<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202407021347165.png" alt="image-20240702134726003" style="zoom:80%;" />

## 缓存穿透

...详细看redis篇
