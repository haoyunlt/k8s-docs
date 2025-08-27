## API Server 测试指南（中文）

### 本地运行与快速验证

- 本地起服（开发用）：`hack/local-up-cluster.sh`（依赖环境详见脚本提示）。
- 健康检查：`kubectl get --raw /readyz`、`/livez`。

### 单元测试

```bash
make test WHAT=./cmd/kube-apiserver
go test ./pkg/kubeapiserver/... -run TestNAME -v
```

### 集成测试

- 目录：`test/integration/...`
- 选择相关用例运行：

```bash
go test ./test/integration/apiserver/... -run TestNAME -v -timeout=20m
```

### 端到端（E2E）

- 建议使用 kind/minikube/kubeadm 启集群后运行：

```bash
go test ./test/e2e/... -run "\bAPIServer\b" -timeout=1h
```

### 性能与压力

- 指标：`/metrics` 抓取 apiserver_request_*、watch 缓存命中率；
- 压测：并发创建/更新对象，观察 P99 延迟与 etcd 写入耗时。


