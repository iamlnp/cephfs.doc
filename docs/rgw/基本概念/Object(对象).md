# 1 基本概念
`class Object` 是 **SAL (Storage Abstraction Layer，存储抽象层)** 架构中的核心接口之一。它的核心作用是**抽象并封装对单个对象（Object）的所有操作**，使得上层的 REST API 处理逻辑（如 S3/Swift）无需关心底层存储引擎的具体实现（如 RADOS、数据库或本地文件系统）。  

`class Object` 数据存储的基本单位，表示一块数据、一组元数据和一系列k-v属性的集合，见 ` rgw_sal.h ` 的定义。  

`class Object` 定义了一系列纯虚函数，其具体实现由各存储后端（如 `RadosObject`）提供。它主要管理以下几类操作 [](https://deepwiki.com/ceph/ceph/4.1-rados-gateway-\(rgw\))：

| 功能类别           | 主要方法 / 操作                                        | 功能描述                                                                      |
| -------------- | ------------------------------------------------ | ------------------------------------------------------------------------- |
| **对象元数据与属性**   | `get_obj_attrs`, `set_obj_attrs`, `get_metadata` | 获取或设置对象的元数据，包括用户自定义的元数据（`x-amz-meta-*`）、系统属性（如 `Content-Type`）、ACL、存储级别等。 |
| **数据读写 (I/O)** | `ReadOp::prepare/read`, `WriteOp::prepare/write` | 准备并执行对象的读写操作。`ReadOp` 支持处理 Range 请求、加密解密和获取对象校验和。`WriteOp` 负责处理数据流的上传。    |
| **对象生命周期与状态**  | `delete_object`, `copy_object`, `restore_obj`    | 执行对象的删除、拷贝，或管理云分层（Cloud Tiering）对象的取回（restore）和过期（expirer）逻辑 。            |
| **版本控制与多部分上传** | `list_parts`, `abort_multipart`                  | 支持多部分上传（Multipart Upload）的管理，例如列出已上传的分块，以及处理版本控制逻辑下的对象状态。                 |
| **状态管理**       | `get_obj_state`, `load_obj_state`                | 获取对象的状态信息（如大小、修改时间、存储类、版本 ID 等）。早期直接暴露内部状态指针，后来为了安全性和清晰性，重构为独立的加载与访问方法    |

在 RGW 处理一次 S3 或 Swift 请求时，`RGWOp` 类的派生类（如处理 GET 请求的 `RGWGetObj_ObjStore_S3`）会通过 `req_state` 结构体持有一个指向 `rgw::sal::Object` 的指针 (`s->object`)。整个请求处理流程中，操作对象（如验证权限后读取数据）都是通过调用该 `Object` 接口的方法来完成的  

# 2 FilterObject和StoreObject  
在 Ceph RGW 的 SAL（存储抽象层）架构中，`class FilterObject` 和 `class StoreObject` 可以看作是 **"功能包装器"** 与 **"存储执行者"** 的关系。它们都实现了 `rgw::sal::Object` 接口。  
简而言之，`StoreObject` 定义了"**如何存储和获取数据**"，而 `FilterObject` 定义了"**在存取数据前后，我们还需要额外做些什么**"。  

### 2.1.1 核心区别与联系

| 特性         | `StoreObject` (以 `RadosObject` 为例)                                | `FilterObject`                                                                                                                                                  |
| ---------- | ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **核心角色**   | 存储后端的具体**执行者**，负责与底层存储系统（如 RADOS）直接交互。                            | 请求处理链中的**包装器/过滤器**，负责拦截请求并在调用实际存储操作前后注入特定逻辑。                                                                                                                    |
| **主要职责**   | 1. 直接读写底层存储引擎的数据和元数据[^1]。  <br>2. 封装 `RGWObjState` 等存储后端特有的状态细节。  | 1. **不直接**访问底层存储。  <br>2. 持有一个指向下一个 `Object` 的指针（通常为 `StoreObject` 或其他 `FilterObject`），将核心操作**委托**给它执行。  <br>3. 在委托前后，处理如**日志记录**、**计量**、**缓存**、**安全检查**等横切关注点。 |
| **SAL 层次** | **底层**：SAL 接口的具体实现层，与特定存储引擎强绑定。                                   | **中间层**：SAL 接口的装饰层，对上层业务逻辑透明。                                                                                                                                   |
| **典型变体**   | `rgw::sal::rados::RadosObject`  <br>`rgw::sal::dbstore::DBObject` | `rgw::sal::filter::LoggingFilterObject`（用于日志记录）等。                                                                                                               |

# 3 多种StoreObject的简要介绍  
这五个类（`DBObject`、`DaosObject`、`MotrObject`、`POSIXObject`、`RadosObject`）都是 `rgw::sal::Object` 接口的**具体实现类**。它们是 RGW SAL（存储抽象层）架构中的"实干家"，负责将统一的对象操作接口（如读、写、删）翻译成对应底层存储引擎的具体调用。[^2]

**它们之间是平等的、可相互替换的关系，而非继承或装饰关系**——都遵循相同的 `rgw::sal::Object` 接口规范，但分别服务于完全不同的存储后端。  

| 类名          | 底层存储后端                       | 核心作用与定位                                                                                |
|-------------|------------------------------|----------------------------------------------------------------------------------------|
| RadosObject | RADOS (Ceph 原生对象存储)          | 默认的核心实现，将对象数据分片为 4MB 的块，通过 librados API 直接存入 Ceph 最底层的 RADOS 集群，是生产环境中 RGW 的标配。        |
| DBObject    | SQLite (嵌入式关系数据库)            | 轻量级测试实现，将对象元数据和数据都存储在 SQLite 数据库的表中。因其性能和扩展性受限，主要用于开发和功能验证，不适合生产环境的大文件场景。              |
| DaosObject  | DAOS (Intel 的分布式异步对象存储)      | 高性能计算 (HPC) 对接，为 Intel 的 DAOS 存储系统提供支持。DAOS 针对 NVMe 等高性能介质做了极致优化，能发挥出远超传统存储的 IOPS 和带宽。 |
| MotrObject  | CORTX Motr (Seagate 的对象存储核心) | 开源存储生态对接，为 Seagate 的开源对象存储系统 CORTX 的核心组件 Motr 提供支持，使 RGW 的 S3 能力可以服务于 CORTX。           |
| POSIXObject | POSIX 文件系统 (如 XFS, ext4)     | 模拟与特殊场景实现，将对象以文件形式直接存储在服务器本地文件系统上，用文件名和目录树模拟桶和对象的层级结构，可用于开发调试或特定的轻量级部署。                |



[^1]: https://deepwiki.com/ceph/ceph/4.1-rados-gateway-(rgw)
[^2]: https://blog.csdn.net/tph5559/article/details/141642390
