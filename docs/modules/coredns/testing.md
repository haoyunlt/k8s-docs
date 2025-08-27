## CoreDNS 测试指南（中文）

### 功能验证

- Service 与 Pod 域名解析：A/AAAA/SRV 记录；
- 自定义域与上游转发（`forward`/`proxy` 插件）。

### 压力测试

- 并发 `dig` 查询延迟曲线；
- 缓存命中率与 QPS 上限。


