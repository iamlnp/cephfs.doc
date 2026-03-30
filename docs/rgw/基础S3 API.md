# 1 桶相关  
## 1.1 创建桶  
方式 1：  radosgw-admin  
```bash
# 创建用户，执行后会返回 JSON 格式的输出，其中包含关键的访问凭证：
radosgw-admin user create --uid=<用户名> --display-name="显示名称"


```

# 2 对象相关