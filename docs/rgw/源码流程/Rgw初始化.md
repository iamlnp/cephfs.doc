---
年份:
  - "2026"
月份:
  - 4月
imageNameKey: Rgw初始化
tags:
  - ceph20
---
# 1 main函数  
```c++
int main()
    -> rgw::AppMain main(&dp);
    -> main.init_storage()
        -> env.driver = DriverManager::get_storage()
            -> rgw::sal::Driver* driver = init_storage_provider
                -> if (cfg.store_name.compare("rados") == 0) -> new rgw::sal::RadosStore(), new RGWRados();
                -> else if (cfg.store_name.compare("d3n") == 0) -> new rgw::sal::RadosStore(); new D3nRGWDataCache<RGWRados>
                -> else if ...
    -> main.cond_init_apis();
        -> 从rgw_enable_apis配置项中读取支持的api, 默认值： s3, s3website, swift, swift_auth, admin, sts, iam, notifications
        -> register_default_mgr注册默认mgr
```
# 2 资源管理  
## 2.1 资源注册  
`rgw::AppMain::cond_init_apis()`  ,检查 `apis_map`，`apis_map` 根据 `g_conf()->rgw_enable_apis` 获取
1. 默认mgr ： `RGWRESTMgr_S3` 或者 `RGWRESTMgr_SWIFT`
2. swift： 
```c++
RGWRESTMgr_SWIFT* const swift_resource = new RGWRESTMgr_SWIFT;
    - "crossdomain.xml" -> RGWRESTMgr_SWIFT_CrossDomain
    - "healthcheck" -> RGWRESTMgr_SWIFT_HealthCheck
    - "info" -> RGWRESTMgr_SWIFT_Info
    - g_conf()->rgw_swift_url_prefix -> swift_resource
```
3. swift_auth：  `rgw_swift_auth_entry -> RGWRESTMgr_SWIFT_Auth`
4. admin: 
```c++
RGWRESTMgr_Admin * admin_resource = new RGWRESTMgr_Admin;
    -  "info" -> RGWRESTMgr_Info
    - "usage" -> RGWRESTMgr_Usage
    - "account" -> RGWRESTMgr_Account
    - g_conf()->rgw_admin_entry -> admin_resource
```
5. zero -> `RESTMgr_Zero `