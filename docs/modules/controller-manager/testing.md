## Controller Manager 测试指南（中文）

### 单元测试

```bash
make test WHAT=./cmd/kube-controller-manager
go test ./pkg/controller/... -run TestNAME -v
```

### 集成与伪集成

- 使用 fake client、informer 与 workqueue 构造控制回路；
- 关注：幂等、重试、最终一致与资源版本冲突处理。

### E2E 场景建议

- Deployment/StatefulSet/Job 生命周期；
- PodDisruptionBudget 与扩缩容；
- 级联删除与垃圾回收（GC）。


