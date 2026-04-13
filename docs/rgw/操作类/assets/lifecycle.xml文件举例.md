```xml
<?xml version="1.0" encoding="UTF-8"?>
<LifecycleConfiguration>
    <!-- 规则 1：针对特定前缀的过期删除 -->
    <Rule>
        <ID>delete-temp-logs-after-7-days</ID>  <!-- 规则唯一标识符 -->
        <Prefix>logs/</Prefix>                  <!-- 仅对 logs/ 文件夹下的对象生效 -->
        <Status>Enabled</Status>                <!-- 启用规则 -->
        <Expiration>
            <Days>7</Days>                      <!-- 7天后删除 -->
        </Expiration>
    </Rule>

    <!-- 规则 2：针对整个桶，将30天前的对象转存到冷存储 -->
    <Rule>
        <ID>transition-to-cold-after-30-days</ID>
        <Prefix></Prefix>                       <!-- 空前缀表示匹配桶内所有对象 -->
        <Status>Enabled</Status>
        <Transition>
            <Days>30</Days>                     <!-- 30天后 -->
            <StorageClass>COLD</StorageClass>   <!-- 转存到冷存储层（具体名称依服务商而定） -->
        </Transition>
    </Rule>

    <!-- 规则 3：管理非当前版本的对象（需开启版本控制） -->
    <Rule>
        <ID>cleanup-old-versions</ID>
        <Prefix></Prefix>
        <Status>Enabled</Status>
        <NoncurrentVersionExpiration>
            <NoncurrentDays>30</NoncurrentDays> <!-- 对象变成“非当前版本”30天后，将其永久删除 -->
        </NoncurrentVersionExpiration>
    </Rule>

    <!-- 规则 4：自动中止未完成的分段上传 -->
    <Rule>
        <ID>abort-incomplete-multipart-upload</ID>
        <Prefix></Prefix>
        <Status>Enabled</Status>
        <AbortIncompleteMultipartUpload>
            <DaysAfterInitiation>7</DaysAfterInitiation> <!-- 上传开始7天后未完成则自动中止 -->
        </AbortIncompleteMultipartUpload>
    </Rule>
</LifecycleConfiguration>
```