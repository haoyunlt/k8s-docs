## API Server 排查指南（中文）

面向：集群管理员与开发同学。聚焦启动失败、请求超时、鉴权异常、etcd 相关问题。

### 快速体征

- Pod 状态：`kubectl -n kube-system get pod -l component=kube-apiserver -o wide`
- 就绪检查：`kubectl get --raw /readyz?verbose | cat`
- 存活检查：`kubectl get --raw /livez?verbose | cat`
- 日志：`kubectl -n kube-system logs <kube-apiserver-pod> --previous`（若重启）
- 审计日志（若启用）：检查 `--audit-log-path` 指定文件

### 核心检查清单

1) 证书与时钟：
   - 报错含 `x509: certificate has expired or is not yet valid` → 校准 NTP、轮换证书；
   - 确认 `--client-ca-file`、`--tls-cert-file`、`--tls-private-key-file`、`--etcd-cafile` 有效。
2) etcd 可用性：
   - 连接参数：`--etcd-servers`；
   - 端到端：`etcdctl --endpoints=<...> endpoint health`，关注延迟与告警；
   - 慢查询：apiserver 日志中的 `etcd` latency 指标。
3) 鉴权与准入：
   - 认证：Webhook/OIDC 配置是否匹配；
   - 鉴权：RBAC 规则是否覆盖；
   - 准入：变更/校验 webhook 是否超时（`timeout`）、证书是否匹配。
4) 资源压力：
   - QPS/并发：`--max-mutating-requests-inflight`、`--max-requests-inflight`；
   - 内存 OOM：容器/节点是否被驱逐；
   - 指标：抓取 `/metrics` 中的 apiserver_request_total、apiserver_request_duration_seconds_*。

### 常见问题与处置

- 429/限流：调大 inflight、优化控制器重试、在客户端引入指数退避；
- 资源版本冲突：客户端使用乐观并发，重试带 `resourceVersion`；
- CRD 不可用：确认 apiextensions-apiserver 正常、CRD 版本与 schema 校验；
- Webhook 超时：缩短变更对象体积、提升副本数、网络连通性检查；
- etcd 写入慢：磁盘/网络/压缩延迟，必要时扩容与分层存储。

### 深入定位

- 清单路径：`/etc/kubernetes/manifests/kube-apiserver.yaml` 或对应管理平面；
- 关键参数：认证/鉴权/准入、etcd、审计、特性门控（FeatureGates）；
- 请求追踪：启用审计日志与请求 ID，结合反向代理或 OTel 追踪。

### 参考源码

- 入口：`cmd/kube-apiserver/apiserver.go`（`NewAPIServerCommand`）
- 通用服务器：`k8s.io/apiserver/pkg/server`
- 存储：`k8s.io/apiserver/pkg/storage/etcd3`
- 聚合层：`k8s.io/kube-aggregator`
- CRD 服务：`staging/src/k8s.io/apiextensions-apiserver`


