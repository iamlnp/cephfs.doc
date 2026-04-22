---
年份:
  - "2026"
月份:
  - 4月
imageNameKey: Rgw put和get的源码流程
tags:
  - rgw
  - ceph20
---


Ceph RGW 的 **Put Object（上传）** 与 **Get Object（下载）** 是对象存储的核心操作，整体遵循 “**请求接入 → 权限校验 → 元数据 / 索引处理 → 数据读写 → 响应返回**” 的流程，底层基于 **librados** 与 RADOS 集群交互。  

所有 RGW 请求（含 Put/Get）均从前端（如 Civetweb、Beast）接入，经统一调度后分发到对应操作类执行。  
# 1 概要  
## 1.1 处理框架
### 1.1.1 前端**Frontend**[^1]入口  
```c++
main() //rgw_main.cc
    -> rgw::AppMain::init_frontends1()
    -> rgw::AppMain::init_frontends2()
        -> if framework == "loadgen" => fe = new RGWLoadGenFrontend()
        -> else if framework == "beast"   => fe = new RGWAsioFrontend()
        -> else if framework == "rgw-nfs" => fe = new RGWLibFrontend()
        -> fe->init()
        -> fe->run()
```

Civetweb vs Beast[^2]是 **Ceph RGW (RADOS Gateway)** 支持的**前端（Frontend）**，负责处理 HTTP/HTTPS 请求并与 S3/Swift API 交互。  

Civetweb 配置方式：  
```bash
[client.rgw.tosa]
rgw_frontends = civetweb port=7480
# 带 SSL
rgw_frontends = civetweb port=7480 ssl_port=7481 ssl_certificate=/etc/ceph/cert.pem
# 更多选项
rgw_frontends = civetweb port=7480 num_threads=100 request_timeout_ms=30000
```
Beast 配置方式：  
```bash
[client.rgw.tosa]
rgw_frontends = beast port=8184
# 带 SSL
rgw_frontends = beast port=8184 ssl_port=8182 ssl_certificate=/etc/ceph/cert.pem
# 多线程（推荐）
rgw_frontends = beast port=8184 ssl_port=8182 ssl_certificate=/etc/ceph/cert.pem num_threads=24
```

Civetweb（多线程模型）  
```bash
客户端请求 → 主线程接收 → 分配线程 → 线程处理请求 → 返回响应
                ↓
        每增加一个连接，增加一个线程
        10,000 连接 = 10,000 个线程
```
Beast（异步模型）  
```bash
客户端请求 → IO 线程接收 → 注册回调 → 继续处理其他请求 → 回调返回响应
                ↓
        少量线程（如 24 个）处理数万连接
        10,000 连接 = 24 个线程
```

### 1.1.2 请求处理  
```c++
AsioFrontend::accept()  // rgw_aio_frontend.cc
    -> AsioFrontend::on_accept() // rgw_aio_frontend.cc
        -> void handle_connection() // rgw_aio_frontend.cc
            -> int process_request() // 核心处理函数：rgw_process.cc, 日志： starting new request req 
```
### 1.1.3 路由分发  

`rest->get_handler()` 根据 URL/Method 匹配 Handler（如 `RGWHandler_REST_S3`） → `handler->get_op()` 实例化操作类（`RGWPutObj` / `RGWGetObj`）  

class RGWOP 是虚类，各种操作对该类进行实现，比如 RGWGetObj(get操作)、RGWPutObj(put 操作）、RGWCreateBucket(创建桶操作)等等

```c++
class RGWRESTMgr {
    virtual RGWHandler_REST* get_handler()
}

class RGWOp : public DoutPrefixProvider {
protected:
    req_state *s;
public:
    virtual void send_response() {}
    virtual void pre_exec() {}
    virtual void execute(optional_yield y) = 0;
    virtual void complete() {
        send_response();
    }
}

int process_request() //rgw_process.cc
       /* 根据请求方法（PUT）、URL（bucket/object）匹配 S3 处理器，在RGWRESTMgr_S3::get_handler()或者
       RGWRESTMgr_SWIFT::get_handler()等返回不同类型的handler
       TDOD: RGWHandler_REST的初始位置待确认
       **/
    -> RGWHandler_REST *handler = rest->get_handler() 
        -> preprocess()
        -> RGWRESTMgr *m = mgr.get_manager() //基于frontend_prefix、decoded_uri和relative_uri获取RGWRESTMgr
        -> RGWHandler_REST* handler = m->get_handler(driver, s, auth_registry, frontend_prefix);
        -> handler->init()
    -> op = handler->get_op(); // 获取实例化操作类，比如RGWPutObj/RGWGetObj
    -> rgw::lua::request::execute()
    -> op->verify_requester(); //认证
    -> rgw_process_authenticated() //认证完成后的op执行流程
    -> （RGWRestfulIO)client_io->complete_request() //完成请求
```
#### 1.1.3.1 RGWHandler_REST::get_op  
路由分发的主体函数之一，根据op类型，生成不同的RGWOp  
```c++
RGWOp* RGWHandler_REST::get_op(void)
{
    RGWOp *op;
    switch (s->op) {
        case OP_GET:
            op = op_get();
            break;
        case OP_PUT:
            op = op_put();
            break;
        case OP_DELETE:
            op = op_delete();
            break;
        case OP_HEAD:
            op = op_head();
            break;
        case OP_POST:
            op = op_post();
            break;
        case OP_COPY:
            op = op_copy();
            break;
        case OP_OPTIONS:
            op = op_options();
            break;
        default:
            return NULL;
        }
    
    if (op) {
        op->init(driver, s, this);
    }
    return op;
}
```

### 1.1.4 执行流程  
```c++
rgw_process_authenticated() //认证完成后的op执行流程
    -> handler->init_permissions()
    -> op->init_processing()
    -> op->verify_op_mask()
    -> op->verify_permission(y);
    -> op->verify_params()
    -> op-> () //step1: 预处理，前置校验与准备
    -> op->execute()  //step2: 数据读写核心流程
    -> op->complete() //step3: 元数据更新与收尾
```

## 1.2 强一致性保证  
- **强一致性**：为了保证对象的上传/删除与索引更新的原子性，RGW设计了一套**3步索引事务（Index Transaction）** 机制[^3]。
    1. **Prepare**：在索引中预操作，将对象标记为 `pending` （待处理）状态。
    2. **Write/Delete**：在数据池中执行实际的对象数据写入或删除。
    3. **Commit/Cancel**：若数据操作成功，则提交事务，将索引条目状态更新为 `completed` （已完成）；若失败，则取消事务。
- **故障自愈**：如果RGW在处理过程中崩溃，可能会导致某些索引条目长时间停留在 `pending` 状态。当客户端后续列出桶时，RGW会检测到这些 `pending` 条目，并去检查对应的实际对象是否存在，然后用检查结果来“修正”索引（这个过程称为 `dir suggest` ）  [^4]

# 2 [Put上传流程](Put上传流程.md)  

# 3 [Get下载流程](Get下载流程.md)

# 4 [创建Bucket流程](创建Bucket流程.md)  

# 5 List buckets-列举桶
RGWListBuckets_ObjStore_S3
**核心职责**：返回用户拥有的桶列表，支持分页、前缀过滤等参数

对应操作： python3 /usr/bin/s3cmd ls s3://my-new-bucket

```c++
class RGWListBuckets_ObjStore_S3 : public RGWListBuckets_ObjStore {

}

class RGWListBuckets_ObjStore : public RGWListBuckets {

}

class RGWListBuckets : public RGWOp {
public:
    void pre_exec() override;
    void execute(optional_yield y) override;
}
```

类继承体系：  

```mermaid
classDiagram
    class RGWOp {
        <<abstract>>
        +verify_permission()
        +execute()
        +send_response()
    }
    class RGWListBuckets {
        +verify_permission()
        +execute()
    }
    class RGWListBuckets_ObjStore {
        <<abstract>>
    }
    class RGWListBuckets_ObjStore_S3 {
        +send_response()
    }

    RGWOp <|-- RGWListBuckets
    RGWListBuckets <|-- RGWListBuckets_ObjStore
    RGWListBuckets_ObjStore <|-- RGWListBuckets_ObjStore_S3
```



### 5.1.1 核心执行阶段  
#### 5.1.1.1 RGWListBucket::execute  

# 6 RGWListBucket-查询桶内容  
**核心职责**：返回指定桶内的对象列表，支持分页、前缀过滤、版本控制等参数。  

**类继承体系**  
```mermaid
classDiagram
    class RGWOp {
        <<abstract>>
        +verify_permission()
        +execute()
        +send_response()
    }
    class RGWListBucket {
        +verify_permission()
        +execute()
    }
    class RGWListBucket_ObjStore {
        <<abstract>>
    }
    class RGWListBucket_ObjStore_S3 {
        +send_response()
    }

    RGWOp <|-- RGWListBucket
    RGWListBucket <|-- RGWListBucket_ObjStore
    RGWListBucket_ObjStore <|-- RGWListBucket_ObjStore_S3
```

# 7 核心操作对比  

## 7.1 上传、下载、创建桶、查询桶列表和查询桶内容对比

| 维度     | RGWGetObj_ObjStore                | RGWCreateBucket                      | RGWListBuckets           | RGWListBucket                                 |
| ------ | --------------------------------- | ------------------------------------ | ------------------------ | --------------------------------------------- |
| 核心操作类型 | 数据读（Data Read）                    | 元数据写（Meta Write）                     | 元数据读（Meta Read）          | 索引读（Index Read）                               |
| 涉及的存储池 | data + index                      | meta + index                         | meta                     | index                                         |
| 主要数据结构 | rgw_bucket_dir_entry<br>RADOS对象分片 | RGWBucketEntryPoint<br>RGWBucketInfo | 用户桶列表对象<br>RGWBucketInfo | rgw_bucket_dir_header<br>rgw_bucket_dir_entry |
| 性能瓶颈   | 磁盘I/O + 网络带宽                      | 元数据池的写入延迟                            | 用户桶列表对象的大小               | 索引分片数量 + 对象数量                                 |
| 特殊功能   | Range请求、加密解密、压缩解压                 | 桶分片、存储类、对象锁                          | 分页、前缀过滤、缓存               | delimiter目录模拟、版本控制分页                          |
| 幂等性    | GET 天然幂等                          | 重复创建返回错误                             | 天然幂等                     | GET 天然幂等                                      |

## 7.2 其他操作对比  
下表对这些操作按功能领域进行了分类和梳理[^9]：  

| 功能领域  | 对应操作                                                                | 核心职责                             | 涉及的关键数据/组件                                          |
| ----- | ------------------------------------------------------------------- | -------------------------------- | --------------------------------------------------- |
| 对象操作  | RGWDeleteObj                                                        | 删除对象                             | 桶索引 (Bucket Index), GC (Garbage Collection) 队列[^10] |
|       | RGWCopyObj                                                          | 在桶内或跨桶复制对象                       | 源/目标对象的元数据和数据                                       |
| 分段上传  | RGWInitMultipart                                                    | 初始化分段上传，生成 UploadId              | RGWBucketInfo, 上传队列[^11]                            |
|       | RGWPutObj                                                           | 上传一个分片 (Part)                    | RGWPutObjProcessor_Multipart, 临时存储对象                |
|       | RGWCompleteMultipart                                                | 组合所有分片，完成上传                      | 对象清单 (manifest), 最终对象元数据                            |
|       | RGWAbortMultipart                                                   | 取消上传，清理已上传的分片                    | GC 队列                                               |
|       | RGWListMultipart                                                    | 列出一个正在进行中的上传任务的分片信息              | 上传任务的元数据对象                                          |
|       | RGWListBucketMultiparts                                             | 列出一个桶中所有正在进行中的上传任务               | 桶索引的 omap                                           |
| 桶管理   | RGWDeleteBucket                                                     | 删除一个空桶                           | RGWBucketEntryPoint, RGWBucketInfo                  |
|       | RGWStatBucket                                                       | 获取桶的元数据信息（如大小、对象数）               | RGWBucketInfo, 桶索引头                                 |
|       | RGWGetBucketLogging / RGWSetBucketLogging                           | 获取/设置桶的日志记录配置                    | 桶属性 (xattrs)                                        |
|       | RGWGetBucketVersioning / RGWSetBucketVersioning                     | 获取/设置桶的版本控制状态                    | RGWBucketInfo                                       |
| 策略与权限 | RGWGetACLs / RGWPutACLs                                             | 获取/设置桶或对象的访问控制列表 (ACL)           | 对象的 RGW_ATTR_ACL 属性                                 |
|       | RGWGetCORS / RGWPutCORS / RGWDeleteCORS                             | 获取/设置/删除桶的跨域资源共享 (CORS) 配置       | 桶的 RGW_ATTR_CORS 属性                                 |
|       | RGWGetRequestPayment / RGWSetRequestPayment                         | 获取/设置请求者付款功能                     | 桶属性                                                 |
| 元数据管理 | RGWPutMetadataAccount / RGWPutMetadataBucket / RGWPutMetadataObject | 更新账户、桶、对象的元数据                    | 账户/桶/对象的 xattrs                                     |
| 其他    | RGWOptionsCORS                                                      | 处理 CORS 预检请求 (Preflight Request) | CORS 配置                                             |
|       | RGWDeleteMultiObj                                                   | 批量删除对象                           | 桶索引, GC 队列                                          |
|       | RGWSetTempURL                                                       | 为 Swift 接口设置临时 URL 密钥            | 账户的元数据                                              |
|       | RGWStatAccount                                                      | 获取账户的元数据信息（如桶个数、用量）              | 用户元数据对象                                             |

# 8 [Rgw元数据管理](Rgw元数据管理.md)


[^1]: RGW Frontend 是 Ceph RGW 的 **HTTP 服务器组件**，负责：
    - 监听端口接收 S3/Swift API 请求
    - 解析 HTTP 请求
    - 调用 RGW 核心逻辑处理请求
    - 返回响应给客户端  

[^2]: Civetweb 和 Beast 比较

    | 对比项       | Civetweb            | Beast                  |
    |-----------|---------------------|------------------------|
    | 默认版本      | Luminous (12.x) 及之前 | Nautilus (14.x) 及之后    |
    | 并发模型      | 多线程（每连接一线程）         | 异步非阻塞（少量线程）            |
    | HTTP/2    | ❌ 不支持               | ✅ 支持                   |
    | WebSocket | ❌ 不支持               | ✅ 支持                   |
    | SSL/TLS   | OpenSSL（性能一般）       | OpenSSL/BoringSSL（优化好） |
    | 高并发性能     | 一般（~10k 连接）         | 优秀（>100k 连接）           |
    | 延迟        | 中等（线程切换开销）          | 低（事件驱动）                |
    | 内存占用      | 每连接独立栈（较高）          | 共享事件循环（较低）             |
    | CPU 利用率   | 线程竞争较多              | 更平滑                    |
    | 配置复杂度     | 简单                  | 中等                     |
    | 调试难度      | 简单                  | 中等                     |
    | 开发活跃度     | 低（仅维护）              | 高（主推）                  |

[^3]: https://cephdocs.readthedocs.io/en/latest/dev/radosgw/bucket_index/
[^4]: https://cephdocs.readthedocs.io/en/latest/dev/radosgw/bucket_index/
[^5]: https://www.programmersought.com/article/37449037100/
[^6]: https://deepwiki.com/ceph/ceph/4-client-interfaces
[^7]: https://deepwiki.com/ceph/ceph/4-client-interfaces
[^8]: 参阅：[Bucket(桶)简要介绍](../基本概念/Bucket(桶)简要介绍.md)
[^9]: https://deepwiki.com/ceph/ceph/4.1-rados-gateway-(rgw)
[^10]: https://cloud.tencent.cn/developer/article/1032873?from=15425
[^11]: https://blog.csdn.net/weixin_34270606/article/details/92578903  
