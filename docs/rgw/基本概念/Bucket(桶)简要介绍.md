bucket_tenant和bucket_name一起组成全局唯一的桶全名  
- **bucket_name = 桶的名字**
    - 就是你 `aws s3 mb s3://abc` 里的 **abc**
    - 用户可见、S3 协议里的桶名
- **bucket_tenant = 租户（多用户隔离用）**
    - 大多数场景 **为空（""）**
    - 只有 **多租户、swift 用户、子用户** 才会有值
    - 作用：**让不同租户可以使用相同的桶名**

在 Ceph RGW 中，**StoreBucket** 是**标准存储桶**，用于持久化存储对象；**FilterBucket** 是**过滤桶（索引桶）**  

| 维度    | StoreBucket（标准存储桶）                    | FilterBucket（过滤桶 / 索引桶）                     |
|-------|---------------------------------------|---------------------------------------------|
| 核心作用  | 存储、托管对象，提供持久化与基本管理                    | 按前缀过滤对象，加速列表 / 查询，不存储原始数据                   |
| 数据存放  | 存对象本体（default.rgw.buckets.data 池）Ceph | 存索引 / 标记（default.rgw.buckets.index 池），无原始数据 |
| 典型场景  | 常规对象存储、数据归档、业务数据托管                    | 海量小对象列表、按前缀快速筛选、多区同步细粒度控制                   |
| 生命周期  | 永久存在，需手动创建 / 删除                       | 随主桶创建，与主桶同生命周期，自动清理                         |
| 性能影响  | 写放大小，读稳定；列表操作随对象数增长                   | 写放大略增（索引维护），列表 / 前缀查询更快                     |
| 配置与依赖 | 基础配置（配额、生命周期、ACL）                     | 依赖索引池（index_pool），支持分片与多区策略                 |

# 1 五种桶简要介绍  
在 Ceph RGW 中，**POSIXBucket、RadosBucket、MotrBucket、DaosBucket、DBBucket** 是五种底层存储引擎 / 语义不同的桶类型，核心差异在于 **创建方式、底层存储、API 兼容性、适用场景**  
创建方式 + 核心区别：  

| 桶类型         | 创建方式                         | 底层存储                      | 核心语义             | S3 兼容 | 适用场景            |
|-------------|------------------------------|---------------------------|------------------|-------|-----------------|
| RadosBucket | S3 命令 /radosgw-admin（默认）     | RADOS 对象（.dir 索引 + 数据）    | S3 对象（扁平、不可改）    | ✅ 完全  | 标准对象存储、S3 业务    |
| POSIXBucket | radosgw-admin --posix-flags  | CephFS + RGW 映射           | POSIX 文件（目录、可追加） | ❌ 有限  | 文件共享、NFS 代理、可修改 |
| MotrBucket  | radosgw-admin + Motr 集群      | Seagate Motr 分布式对象        | Motr 对象接口        | ✅ 部分  | 高性能、低时延、海量对象    |
| DaosBucket  | radosgw-admin + DAOS 集群      | DAOS 分布式对象存储              | DAOS KV / 对象     | ✅ 部分  | 高性能计算（HPC）、AI   |
| DBBucket    | radosgw-admin --bucket-attrs | RocksDB/LevelDB（元数据 + 数据） | KV 数据库语义         | ❌ 有限  | 元数据密集、小对象、强事务   |

## 1.1 详细介绍
### 1.1.1 RadosBucket（标准桶 / StoreBucket）  
1. 创建
```bash
# S3 API（默认）
aws s3 mb s3://my-bucket
s3cmd mb s3://my-bucket

# radosgw-admin（默认）
radosgw-admin bucket create --bucket=my-bucket --uid=user1
```
2. 底层
    - RADOS 池：`default.rgw.buckets.index` + `default.rgw.buckets.data`
    - 索引：`.dir.{bucket_id}` 对象（OMAP 键值）
    - 数据：分片 RADOS 对象
3. 特性
    - 扁平结构、**无真实目录**（仅前缀）
    - 一次写入、不可修改（只能覆盖）
    - 强一致性、版本、生命周期、跨域、多区同步
4. 查询
```bash
radosgw-admin bucket stats --bucket=my-bucket 
# "type": "RadosBucket", "posix": ""
```

### 1.1.2 POSIXBucket（文件兼容桶）  
1. 创建（必须显式指定）
```bash
radosgw-admin bucket create \
  --bucket=px-bucket \
  --uid=user1 \
  --posix-flags=all  # enable posix mode
```
2. 底层
    - CephFS（MDS 管理目录 / 权限）+ RGW 元数据映射
    - 真实 inode、目录树、硬 / 软链接
3. 特性
    - 支持：mkdir/rmdir/link/chmod/chown/append/truncate
    - 可通过 NFS/S3/FUSE 挂载
    - 不兼容 S3 完整语义（如版本、分段上传受限）
4. 查看
```bash
radosgw-admin bucket stats --bucket=px-bucket
# "type": "POSIXBucket", "posix": "enabled"
```
### 1.1.3 MotrBucket（Seagate Motr 引擎桶）
1. 创建（需对接 Motr 集群）
```bash
radosgw-admin bucket create \
  --bucket=motr-bucket \
  --uid=user1 \
  --placement-id=motr-pool  # 指定Motr布局
```
2. 底层
    - Seagate Motr 分布式对象存储（替代 RADOS）
    - 高并发、低时延、海量对象（百亿级）
3. 特性
    - 兼容 S3 基础接口
    - 无 RADOS 开销、更高 IOPS / 带宽
    - 企业级高性能场景
4. 查看
```bash
radosgw-admin bucket stats --bucket=motr-bucket
# "type": "MotrBucket"
```
### 1.1.4 DaosBucket（DAOS 引擎桶）
1. 创建（需对接 DAOS 集群）
```bash
radosgw-admin bucket create \
  --bucket=daos-bucket \
  --uid=user1 \
  --placement-id=daos-pool
```
2. 底层
    - DAOS（Distributed Asynchronous Object Storage）
    - 自研用户态、PMem/SSD 优化、无 OSD
3. 特性
    - HPC/AI 优化：高带宽、低时延、小对象聚合
    - 兼容 S3 基础接口
    - 强事务、快照、克隆
4. 查看
```bash
radosgw-admin bucket stats --bucket=daos-bucket
# "type": "DaosBucket"
```
### 1.1.5 DBBucket（数据库桶）
1. 创建（KV 数据库语义）
```bash
radosgw-admin bucket create \
  --bucket=db-bucket \
  --uid=user1 \
  --bucket-attrs=db_backend=rocksdb
```
2. 底层
    - RocksDB/LevelDB：元数据 + 数据均存 KV
    - 单桶单 DB、强事务、范围查询
3. 特性
    - 小对象性能优、元数据密集场景
    - 支持事务、条件更新、范围扫描
    - S3 兼容差（无版本、无分段）
3. 查看
```bash
radosgw-admin bucket stats --bucket=db-bucket
# "type": "DBBucket", "db_backend": "rocksdb"
```