## CoreDNS 排查指南（中文）

### 快速体征

- Pod：`kubectl -n kube-system get pod -l k8s-app=kube-dns -o wide`
- 日志：`kubectl -n kube-system logs deploy/coredns`
- 配置：`kubectl -n kube-system get cm coredns -o yaml`

### 常见问题

- 域名解析失败：
  - `kube-dns` Service/Endpoints 是否存在且正常；
  - Corefile 配置语法或上游 `forward` 配置错误；
  - NetworkPolicy 导致 Pod 到 CoreDNS 的 53/UDP 受限。
- 延迟高：
  - 上游 DNS 超时；
  - Pod/Node 网络抖动；
  - 负载过高，需横向扩容或启用缓存优化。

### 验证命令

- Pod 内：`nslookup kubernetes.default` 或 `dig kubernetes.default.svc.cluster.local @<coredns-ip>`
- 观察事件：`kubectl -n kube-system describe deploy coredns`


