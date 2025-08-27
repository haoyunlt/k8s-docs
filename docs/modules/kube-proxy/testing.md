## Kube-Proxy 测试指南（中文）

### 单元测试

```bash
make test WHAT=./cmd/kube-proxy
go test ./pkg/proxy/... -run TestNAME -v
```

### 集成与 E2E

- 验证 ClusterIP/NodePort/LoadBalancer 的转发行为；
- 双栈/IPv6 环境下的规则正确性；
- 大规模 Service/Endpoints 性能与内存占用。


