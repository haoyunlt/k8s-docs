## Kube-Proxy 排查指南（中文）

### 快速体征

- Pod：`kubectl -n kube-system get pod -l k8s-app=kube-proxy -o wide`
- 日志：`kubectl -n kube-system logs <kube-proxy-pod>`
- 模式：`iptables` 或 `ipvs`，查看对应系统规则以验证同步是否正确。

### 常见问题

- Service 不通：
  - Endpoints/EndpointSlice 是否为空；
  - 选择器与 Pod 标签匹配；
  - 节点网络、路由与策略（NetworkPolicy）。
- 节点规则未同步：
  - kube-proxy 权限或 API 限流导致 watch 失败；
  - 大规模 Service 导致规则过多，建议使用 ipvs 并调优内核参数。

### 检查命令

- iptables：`iptables-save | grep -E "KUBE-SVC|KUBE-SEP|KUBE-CLUSTER-IP"`
- ipvs：`ipvsadm -Ln`
- 路由：`ip route`，`sysctl net.ipv4.ip_forward`

### 源码位置

- 入口：`cmd/kube-proxy/`
- 实现：`pkg/proxy/iptables`、`pkg/proxy/ipvs`、`pkg/proxy/endpoints`。


