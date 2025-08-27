## 业务流程：Pod 全生命周期（中文）

涵盖创建、调度、启动、就绪、扩缩、终止等关键步骤以及对应组件、事件与指标。

### 流程图

```mermaid
sequenceDiagram
  autonumber
  participant U as User/kubectl
  participant APIS as API Server
  participant ETCD as etcd
  participant CM as Controller
  participant SCH as Scheduler
  participant KL as Kubelet
  participant CRI as Runtime

  U->>APIS: apply Pod/Deployment
  APIS->>ETCD: Persist
  CM-->>APIS: Watch desired state (Deployment/RS)
  SCH-->>APIS: Watch Pending Pods
  SCH->>APIS: Bind Pod(NodeName)
  APIS->>ETCD: Update Binding
  KL-->>APIS: Watch Pods for this Node
  KL->>CRI: RunPodSandbox/CreateContainer/StartContainer
  KL->>APIS: Update PodStatus
  Note over KL: Probes start; Volume/CNI ready
  U->>APIS: get/describe Pod (events)
```

### 关键检查点与指标

- 调度阶段：`schedule_attempts_total`、各插件延迟；
- 启动阶段：镜像拉取时间、容器创建时间、Pod 启动延迟；
- 就绪阶段：readiness/liveness/startup 成功率与失败原因；
- 终止阶段：grace period、强制终止比例、卷卸载耗时。


