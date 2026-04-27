# 1 OP类继承体系  
```plantuml
@startuml
set namespaceSeparator none
class RGWOp {
    +virtual int init_processing(optional_yield y)
    +virtual void init
    +virtual int verify_params()
    +virtual int verify_permission(optional_yield y) = 0
    +virtual void pre_exec()
    +virtual void execute(optional_yield y) = 0
    +virtual void complete()
    +void send_response()
}
class RGWDeleteBucket {
    #RGWObjVersionTracker objv_tracker
    +int verify_permission(optional_yield y) override
    +void pre_exec() override
    +void execute(optional_yield y) override
    +void send_response() override = 0;
}
class RGWDeleteBucket_ObjStore {

}
class RGWDeleteBucket_ObjStore_S3 {
    +void send_response() override
}

RGWOp <|-- RGWDeleteBucket
RGWDeleteBucket <|-- RGWDeleteBucket_ObjStore
RGWDeleteBucket_ObjStore <|-- RGWDeleteBucket_ObjStore_S3
@enduml
```

- `RGWOp` 是所有RGW操作的**抽象基类**  
- `RGWDeleteBucket` 是**直接继承自 `RGWOp` 的第一个子类**，它实现了**删除存储桶的通用核心逻辑**，实现了 `execute()` 函数，不关心具体的协议
- `RGWDeleteBucket_ObjStore` 继承了 `RGWDeleteBucket`，是**从通用操作到具体存储逻辑的桥梁**。它引入**对象存储语义**，将上层请求适配到RGW对象存储模型，负责解析对象存储相关的参数（如存储类、元数据等），并实现执行过程中与对象存储模型相关的逻辑。
- `RGWDeleteBucket_ObjStore_S3` 继承了 `RGWDeleteBucket_ObjStore`，是**整个继承链的最终环节**，专门负责处理**S3协议**的创建存储桶请求。它实现了S3协议特有的细节，如覆盖 `send_response()` 方法以构造符合S3规范的响应。

>  本文只涉及RadosBucket
# 2 整体删除流程梳理
`RGWOp` 是所有RGW操作的**抽象基类**，抽象了**pre_exec、execute、complete**三阶段，其中主体处理逻辑在 `execute`：`RGWDeleteBucket::execute(optional_yield y)`  。  

使用 `s3cmd rb s3://my-bucket` 可以删除空桶
```bash
[root@ceph-221 /]# s3cmd rb s3://my-bucket
Bucket 's3://my-bucket/' removed
```
### 2.1.1 阶段 1:pre_exec   

### 2.1.2 阶段2:核心流程-execute  
#### 2.1.2.1 简要流程图
```dot
digraph G {
    rankdir=TB;
    node [shape=box, style="filled", fillcolor=lightcyan];
    
    start[shape=oval,label="RGWDeleteBucket::execute( )"]
    end[shape=doublecircle, label="结束"]
    
    check1[shape=diamond, label="s->bucket_exists"]
    check2[shape=diamond, label="own_bucket==true"]
    check3[shape=diamond, label="再次判断own_bucket==true"]
    
    process1[label="获取own_bucket的值"]
    process2[label="同步ower统计: \n s->bucket->sync_owner_stats(...)"]
    process3[label="判断桶是否为空: \n s->bucket->check_empty(...)"]
    process4[label="转发请求到master: \n rgw_forward_request_to_master(...)"]
    process5[label="删除sse key \n rgw_remove_sse_s3_bucket_key(...)"]
    process6[label="删除bucket \n s->bucket->remove(...)"]
    
    start -> check1
    check1 -> end[label="否"]
    check1 -> process1[label="是"]
    process1 -> check2
    check2 -> process2[label="是"]
    process2 -> process3
    check2 -> process4[label="否"]
    process3 -> process4
    process4 -> check3
    check3 -> process5[label="是"]
    check3 -> process6[label="否"]
    process5 -> process6
    process6 -> end
}
```

#### 2.1.2.2 详细分步描述   
1. 判断own_bucket，如果等于true
    - s->bucket->sync_owner_stats(...)
    - s->bucket->check_empty(...)
2. 转发消息到master： `rgw_forward_request_to_master(...)`
3. 如果own_bucket == true：
    - `rgw_remove_sse_s3_bucket_key(...)`
4. 删除bucket：`RadosBucket::remove()`
    - 加载bucket：`RadosBucket::load_bucket(...)`
    - 如果delete_children == true，即删除非空桶，则循环
        - 获取桶中对象： `RadosBucket::list(...)`
        - 删除对象：`rgw_remove_object(...)`
    - 如果own_bucket == true：
        - 放弃分段上传：`RadosBucket::abort_multiparts(...)`
    - remove lifecycle config：`store->getRados()->get_lc()->remove_bucket_config(...)`
    - remove bucket-topic mapping
    - 如果own_bucket == true：
        - 同步owner统计：`store->ctl()->bucket->sync_owner_stats(...)`
    - 删除桶实例信息： `store->getRados()->delete_bucket(...)`:  `RGWRados::delete_bucket(...)`
    - 删除桶entrypoint： `store->ctl()->bucket->unlink_bucket(...)`:  `RGWBucketCtl::unlink_bucket(...)`
### 2.1.3 阶段3:complete  
调用基类的void complete() 函数，最终执行void send_response()，RGWDeleteBucket的send_response() 为纯虚函数 `void send_response() override = 0`，最终需要执行子类的send_response()，比如：`RGWDeleteBucket_ObjStore_S3::send_response(...)` 或者 `RGWDeleteBucket_ObjStore_SWIFT::send_response(...)  

### 疑问  
1. 如何保证原子性