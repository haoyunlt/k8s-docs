## API Server 代码剖析（中文）

### 入口与启动流程

- 入口：`cmd/kube-apiserver/apiserver.go`
  - `NewAPIServerCommand` → 解析 flags → `Run` → 构建 `GenericAPIServer`；
- 通用服务器栈：`k8s.io/apiserver/pkg/server`
  - 安装认证、鉴权、审计、handler 链；
  - 聚合与 CRD API 通过独立 apiserver 注册对接。

### 存储与对象

- 存储实现：`k8s.io/apiserver/pkg/storage/etcd3`
- 类型注册与编解码：`pkg/apis`、`k8s.io/apimachinery/pkg/runtime/serializer`
- 缓存：`k8s.io/client-go/tools/cache`（informer / reflector / store）。

### 请求处理管线（简化）

1) 认证（Authenticator）→ 2) 鉴权（Authorizer）→ 3) 准入（Admission）→ 4) Handler；
5) RESTStorage 调用（Create/Update/Patch/List/Watch）→ 6) etcd；
7) 通过 Watch 广播给订阅者（Controller/Scheduler/Kubelet 等）。

### 扩展点

- 准入 Webhook：`ValidatingAdmissionPolicy` 与外部 Webhook；
- 聚合层：`k8s.io/kube-aggregator` 注册外部 API；
- CRD：`staging/src/k8s.io/apiextensions-apiserver` 提供自定义类型与校验；
- 审计：多后端与策略；
- 特性门控：通过 `--feature-gates` 渐进启用。

### 关键中间件实现与位置

- 认证与鉴权 filter：
  - `staging/src/k8s.io/apiserver/pkg/server/config.go`
    - `WithAuthorization(handler, ...)`、`WithAuthentication(handler, ...)` 按顺序包裹；
  - 具体实现：`staging/src/k8s.io/apiserver/pkg/endpoints/filters/authorization.go:WithAuthorization`
    与 `.../filters/authentication_test.go` 展示调用方式；
- Admission：在建链后于 REST 层面拦截对象变更；

### RESTStorage 到 etcd 的落盘

- Generic store：`staging/src/k8s.io/apiserver/pkg/registry/generic/registry`；
- 存储驱动：`k8s.io/apiserver/pkg/storage/etcd3` 负责与 etcd 交互；
- 事务与并发：对象 `resourceVersion` 对应 etcd `revision`，依赖乐观并发更新。

### 性能与观测点

- 指标：`/metrics` 暴露 `apiserver_request_duration_seconds`、`watch_cache_*` 等；
- 审计：`--audit-*` 开启后可追踪请求链路；
- 过滤器时延追踪：`filterlatency.TrackStarted/TrackCompleted`。


