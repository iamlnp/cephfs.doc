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

## 1.2 请求处理  
```c++
AsioFrontend::accept()  // rgw_aio_frontend.cc
    -> AsioFrontend::on_accept() // rgw_aio_frontend.cc
        -> void handle_connection() // rgw_aio_frontend.cc
            -> int process_request() // 核心处理函数：rgw_process.cc
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



# 3 杂记  
1. 在rgw::sal::Driver* newBaseFilter中生成rgw::sal::FilterDriver* driver对象，FilterDriver继承自class Driver
2. 调用链
```c++
static rgw::sal::Driver* get_storage() //Get a full driver by service name
    -> rgw::sal::Driver* DriverManager::init_storage_provider()
        -> rgw::sal::Driver* newBaseFilter()
```

3. rgw启动调用链
```c++
//以下是ceph15
int main() //radosgw.cc
    -> int radosgw_Main() //rgw_main.cc
        -> global_init()
        -> 初始化frontend，默认civetweb，根据rgw_frontends配置
            -> 创建RGWFrontendConfig *config对象
            -> config->init()
        -> store->getRados()->register_to_service_map("rgw", service_map_meta)
        
//ceph20
int main()
    -> rgw::AppMain main(&dp);
    -> driver = DriverManager::get_storage()
```
4. op返回值
```c++
// 返回各种底层操作OP
int process_request()
    -> RGWHandler_REST::get_op()
        -> case OP_PUT => RGWHandler_REST_Obj_S3::op_put()

```


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
