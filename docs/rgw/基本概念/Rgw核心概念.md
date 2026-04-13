# 1 顶层模型

- **Tenant（租户）**：多租户隔离，`tenant$user` 形式
- **User（用户）**：身份实体，拥有 AK/SK
- **Subuser（子用户）**：用于 Swift 子账户 / 权限细分
- [Bucket(桶)](Bucket(桶)简要介绍.md)：命名空间、容器、权限边界
- [Object(对象)](Object(对象).md)：数据主体，key-value + metadata
- **Bucket Index（桶索引）**：对象列表 / 元数据索引（OMAP）

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