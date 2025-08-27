## Scheduler 排查指南（中文）

### 快速体征

- Pod：`kubectl -n kube-system get pod -l component=kube-scheduler -o wide`
- 日志：`kubectl -n kube-system logs <ks-pod>`
- 指标：调度尝试次数、失败原因统计、各阶段耗时（Filter/Score/Bind）。

### 常见问题

- Pod 长期 Pending：
  - 资源不足/配额不足；
  - 亲和性/反亲和性冲突、拓扑域约束；
  - 污点/容忍度不匹配；
  - 调度器插件误配置或 webhook 延迟。
- 调度慢：
  - 缓存同步缓慢；
  - Score 插件耗时过长；
  - 多队列与优先级抢占策略导致抖动。

### 排查步骤

1) 描述 Pending Pod：`kubectl describe pod <name> -n <ns>` 查看 `Events`；
2) 启用调度日志/配置更高的 `--v` 级别；
3) 观察 `/metrics`：调度延迟直方图、失败原因 label；
4) 若使用自定义插件：本地复现场景并对可疑阶段做 CPU/内存剖析。

### 源码位置

- 入口：`cmd/kube-scheduler/`
- 框架：`pkg/scheduler/framework`（扩展点）；
- 默认插件：`pkg/scheduler/framework/plugins/...`；
- 缓存：`pkg/scheduler/internal/cache`。


