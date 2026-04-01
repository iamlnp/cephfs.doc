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

```c++
class RGWRESTMgr {
    virtual RGWHandler_REST* get_handler()
}

int process_request()
    -> RGWHandler_REST *handler = rest->get_handler()
    -> op = handler->get_op();
    -> op->verify_requester(); //认证
    -> rgw_process_authenticated() //授权
```


## 1.4 执行流程  




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
