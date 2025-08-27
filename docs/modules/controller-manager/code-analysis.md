## Controller Manager 代码剖析（中文）

### 入口与框架

- 入口：`cmd/kube-controller-manager/app`
- 通过 `NewControllerInitializers` 注册控制器；按特性门控与配置选择开启。

### 控制器通用模式

- informer 监听 → workqueue 入队 → 幂等 reconcile → 失败重试；
- 关键库：`k8s.io/client-go/tools/cache` 与 `util/workqueue`；
- 资源版本与冲突：使用乐观并发与指数退避重试。

### 典型控制器路径

- Deployment：`pkg/controller/deployment`（扩缩容与滚动更新）；
- StatefulSet：`pkg/controller/statefulset`（有序编号与稳定身份）；
- Job/CronJob：`pkg/controller/job`、`pkg/controller/cronjob`；
- GarbageCollector：`pkg/controller/garbagecollector`（级联删除与 ownerRef）。

### 关键控制器调用链（示例：Deployment）

1) 事件进入：informer 监听 `ReplicaSet/Pod/Deployment` 变化，入队 key；
2) 工作者出队执行 `syncDeployment`：计算新旧 RS、期望副本数、滚动策略；
3) 与 RS 控制器协作：`pkg/controller/replicaset` 负责具体 Pod 整数控制；
4) 终止副本字段：`features.DeploymentReplicaSetTerminatingReplicas` 打开时统计 `terminatingReplicas` 参与进度评估（见 `pkg/controller/replicaset/replica_set_utils.go`）。

### 垃圾回收（GC）

- 基于 `ownerReferences` 维护级联删除；
- 控制器：`pkg/controller/garbagecollector` 遍历/监听对象图并清理悬挂对象；
- 常见交互：删除 Deployment 时由 GC 级联清理其 RS 与 Pod（取决于 `PropagationPolicy`）。


