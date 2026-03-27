# 1 客户端选择mds

`Client::choose_target_mds()`
- 随机模式：`Client::_get_random_up_mds()`, 从up的rank中随机选择一个，可直接通过`client_use_random_mds`控制
- hash模式：同时满足如下条件：
    1. is_hash：有如下几种场景之一
        1. req->inode，且req->path.depth()：
        2. req->dentry()，且req->dentry()->inode == nullptr
        3. req->inode，且in->snapid != CEPH_NOSNAP
    2. in是目录
    3. !in->fragmap.empty() || !in->frag_repmap.empty()
    
## 1.1 典型模式 - 以mkdir为例

## 1.2 session管理



## 1.3 proto结构

```c++
//client/MetaRequest.h
struct MetaRequest {
  
  
}
```


