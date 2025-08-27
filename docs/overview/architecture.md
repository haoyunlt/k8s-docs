## Kubernetes 架构总览（中文）

本文面向阅读 Kubernetes 源码与运维集群的同学，以中文方式描述核心组件、数据流、扩展点与观测面。

### 控制平面（Control Plane）

- API Server（`cmd/kube-apiserver`，`pkg/kubeapiserver`）：
  - 统一的 REST 接口，所有组件通过它读写集群状态；
  - 认证/鉴权/准入（Authentication/Authorization/Admission）；
  - 对象存储在 etcd，提供乐观并发与版本控制；
  - 聚合层与自定义资源（CRD）扩展。
- Controller Manager（`cmd/kube-controller-manager`，`pkg/controller`）：
  - 期望态驱动：监听对象变化，持续调谐（reconcile）；
  - 常见控制器：Deployment/ReplicaSet/Job/Node/Lifecycle/ServiceAccount 等；
  - 通过 informer 与 workqueue 实现缓存、去抖与幂等控制回路。
- Scheduler（`cmd/kube-scheduler`，`pkg/scheduler`）：
  - 为未绑定的 Pod 选择符合约束与最优打分的 Node；
  - 调度框架（Scheduling Framework）提供丰富的扩展点（Filter/Score/Bind/Reserve 等）。
- etcd（外部组件）：
  - 强一致 KV 存储；Kubernetes 所有对象的持久化后端。

### Node 节点组件

- Kubelet（`cmd/kubelet`，`pkg/kubelet`）：
  - 节点级控制回路，拉取 Pod 期望态并驱动容器运行时（CRI）；
  - 上报 Node/Pod 状态、执行探针、挂载存储（CSI）与网络（CNI）。
- Kube-Proxy（`cmd/kube-proxy`，`pkg/proxy`）：
  - 基于 Service/Endpoints(或 EndpointSlice) 实现四层流量转发；
  - 模式：iptables、ipvs、userspace（历史）。
- 容器运行时（CRI）：containerd、CRI-O 等；
- 网络（CNI）：Flannel/Calico/Cilium 等；
- 存储（CSI）：RBD/EBS/NFS 等。

### 典型对象与数据流

1) 用户或控制器对 API Server 发起变更（kubectl/Operator/Controller）。
2) API Server 认证鉴权与准入后写入 etcd，并通过 Watch 通知订阅者。
3) Controller 根据对象期望态与实际态差异进行调谐（如创建/更新 Pod）。
4) Scheduler 为 Pending Pod 选出目标 Node 并绑定。
5) Kubelet 在目标 Node 上创建 Pod（通过 CRI），并周期上报状态。
6) Kube-Proxy 同步 Service/Endpoints，更新数据面（iptables/ipvs）。
7) CoreDNS 作为集群 DNS，解析 Service/Pod 域名。 

### 扩展机制

- CRD + Controller：按 CR 设计业务资源与控制回路；
- 准入扩展：Webhook 验证/变更对象；
- 调度框架：实现自定义调度插件；
- CNI/CSI/CRI：网络、存储、运行时解耦；
- 聚合层与 API 扩展：注册额外的 API 服务。

### 观测与安全

- 可观测性：
  - 指标：`/metrics` (Prometheus)；
  - 日志：组件日志、审计日志（apiserver）；
  - 事件：`kubectl get events`；
  - 追踪：OpenTelemetry（部分组件/自研接入）。
- 安全：
  - 访问控制：RBAC；
  - 安全基线：PodSecurity 标准（替代 PSP）、安全上下文；
  - 机密管理：Secret（建议集成外部 KMS）；
  - 网络安全：NetworkPolicy，数据面依据 CNI 实现。

### 源码导览（按本仓库路径）

- API Server：`cmd/kube-apiserver/`，`pkg/kubeapiserver/`
- Controller：`cmd/kube-controller-manager/`，`pkg/controller/`
- Scheduler：`cmd/kube-scheduler/`，`pkg/scheduler/`
- Kubelet：`cmd/kubelet/`，`pkg/kubelet/`
- Kube-Proxy：`cmd/kube-proxy/`，`pkg/proxy/`
- Control Plane 其他：`pkg/controlplane/`，`pkg/registry/`，`pkg/apis/`

附：英文历史文档位于 `docs/architecture.md`，本文为中文详解版本。


