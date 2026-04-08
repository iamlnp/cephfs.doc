# 1 顶层模型

- **Tenant（租户）**：多租户隔离，`tenant$user` 形式
- **User（用户）**：身份实体，拥有 AK/SK
- **Subuser（子用户）**：用于 Swift 子账户 / 权限细分
- **Bucket（桶）**：命名空间、容器、权限边界
- **Object（对象）**：数据主体，key-value + metadata
- **Bucket Index（桶索引）**：对象列表 / 元数据索引（OMAP）

## 1.1 桶  
在 Ceph RGW 中，**StoreBucket** 是**标准存储桶**，用于持久化存储对象；**FilterBucket** 是**过滤桶（索引桶）**  

| 维度    | StoreBucket（标准存储桶）                    | FilterBucket（过滤桶 / 索引桶）                     |
|-------|---------------------------------------|---------------------------------------------|
| 核心作用  | 存储、托管对象，提供持久化与基本管理                    | 按前缀过滤对象，加速列表 / 查询，不存储原始数据                   |
| 数据存放  | 存对象本体（default.rgw.buckets.data 池）Ceph | 存索引 / 标记（default.rgw.buckets.index 池），无原始数据 |
| 典型场景  | 常规对象存储、数据归档、业务数据托管                    | 海量小对象列表、按前缀快速筛选、多区同步细粒度控制                   |
| 生命周期  | 永久存在，需手动创建 / 删除                       | 随主桶创建，与主桶同生命周期，自动清理                         |
| 性能影响  | 写放大小，读稳定；列表操作随对象数增长                   | 写放大略增（索引维护），列表 / 前缀查询更快                     |
| 配置与依赖 | 基础配置（配额、生命周期、ACL）                     | 依赖索引池（index_pool），支持分片与多区策略                 |


# 2 数据结构

- **Object（数据对象）**：实际数据
- **Head Object（对象头）**：元数据 + manifest 指针
- **Manifest（清单）**：大对象分片索引
- **Shadow Object（影子对象）**：大对象分片实体
- **Multipart（分段）**：大文件上传（part + 合并）

# 3 存储位置

- **.rgw.root**：全局元数据（用户、桶、配额）
- **.rgw.control**：同步 / 分片信息
- **.rgw.bucket.index**：桶索引
- **.rgw.bucket.data**：对象数据（可自定义池）

# 4 核心流程抽象

- **Put**：数据 → manifest → head → bucket index
- **Get**：head → manifest → 读分片 → 返回
- **List**：遍历 bucket index（OMAP）
- **Delete**：写墓碑（tombstone）→ 异步 GC