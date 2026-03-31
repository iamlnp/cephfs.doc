# 启动流程  
1. 创建存储池
```bash
*.tos.log
*.tos.meta
*.tos.control
*.tos.non-ec
*.tos.otp
.tos.root
```
2. 创建网关
创建时需要指定 http 和 https 的端口号
```bash
[root@node2 ~]# ceph -s
  cluster:
    id:     328c14a2-21e4-11f1-92e2-6cfe543d5bc8
    health: HEALTH_OK

  services:
    rgw: 2 daemons active (tosa, tosb)

  task status:

  data:
    pools:   11 pools, 10689 pgs
    objects: 26.72M objects, 99 GiB
    usage:   2.2 TiB used, 308 TiB / 311 TiB avail
    pgs:     10689 active+clean

  io:
    client:   415 MiB/s rd, 818 KiB/s wr, 11.16k op/s rd, 54 op/s wr
    
 # 网关，目前创建了2个
 [root@node2 ~]# onnode all ps -axu | grep /usr/bin/radosgw

>> NODE: 192.168.124.1 
ceph     1686311  1.3  0.0 5387904 111656 ?      Ssl  15:21   0:01 /usr/bin/radosgw -f --cluster ceph --name client.rgw.tosa --setuser ceph --setgroup ceph
>> NODE: 192.168.124.2 
ceph     3044787  1.4  0.0 5387908 110868 ?      Ssl  15:22   0:00 /usr/bin/radosgw -f --cluster ceph --name client.rgw.tosb --setuser ceph --setgroup ceph
>> NODE: 192.168.124.3 

# 查询网关服务状态
[root@node2 ~]# systemctl status ceph-radosgw@rgw.tosb
● ceph-radosgw@rgw.tosb.service - Ceph rados gateway
   Loaded: loaded (/usr/lib/systemd/system/ceph-radosgw@.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2026-03-27 15:22:27 CST; 4min 10s ago
 Main PID: 3044787 (radosgw)
    Tasks: 609
   Memory: 85.3M
   CGroup: /system.slice/system-ceph\x2dradosgw.slice/ceph-radosgw@rgw.tosb.service
           └─3044787 /usr/bin/radosgw -f --cluster ceph --name client.rgw.tosb --setuser ceph --setgroup ceph

Mar 27 15:22:27 node2 systemd[1]: Started Ceph rados gateway.
Mar 27 15:22:27 node2 radosgw[3044787]: 2026-03-27T15:22:27.936+0800 7fdc9f5ada80 -1 ssl_private_key was not found: rgw/cert/magnascale_realm/magnascale.key
    
```
3. 创建路由

4. 创建存储策略

5. 创建对象用户
