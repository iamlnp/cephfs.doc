---
写作年份:
  - "2026"
imageNameKey: 预签名URL
tags:
  - rgw
  - 对象存储
---
RGW（RADOS Gateway）的预签名URL功能，是一种安全、可控的临时授权机制，允许持有有效凭证的用户为特定对象（Object）生成一个带有有效期和操作权限的URL，并将此URL分享给第三方，供其在有效期内进行特定操作（如下载、上传）[^1]

# 1 核心原理  
预签名URL的核心在于“签名”。生成时，服务端（或客户端SDK）会使用请求者的**Secret Key**对请求的核心要素进行加密签名[^2]。这个签名被作为查询参数附加到URL中，当RGW接收到请求时，会重新计算签名并与URL中的比对，以此验证请求的合法性和完整性。

生成的预签名URL会携带特定的查询参数，主要包括：
- **X-Amz-Algorithm**：签名算法，通常为 `AWS4-HMAC-SHA256`。
- **X-Amz-Credential**：包含访问密钥ID、日期和区域信息。
- **X-Amz-Date**：签名生成的日期和时间。
- **X-Amz-Expires**：URL的有效期，以秒为单位。
- **X-Amz-Signature**：根据上述信息计算出的签名，用于验证[^3]。
- **X-Amz-SignedHeaders**：签名计算中包含的HTTP头，通常是 `host`。
RGW目前主要支持**AWS签名v4（SigV4）** 标准，该标准提供了比旧版v2更高的安全性[^4]。部分旧工具（如 `s3cmd`）可能仍使用v2签名，建议升级或更换

```bash
用户（持有 AWS Key）                S3 服务
      │                                 │
      ├─ 1. 请求 signurl ──────────────→│（本地计算，不联网）
      │                                 │
      ├─ 2. 生成签名 URL ───────────────│
      │   (基于 Secret Key 计算签名)     │
      │                                 │
      │   【将 URL 分享给其他人】         │
      │                                 │
第三方 ──→ 3. 访问 URL ─────────────────→│
      │                                 │
      │   ← 4. 验证签名和过期时间 ────────│
      │                                 │
      │   ← 5. 返回文件内容 ──────────────│
```

与其他命令的对比：  

| 命令                        | 用途         | 是否需要凭证访问      |
|---------------------------|------------|---------------|
| s3cmd get                 | 下载文件（需要凭证） | ✅ 需要          |
| s3cmd signurl             | 生成临时下载链接   | ❌ 不需要（但生成时需要） |
| s3cmd setacl --acl-public | 永久公开文件     | ❌ 不需要         |
| s3cmd ls                  | 列出文件（需要凭证） | ✅ 需要          |

# 2 使用方式  
可以通过多种方式生成预签名URL：
- 使用命令行工具：这是最快捷的方式。
    - AWS CLI: `aws s3 presign s3://bucket-name/object-key --expires-in 3600`。
    - s3cmd: `s3cmd signurl s3://BUCKET/OBJECT <expiry_epoch|+expiry_offset>` 。

- 使用编程语言SDK：这提供了最高的灵活性。
    - Python (Boto3)：使用 `generate_presigned_url` 方法[^5]。支持 `put_object`（上传）、`get_object`（下载）等多种操作。
    - Go (aws-sdk-go-v2)：使用 `s3.NewPresignClient` 并调用 `PresignPutObject` 等方法[^6]。
    - Java (AWS SDK for Java): 使用 `AmazonS3.generatePresignedUrl` 方法

# 3 相关功能  
**Swift Temp URL**: RGW也支持OpenStack Swift的Temp URL功能，原理类似，但管理模型不同（密钥与用户而非租户关联）  。  


[^1]: https://www.javaxxw.com/recommend/d7d8381/ceph%e7%9b%b8%e5%85%b3-s-%e9%a2%84%e7%ad%be%e5%90%8durl-presign
[^2]: https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/HVLFGODPWVICWGQGD3IEEBDPK7RXZ7CA/?sort=date
[^3]: https://tracker.ceph.com/issues/62033#change-242431
[^4]: https://www.javaxxw.com/recommend/d7d8381/ceph%e7%9b%b8%e5%85%b3-s-%e9%a2%84%e7%ad%be%e5%90%8durl-presign
[^5]: https://tracker.ceph.com/attachments/6779/cors-preflight-ceph.py
[^6]: https://blog.csdn.net/weixin_59631089/article/details/154697943
