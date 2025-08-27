## Scheduler 测试指南（中文）

### 单元测试

```bash
make test WHAT=./cmd/kube-scheduler
go test ./pkg/scheduler/... -run TestNAME -v
```

### 集成测试

- 重点：调度框架插件之间的交互、缓存一致性、抢占逻辑；
- 可在 `test/integration/scheduler` 目录下运行：

```bash
go test ./test/integration/scheduler/... -run TestNAME -v -timeout=30m
```

### E2E 建议

- 资源约束/亲和性/拓扑/抢占/优先级场景；
- 大规模 Pod 创建压力与延迟曲线。


