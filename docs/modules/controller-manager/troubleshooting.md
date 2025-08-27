## Controller Manager 排查指南（中文）

### 快速体征

- Pod：`kubectl -n kube-system get pod -l component=kube-controller-manager -o wide`
- 日志：`kubectl -n kube-system logs <kcm-pod>`
- 指标：`/metrics` 关注 workqueue 深度与 requeue 次数。

### 常见问题

- 大量重试/reconcile 变慢：检查 informer 是否卡顿、API 限流、队列拥塞；
- 准入/配额冲突导致创建失败：结合 apiserver 审计日志定位；
- 云控制器依赖（如服务/负载均衡）异常：确认 cloud-provider 集成正确。

### 定位要点

- 关键控制器：Deployment/ReplicaSet/Job/StatefulSet/ServiceAccount/Node/Lifecycle；
- 关注 workqueue：`k8s.io/client-go/util/workqueue` 指标与日志；
- 幂等保障：确保外部系统失败时不会产生漂移或泄漏资源。

### 源码位置

- 入口：`cmd/kube-controller-manager/`
- 控制器实现：`pkg/controller/...`


