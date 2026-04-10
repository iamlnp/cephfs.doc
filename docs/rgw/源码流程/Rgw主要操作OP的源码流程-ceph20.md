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
## 1.1 前端**Frontend**[^1]入口  
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

## 1.2 请求处理  
```c++
AsioFrontend::accept()  // rgw_aio_frontend.cc
    -> AsioFrontend::on_accept() // rgw_aio_frontend.cc
        -> void handle_connection() // rgw_aio_frontend.cc
            -> int process_request() // 核心处理函数：rgw_process.cc, 日志： starting new request req 
```
## 1.3 路由分发  

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

int process_request()
    -> RGWHandler_REST *handler = rest->get_handler() //根据请求方法（PUT）、URL（bucket/object）匹配 S3 处理器
        -> preprocess()
    -> op = handler->get_op(); // 获取实例化操作类，比如RGWPutObj/RGWGetObj
    -> rgw::lua::request::execute()
    -> op->verify_requester(); //认证
    -> rgw_process_authenticated() //认证完成后的op执行流程
    -> （RGWRestfulIO)client_io->complete_request() //完成请求
```


## 1.4 执行流程  
```c++
rgw_process_authenticated() //认证完成后的op执行流程
    -> handler->init_permissions()
    -> op->init_processing()
    -> op->verify_op_mask()
    -> op->verify_permission(y);
    -> op->verify_params()
    -> op->pre_exec() //step1: 预处理，前置校验与准备
    -> op->execute()  //step2: 数据读写核心流程
    -> op->complete() //step3: 元数据更新与收尾
```

# 2 Put 流程  
## 2.1 RGWPutObj - REST 前端操作（S3/Swift 入口）
```c++
class RGWPutObj : public RGWOp {

public:
    void pre_exec() override;
    void execute(optional_yield y) override;
}
```

### 2.1.1 核心执行阶段  
PUT 上传分**pre_exec、execute、complete**三阶段：  
#### 2.1.1.1 阶段 1：pre_exec  
```c++
// s是基类RGWOP中字段：req_state *s
void RGWPutObj::pre_exec()
{
    rgw_bucket_object_pre_exec(s);
}
```

#### 2.1.1.2 阶段2：execute  
```c++
//数据读写核心流程
void RGWPutObj::execute(optional_yield y)
{
    /* 主要处理如下：
    1. get_system_versioning_params()
    2. s->bucket->check_quota() // 配额检查
    3. s->object->swift_versioning_copy()
    4. res->publish_reserve()
    5. if (multipart)
            5.1 upload = s->bucket->get_multipart_upload()
            5.2 std::unique_ptr<rgw::sal::Writer> processor = upload->get_writer
        else if (append)
            5.3 std::unique_ptr<rgw::sal::Writer> processor = driver->get_append_writer
        else
            5.4 std::unique_ptr<rgw::sal::Writer> processor = driver->get_atomic_writer
    6. processor->prepare
    // rgw::sal::DataProcessor *filter = processor.get();
    7. 循环处理data
        7.1 RGWPutObj::get_data()
        7.2 filter->process()
    8. 处理压缩
    9. processor->complete()
    10. res->publish_commit
    */
}
```
##### 2.1.1.2.1 RGWPutObj::get_data  

1. read_op->prepare()
2. rgw_compression_info_from_attrset()
3. obj->range_to_ofs()
4. filter->fixup_range()
5. read_op->iterate()
6. filter->flush()



#### 2.1.1.3 阶段3：complete
## 2.2 RGWPutObj_ObjStore - 底层存储驱动（直接写 RADOS） 

 创建位置：

| 维度   | RGWPutObj                                                                | RGWPutObj_ObjStore                                                                          |
|------|--------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| 归属模块 | REST 前端（S3/Swift 协议处理）                                                   | RADOS 存储驱动（后端存储）                                                                            |
| 核心职责 | 1. 解析 HTTP 请求 / 头2. 权限 / 条件 / 配额校验3. 选择原子 / 分段处理器4. 读数据、限流、校验5. 调用底层存储接口 | 1. 纯数据写入（条带化、AIO）2. 管理 obj_manifest3. 写 RADOS 数据 / Shadow 对象4. 计算 ETag/MD5/CRC5. 原子提交、更新元数据 |
| 生命周期 | 每个 HTTP PUT 请求创建一个                                                       | 被 RGWPutObj 创建并调用                                                                           |
| 关键方法 | pre_exec()<br>execute()<br>complete()                                    | prepare()<br>process_data()<br>finish_data()<br>complete()                                  |
| 调用关系 | 上层入口 → 调用下层                                                              | 下层实现 → 被上层调用                                                                                |
| 异常处理 | HTTP 状态码（403/412/400/500）                                                | 内部错误码，抛给上层处理                                                                                |

# 3 List buckets  - list bucket中的内容
RGWListBuckets_ObjStore_S3  

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

### 3.1.1 核心执行阶段  
#### 3.1.1.1 RGWListBucket::execute  
 


# 4 创建Bucket流程  
用 S3 命令（s3cmd /awscli/boto3 等）创建的是 RadosBucket（标准对象存储桶）[^3]

```c++
class RGWCreateBucket_ObjStore_S3 : public RGWCreateBucket_ObjStore {
public:
    RGWCreateBucket_ObjStore_S3() {}
    ~RGWCreateBucket_ObjStore_S3() override {}

    int get_params(optional_yield y) override;
    void send_response() override;
};


class RGWCreateBucket_ObjStore : public RGWCreateBucket {
public:
  RGWCreateBucket_ObjStore() {}
  ~RGWCreateBucket_ObjStore() override {}

  virtual std::string canonical_name() const override { return fmt::format("REST.{}.BUCKET", s->info.method); }
};

class RGWCreateBucket : public RGWOp {
    int verify_permission(optional_yield y) override;
    void pre_exec() override;
    void execute(optional_yield y) override;
    void init(rgw::sal::Driver* driver, req_state *s, RGWHandler *h) override {
        RGWOp::init(driver, s, h);
        relaxed_region_enforcement =
        s->cct->_conf.get_val<bool>("rgw_relaxed_region_enforcement");
    }
    virtual int get_params(optional_yield y) { return 0; }
    void send_response() override = 0;
}

// rgw_sal_rados.h
class RadosBucket : public StoreBucket {

}
```

## 4.1 创建整体简要流程  
### 4.1.1 API 入口与权限校验
RGWCreateBucket::verify_permission()  
1. 用户权限检查： verify_user_permission()
2. 检查用户的桶数量上限： check_owner_max_buckets()

### 4.1.2 核心流程  
RGWCreateBucket::execute(optional_yield y)  
1. **选择placement**  
select_bucket_placement() //select and validate the placement target

2. **检查是否已经存在待创建的桶**    
    - 加载处理：driver->load_bucket(), 即RadosBucket::load_bucket()  
        - if bucket_id为空，则 `store->ctl()->bucket->read_bucket_info`
            - read_bucket_entrypoint_info()
            - read_bucket_instance_info()
                - do_read_bucket_instance_info
        - 否则：`store->ctl()->bucket->read_bucket_instance_info`
    - 如果存在：获取已存在桶的部分属性作为待创建桶的参数
        - swift_ver_location
        - placement_rule

3. **组装创建桶需要的参数**    
    - zonegroup_id
    - zone_placement
    - swift_ver_location
    - placement_rule
    - owner
    - attrs(不局限于如下两个)
        - RGW_ATTR_ACL
        - RGW_ATTR_CORS
    - quota

4. **检查是否是master zone，如果不是，则优先转发给master创建**
    - rgw_forward_request_to_master()
    - 如下从master获取
        - marker
        - bucket_id
        - zonegroup_id
        - obj_lock_enabled
        - quota
        - creation_time
5. **创建桶并且持久化    
    - 创建bucket:  s->bucket->create(this, createparams, y)
        - RadosBucket::create()  
            - store->get_rados()->create_bucket():  RGWRados::create_bucket(...)
                - 生成版本号： generate_new_write_ver()
                - 创建桶id： create_bucket_id()
                - RGWRados::put_linked_bucket_info()
                    - 元数据持久化 BucketInfo结构：RGWRados::put_bucket_instance_info
                    - ctl.bucket->store_bucket_entrypoint_info()
            - RadosBucket::link()
                - store->ctl()->bucket->link_bucket() - RGWBucketCtl::link_bucket() 
                    - 持久化bucket entrypoint结构： - svc.bucket->store_bucket_entrypoint_info()
            - store->ctl()->bucket->read_bucket_entrypoint_info()

扩展属性（xattrs/attrs）处理 ？
原子提交与返回  ？

### 4.1.3 元数据  

#### 4.1.3.1 BucketEntryPoint  
Bucket 的入口信息（BucketEntryPoint），也就是桶的 “身份证”  

读取函数：   
RGWSI_Bucket_SObj::read_bucket_entrypoint_info(),   
```c++
int RGWSI_Bucket_SObj::read_bucket_entrypoint_info(
    const string& key,                          // 桶的key：root/bucket.{bucket_name}
    RGWBucketEntryPoint *entry_point,           // 输出：解码后的桶入口信息
    RGWObjVersionTracker *objv_tracker,         // 对象版本追踪
    real_time *pmtime,                          // 输出：修改时间
    map<string, bufferlist> *pattrs,            // 输出：扩展属性 xattrs
    optional_yield y,                           // 协程yield
    const DoutPrefixProvider *dpp,              // 日志
    rgw_cache_entry_info *cache_info,           // 缓存
    boost::optional<obj_version> refresh_version)
```
其中RGW 存储桶入口元数据的 RADOS 池， 默认 default.rgw.meta，const rgw_pool& pool = svc.zone->get_zone_params().domain_root

可以使用 `radosgw-admin zone get --default` 查询  :  
```bash
{
    "id": "0f2f1491-1607-439d-a1b0-6e88fe76cc6d",
    "name": "default",
    "domain_root": "default.rgw.meta:root",
    "control_pool": "default.rgw.control",
    "dedup_pool": "default.rgw.dedup",
    "gc_pool": "default.rgw.log:gc",
    ...
}
```

读取omap： RGWSI_SysObj_Core::read()  
- librados::ObjectReadOperation op 
    - op.read(ofs, len, bl, nullptr)
    - op.getxattrs()
- RGWSI_SysObj_Core::get_rados_obj
- RGWSI_SysObj_Core::rgw_rados_operate

#### 4.1.3.2 Bucket信息-RGWBucketInfo  

#### 4.1.3.3 核心元数据  

#### 4.1.3.4 桶索引对象  

#### 4.1.3.5 名称映射 

### 4.1.4 扩展属性  

两类属性：系统 vs 用户    
- **系统属性（System attrs）**：RGW 内部使用
    - `user`、`owner`、`owner_display_name`
    - `acl`、`policy`、`creation_time`
    - `quota`、`max_size`、`max_objects`
    - `placement_id`、`zonegroup`、`region`
    - `versioning`、`encryption`、`lock_mode`
- **用户扩展属性（User xattrs）**
    - S3：`x-amz-meta-key: value`
    - Swift：`X-Object-Meta-*`
    - Bucket tags（`tag1=val1&tag2=val2`
存储位置（关键）：  



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

​   

[^3]: 参阅：[对象桶简要介绍](../基本概念/对象桶简要介绍.md)
