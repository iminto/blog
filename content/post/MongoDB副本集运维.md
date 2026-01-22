---
date : "2026-01-22T18:40:30+08:00"
title : "MongoDB副本集运维"
archives: "2026"
tags: [运维]
---

# 一、MongoDB搭建三节点副本集

## 1.架构图

```tex
       Your App
           │
           ▼
   ┌───────────────┐
   │  MongoDB RS   │ ← 所有读写都发到这里（连任意节点）
   ├───────────────┤
   │ Node1 (Primary)│ ← 当前主节点（可写）
   │ Node2 (Secondary)│ ← 从节点（可读，自动同步）
   │ Node3 (Secondary)│ ← 从节点（可读，自动同步）
   └───────────────┘
```

- **3 台物理机**：每台运行一个 `mongod` 实例；
- **无 mongos、无 config server**：因为不是分片集群；
- **自动选主**：如果 Node1 宕机，Node2 或 Node3 自动升为 Primary。

即使是单机，也能配置副本集

## 2.步骤

### 1.在三台机器上安装 MongoDB

```bash
sudo mkdir -p /var/log/mongodb
sudo chown -R $(whoami) /develop/data/mongo/ /var/log/mongodb/
```



### 2.配置每台机器的 `mongod.conf`

假设三台机器 IP：

- Node1: `192.168.1.101`
- Node2: `192.168.1.102`
- Node3: `192.168.1.103`

在 **所有三台** 上编辑 `/etc/mongod.conf`：
```yaml
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true

net:
  port: 27017
  bindIp: 0.0.0.0  # 允许其他节点连接

replication:
  replSetName: "rs0"  # 副本集名称，必须一致！

security:
  authorization: enabled  # 启用认证（可选但推荐）
```

### 3.启动所有 mongod 服务

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

### 4.初始化副本集（只需在一台机器上操作）

```bash
# 连接到任意一台（如 Node1）
mongo 192.168.1.101

# 切换到 admin
use admin

# 初始化副本集，注意不要用主机名
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "m1.bigdata.com:27017" },
    { _id: 1, host: "m2.bigdata.com:27017" },
    { _id: 2, host: "m3.bigdata.com:27017" }
  ]
})

# 等待至少30秒，查看状态（选举需要时间）
rs.status()
```

至少等待30秒后，成功后你会看到：

- 一个 `PRIMARY`
- 两个 `SECONDARY`

每个副本集成员可以设置 priority 值（默认为 1），priority 越高，越容易被选为 PRIMARY。

如果这么写

```
 { _id: 2, host: "localhost:27019", arbiterOnly: true }  // ← 关键
```

那么其中一个就是仲裁节点。

如果要加入新的副本，注意一次只加一个，等同步完成再加下一个

```js
// 方法 1：简单添加（使用默认配置）
rs.add("new_node_ip:27017")
// 方法 2：指定更多选项（推荐）
rs.add({
  host: "new_node_ip:27017",
  priority: 1,        // 可参与选举（0 = 仲裁/只读）
  votes: 1,           // 有投票权
  tags: { dc: "rack3" } // 可选标签
})
```



### 5.创建用户（启用 auth 后必需）

```bash
// 在 PRIMARY 上执行
use admin
db.createUser({
  user: "admin",
  pwd: "your_strong_password",
  roles: [ { role: "root", db: "admin" } ]
})

// 退出重连（带认证）
exit
mongo -u admin -p --authenticationDatabase admin 192.168.1.101
//修改配置文件，按从主依次重启节点
```

root 角色只能在 admin 库授予，也可以在业务数据库创建用户

注意：MongoDB 启动后、创建用户前，是完全开放的！如果 admin 数据库中没有任何用户，那么来自 localhost 的连接可以绕过认证，执行创建用户的操作。

当你启用 **副本集（replica set）** 并开启 **用户认证（`authorization: enabled`）** 时，**社区版 MongoDB 要求必须配置 `keyFile`**，步骤可参考分片集群。

代码连接示例

```python
client = MongoClient(
    "mongodb://user:pwd@host1,host2,host3/?replicaSet=myRepl&authSource=admin"
)
# 或者 优先从从读，从不可用时读主
client = MongoClient(
    "mongodb://...?replicaSet=rs0&authSource=admin&readPreference=secondaryPreferred"
)
```

注意：开启认证后，rs.status()会报错，需要先认证才能查询。

### 6.连接方式

不要只连一个节点

```bash
mongo "mongodb://host1:27017,host2:27017,host3:27017/mydb?replicaSet=rs0"
//开了认证
mongo "mongodb://username:password@host1:27017,host2:27017,host3:27017/mydb?authSource=admin&replicaSet=rs0"
```

注意：老版本mongo不支持`readPreference=secondary`这类参数，secondary节点默认是不可读的，需要可读，最兼容的方式是连接后执行如下命令

```js
rs.slaveOk()
```

而且如mongo 3.0以下连接方式应该是

```bash
mongo --host rs0/host1:27017,host2:27017,host3:27017 mydb
```



# 二、MongoDB创建三节点集群

## 1.架构图

✅ **目标架构**：1 个 Config Server Replica Set + 1 个 Shard Replica Set + 1 个 mongos
 ✅ **资源**：3 台物理机（每台运行多个 MongoDB 进程）

```latex
+------------------+     +------------------+     +------------------+
|    Node A        |     |    Node B        |     |    Node C        |
|                  |     |                  |     |                  |
|  ConfigSvr (P)   |<--->|  ConfigSvr (S)   |<--->|  ConfigSvr (S)   |
|                  |     |                  |     |                  |
|  Shard0 (P)      |<--->|  Shard0 (S)      |<--->|  Shard0 (S)      |
|                  |     |                  |     |                  |
|  mongos          |     |  mongos          |     |  mongos          |
+------------------+     +------------------+     +------------------+
        ↑                        ↑                        ↑
        └─────── 应用连接任意 mongos ────────────────┘
```

**端口分配总结**

| 组件          | 端口  | 作用         |
| ------------- | ----- | ------------ |
| mongos        | 27017 | 应用连接入口 |
| Shard0        | 27018 | 数据存储     |
| Config Server | 27019 | 元数据存储   |

## 2.步骤

### 1.在三台机器上安装 MongoDB

```bash
# 1. 安装 MongoDB（以 Ubuntu 为例，三台都装）
sudo apt update && sudo apt install -y mongodb-org

# 2. 创建目录
sudo mkdir -p /data/configdb /data/shard0 /logs
sudo chown -R mongodb:mongodb /data /logs

# 3. 关闭防火墙 or 开放端口（27017~27020）
sudo ufw allow from 192.168.1.0/24 to any port 27017:27020
```

### 2.生成并分发 keyFile（内部认证密钥）

```bash
# 在 Node A 生成
sudo openssl rand -base64 756 > /etc/mongodb-keyfile
sudo chmod 400 /etc/mongodb-keyfile
sudo chown mongodb:mongodb /etc/mongodb-keyfile

# 复制到 Node B 和 Node C
scp /etc/mongodb-keyfile nodeB:/etc/
scp /etc/mongodb-keyfile nodeC:/etc/
# （在 B/C 上同样执行 chmod 400 + chown）
```

### 3.配置并启动 Config Server（3 节点 RS）

在 所有三台 创建 /etc/mongod-config.conf：

```yaml
storage:
  dbPath: /data/configdb
  journal:
    enabled: true

net:
  port: 27019
  bindIp: 0.0.0.0

replication:
  replSetName: "configRepl"

sharding:
  clusterRole: configsvr

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile
```

启动 Config Server（三台都执行）：

```bash
sudo -u mongodb mongod -f /etc/mongod-config.conf --fork --logpath /logs/config.log
```

初始化 Config Replica Set（只在 Node A 执行）

```bash
mongo --port 27019
> rs.initiate({
    _id: "configRepl",
    configsvr: true,
    members: [
      { _id: 0, host: "nodeA:27019" },
      { _id: 1, host: "nodeB:27019" },
      { _id: 2, host: "nodeC:27019" }
    ]
  })
> rs.status()  // 等待 PRIMARY 出现
```

### 4.配置并启动 Shard（3 节点 RS）

在 所有三台 创建 /etc/mongod-shard0.conf

```yml
storage:
  dbPath: /data/shard0
  journal:
    enabled: true

net:
  port: 27018
  bindIp: 0.0.0.0

replication:
  replSetName: "shard0Repl"

sharding:
  clusterRole: shardsvr

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile
```

启动 Shard（三台都执行）：

```bash
sudo -u mongodb mongod -f /etc/mongod-shard0.conf --fork --logpath /logs/shard0.log
```

初始化 Shard Replica Set（只在 Node A 执行）：

```bash
mongo --port 27018
> rs.initiate({
    _id: "shard0Repl",
    members: [
      { _id: 0, host: "nodeA:27018" },
      { _id: 1, host: "nodeB:27018" },
      { _id: 2, host: "nodeC:27018" }
    ]
  })
> rs.status()
```

### 5.启动 mongos（路由）

在 所有三台 创建 /etc/mongos.conf：

```yaml
sharding:
  configDB: configRepl/nodeA:27019,nodeB:27019,nodeC:27019

net:
  port: 27017
  bindIp: 0.0.0.0

security:
  keyFile: /etc/mongodb-keyfile
```

⚠️ 注意：mongos **不需要 `authorization`**，它从 shard 获取用户

启动 mongos（三台都执行）：

```bash
sudo -u mongodb mongos -f /etc/mongos.conf --fork --logpath /logs/mongos.log
```

### 6.在mongos添加 Shard 并创建用户

连接任意 mongos（如 Node A）：

```bash
mongo nodeA:27017
```

添加 Shard

```bash
// 添加 shard0 到集群
sh.addShard("shard0Repl/nodeA:27018,nodeB:27018,nodeC:27018")
sh.status()  // 查看是否成功
```

创建管理员用户

```bash
use admin
db.createUser({
  user: "admin",
  pwd: "your_strong_password",
  roles: [ "root" ]
})
```

添加sharding后，还需要做**分片键（Shard Key）的选择 & 分片开启**

```js
// 必须在 mongos 上手动执行：
sh.enableSharding("myapp") // 为数据库启用分片
sh.shardCollection("myapp.orders", { customerId: 1 })  //指定分片键，范围分片
sh.shardCollection("myapp.orders", { customerId: "hashed" }) //hash分片
```

分片键一旦选定，几乎无法更改，除非重新导入导出。而且唯一索引必须包含分片键字段。

customerId: 1 表示使用文档中的字段 customerId 作为分片依据，且升序，1是固定写法，根据实际插入的数据动态计算中位数。

如果需要按 customerId 的前缀（如前2位）代表地区，希望不同地区的数据落在不同 shard 上，在 MongoDB 原生分片中 无法直接实现，需要插入数据时，显式计算并存储地区码，然后用 region 作分片键，也可以做联合分片

```js
sh.shardCollection("myapp.orders", { region: 1, customerId: 1 })
```



### 7.应用连接方式

```python
from pymongo import MongoClient

client = MongoClient(
    "mongodb://admin:your_strong_password@nodeA:27017,nodeB:27017,nodeC:27017/"
    "?replicaSet=none&authSource=admin"
)
# 注意：连 mongos 时 replicaSet 参数可省略或设为 none
```

验证：

```bash
// 连 mongos
mongo -u admin -p --authenticationDatabase admin nodeA:27017

// 查看分片状态
sh.status()

// 插入测试数据（会自动分片）
use testdb
db.testcoll.insertMany([{x:1}, {x:2}])
db.testcoll.getShardDistribution()
```

# 三、基本的用户管理和监控

## 1.用户管理

MongoDB 的用户不是全局的，而是属于某个特定数据库（称为 认证源 authentication database），例如：在 admin 库创建的用户，登录时必须指定 authSource=admin。

- 用户通过 **角色** 获得权限；
- 角色可以是：
  - **内置角色**（如 `readWrite`, `dbAdmin`, `root`）
  - **自定义角色**

### **权限作用域**

| 角色类型                          | 作用范围                              |
| --------------------------------- | ------------------------------------- |
| `read`, `readWrite`               | 仅限**创建用户所在的数据库**          |
| `dbAdmin`, `userAdmin`，`dbOwner` | 仅限所在库                            |
| `clusterAdmin`, `root`            | **整个集群**（必须在 `admin` 库授予） |

查看用户

```js
// 查看当前库的所有用户
db.getUsers()
// 查看特定用户
db.getUser("app_user")
```

修改用户密码

```js
use admin
db.changeUserPassword("admin", "NewStrongPass789!")
```

修改角色

```js
use myapp
db.grantRolesToUser("app_user", [ { role: "dbAdmin", db: "myapp" } ])
db.revokeRolesFromUser("app_user", [ "readWrite" ])
```

## 2.常用查询监控

```js
#列出所有非空数据库
show dbs
// 检查 admin 库是否有用户（判断是否初始化过，system.users是内置集合）
use admin
db.system.users.countDocuments({})
//查询集合
show collections
//查询存贮空间
db.products.stats()
db.getReplicationInfo()
```

常用增删改查

```js
db.products.insertOne({
  name: "笔记本电脑",
  price: 5999,
  category: "Electronics",
  tags: ["办公", "高性能"],
  inStock: true,
  specs: {
    cpu: "Intel i7",
    ram: "16GB",
    storage: "512GB SSD"
  }
})
//插入多个文档
db.products.insertMany([
  {
    name: "无线鼠标",
    price: 89,
    category: "Accessories",
    tags: ["外设"],
    inStock: true
  },
  {
    name: "机械键盘",
    price: 399,
    category: "Accessories",
    tags: ["外设", "游戏"],
    inStock: false
  }
])
//
db.products.find()
db.collection_name.find().sort({ _id: -1 }).limit(10)
db.users.findOne({ _id: ObjectId("673d1a2b8f4e1c0012345678") })//_id可以自定义
db.products.find({ category: "Electronics" })
// 将“无线鼠标”的价格改为 79 元
db.products.updateOne(
  { name: "无线鼠标" },           // 查询条件
  { $set: { price: 79 } }         // 更新操作
)
```

## 3.数据导入导出

MongoDB 的数据存储目录结构 不是按数据库（Database）分文件夹，而是采用 统一的存储引擎格式（默认是 WiredTiger），将所有数据库的数据混合存储在同一个目录下，通过内部命名空间（namespace）来区分。

导入导出工具下载地址：https://www.mongodb.com/try/download/database-tools/releases/archive

如果启用了认证，加上 `--username` 和 `--password`以及`--authenticationDatabase admin`

### 导出备份

```bash
./mongodump --host 127.0.0.1 --port 27017 --db test --out /tmp/test
```

如果是分片集群，必须连接 mongos 路由节点

副本集

```bash
mongodump \
  --host "myReplSet/192.168.1.10:27017,192.168.1.11:27017,192.168.1.12:27017" \
  --db myapp \
  --out /backup/
```

当然，也可以直接连 Primary（如果你知道是谁），从 Secondary 备份（减轻主库压力）

分片集群

```bash
mongodump \
  --host "192.168.10.100:27017" \  # 任意一个 mongos 地址
  --db myapp \
  --out /backup/
```

批量导出
```bash
#!/bin/bash

set -euo pipefail 

MONGO_URI="mongodb://user:pass@host:27017"
DB_NAMES=("db1" "db2" "myapp")
BACKUP_TIME=$(date +%Y%m%d_%H%M%S)
OUTPUT_ROOT="./mongodb_backups/$BACKUP_TIME"

mkdir -p "$OUTPUT_ROOT"

echo "备份目标: $OUTPUT_ROOT"
for db in "${DB_NAMES[@]}"; do
    if [[ -z "$db" ]]; then continue; fi

    echo "Dumping database: $db ..."
    
    # 关键：--out 指向根目录，mongodump 自动创建 $OUTPUT_ROOT/$db/
    if mongodump --uri="$MONGO_URI" --db="$db" --out="$OUTPUT_ROOT"; then
        echo "$db -> $OUTPUT_ROOT/$db/"
    else
        echo "备份 $db 失败！" >&2
        exit 1
    fi
done

echo "所有数据库备份完成！"
ls -l "$OUTPUT_ROOT"
```

MONGO_URI 参数很重要，如果没有认证就 **不要加 `?authSource=`**这类的参数

当然，也能用 --host 的方式去写。



### 导入恢复

注意：必须指定完整 dump 路径

```bash
mongorestore --host localhost --port 27017 --db myapp /backup/myapp
```

如果是副本集，~~连任意一个 主节点（Primary） 或 从节点（Secondary）~~，同样应该使用副本集连接字符串。

```bash
mongorestore \
  --host "rsName/host1:port,host2:port,host3:port" \
  --username user \
  --password 'xxx' \
  --authenticationDatabase admin \
  /path/to/backup
```

导入时还能重命名

```bash
--nsFrom "prod_files.*" \
--nsTo "test_files.*" \
```



### 加速导入导出

```bash
mongorestore --noIndexRestore ...
# 然后在 mongo shell 中：
use your_db
// 查看原库有哪些索引（从 2.6 备份前记录，或从其他环境获取）
db.fs.files.getIndexes()
// 手动创建（示例）
db.mycol.createIndex({ name: 1 })
db.fs.chunks.createIndex({ files_id: 1, n: 1 }, { unique: true })
```

fs.files 集合已有默认_id索引，除非特殊需求，否则不需要重建索引。

```js
db.fs.files.createIndex({ filename: 1 })
```

**但是：现代 mongorestore 已优化为“后建索引”模式，所以这个优化无效**

还可以增大缓存配置（但意义不大）：

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # 默认是 (RAM - 1GB) / 2，可适当调大
```

可以使用如下命令查看缓存配置

```bash
db.serverStatus().wiredTiger.cache
```

还可以在导入时，允许并行导入

```bash
--numParallelCollections 4 \          # 并行恢复 4 个集合
--writeConcern '{"w":0}' # 大水漫灌，效果？
```

同时尝试锁定primary，减少网络漂移，测试能节约10-20%的时间。

```bash
cfg = rs.conf()
cfg.members[0].priority = 10    // 默认是 1，设为最高
rs.reconfig(cfg)
```

或者尝试先单副本，再后台同步

```bash
// 在 Primary (A) 上执行
rs.remove("B:27017")
rs.remove("C:27017")
// 在 B、C 服务器上执行（停止 mongod 后）
sudo systemctl stop mongod
sudo rm -rf /var/lib/mongodb/*     # 清空数据目录
sudo systemctl start mongod        # 重启（此时是 standalone 模式）
//在 A 上执行 mongorestore
rs.add("B:27017")//重新加入副本集
```

或者直接单节点设置副本集，导入后再增加。

1963个文件+108185文件块，空间占用26G，16核16G虚拟机，三节点数据导入并恢复索引测试

| 参数与优化                                                   | 导入耗时 |
| ------------------------------------------------------------ | -------- |
| 默认参数                                                     | 62分钟   |
| --numParallelCollections=4<br />--writeConcern '{"w":0}'<br />锁定主节点，确保本地导入 | 49分钟   |
| 先单副本导入，然后添加从节点同步                             | 51分钟   |
| --numParallelCollections=8<br />--numInsertionWorkersPerCollection=8<br />锁定主节点，确保本地导入 | 10分钟   |



## 4.mongofiles工具

如果要存储 **大于 16MB 的文件**（如视频、音频、大型日志、备份文件等），就需要使用 **GridFS**。

> ✅ **GridFS 不是单独的文件系统**，而是 MongoDB 的一种**规范**：
>  它会自动将大文件 **切分成多个小块（默认 255KB/块）**，存入两个集合：
>
> - `fs.files`：存储文件元信息（文件名、大小、上传时间等）
> - `fs.chunks`：存储文件的实际数据块

`mongofiles` 就是用来 **在本地文件系统 和 GridFS 之间传输文件** 的 CLI 工具。

支持的操作：

| 命令     | 作用                     |
| -------- | ------------------------ |
| `put`    | 上传本地文件到 GridFS    |
| `get`    | 从 GridFS 下载文件到本地 |
| `list`   | 列出 GridFS 中的所有文件 |
| `search` | 按文件名模糊搜索         |
| `delete` | 删除 GridFS 中的文件     |
| `del`    | 同 `delete`              |

------

比如

```bash
#将本地 文件 上传到 test 库的 GridFS
./mongofiles -d test put /home/koudai/下载/md5.gif  
./mongofiles -d test put -l /home/koudai/下载/md5.gif  -r md5.gif
```

执行后：

- 在 `test.fs.files` 中插入一条元数据记录
- 在 `test.fs.chunks` 中插入多个数据块

查询

```bash
./mongofiles -d test list
```

也可以直接用 `db.fs.files.find()` 查看文件列表！

## 5.增量备份恢复

先全局备份，假设    

12:00:00：开始执行 mongodump --oplog
13:00:00：mongodump 完成，生成了 dump目录 + oplog.bson

```bash
echo "Backup started at $(date -Iseconds)" >> backup.log
mongodump --host "rs0/localhost:27017" \
--oplog \
--out /backup/full_$(date +%Y%m%d_%H%M%S)
```

--oplog 备份的 oplog 范围是 [备份开始时刻, 备份结束时刻],`oplog.bson` = 备份期间产生的所有增量变更

mongodump只会dump 所有业务数据库（`admin`, `mydb`, ...）

恢复数据需要排除local库

```bash
mongorestore  --oplogReplay ./dum
```

但是需要注意`--oplogReplay`选项不能与 `--nsInclude`等指令同时出现，否则会报错。MongoDB **强制要求**：使用 `--oplogReplay` 时，必须恢复**完整的备份目录**，不能筛选。



### 备份恢复之后的增量更新

假设T0开始DUMP，T1时刻DUMP完毕拿去恢复，mongorestore只能得到T1时刻前的数据，T1后的更新需要迁移到新集群。

mongodump完毕后立刻执行，获取T1时间戳和序列

```bash
#!/bin/bash
SRC="your_2.6_host:27017"
# 创建临时 JS 脚本
cat > /tmp/get_ts.js <<'EOF'
var db = db.getSiblingDB("local");
var last = db.oplog.rs.find().sort({$natural: -1}).limit(1).next();
if (last && last.ts) {
    print(tojson(last.ts));
} else {
    print("null");
}
EOF

# 执行
T0_STR=$(mongo --host "$SRC" /tmp/get_ts.js 2>/dev/null)
T0_STR=$(echo "$T0_STR" | tr -d '\n\r ')

# 验证格式
if [[ "$T0_STR" =~ ^Timestamp$\([0-9]+,[[:space:]]*[0-9]+$\) ]]; then
    T_SEC=$(echo "$T0_STR" | sed 's/Timestamp(\([0-9]*\),[[:space:]]*\([0-9]*\))/\1/')
    T_INC=$(echo "$T0_STR" | sed 's/Timestamp(\([0-9]*\),[[:space:]]*\([0-9]*\))/\2/')
    echo "T0.t = $T_SEC"
    echo "T0.i = $T_INC"
else
    echo "ERROR: Failed to get valid Timestamp: '$T0_STR'"
    exit 1
fi
```

得到了这个时间戳，我们就能增量备份了

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
"""
MongoDB Oplog Tailing Sync Tool
Supports regular collections and GridFS.
Compatible with Python 2.7+ and PyMongo 3.x.
局限：运行后新增的数据库无法同步，考虑代码复杂性不做修改
"""

import sys
import os
import time
import logging
import argparse
from pymongo import MongoClient, CursorType
from bson.timestamp import Timestamp
import threading
from collections import deque
import re

# Default config (can be overridden by CLI)
DEFAULT_SRC_URI = "mongodb://manager125.bigdata.com:27017"
DEFAULT_DST_URI = "mongodb://root:Bigdata123@m1.bigdata.com:27017,m2.bigdata.com:27017,m3.bigdata.com:27017/test?authSource=admin&replicaSet=rs0"
DEFAULT_CHECKPOINT_FILE = "/tmp/oplog_tail_ts.txt"

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S"
)
logger = logging.getLogger(__name__)
running = True

def signal_handler(signum, frame):
    global running
    logger.info("Received signal %d, shutting down gracefully...", signum)
    running = False


class ProgressTracker(object):
    def __init__(self, db_name, report_interval=10):
        self.db_name = db_name
        self.report_interval = report_interval
        self.last_report_time = time.time()
        self.op_count = 0
        self.total_op_count = 0

    def record_op(self):
        self.op_count += 1
        self.total_op_count += 1

    def should_report(self):
        return time.time() - self.last_report_time >= self.report_interval

    def report(self, last_ts):
        current_time = time.time()
        lag_seconds = int(current_time - last_ts.time)
        elapsed = current_time - self.last_report_time
        ops_per_sec = float(self.op_count) / elapsed if elapsed > 0 else 0.0

        readable_time = time.strftime(
            "%Y-%m-%d %H:%M:%S",
            time.localtime(last_ts.time)
        )
        logger.info(
            "Progress:  total=%d | rate=%.1f ops/s | lag=%d sec | latest_time=%s|db=%s",
            self.total_op_count,
            ops_per_sec,
            lag_seconds,
            readable_time,self.db_name,
        )

        self.op_count = 0
        self.last_report_time = current_time

    def force_report_if_needed(self, last_ts):
        """即使没有新操作，也按需报告进度（用于空闲时心跳）"""
        if self.should_report():
            self.report(last_ts)

class OplogTailer(object):
    def __init__(self, src_uri, dst_uri, db_name, checkpoint_file):
        self.src_uri = src_uri
        self.dst_uri = dst_uri
        self.db_name = db_name
        self.checkpoint_file = checkpoint_file
        self.src_client = None
        self.dst_client = None

    def connect(self):
        self.src_client = MongoClient(self.src_uri)
        self.dst_client = MongoClient(self.dst_uri)

    def close(self):
        if self.src_client:
            self.src_client.close()
        if self.dst_client:
            self.dst_client.close()

    def get_latest_oplog_ts(self):
        local_db = self.src_client["local"]
        doc = local_db.oplog.rs.find_one(
            {"ns": {"$regex": "^" + self.db_name + r"\."}},
            sort=[("ts", -1)]
        )
        if doc:
            return doc["ts"]
        else:
            return Timestamp(int(time.time()), 0)

    def read_last_ts_from_file(self):
        try:
            if os.path.exists(self.checkpoint_file):
                with open(self.checkpoint_file, "r") as f:
                    line = f.read().strip()
                    if line:
                        parts = line.split(",")
                        if len(parts) == 2:
                            return Timestamp(int(parts[0]), int(parts[1]))
        except Exception as e:
            logger.warning("Failed to read checkpoint file: %s", e)
        return None

    def write_last_ts_to_file(self, ts):
        try:
            with open(self.checkpoint_file, "w") as f:
                f.write("{},{}".format(ts.time, ts.inc))
        except Exception as e:
            logger.warning("Failed to write checkpoint file: %s", e)

    def replay_op(self, op):
        ns = op.get("ns", "")
        if not ns.startswith(self.db_name + "."):
            return
        coll_name = ns.split(".", 1)[1]

        target_coll = self.dst_client[self.db_name][coll_name]
        src_coll = self.src_client[self.db_name][coll_name]

        try:
            if op["op"] == "i":
                if coll_name in ("fs.files", "fs.chunks"):
                    real_doc = src_coll.find_one({"_id": op["o"]["_id"]})
                    if real_doc:
                        target_coll.replace_one({"_id": real_doc["_id"]}, real_doc, upsert=True)
                else:
                    target_coll.insert_one(op["o"])
            elif op["op"] == "u":
                filter_ = op.get("o2", {})
                target_coll.update_many(filter_, op["o"])
            elif op["op"] == "d":
                target_coll.delete_many(op["o"])
        except Exception as e:
            if "duplicate key" not in str(e).lower():
                logger.error("Replay failed (ns=%s): %s", ns, e)

    def tail(self, start_ts=None):
        if start_ts is None:
            start_ts = self.read_last_ts_from_file()
            if start_ts is None:
                start_ts = self.get_latest_oplog_ts()
                logger.info("No checkpoint found, starting from latest oplog: %s", start_ts)
            else:
                logger.info("Resuming from checkpoint: %s", start_ts)

        logger.info("Starting to tail oplog from ts: %s", start_ts)
        last_applied_ts = start_ts
        tracker = ProgressTracker(self.db_name,report_interval=10)
        local_db = self.src_client["local"]
        query_filter = {
            "ts": {"$gt": last_applied_ts},
            "ns": {"$regex": "^" + re.escape(self.db_name) + r"\."}  # 防止正则注入
        }

        while running:
            try:
                cursor = local_db.oplog.rs.find(
                    query_filter,
                    cursor_type=CursorType.TAILABLE_AWAIT,
                    oplog_replay=True
                )

                while running and cursor.alive:
                    try:
                        doc = cursor.next()
                        last_applied_ts = doc["ts"]
                        self.replay_op(doc)
                        self.write_last_ts_to_file(last_applied_ts)

                        tracker.record_op()
                        if tracker.should_report():
                            tracker.report(last_applied_ts)

                    except StopIteration:
                        tracker.force_report_if_needed(last_applied_ts)
                        time.sleep(0.1)
                    except Exception as e:
                        logger.error("Error processing oplog document: %s", e)
                        time.sleep(0.5)

            except Exception as e:
                if not running:
                    break
                logger.error("Cursor error (oplog may be rolled over or network issue): %s", e)
                logger.info("Retrying in 2 seconds...")
                time.sleep(2)
            finally:
                try:
                    cursor.close()
                except:
                    pass
        logger.info("Oplog tailing stopped")


class MultiDBOplogSyncer(object):
    SYSTEM_DBS = {"admin", "config", "local"}

    def __init__(self, src_uri, dst_uri, db_names, checkpoint_base, start_ts=None):
        self.src_uri = src_uri
        self.dst_uri = dst_uri
        self.db_names = db_names
        self.checkpoint_base = checkpoint_base
        self.start_ts = start_ts
        self.threads = []
        self.tailers = []

    def resolve_databases(self, client):
        """Resolve 'all' to actual user databases."""
        if len(self.db_names) == 1 and self.db_names[0].lower() == 'all':
            all_dbs = client.database_names()
            user_dbs = [db for db in all_dbs if db not in self.SYSTEM_DBS]
            logger.info("Resolved 'all' to %d user databases: %s", len(user_dbs), user_dbs)
            return user_dbs
        else:
            # Validate that specified DBs exist (optional, but helpful)
            existing = set(client.database_names())
            valid_dbs = []
            for db in self.db_names:
                if db in self.SYSTEM_DBS:
                    logger.warning("Skipping system database: %s", db)
                    continue
                if db not in existing:
                    logger.warning("Database '%s' does not exist on source, will still attempt to sync.", db)
                valid_dbs.append(db)
            return valid_dbs

    def start(self):
        src_client = MongoClient(self.src_uri)
        try:
            dbs_to_sync = self.resolve_databases(src_client)
        finally:
            src_client.close()

        if not dbs_to_sync:
            logger.error("No valid databases to sync.")
            return

        for db in dbs_to_sync:
            checkpoint_file = os.path.join(self.checkpoint_base, "oplog_tail_{}.txt".format(db))
            tailer = OplogTailer(
                src_uri=self.src_uri,
                dst_uri=self.dst_uri,
                db_name=db,
                checkpoint_file=checkpoint_file
            )
            thread = threading.Thread(target=self._run_tailer, args=(tailer,))
            thread.daemon = True
            self.threads.append(thread)
            self.tailers.append(tailer)
            thread.start()
            logger.info("Started sync thread for database: %s", db)

    def _run_tailer(self, tailer):
        try:
            tailer.connect()
            tailer.tail(start_ts=self.start_ts)
        except Exception as e:
            logger.exception("Tailer for DB %s crashed: %s", tailer.db_name, e)
        finally:
            tailer.close()

    def stop(self):
        global running
        running = False
        for t in self.threads:
            t.join(timeout=5)
        logger.info("All sync threads stopped.")

def parse_args():
    parser = argparse.ArgumentParser(description="Tail MongoDB oplog and sync to another instance (multi-db support).")
    parser.add_argument("--src", default=DEFAULT_SRC_URI, help="Source MongoDB URI")
    parser.add_argument("--dst", default=DEFAULT_DST_URI, help="Destination MongoDB URI")
    parser.add_argument("--db", required=True, nargs='+', help="Database name(s) to sync, or 'all' for all non-system DBs")
    parser.add_argument("--checkpoint-base", default="/tmp/oplog_sync", help="Base directory for checkpoint files (one per DB)")
    parser.add_argument("--start-ts", metavar="TIME,INC", help="Start from specific Timestamp (e.g., 1765504800,123)")
    return parser.parse_args()


def main():
    args = parse_args()

    # Parse --start-ts if provided
    start_ts = None
    if args.start_ts:
        try:
            time_part, inc_part = args.start_ts.split(",", 1)
            start_ts = Timestamp(int(time_part), int(inc_part))
            logger.info("Using explicit start timestamp: %s", start_ts)
        except Exception as e:
            logger.error("Invalid --start-ts format. Use 'time,inc' (e.g., 1765504800,123): %s", e)
            sys.exit(1)

    # Ensure checkpoint base dir exists
    if not os.path.exists(args.checkpoint_base):
        os.makedirs(args.checkpoint_base)

    import signal
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

    syncer = MultiDBOplogSyncer(
        src_uri=args.src,
        dst_uri=args.dst,
        db_names=args.db,
        checkpoint_base=args.checkpoint_base,
        start_ts=start_ts
    )

    try:
        syncer.start()
        # Keep main thread alive
        while running:
            time.sleep(1)
    except KeyboardInterrupt:
        logger.info("Main thread interrupted.")
    finally:
        syncer.stop()


if __name__ == "__main__":
    main()
```

如何确认呢，连接两边的数据库查询count即可

如果是gridFS，也可查看最新的文件时间戳

```bash
db.system.users.estimatedDocumentCount()//4.0+支持
db.fs.files.find().count()
db.fs.files.find().sort({uploadDate:-1}).limit(3)
```



### oplog管理

oplog 不按时间保留，而是按空间。一旦写满，旧记录会被覆盖。可以通过以下命令查看 oplog 大小和时间范围：

```js
use local
db.oplog.rs.stats()
// 查看第一条和最后一条的时间
db.oplog.rs.find().sort({$natural: 1}).limit(1)
db.oplog.rs.find().sort({$natural: -1}).limit(1)
```



### 分片集群备份

如果是分片集群，需要连mongos备份。但是恢复步骤就不同了，需要**先搭建好目标分片集群拓扑**，即部署相同数量的 `config server`、`shard`、`mongos`，然后通过 mongos 恢复。

```bash
# 备份整个集群（所有数据库）
mongodump \
  --host "mongos1:27017" \          # 任意一个 mongos 地址
  --username admin \
  --password 'xxx' \
  --authenticationDatabase admin \
  --out /backup/full_$(date +%Y%m%d)
# 恢复
mongorestore \
  --host "new_mongos:27017" \
  --username admin \
  --password 'xxx' \
  --authenticationDatabase admin \
  /backup/full_20251207  
  
```

这时，mongorestore 会自动：

    恢复 config 库 → 重建分片元数据（chunk 范围、shard 位置等）
    恢复业务库数据 → 自动路由到正确的 shard
但是**分片集群不支持 `--oplog`** 。

## 6.修改副本集

不建议手动修改。

```js
// 1. 先获取当前配置
cfg = rs.conf()

// 2. 修改各成员的 priority
cfg.members[0].priority = 10   // 192.168.1.101（希望它当主）
cfg.members[1].priority = 5    // 192.168.1.102
cfg.members[2].priority = 5    // 192.168.1.103

// 3. 重新应用配置（必须在 PRIMARY 上执行！）
rs.reconfig(cfg)
// 强制在 SECONDARY 上 reconfig（不推荐常规使用）
//rs.reconfig(cfg, { force: true })
```

如果你已有 PRIMARY，但想主动切换主节点（比如维护升级），可以：

```js
// 在当前 PRIMARY 上执行，让其主动降级，并触发选举
rs.stepDown(60)  // 60秒内不参与选举，其他节点会选新主
```

如果你不想删数据，只想移除副本集身份（变成 standalone 模式）

```js
//停止所有 secondary；
//在 primary 上执行
use admin
db.runCommand({ "replSetFreeze": 0 })  // 解冻
db.runCommand({ "replSetStepDown": 0 }) // 降级
//修改 mongod.conf，注释或删除 replication段,重启该节点 → 它就变成普通单机实例了,其他节点同理处理
```

## 7.批量插入数据测试

以下均针对mongo2.6.9 测试，使用python2.7版本

### 插入普通数据

```js
// generate_data.js
// 兼容 MongoDB 2.6 的数据生成脚本

var dbName = "testdb";
var collName = "users";
var totalDocs = 1000000; // 生成 100 万条
var batchSize = 1000;    // 每批插入 1000 条（避免内存溢出）

var db = db.getSiblingDB(dbName);
var coll = db[collName];

print("开始生成 " + totalDocs + " 条测试数据...");

var docs = [];
for (var i = 1; i <= totalDocs; i++) {
    docs.push({
        _id: i,
        name: "user_" + i,
        email: "user" + i + "@example.com",
        age: Math.floor(Math.random() * 50) + 18, // 18-67 岁
        created_at: new Date(),
        tags: ["tagA", "tagB", "tagC"][Math.floor(Math.random() * 3)],
        isActive: Math.random() > 0.3,
        address: {
            city: ["Beijing", "Shanghai", "Guangzhou", "Shenzhen"][Math.floor(Math.random() * 4)],
            zip: "100000" + (i % 1000)
        }
    });

    // 每 batch 插入一次
    if (i % batchSize === 0 || i === totalDocs) {
        coll.insert(docs);
        print("已插入 " + i + " 条");
        docs = []; // 清空数组
    }
}

print("数据生成完成！");
mongo localhost:27017/testdb generate_data.js
```

检查数据

```js
use your_database_name
db.your_collection_name.count()//查看集合条数
```

这样操作后，可能会导致 local库体积很大，所有写操作都会被记录到 oplog（操作日志）中。默认会占用5% 的可用磁盘空间。

MongoDB 4.0+（特别是 4.2 起）改变了 oplog 的初始化策略，local库占用就不会很大。

查看验证

```js
use local
db.oplog.rs.count() //或者db.oplog.rs.stats()
// 查看 oplog 大小和使用情况
rs.printReplicationInfo()
```

### 插入文件

内网环境先离线安装pip依赖

```bash
tar -xzf setuptools-1.4.2.tar.gz && cd setuptools-1.4.2 && python setup.py install
tar -xzf  pymongo-2.9.tar.gz && cd  pymongo-2.9 && python setup.py install 
```

python脚本如下

```py
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import os
import sys
from pymongo import MongoClient
from gridfs import GridFS
import argparse
from datetime import datetime

def upload_directory_to_gridfs(mongo_uri, db_name, directory_path):
    """
    将指定目录下的所有文件存入 MongoDB 的 GridFS
    """
    # 连接 MongoDB
    client = MongoClient(mongo_uri)
    db = client[db_name]
    fs = GridFS(db)  # 使用 GridFS

    if not os.path.isdir(directory_path):
        print("错误: 路径不是目录 - {}".format(directory_path))
        return

    uploaded = 0
    skipped = 0

    for root, dirs, files in os.walk(directory_path):
        for filename in files:
            file_path = os.path.join(root, filename)
            rel_path = os.path.relpath(file_path, directory_path)

            # 跳过空文件
            if os.path.getsize(file_path) == 0:
                print("跳过空文件: {}".format(rel_path))
                skipped += 1
                continue

            try:
                with open(file_path, 'rb') as f:
                    # 存入 GridFS，保留原始文件名和相对路径
                    file_id = fs.put(
                        f,
                        filename=filename,
                        metadata={
                            "relative_path": rel_path,
                            "upload_time": datetime.utcnow(),
                            "size": os.path.getsize(file_path)
                        }
                    )
                print("已上传: {} (ID: {})".format(rel_path, file_id))
                uploaded += 1

            except Exception as e:
                print("上传失败 {}: {}".format(rel_path, e))
                skipped += 1

    print("\n--- 完成 ---")
    print("成功上传: {} 个文件".format(uploaded))
    print("跳过/失败: {} 个文件".format(skipped))

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="将目录下所有文件上传到 MongoDB GridFS")
    parser.add_argument("directory", help="要上传的本地目录路径")
    parser.add_argument("--host", default="localhost", help="MongoDB 主机 (默认: localhost)")
    parser.add_argument("--port", type=int, default=27017, help="MongoDB 端口 (默认: 27017)")
    parser.add_argument("--db", default="filedb", help="数据库名 (默认: filedb)")

    args = parser.parse_args()

    mongo_uri = "mongodb://{}:{}".format(args.host, args.port)
    upload_directory_to_gridfs(mongo_uri, args.db, args.directory)
```

可以用如下脚本生成随机文件

```bash
#!/bin/bash
for i in {1..100}; do
    # 生成 10~500 之间的随机数（单位：MB）
    size=$(( RANDOM % 491 + 10 ))
    echo "Creating file_$i.dat ($size MB)"
    dd if=/dev/urandom of="file_$i.dat" bs=1M count=$size status=progress 2>/dev/null
done
```

如果是mongo4.4版本，只需要更新下pytmongo依赖即可

```bash
 pip install pymongo-3.12.3-cp27-cp27mu-manylinux1_x86_64.whl --no-index --find-links ./
```


