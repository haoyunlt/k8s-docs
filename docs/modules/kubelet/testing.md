## Kubelet 测试指南（中文）

### 单元测试

```bash
make test WHAT=./cmd/kubelet
go test ./pkg/kubelet/... -run TestNAME -v
```

### 节点级集成测试

- 使用 kind 节点或真实节点：重点验证 Pod 生命周期、探针、卷挂载、镜像拉取；
- 日志与指标：关注 kubelet 日志中的错误堆栈与 `/metrics` 延迟指标。

### E2E 建议

- `test/e2e_node/...`：覆盖 cgroups、内存压力、镜像管理、CRI 行为等；
- 大量 Pod 创建/删除的稳定性与清理能力。


