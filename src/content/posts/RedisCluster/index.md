---
title: Redis 集群深度学习指南
published: 2025-09-20
description: "Redis 集群是 Redis 官方提供的分布式解决方案，用于实现数据的水平扩展和高可用性。"
image: "./cover.png"
tags: ["Redis", "集群"]
category: Guides
draft: false
---

# Redis 集群深度学习指南

## 1. Redis 集群概述

### 1.1 Redis 集群的基本概念

Redis 集群（Redis Cluster）是 Redis 官方提供的分布式解决方案，它通过将数据分散存储在多个 Redis 节点上来实现数据的水平扩展。Redis 集群的核心思想是**数据分片**和**去中心化**。

**底层原理**：

- Redis 集群采用无中心架构，每个节点都保存集群的完整拓扑信息
- 使用 Gossip 协议进行节点间通信和状态同步
- 通过 CRC16 算法将键映射到 16384 个哈希槽中

### 1.2 Redis 集群与单机模式的区别

| 特性         | 单机模式                 | 集群模式                 |
| ------------ | ------------------------ | ------------------------ |
| **数据存储** | 所有数据存储在单个实例中 | 数据分片存储在多个节点中 |
| **可扩展性** | 垂直扩展（升级硬件）     | 水平扩展（增加节点）     |
| **单点故障** | 存在单点故障风险         | 通过主从复制避免单点故障 |
| **内存限制** | 受单机内存限制           | 可突破单机内存限制       |
| **网络开销** | 无网络开销               | 存在节点间通信开销       |

### 1.3 Redis 集群的优缺点

**优点**：

- **高可用性**：支持主从复制和自动故障转移
- **水平扩展**：可以线性增加存储容量和处理能力
- **负载分散**：读写请求分散到多个节点
- **数据安全**：数据分片存储，降低数据丢失风险

**缺点**：

- **复杂性增加**：部署、维护、监控复杂度提升
- **跨节点操作限制**：不支持跨 slot 的批量操作
- **网络延迟**：节点间通信带来额外延迟
- **数据一致性**：分布式环境下的一致性保证更复杂

### 1.4 Redis 集群的应用场景

- **大规模缓存**：需要存储大量热点数据
- **高并发读写**：单机无法承载的高并发场景
- **数据量超过单机内存**：TB 级别的数据存储需求
- **高可用要求**：对服务可用性要求极高的业务

## 2. Redis 集群架构深度解析

### 2.1 Redis 集群的组成

#### 2.1.1 主节点（Master Node）

**底层机制**：

- 每个主节点负责处理特定 hash slots 的读写请求
- 主节点维护自己负责的 slot 范围信息
- 主节点通过 Gossip 协议与其他节点交换集群状态信息

```bash
# 主节点核心配置
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000
cluster-announce-ip 192.168.1.100
cluster-announce-port 6379
```

#### 2.1.2 从节点（Slave Node）

**复制原理**：

- 从节点通过**异步复制**同步主节点数据
- 使用 RDB 快照+AOF 增量的方式进行数据同步
- 从节点定期向主节点发送`REPLCONF ACK`确认复制进度

**故障转移机制**：

```
主节点故障检测流程：
1. 从节点检测主节点超时（cluster-node-timeout）
2. 从节点标记主节点为PFAIL状态
3. 通过Gossip协议传播故障信息
4. 当大多数节点确认故障时，标记为FAIL
5. 从节点发起选举，成为新主节点
```

#### 2.1.3 代理节点概念澄清

**注意**：Redis 官方集群没有专门的代理节点，每个节点都具有路由功能。客户端可以连接任意节点，节点会将请求转发到正确的节点。

### 2.2 数据分片（Sharding）深度解析

#### 2.2.1 Hash Slot 机制

**底层实现**：

```c
// Redis源码中的slot计算
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e;
    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;
    if (s == keylen) return crc16(key,keylen) & 0x3FFF;
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;
    if (e == keylen || e == s+1) return crc16(key,keylen) & 0x3FFF;
    return crc16(key+s+1,e-s-1) & 0x3FFF;
}
```

**Hash Tag 机制**：

```bash
# 这些key会被分配到同一个slot
user:{123}:profile
user:{123}:settings
user:{123}:orders
```

#### 2.2.2 Slot 分配策略

**均匀分配算法**：

```
总slot数：16384
节点数：n
每个节点slot数 ≈ 16384 / n

例如：6个节点的分配
Node1: 0-2729     (2730个slot)
Node2: 2730-5459  (2730个slot)
Node3: 5460-8191  (2732个slot)
Node4: 8192-10921 (2730个slot)
Node5: 10922-13651(2730个slot)
Node6: 13652-16383(2732个slot)
```

## 3. 集群部署与配置实战

### 3.1 手动搭建 Redis 集群环境

#### 3.1.1 环境准备

```bash
# 1. 创建集群目录结构
mkdir -p /opt/redis-cluster/{7000,7001,7002,7003,7004,7005}

# 2. 准备Redis配置文件模板
cat > /opt/redis-cluster/redis-template.conf << 'EOF'
port PORT_PLACEHOLDER
cluster-enabled yes
cluster-config-file nodes-PORT_PLACEHOLDER.conf
cluster-node-timeout 15000
cluster-announce-ip 192.168.1.100
cluster-announce-port PORT_PLACEHOLDER
cluster-announce-bus-port 1PORT_PLACEHOLDER
appendonly yes
appendfilename "appendonly-PORT_PLACEHOLDER.aof"
dbfilename dump-PORT_PLACEHOLDER.rdb
dir /opt/redis-cluster/PORT_PLACEHOLDER
pidfile /var/run/redis-PORT_PLACEHOLDER.pid
logfile /opt/redis-cluster/PORT_PLACEHOLDER/redis-PORT_PLACEHOLDER.log
EOF
```

#### 3.1.2 生成配置文件

```bash
# 批量生成配置文件
for port in 7000 7001 7002 7003 7004 7005; do
    sed "s/PORT_PLACEHOLDER/$port/g" /opt/redis-cluster/redis-template.conf > /opt/redis-cluster/$port/redis-$port.conf
done
```

#### 3.1.3 启动 Redis 节点

```bash
# 启动所有节点
for port in 7000 7001 7002 7003 7004 7005; do
    redis-server /opt/redis-cluster/$port/redis-$port.conf &
done

# 验证节点启动
ps aux | grep redis-server
netstat -tlnp | grep :700
```

### 3.2 创建集群

#### 3.2.1 使用 redis-cli 创建集群

```bash
# Redis 5.0+版本
redis-cli --cluster create \
192.168.1.100:7000 \
192.168.1.100:7001 \
192.168.1.100:7002 \
192.168.1.100:7003 \
192.168.1.100:7004 \
192.168.1.100:7005 \
--cluster-replicas 1

# 参数说明：
# --cluster-replicas 1: 每个主节点有1个从节点
# 结果：3个主节点，3个从节点
```

#### 3.2.2 集群初始化底层过程

```
1. 节点发现：各节点通过MEET命令建立连接
2. Slot分配：16384个slot均匀分配给主节点
3. 主从关系建立：从节点复制指定主节点
4. 集群状态同步：通过Gossip协议同步拓扑信息
```

### 3.3 集群配置深度解析

#### 3.3.1 关键配置参数

```bash
# 集群基础配置
cluster-enabled yes                    # 启用集群模式
cluster-config-file nodes.conf        # 集群配置文件
cluster-node-timeout 15000           # 节点超时时间(ms)
cluster-require-full-coverage no     # 是否要求完整覆盖

# 网络配置
cluster-announce-ip 192.168.1.100    # 对外宣告IP
cluster-announce-port 6379            # 对外宣告端口
cluster-announce-bus-port 16379      # 集群总线端口

# 故障转移配置
cluster-slave-validity-factor 10      # 从节点有效性因子
cluster-migration-barrier 1          # 主节点最少从节点数
cluster-replica-no-failover no       # 从节点是否参与故障转移
```

#### 3.3.2 网络通信机制

**集群总线协议**：

- 每个 Redis 节点都有两个 TCP 端口：服务端口和集群总线端口
- 服务端口：处理客户端请求（如 6379）
- 集群总线端口：节点间通信（服务端口+10000，如 16379）

**通信协议栈**：

```
应用层：Gossip协议消息
传输层：TCP连接
网络层：IP路由
数据链路层：以太网帧
```

## 4. 集群管理与操作实战

### 4.1 集群状态监控

#### 4.1.1 查看集群基本信息

```bash
# 连接任意集群节点
redis-cli -c -h 192.168.1.100 -p 7000

# 查看集群信息
CLUSTER INFO
```

**关键指标解读**：

```
cluster_state:ok                    # 集群状态
cluster_slots_assigned:16384       # 已分配slot数
cluster_slots_ok:16384             # 正常slot数
cluster_slots_pfail:0              # 疑似故障slot数
cluster_slots_fail:0               # 故障slot数
cluster_known_nodes:6              # 已知节点数
cluster_size:3                     # 集群大小（主节点数）
```

#### 4.1.2 查看节点详细信息

```bash
# 查看集群节点
CLUSTER NODES

# 输出解析：
# 节点ID 节点IP:端口@集群端口 角色 主节点ID slot范围 连接状态
```

### 4.2 动态添加节点

#### 4.2.1 添加主节点

```bash
# 1. 启动新节点
redis-server /opt/redis-cluster/7006/redis-7006.conf &

# 2. 添加节点到集群
redis-cli --cluster add-node 192.168.1.100:7006 192.168.1.100:7000

# 3. 重新分配slot
redis-cli --cluster reshard 192.168.1.100:7000
```

#### 4.2.2 添加从节点

```bash
# 添加从节点并指定主节点
redis-cli --cluster add-node 192.168.1.100:7007 192.168.1.100:7000 --cluster-slave --cluster-master-id <master-node-id>
```

### 4.3 数据迁移与重新分配（Resharding）

#### 4.3.1 Resharding 底层机制

```
1. 源节点准备迁移：标记slot为MIGRATING状态
2. 目标节点准备接收：标记slot为IMPORTING状态
3. 获取slot中的所有key：CLUSTER GETKEYSINSLOT
4. 逐个迁移key：MIGRATE命令
5. 更新slot归属：CLUSTER SETSLOT
6. 广播slot变更：Gossip协议同步
```

#### 4.3.2 手动迁移 slot

```bash
# 1. 设置源节点slot状态为migrating
CLUSTER SETSLOT 1000 MIGRATING <target-node-id>

# 2. 设置目标节点slot状态为importing
CLUSTER SETSLOT 1000 IMPORTING <source-node-id>

# 3. 获取slot中的key
CLUSTER GETKEYSINSLOT 1000 100

# 4. 迁移key
MIGRATE 192.168.1.100 7001 key 0 5000

# 5. 完成迁移
CLUSTER SETSLOT 1000 NODE <target-node-id>
```

### 4.4 删除节点

#### 4.4.1 删除从节点

```bash
# 直接删除从节点
redis-cli --cluster del-node 192.168.1.100:7000 <node-id>
```

#### 4.4.2 删除主节点

```bash
# 1. 先迁移slot到其他节点
redis-cli --cluster reshard 192.168.1.100:7000 \
--cluster-from <node-id> \
--cluster-to <target-node-id> \
--cluster-slots 5461

# 2. 删除节点
redis-cli --cluster del-node 192.168.1.100:7000 <node-id>
```

## 5. 集群的高可用性与容错机制

### 5.1 主从复制深度解析

#### 5.1.1 复制过程

```
全量同步过程：
1. 从节点发送PSYNC命令
2. 主节点判断是否首次同步
3. 主节点执行BGSAVE生成RDB
4. 主节点发送RDB给从节点
5. 从节点载入RDB数据
6. 主节点发送复制积压缓冲区数据

增量同步过程：
1. 主节点记录写命令到复制积压缓冲区
2. 主节点异步发送命令到从节点
3. 从节点执行收到的命令
```

#### 5.1.2 复制配置优化

```bash
# 主节点配置
replica-serve-stale-data yes        # 从节点断线时是否继续服务
replica-read-only yes               # 从节点只读模式
repl-ping-replica-period 10        # ping周期
repl-timeout 60                     # 复制超时时间

# 复制积压缓冲区
repl-backlog-size 1mb              # 积压缓冲区大小
repl-backlog-ttl 3600              # 积压缓冲区TTL
```

### 5.2 故障检测与转移

#### 5.2.1 故障检测机制

**PFAIL 检测**：

```c
// 节点标记为PFAIL的条件
if (now - node->ping_sent > server.cluster_node_timeout/2 &&
    now - node->pong_received > server.cluster_node_timeout) {
    node->flags |= CLUSTER_NODE_PFAIL;
}
```

**FAIL 确认**：

```
1. 节点收到其他节点的PFAIL报告
2. 统计报告PFAIL的节点数量
3. 如果超过集群节点数量的一半，标记为FAIL
4. 广播FAIL状态到整个集群
```

#### 5.2.2 自动故障转移

**选举过程**：

```
1. 从节点检测主节点FAIL
2. 从节点延迟选举（避免同时选举）
3. 从节点广播选举请求
4. 主节点投票给从节点
5. 获得大多数票的从节点成为新主节点
6. 新主节点广播成为主节点的消息
```

**延迟计算公式**：

```
DELAY = 500ms + random(0~500ms) + SLAVE_RANK * 1000ms
```

### 5.3 数据一致性保证

#### 5.3.1 一致性级别

```bash
# CAP定理在Redis集群中的体现
# C（一致性）：最终一致性，非强一致性
# A（可用性）：在网络分区时保持服务可用
# P（分区容错性）：能够处理网络分区

# 一致性相关配置
min-replicas-to-write 1           # 最少写入从节点数
min-replicas-max-lag 10           # 从节点最大延迟秒数
```

#### 5.3.2 脑裂防护

```bash
# 防止脑裂的配置
cluster-require-full-coverage yes  # 要求完整slot覆盖
cluster-node-timeout 15000        # 合理的超时时间
cluster-slave-validity-factor 0   # 从节点有效性检查
```

## 6. Redis 集群监控与优化

### 6.1 性能监控体系

#### 6.1.1 关键性能指标

```bash
# 1. 集群级别指标
redis-cli --cluster info 192.168.1.100:7000

# 2. 节点级别指标
redis-cli -h 192.168.1.100 -p 7000 INFO stats

# 3. 实时监控命令
redis-cli -h 192.168.1.100 -p 7000 --latency-history -i 1
```

**核心监控指标**：

- **QPS**：每秒查询数
- **延迟**：命令执行延迟分布
- **内存使用率**：各节点内存使用情况
- **网络流量**：集群间通信流量
- **错误率**：命令执行错误比例
- **slot 分布**：数据分布均匀性

#### 6.1.2 监控脚本示例

```bash
#!/bin/bash
# Redis集群监控脚本

NODES="192.168.1.100:7000 192.168.1.100:7001 192.168.1.100:7002"

for node in $NODES; do
    echo "=== $node ==="
    redis-cli -h ${node%:*} -p ${node#*:} INFO memory | grep used_memory_human
    redis-cli -h ${node%:*} -p ${node#*:} INFO stats | grep instantaneous_ops_per_sec
    redis-cli -h ${node%:*} -p ${node#*:} CLUSTER INFO | grep cluster_state
done
```

### 6.2 性能瓶颈识别与优化

#### 6.2.1 常见性能瓶颈

**热点数据问题**：

```bash
# 识别热点slot
redis-cli --cluster call 192.168.1.100:7000 CLUSTER COUNTKEYSINSLOT {slot_number}

# 解决方案：
# 1. 使用Hash Tag分散热点
# 2. 增加从节点分担读压力
# 3. 使用Redis Streams等数据结构
```

**网络带宽瓶颈**：

```bash
# 监控网络使用
iftop -i eth0
nethogs eth0

# 优化策略：
# 1. 压缩数据传输
# 2. 减少跨节点操作
# 3. 优化序列化方式
```

#### 6.2.2 内存优化

```bash
# 内存使用分析
redis-cli -h 192.168.1.100 -p 7000 MEMORY USAGE keyname
redis-cli -h 192.168.1.100 -p 7000 MEMORY STATS

# 内存优化配置
maxmemory 2gb                      # 最大内存限制
maxmemory-policy allkeys-lru       # 内存淘汰策略
hash-max-ziplist-entries 512       # 压缩列表优化
hash-max-ziplist-value 64
```

### 6.3 网络与资源优化

#### 6.3.1 网络优化

```bash
# TCP参数优化
echo 'net.core.somaxconn = 65535' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_max_syn_backlog = 65535' >> /etc/sysctl.conf
echo 'vm.overcommit_memory = 1' >> /etc/sysctl.conf

# Redis网络配置
tcp-backlog 511                    # TCP监听队列长度
tcp-keepalive 300                  # TCP keepalive时间
timeout 0                          # 客户端空闲超时
```

#### 6.3.2 硬件资源配置

```
CPU配置建议：
- 每个Redis实例绑定特定CPU核心
- 避免CPU频繁切换和缓存失效

内存配置建议：
- 预留足够的系统内存（避免OOM）
- 考虑fork操作的内存需求
- 合理设置maxmemory

磁盘配置建议：
- 使用SSD存储AOF和RDB文件
- 分离不同节点的存储路径
- 定期清理日志文件
```

## 7. 集群在业务中的应用实践

### 7.1 客户端连接与使用

#### 7.1.1 Java 客户端示例（Jedis）

```java
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.HostAndPort;

public class RedisClusterExample {
    public static void main(String[] args) {
        // 集群节点配置
        Set<HostAndPort> jedisClusterNodes = new HashSet<>();
        jedisClusterNodes.add(new HostAndPort("192.168.1.100", 7000));
        jedisClusterNodes.add(new HostAndPort("192.168.1.100", 7001));
        jedisClusterNodes.add(new HostAndPort("192.168.1.100", 7002));

        // 连接配置
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(20);
        poolConfig.setMaxIdle(10);
        poolConfig.setMinIdle(5);

        // 创建集群连接
        JedisCluster jedisCluster = new JedisCluster(
            jedisClusterNodes,
            2000, 2000, 3,
            poolConfig
        );

        // 使用Hash Tag确保相关数据在同一slot
        jedisCluster.set("user:{123}:profile", "张三");
        jedisCluster.set("user:{123}:score", "95");

        // 批量操作（需要在同一slot）
        jedisCluster.mset(
            "order:{456}:info", "订单信息",
            "order:{456}:status", "已支付"
        );

        jedisCluster.close();
    }
}
```

#### 7.1.2 Python 客户端示例（redis-py-cluster）

```python
from rediscluster import RedisCluster
import time

# 集群节点配置
startup_nodes = [
    {"host": "192.168.1.100", "port": "7000"},
    {"host": "192.168.1.100", "port": "7001"},
    {"host": "192.168.1.100", "port": "7002"},
]

# 创建集群连接
rc = RedisCluster(
    startup_nodes=startup_nodes,
    decode_responses=True,
    skip_full_coverage_check=True,
    health_check_interval=30
)

# 分布式计数器示例
def distributed_counter(key, increment=1):
    with rc.pipeline() as pipe:
        while True:
            try:
                pipe.watch(key)
                current_value = pipe.get(key) or 0
                pipe.multi()
                pipe.set(key, int(current_value) + increment)
                pipe.execute()
                break
            except:
                continue
    return int(current_value) + increment

# 使用示例
counter_value = distributed_counter("global:counter")
print(f"当前计数值: {counter_value}")
```

### 7.2 分布式锁实现

#### 7.2.1 基于 SET 命令的分布式锁

```python
import uuid
import time
from rediscluster import RedisCluster

class RedisDistributedLock:
    def __init__(self, redis_client, key, timeout=10):
        self.redis = redis_client
        self.key = key
        self.timeout = timeout
        self.identifier = str(uuid.uuid4())

    def acquire(self):
        """获取锁"""
        end = time.time() + self.timeout
        while time.time() < end:
            # 使用SET命令的NX和EX参数实现原子操作
            if self.redis.set(self.key, self.identifier, nx=True, ex=self.timeout):
                return True
            time.sleep(0.001)
        return False

    def release(self):
        """释放锁"""
        # 使用Lua脚本保证原子性
        script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """
        return self.redis.eval(script, 1, self.key, self.identifier)

# 使用示例
def critical_section():
    lock = RedisDistributedLock(rc, "resource:lock", timeout=30)
    if lock.acquire():
        try:
            print("获取锁成功，执行关键代码")
            # 执行需要互斥的代码
            time.sleep(5)
        finally:
            lock.release()
            print("释放锁成功")
    else:
        print("获取锁失败")
```

#### 7.2.2 红锁（Redlock）算法实现

```python
import time
import random
from rediscluster import RedisCluster

class Redlock:
    def __init__(self, redis_instances):
        self.redis_instances = redis_instances
        self.quorum = len(redis_instances) // 2 + 1

    def acquire_lock(self, key, ttl, identifier):
        """获取红锁"""
        start_time = time.time()
        acquired = 0

        for redis_instance in self.redis_instances:
            try:
                if redis_instance.set(key, identifier, nx=True, px=ttl):
                    acquired += 1
            except:
                pass

        # 计算获取锁的时间
        elapsed_time = (time.time() - start_time) * 1000

        # 检查是否获得了大多数锁，且剩余时间充足
        if acquired >= self.quorum and elapsed_time < ttl:
            return True
        else:
            # 释放已获得的锁
            self.release_lock(key, identifier)
            return False

    def release_lock(self, key, identifier):
        """释放红锁"""
        script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """

        for redis_instance in self.redis_instances:
            try:
                redis_instance.eval(script, 1, key, identifier)
            except:
                pass
```

### 7.3 数据一致性解决方案

#### 7.3.1 最终一致性模式

```python
def update_user_cache(user_id, user_data):
    """更新用户缓存，保证最终一致性"""

    # 1. 先删除缓存
    cache_key = f"user:{user_id}"
    rc.delete(cache_key)

    # 2. 更新数据库
    update_user_in_database(user_id, user_data)

    # 3. 异步更新缓存
    import threading
    def async_update_cache():
        time.sleep(0.1)  # 短暂延迟
        fresh_data = get_user_from_database(user_id)
        rc.setex(cache_key, 3600, json.dumps(fresh_data))

    threading.Thread(target=async_update_cache).start()

def get_user_with_consistency(user_id):
    """获取用户数据，保证一致性"""
    cache_key = f"user:{user_id}"

    # 1. 尝试从缓存获取
    cached_data = rc.get(cache_key)
    if cached_data:
        return json.loads(cached_data)

    # 2. 缓存未命中，从数据库获取
    user_data = get_user_from_database(user_id)
    if user_data:
        # 设置缓存，并添加随机过期时间防止缓存雪崩
        expire_time = 3600 + random.randint(0, 600)
        rc.setex(cache_key, expire_time, json.dumps(user_data))

    return user_data
```

#### 7.3.2 事务一致性保证

```python
def transfer_points(from_user_id, to_user_id, points):
    """积分转账，保证事务一致性"""

    # 使用Hash Tag确保相关数据在同一slot
    from_key = f"points:{{transfer}}:{from_user_id}"
    to_key = f"points:{{transfer}}:{to_user_id}"
    lock_key = f"lock:{{transfer}}:transfer"

    # 获取分布式锁
    lock = RedisDistributedLock(rc, lock_key, timeout=30)
    if not lock.acquire():
        raise Exception("获取锁失败，转账被取消")

    try:
        # 使用管道保证原子性
        with rc.pipeline() as pipe:
            # 监听相关键
            pipe.watch(from_key, to_key)

            # 获取当前余额
            from_balance = int(pipe.get(from_key) or 0)
            to_balance = int(pipe.get(to_key) or 0)

            # 检查余额是否充足
            if from_balance < points:
                raise Exception("余额不足")

            # 开始事务
            pipe.multi()
            pipe.set(from_key, from_balance - points)
            pipe.set(to_key, to_balance + points)

            # 记录转账日志
            log_key = f"transfer_log:{{transfer}}:{int(time.time())}"
            pipe.hset(log_key, mapping={
                'from_user': from_user_id,
                'to_user': to_user_id,
                'points': points,
                'timestamp': int(time.time())
            })

            # 执行事务
            pipe.execute()

    except Exception as e:
        print(f"转账失败: {e}")
        raise
    finally:
        lock.release()

# 使用示例
try:
    transfer_points("user1", "user2", 100)
    print("转账成功")
except Exception as e:
    print(f"转账失败: {e}")
```

## 8. 常见问题与集群故障排查

### 8.1 节点故障处理

#### 8.1.1 节点掉线检测与恢复

```bash
#!/bin/bash
# 节点健康检查脚本

check_node_health() {
    local host=$1
    local port=$2

    # 检查节点是否响应
    if redis-cli -h $host -p $port ping 2>/dev/null | grep -q "PONG"; then
        echo "[$host:$port] 节点正常"
        return 0
    else
        echo "[$host:$port] 节点异常，尝试重启"

        # 尝试重启节点
        systemctl restart redis-server@$port
        sleep 5

        # 再次检查
        if redis-cli -h $host -p $port ping 2>/dev/null | grep -q "PONG"; then
            echo "[$host:$port] 节点重启成功"

            # 检查是否需要重新加入集群
            cluster_state=$(redis-cli -h $host -p $port CLUSTER INFO | grep cluster_state)
            if [[ $cluster_state == *"fail"* ]]; then
                echo "[$host:$port] 需要手动修复集群状态"
                # 这里可以添加自动修复逻辑
            fi
        else
            echo "[$host:$port] 节点重启失败，需要人工介入"
            # 发送告警通知
            send_alert "Redis节点 $host:$port 重启失败"
        fi
        return 1
    fi
}

# 检查所有节点
nodes=("192.168.1.100:7000" "192.168.1.100:7001" "192.168.1.100:7002"
       "192.168.1.100:7003" "192.168.1.100:7004" "192.168.1.100:7005")

for node in "${nodes[@]}"; do
    IFS=':' read -ra ADDR <<< "$node"
    check_node_health "${ADDR[0]}" "${ADDR[1]}"
done
```

#### 8.1.2 主从切换异常处理

```bash
# 强制故障转移
redis-cli -h 192.168.1.100 -p 7001 CLUSTER FAILOVER FORCE

# 手动设置从节点为主节点
redis-cli -h 192.168.1.100 -p 7001 CLUSTER FAILOVER TAKEOVER

# 重置节点状态（谨慎使用）
redis-cli -h 192.168.1.100 -p 7001 CLUSTER RESET SOFT
```

### 8.2 数据一致性问题排查

#### 8.2.1 检测数据不一致

```python
def check_data_consistency():
    """检查集群数据一致性"""

    # 获取所有主节点
    cluster_nodes = rc.cluster_nodes()
    master_nodes = []

    for node_id, node_info in cluster_nodes.items():
        if 'master' in node_info['flags']:
            master_nodes.append({
                'id': node_id,
                'host': node_info['host'],
                'port': node_info['port'],
                'slots': node_info['slots']
            })

    inconsistent_keys = []

    # 检查每个key在主从之间是否一致
    for master in master_nodes:
        master_client = redis.Redis(host=master['host'], port=master['port'])

        # 获取该主节点的所有从节点
        slaves = get_slave_nodes(master['id'])

        # 扫描主节点的所有key
        for key in master_client.scan_iter(count=100):
            master_value = master_client.get(key)

            # 检查每个从节点的对应key
            for slave in slaves:
                slave_client = redis.Redis(host=slave['host'], port=slave['port'])
                slave_value = slave_client.get(key)

                if master_value != slave_value:
                    inconsistent_keys.append({
                        'key': key,
                        'master_value': master_value,
                        'slave_value': slave_value,
                        'master_node': f"{master['host']}:{master['port']}",
                        'slave_node': f"{slave['host']}:{slave['port']}"
                    })

    return inconsistent_keys

def repair_inconsistent_data(inconsistent_keys):
    """修复不一致数据"""
    for item in inconsistent_keys:
        print(f"修复key: {item['key']}")

        # 从主节点重新同步到从节点
        master_host, master_port = item['master_node'].split(':')
        slave_host, slave_port = item['slave_node'].split(':')

        # 触发部分重同步
        slave_client = redis.Redis(host=slave_host, port=slave_port)
        slave_client.execute_command('REPLICAOF', master_host, master_port)
```

#### 8.2.2 脑裂检测与修复

```bash
#!/bin/bash
# 脑裂检测脚本

detect_split_brain() {
    declare -A masters
    node_count=0

    # 检查每个节点认为的主节点情况
    for node in "${nodes[@]}"; do
        IFS=':' read -ra ADDR <<< "$node"
        host=${ADDR[0]}
        port=${ADDR[1]}

        # 获取节点信息
        cluster_info=$(redis-cli -h $host -p $port CLUSTER NODES 2>/dev/null)
        if [ $? -eq 0 ]; then
            node_count=$((node_count + 1))

            # 统计主节点数量
            master_count=$(echo "$cluster_info" | grep "master" | wc -l)
            masters[$node]=$master_count

            echo "节点 $node 看到的主节点数: $master_count"
        else
            echo "节点 $node 无法连接"
        fi
    done

    # 检查是否存在脑裂
    expected_masters=3  # 期望的主节点数
    split_brain=false

    for node in "${!masters[@]}"; do
        if [ "${masters[$node]}" -ne $expected_masters ]; then
            echo "警告：节点 $node 的主节点数量异常: ${masters[$node]}"
            split_brain=true
        fi
    done

    if [ "$split_brain" = true ]; then
        echo "检测到脑裂现象，需要人工介入修复"
        # 这里可以添加自动修复逻辑或发送告警
        return 1
    else
        echo "集群状态正常，未检测到脑裂"
        return 0
    fi
}

# 执行脑裂检测
detect_split_brain
```

### 8.3 网络通信问题排查

#### 8.3.1 节点通信诊断

```bash
#!/bin/bash
# 集群网络诊断脚本

diagnose_cluster_network() {
    local nodes=("$@")

    echo "=== 集群网络诊断 ==="

    for i in "${!nodes[@]}"; do
        for j in "${!nodes[@]}"; do
            if [ $i -ne $j ]; then
                IFS=':' read -ra FROM <<< "${nodes[$i]}"
                IFS=':' read -ra TO <<< "${nodes[$j]}"

                from_host=${FROM[0]}
                from_port=${FROM[1]}
                to_host=${TO[0]}
                to_port=${TO[1]}

                echo "检查 $from_host:$from_port -> $to_host:$to_port"

                # 1. 检查服务端口连通性
                if timeout 3 bash -c "</dev/tcp/$to_host/$to_port"; then
                    echo "  ✓ 服务端口 $to_port 连通"
                else
                    echo "  ✗ 服务端口 $to_port 不通"
                fi

                # 2. 检查集群总线端口连通性
                bus_port=$((to_port + 10000))
                if timeout 3 bash -c "</dev/tcp/$to_host/$bus_port"; then
                    echo "  ✓ 集群总线端口 $bus_port 连通"
                else
                    echo "  ✗ 集群总线端口 $bus_port 不通"
                fi

                # 3. 检查网络延迟
                ping_result=$(ping -c 3 -W 1000 $to_host 2>/dev/null | grep "avg")
                if [ $? -eq 0 ]; then
                    echo "  ✓ 网络延迟: $ping_result"
                else
                    echo "  ✗ 网络不可达"
                fi

                echo ""
            fi
        done
    done
}

# 执行网络诊断
nodes=("192.168.1.100:7000" "192.168.1.100:7001" "192.168.1.100:7002")
diagnose_cluster_network "${nodes[@]}"
```

#### 8.3.2 集群状态修复

```bash
# 修复集群状态的常用命令

# 1. 重置节点状态
redis-cli -h 192.168.1.100 -p 7000 CLUSTER RESET SOFT

# 2. 手动设置slot归属
redis-cli -h 192.168.1.100 -p 7000 CLUSTER ADDSLOTS {0..5460}

# 3. 修复集群拓扑
redis-cli --cluster fix 192.168.1.100:7000

# 4. 重新分配slots
redis-cli --cluster rebalance 192.168.1.100:7000 --cluster-use-empty-masters

# 5. 检查集群完整性
redis-cli --cluster check 192.168.1.100:7000 --cluster-search-multiple-owners
```

### 8.4 性能问题排查

#### 8.4.1 慢查询分析

```python
def analyze_slow_queries():
    """分析集群慢查询"""

    cluster_nodes = rc.cluster_nodes()
    all_slow_queries = []

    for node_id, node_info in cluster_nodes.items():
        if 'master' in node_info['flags']:
            host = node_info['host']
            port = node_info['port']

            # 连接到具体节点
            node_client = redis.Redis(host=host, port=port)

            # 获取慢查询日志
            slow_queries = node_client.slowlog_get(100)

            for query in slow_queries:
                all_slow_queries.append({
                    'node': f"{host}:{port}",
                    'id': query['id'],
                    'start_time': query['start_time'],
                    'duration': query['duration'],
                    'command': ' '.join(str(arg) for arg in query['command'])
                })

    # 按执行时间排序
    all_slow_queries.sort(key=lambda x: x['duration'], reverse=True)

    # 分析结果
    print("=== 慢查询分析报告 ===")
    for i, query in enumerate(all_slow_queries[:10]):
        print(f"{i+1}. 节点: {query['node']}")
        print(f"   执行时间: {query['duration']} μs")
        print(f"   命令: {query['command'][:100]}...")
        print()

    return all_slow_queries

def optimize_slow_queries(slow_queries):
    """慢查询优化建议"""
    command_stats = {}

    for query in slow_queries:
        cmd = query['command'].split()[0].upper()
        if cmd not in command_stats:
            command_stats[cmd] = {'count': 0, 'total_time': 0}

        command_stats[cmd]['count'] += 1
        command_stats[cmd]['total_time'] += query['duration']

    print("=== 优化建议 ===")
    for cmd, stats in sorted(command_stats.items(),
                           key=lambda x: x[1]['total_time'], reverse=True):
        avg_time = stats['total_time'] / stats['count']
        print(f"命令: {cmd}")
        print(f"  出现次数: {stats['count']}")
        print(f"  平均执行时间: {avg_time:.2f} μs")

        # 提供优化建议
        if cmd in ['KEYS', 'FLUSHALL', 'FLUSHDB']:
            print("  建议: 避免使用阻塞性命令")
        elif cmd == 'SORT':
            print("  建议: 考虑使用有序集合替代SORT操作")
        elif cmd in ['SUNION', 'SINTER', 'SDIFF']:
            print("  建议: 减少集合操作的数据量")
        print()
```

#### 8.4.2 内存使用分析

```bash
#!/bin/bash
# 内存使用分析脚本

analyze_memory_usage() {
    echo "=== Redis集群内存使用分析 ==="

    for node in "${nodes[@]}"; do
        IFS=':' read -ra ADDR <<< "$node"
        host=${ADDR[0]}
        port=${ADDR[1]}

        echo "节点: $host:$port"

        # 获取内存信息
        memory_info=$(redis-cli -h $host -p $port INFO memory)

        # 解析关键内存指标
        used_memory=$(echo "$memory_info" | grep "used_memory:" | cut -d: -f2 | tr -d '\r')
        used_memory_human=$(echo "$memory_info" | grep "used_memory_human:" | cut -d: -f2 | tr -d '\r')
        used_memory_peak=$(echo "$memory_info" | grep "used_memory_peak_human:" | cut -d: -f2 | tr -d '\r')
        mem_fragmentation_ratio=$(echo "$memory_info" | grep "mem_fragmentation_ratio:" | cut -d: -f2 | tr -d '\r')

        echo "  已使用内存: $used_memory_human"
        echo "  内存使用峰值: $used_memory_peak"
        echo "  内存碎片率: $mem_fragmentation_ratio"

        # 内存碎片率分析
        if (( $(echo "$mem_fragmentation_ratio > 1.5" | bc -l) )); then
            echo "  ⚠️  警告: 内存碎片率过高，建议重启节点"
        elif (( $(echo "$mem_fragmentation_ratio < 1.0" | bc -l) )); then
            echo "  ⚠️  警告: 可能存在内存交换，检查系统内存"
        else
            echo "  ✓ 内存碎片率正常"
        fi

        # 获取键空间统计
        keyspace_info=$(redis-cli -h $host -p $port INFO keyspace)
        if [ -n "$keyspace_info" ]; then
            echo "  键空间信息:"
            echo "$keyspace_info" | grep "^db" | while read line; do
                echo "    $line"
            done
        fi

        echo ""
    done
}

# 执行内存分析
nodes=("192.168.1.100:7000" "192.168.1.100:7001" "192.168.1.100:7002"
       "192.168.1.100:7003" "192.168.1.100:7004" "192.168.1.100:7005")
analyze_memory_usage
```

## 9. 生产环境最佳实践

### 9.1 部署架构建议

#### 9.1.1 硬件配置建议

```
生产环境推荐配置：

单节点配置：
- CPU: 4核心以上，主频2.4GHz+
- 内存: 16GB以上（Redis使用8GB，系统预留8GB）
- 磁盘: SSD，至少500GB，IOPS > 3000
- 网络: 千兆网卡，低延迟网络环境

集群规模：
- 最小配置: 6节点（3主3从）
- 推荐配置: 9节点（3主6从）或12节点（6主6从）
- 大型集群: 可扩展到1000+节点

网络架构：
- 独立的集群网络VLAN
- 避免跨机房部署（延迟 < 1ms）
- 配置防火墙规则开放必要端口
```

#### 9.1.2 容量规划

```python
def calculate_cluster_capacity():
    """集群容量规划计算器"""

    # 基础参数
    total_data_size_gb = 500  # 总数据量GB
    avg_key_size_bytes = 1024  # 平均key大小
    replication_factor = 2  # 副本因子（1主1从）
    memory_overhead_ratio = 1.2  # 内存开销比例
    growth_factor = 1.5  # 增长预留

    # 计算所需内存
    required_memory_gb = (total_data_size_gb * replication_factor *
                         memory_overhead_ratio * growth_factor)

    # 计算节点数量
    single_node_memory_gb = 16  # 单节点可用内存
    min_master_nodes = max(3, math.ceil(total_data_size_gb / single_node_memory_gb))
    total_nodes = min_master_nodes * replication_factor

    print(f"=== 集群容量规划 ===")
    print(f"总数据量: {total_data_size_gb} GB")
    print(f"所需总内存: {required_memory_gb:.2f} GB")
    print(f"建议主节点数: {min_master_nodes}")
    print(f"建议总节点数: {total_nodes}")
    print(f"单节点平均数据量: {total_data_size_gb/min_master_nodes:.2f} GB")

    # QPS评估
    estimated_qps = 100000  # 预估QPS
    qps_per_node = estimated_qps / min_master_nodes
    print(f"预估总QPS: {estimated_qps}")
    print(f"单节点平均QPS: {qps_per_node:.0f}")

    if qps_per_node > 50000:
        print("⚠️  警告: 单节点QPS过高，建议增加节点数量")

    return {
        'master_nodes': min_master_nodes,
        'total_nodes': total_nodes,
        'required_memory_gb': required_memory_gb
    }

# 执行容量规划
capacity_plan = calculate_cluster_capacity()
```

### 9.2 监控告警体系

#### 9.2.1 监控指标体系

```yaml
# Prometheus监控配置示例
groups:
  - name: redis-cluster
    rules:
      - alert: RedisClusterDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis节点下线"
          description: "Redis节点 {{ $labels.instance }} 已下线超过1分钟"

      - alert: RedisHighMemoryUsage
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis内存使用率过高"
          description: "Redis节点 {{ $labels.instance }} 内存使用率超过90%"

      - alert: RedisHighLatency
        expr: redis_slowlog_length > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Redis响应延迟过高"
          description: "Redis节点 {{ $labels.instance }} 慢查询数量异常"

      - alert: RedisClusterSlotsFail
        expr: redis_cluster_slots_fail > 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Redis集群slot故障"
          description: "Redis集群存在故障slot，影响服务可用性"
```
