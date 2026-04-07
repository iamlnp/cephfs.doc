`radosgw-admin` 是 Ceph 对象网关（RGW）的命令行管理工具，用于管理和维护整个对象网关服务。与通过 `aws s3` 命令操作桶和对象不同，`radosgw-admin` 主要执行**管理类操作**，如创建用户、查看集群元数据、管理配额、删除孤儿数据等。

以下是其核心功能和使用方法：

---
# 1 用户管理

用户管理是 `radosgw-admin` 最基础的功能。创建用户后你会获得 `access_key` 和 `secret_key`，这是后续通过 S 3 客户端访问的凭证。

| 操作         | 命令示例                                                                | 说明                             |
| ---------- | ------------------------------------------------------------------- | ------------------------------ |
| **创建用户**   | `radosgw-admin user create --uid=johndoe --display-name="John Doe"` | 生成 access_key 和 secret_key     |
| **查看用户信息** | `radosgw-admin user info --uid=johndoe`                             | 显示用户的 key、配额、子用户等详情            |
| **修改用户**   | `radosgw-admin user modify --uid=johndoe --max-buckets=2000`        | 可调整配额、邮箱、暂停状态等                 |
| **删除用户**   | `radosgw-admin user rm --uid=johndoe --purge-data`                  | `--purge-data` 会同时删除该用户的所有数据和桶 |
| **列出所有用户** | `radosgw-admin user list`                                           | 返回所有用户的 uid 列表                 |

> **多站点环境注意**：在多站点部署中，始终在 **master zone** 的主机上执行用户操作，否则可能因同步问题导致不一致。

## 1.1 创建 Swift 子用户
如果需要通过 Swift 协议访问，需要额外创建子用户：

```bash
# 创建 Swift 子用户
radosgw-admin subuser create --uid=johndoe --subuser=johndoe:swift --access=full

# 生成 Swift 密钥
radosgw-admin key create --subuser=johndoe:swift --key-type=swift --gen-secret
```

# 2 Zone  
1. 查询当前zone的配置
```bash
[root@node1 ~]# radosgw-admin zone get
{
    "id": "03b4c78e-c70f-4ad7-b229-608bff012f1b",
    "name": "magnascale",
    "domain_root": "magnascale.tos.meta:root",
    "control_pool": "magnascale.tos.control",
    ...
}
```

2. 查询所有zone
```bash
[root@node1 ~]# radosgw-admin zone list
{
    "default_info": "03b4c78e-c70f-4ad7-b229-608bff012f1b",
    "zones": [
        "magnascale"
    ]
}
```
3. 查询当前的zonegroup
```bash
[root@node1 ~]# radosgw-admin zonegroup get
{
    "id": "14ebae29-9348-44fc-84ae-e9174054083b",
    "name": "magnascale",
    "api_name": "magnascale",
    "is_master": "true",
    "endpoints": [],
    "hostnames": [],
    ......
}   
```
4. 创建新 zone（多站点用）
```bash
radosgw-admin zone modify --zone=zone名 --endpoint=http://x.x.x.x:8080
```

5. 删除zone
```bash
radosgw-admin zone delete --zone=zone名
```
# 3 桶管理

| 操作 | 命令示例 | 说明 |
|------|----------|------|
| **列出用户的所有桶** | `radosgw-admin bucket list --uid=johndoe` | 返回桶名列表 |
| **查看桶详情** | `radosgw-admin bucket stats --bucket=mybucket` | 显示桶大小、对象数、所有者等信息 |
| **调整桶索引分片** | `radosgw-admin bucket reshard --bucket=mybucket --num-shards=16` | 当单个桶对象数超过 10 万时，分片可提升性能 |
| **删除空桶** | `radosgw-admin bucket rm --bucket=mybucket` | 仅删除空桶 |
| **强制删除桶及内容** | `radosgw-admin bucket rm --bucket=mybucket --purge-objects` | 递归删除桶内所有对象 |
| **将桶关联给其他用户** | `radosgw-admin bucket link --bucket=mybucket --uid=otheruser` | 不改变所有权，仅关联 |
| **更改桶所有权** | `radosgw-admin bucket chown --bucket=mybucket --uid=otheruser` | 彻底转移桶所有权 |

## 3.1 创建桶 
 1. **桶的所有者用户必须先存在**。如果用户不存在，需要先创建
 ```bash
 # 1. 创建用户（如果还没有）
radosgw-admin user create --uid=4001 --display-name=user1
# 回显
{
     "user_id": "4001",
     "display_name": "user1",
     "email": "",
     "suspended": 0,
     "max_buckets": 1000,
     "subusers": [],
     "keys": [
         {
             "user": "4001",
             "access_key": "KF8LPEA54DGT9SMI2KUY",
             "secret_key": "WqUjZDnOYtEJew6ViW4TnvzmqeGnlyg9UQ6nbwvC"
         }
     ],
 }
 ```
 
 2. 创建桶
 新版 Ceph 已经去掉了 `bucket create` 命令，Ceph 在 **Pacific 版之后** 就废弃了，现在创建桶必须通过 **S 3 协议** 或 **link 关联**。
 ```bash
 # 方法1：使用 s3cmd 创建桶（最简单、推荐）
 s3cmd mb s3://my-new-bucket
 
 # 方法2：使用 radosgw-admin 间接创建（先给用户，再自动生成）
 radosgw-admin bucket link --bucket=my-new-bucket --uid=4001
 ```

# 4 元数据管理

RGW 的内部元数据存储在 RADOS 对象中，可通过 `metadata` 命令查看和修改。

```bash
# 列出所有元数据类型
radosgw-admin metadata list
# 输出示例: ["bucket", "bucket.instance", "user", "account", "roles", ...]

# 列出所有用户元数据
radosgw-admin metadata list user

# 查看具体用户的元数据详情
radosgw-admin metadata get user:johndoe

# 查看具体桶的元数据
radosgw-admin metadata get bucket:mybucket
```

元数据类型包括：
- `user`：用户信息（含密钥、配额等）
- `bucket`：桶名到桶实例 ID 的映射
- `bucket.instance`：桶实例的详细信息
- `account`：账户级信息
- `roles`：IAM 角色定义

# 5 配额管理

配额可限制用户或桶的资源使用量。

```bash
# 设置用户配额：最大对象数 1000，最大容量 10GB
radosgw-admin quota set --uid=johndoe --quota-scope=user \
  --max-objects=1000 --max-size=10G

# 设置桶配额
radosgw-admin quota set --uid=johndoe --quota-scope=bucket \
  --max-objects=500 --max-size=5G

# 查看用户配额
radosgw-admin user info --uid=johndoe | grep -A10 quota

# 禁用配额
radosgw-admin quota disable --uid=johndoe --quota-scope=user
```


# 6 多租户（Multi-tenancy）

从 Jewel 版本开始，RGW 支持多租户，允许多个用户使用相同的桶名（通过租户前缀隔离）。

```bash
# 创建带租户的用户（租户名为 "tenant1"）
radosgw-admin user create --tenant=tenant1 --uid=tester \
  --display-name="Test User" --access_key=TESTKEY --secret=test123

# 或使用 $ 语法（需引号包裹）
radosgw-admin user create --uid='tenant1$tester' --display-name="Test User"
```

租户用户的访问方式：
- **S3 API**：URL 中使用冒号分隔，如 `http://rgw-host/tenant1:mybucket/myobject`
- **公开读的桶**：必须带租户前缀访问，否则无法解析

# 7 日志与使用统计

```bash
# 查看用户使用统计（某时间范围）
radosgw-admin usage show --uid=johndoe \
  --start-date=2025-01-01 --end-date=2025-03-01

# 仅显示汇总（不显示明细条目）
radosgw-admin usage show --show-log-entries=false

# 清理旧的用量日志
radosgw-admin usage trim --uid=johndoe --end-date=2025-01-01
```

# 8 常用运维命令速查

| 场景 | 命令 |
|------|------|
| **检查网关状态** | `systemctl status ceph-radosgw@*` 或查看日志 |
| **重建桶索引** | `radosgw-admin bucket check --bucket=mybucket --fix` | 
| **暂停用户（禁止访问）** | `radosgw-admin user modify --uid=johndoe --suspended=1` |
| **重新生成用户密钥** | `radosgw-admin key create --uid=johndoe --gen-access-key --gen-secret` |
| **查看桶索引分片状态** | `radosgw-admin bucket stats --bucket=mybucket` 查看 `num_shards` 字段 |

# 9 ⚠️ 重要提醒

1. **版本一致性**：确保 `radosgw-admin` 和 Ceph 集群是**相同版本**，混用可能导致元数据损坏
2. **JSON 转义问题**：某些客户端无法处理包含反斜杠 `\` 的密钥。如果生成的密钥包含 JSON 转义符，建议重新生成或手动指定
3. **多站点环境**：用户、桶创建等写操作必须在 **master zone** 执行，备 zone 会自动同步
4. **元数据不可直接编辑**：除 `metadata set` 命令外，不要直接操作 RADOS 对象修改元数据
