# 类型关系图  
```plantuml
set namespaceSeparator none
' 组合关系
rgw_raw_obj *-- rgw_pool: 组合
RGWSI_SysObj *-- RGWSI_SysObj_Core: 组合
rgw_rados_ref *-- rgw_raw_obj: 组合

' 继承关系（箭头方向必须 → 指向父类）
RGWServiceInstance <|-- RGWSI_SysObj: 继承
RGWServiceInstance <|-- RGWSI_SysObj_Core: 继承

' 内部类组合
Obj *-- RGWSI_SysObj_Core: 组合
Obj *-- rgw_raw_obj: 组合
Pool *-- RGWSI_SysObj_Core: 组合
Pool *-- rgw_pool: 组合
WOp *-- Obj: 组合

class RGWServiceInstance {
}

class RGWSI_SysObj_Core {
  #librados::Rados* rados
  #RGWSI_Zone *zone_svc
  #virtual int raw_stat()
  #virtual int read()
  #virtual int remove()
  #virtual int write()
  #virtual int write_data() //调用librados接口写数据
  #virtual int get_attr()
  #virtual int set_attrs()
  #virtual int omap_get_all()
  #virtual int omap_get_vals()
  #virtual int omap_set()
  #virtual int omap_del()
  #int stat()
}

class RGWSI_SysObj {
  #librados::Rados* rados
  #RGWSI_SysObj_Core *core_svc
  +Obj get_obj()
}

class "RGWSI_SysObj::Obj" as Obj {
  -RGWSI_SysObj_Core *core_svc
  -rgw_raw_obj obj
}

class "RGWSI_SysObj::Pool" as Pool {
  -RGWSI_SysObj_Core *core_svc
  -rgw_pool pool
}

class "RGWSI_SysObj::Obj::WOp" as WOp {
  +Obj& source
  +RGWObjVersionTracker *objv_tracker
  +std::map<std::string, bufferlist> attrs
  +ceph::real_time mtime
  +ceph::real_time *pmtime
  +bool exclusive
}

class rgw_rados_ref{
    +librados::IoCtx ioctx
    +rgw_raw_obj obj
}

class rgw_pool {
  +std::string name
  +std::string ns
}

class rgw_raw_obj {
  +rgw_pool pool
  +std::string oid
  +std::string loc
  +void init()
  +void encode()
  +void decode()
  +void decode_from_rgw_obj()
}
@enduml
```

# 接口描述
1. `rgw_put_system_obj()`
函数原型：  
```c++
int rgw_put_system_obj(const DoutPrefixProvider *dpp, RGWSI_SysObj* svc_sysobj,
                       const rgw_pool& pool, const std::string& oid,
                       bufferlist& data, bool exclusive,
                       RGWObjVersionTracker *objv_tracker,
                       real_time set_mtime, optional_yield y,
                       const std::map<std::string, bufferlist> *pattrs = nullptr);
```


