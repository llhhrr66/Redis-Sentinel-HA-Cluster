# Redis Sentinel 高可用集群

基于 Docker Compose 的 Redis Sentinel 高可用集群部署方案，包含 1 个主节点、2 个从节点和 3 个哨兵节点。

## 🏗️ 架构概览

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Redis Master  │    │  Redis Slave1   │    │  Redis Slave2   │
│   172.19.0.2    │◄───┤   172.19.0.3    │    │   172.19.0.4    │
│   Port: 6379    │    │   Port: 6380    │    │   Port: 6381    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲                       ▲                       ▲
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │   Sentinel 1    │    │   Sentinel 2    │    │   Sentinel 3    │
    │   Port: 26379   │    │   Port: 26380   │    │   Port: 26381   │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 📋 功能特性

- ✅ **主从复制**：自动数据同步，读写分离
- ✅ **自动故障转移**：主节点故障时自动选举新主节点
- ✅ **高可用性**：3个哨兵节点确保集群稳定性
- ✅ **数据持久化**：AOF 持久化保证数据安全
- ✅ **容器化部署**：基于 Docker，易于部署和管理

## 🚀 快速开始

### 环境要求

- Docker 20.0+
- Docker Compose 2.0+
- 至少 4GB 可用内存

### 1. 克隆项目

```bash
git clone <your-repo-url>
cd redis_cluster
```

### 2. 启动集群

```bash
# 启动所有服务
docker-compose up -d

# 查看服务状态
docker-compose ps
```

### 3. 获取主节点IP并更新配置

```bash
# 获取主节点IP地址
docker inspect redis-master | grep "IPAddress"
```

将获取到的IP地址更新到所有哨兵配置文件中：
- `sentinel1/sentinel.conf`
- `sentinel2/sentinel.conf`  
- `sentinel3/sentinel.conf`

```conf
sentinel monitor mymaster <主节点IP> 6379 2
```

### 4. 重启服务

```bash
docker-compose down
docker-compose up -d
```

## 🧪 测试验证

### 基础连接测试

```bash
# 测试 Redis 节点
docker exec -it redis-master redis-cli ping
docker exec -it redis-slave1 redis-cli ping
docker exec -it redis-slave2 redis-cli ping

# 测试哨兵节点
docker exec -it sentinel1 redis-cli -p 26379 ping
```

### 主从复制测试

```bash
# 主节点写入数据
docker exec -it redis-master redis-cli SET test:key "Hello Redis"

# 从节点读取数据
docker exec -it redis-slave1 redis-cli GET test:key
docker exec -it redis-slave2 redis-cli GET test:key
```

### 故障转移测试

```bash
# 1. 停止主节点
docker stop redis-master

# 2. 等待故障转移（30-60秒）
sleep 60

# 3. 查看新主节点
docker exec -it sentinel1 redis-cli -p 26379 SENTINEL masters

# 4. 重启原主节点
docker start redis-master

# 5. 验证原主节点变成从节点
docker exec -it redis-master redis-cli INFO replication
```

## 📊 监控命令

### 查看集群状态

```bash
# 查看主节点信息
docker exec -it sentinel1 redis-cli -p 26379 SENTINEL masters

# 查看从节点信息
docker exec -it sentinel1 redis-cli -p 26379 SENTINEL slaves mymaster

# 查看哨兵信息
docker exec -it sentinel1 redis-cli -p 26379 SENTINEL sentinels mymaster
```

### 查看复制状态

```bash
# 主节点复制信息
docker exec -it redis-master redis-cli INFO replication

# 从节点复制信息
docker exec -it redis-slave1 redis-cli INFO replication
```

## 🗂️ 项目结构

```
redis_cluster/
├── README.md                   # 项目说明文档
├── docker-compose.yml          # Docker Compose 配置
├── master/
│   └── redis.conf             # 主节点配置
├── slave1/
│   └── redis.conf             # 从节点1配置
├── slave2/
│   └── redis.conf             # 从节点2配置
├── sentinel1/
│   └── sentinel.conf          # 哨兵1配置
├── sentinel2/
│   └── sentinel.conf          # 哨兵2配置
└── sentinel3/
    └── sentinel.conf          # 哨兵3配置
```

## ⚙️ 配置说明

### 端口映射

| 服务 | 容器端口 | 主机端口 | 说明 |
|------|----------|----------|------|
| redis-master | 6379 | 6379 | 主节点 |
| redis-slave1 | 6379 | 6380 | 从节点1 |
| redis-slave2 | 6379 | 6381 | 从节点2 |
| sentinel1 | 26379 | 26379 | 哨兵1 |
| sentinel2 | 26379 | 26380 | 哨兵2 |
| sentinel3 | 26379 | 26381 | 哨兵3 |

### 关键配置参数

#### Redis 配置
- `replicaof`: 指定主节点地址
- `appendonly yes`: 启用AOF持久化
- `replica-read-only yes`: 从节点只读模式

#### Sentinel 配置
- `sentinel monitor`: 监控的主节点信息
- `down-after-milliseconds`: 判定节点下线时间（30秒）
- `failover-timeout`: 故障转移超时时间（3分钟）
- `parallel-syncs`: 同时同步的从节点数（1个）

## 🔧 故障排除

### 常见问题

#### 1. 哨兵一直重启
**原因**: 哨兵配置中的主节点IP地址不正确

**解决方案**:
```bash
# 获取正确的主节点IP
docker inspect redis-master | grep "IPAddress"

# 更新哨兵配置文件
# 重启服务
docker-compose down && docker-compose up -d
```

#### 2. 从节点无法连接主节点
**原因**: 网络连通性问题或主节点未启动

**解决方案**:
```bash
# 检查网络连通性
docker exec -it redis-slave1 ping redis-master

# 检查主节点状态
docker exec -it redis-master redis-cli ping

# 查看从节点日志
docker-compose logs redis-slave1
```

#### 3. 故障转移不工作
**原因**: 哨兵数量不足或仲裁数配置错误

**解决方案**:
- 确保至少有3个哨兵节点运行
- 检查仲裁数设置（当前为2，适合3个哨兵）

### 查看日志

```bash
# 查看所有服务日志
docker-compose logs

# 查看特定服务日志
docker-compose logs redis-master
docker-compose logs sentinel1

# 实时查看日志
docker-compose logs -f
```

## 🛡️ 生产环境建议

### 安全配置
- 启用密码认证 (`requirepass`)
- 配置防火墙规则
- 使用 TLS 加密连接
- 限制网络访问

### 性能优化
- 根据业务调整 `maxmemory` 和淘汰策略
- 优化 `appendfsync` 策略
- 监控内存和CPU使用情况
- 配置合适的超时值

### 高可用建议
- 将哨兵部署在不同的物理机器上
- 设置合理的故障检测时间
- 定期备份数据
- 监控集群健康状态

## 📈 性能测试

### 基准测试

```bash
# 使用 redis-benchmark 进行性能测试
docker exec -it redis-master redis-benchmark -h localhost -p 6379 -n 10000 -c 100

# 测试读性能（从节点）
docker exec -it redis-slave1 redis-benchmark -h localhost -p 6379 -n 10000 -c 100 -t get
```

## 🤝 贡献指南

1. Fork 本项目
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

## 📄 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情

## 🙏 致谢

- [Redis](https://redis.io/) - 高性能内存数据库
- [Docker](https://www.docker.com/) - 容器化平台
- [Docker Compose](https://docs.docker.com/compose/) - 多容器应用定义工具

## 📞 支持

如果你在使用过程中遇到问题，请：

1. 查看 [故障排除](#🔧-故障排除) 部分
2. 搜索现有的 [Issues](../../issues)
3. 创建新的 [Issue](../../issues/new) 描述你的问题

---

⭐ 如果这个项目对你有帮助，请给个 Star！