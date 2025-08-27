## Kubelet 代码剖析（中文）

### 入口与主循环

- 入口命令：`cmd/kubelet/kubelet.go` → `app.NewKubeletCommand` → `app.RunKubelet`
  - `cmd/kubelet/app/server.go:NewKubeletCommand(...)` 构造 Cobra 命令与 flag；
  - `server.RunKubelet(...)` 完成依赖注入（kubeClient、Informer、ContainerManager、RuntimeService 等）并启动主循环；
  - 远程运行时初始化：`pkg/kubelet/kubelet.go` 中通过
    `remote.NewRemoteRuntimeService(...)` 与 `remote.NewRemoteImageService(...)` 创建 CRI 客户端；
- 主体：`pkg/kubelet/kubelet.go`
  - `(*Kubelet).Run()` 内最终进入 `(*Kubelet).syncLoop(...)` 驱动 Pod 同步（配置、PLEG、定时器等触发）。

### 关键子系统

- 容器运行时（CRI）：通过 `RemoteRuntimeService` 与 `RemoteImageService` 交互；
- 卷与存储（CSI）：`pkg/volume` 与 `k8s.io/mount-utils`；
- 探针：`pkg/kubelet/prober`（liveness/readiness/startup）；
- Cgroups 与资源：`pkg/kubelet/cm`；
- 节点状态：`pkg/kubelet/nodestatus` 与 `status/` 汇报。

### 同步流程要点

1) 从 apiserver/informer 获取期望态；
2) 计算容器变化并调用 CRI 创建/更新/删除；
3) 执行卷挂载、探针与网络准备；
4) 上报 Pod/容器状态与事件。

### 主循环与事件通道

- 主循环：`pkg/kubelet/kubelet.go:syncLoop(ctx, updates, handler)`
  - 关键通道：
    - 配置变更 `updates <-chan kubetypes.PodUpdate`（file/apiserver/http 源合并）；
    - 定时同步 `syncTicker`（周期性全量校对）；
    - housekeeping 定时器 `housekeepingTicker`；
    - PLEG 事件通道 `plegCh <-chan *pleg.PodLifecycleEvent`；
  - 监控点：`syncLoopMonitor atomic.Value` 记录进入/退出迭代的时间戳，便于健康检查；
- 单轮迭代：`pkg/kubelet/kubelet.go:syncLoopIteration(...) bool`
  - 从上述通道 select 读取，转换为对单 Pod 的 `SyncHandler` 调度（见下文 `podWorkers`）；
  - 在迭代首尾写入 `syncLoopMonitor`，配合 watchdog 对 `syncloop` 就绪探测；
- PLEG（Pod Lifecycle Event Generator）：`pkg/kubelet/pleg`
  - `GenericPLEG` 周期 `Relist()` 对比容器运行时快照 → 形成 `ContainerStarted/ContainerDied/PodSync/...` 等事件；
  - 通过 `Watch()` 将事件送入 `plegCh`，触发相应 Pod 的快速同步。

### podWorkers 与并发模型

- 文件：`pkg/kubelet/pod_workers.go`
- 角色：每个 Pod 一个串行 worker，确保同一 Pod 的状态变更顺序与幂等；
- 关键接口：
  - `UpdatePod(UpdatePodOptions)`：提交创建/更新/删除等操作至对应 worker 队列；
  - `SyncKnownPods(desiredPods)`：基于期望态对 worker 集合做对齐；
  - 若 Pod 处于终止序列，worker 在看到来自 PLEG 的终止相关事件后继续推进清理；
- 与主循环关系：`syncLoopIteration(...)` 根据事件类型将目标 Pod 提交给 `podWorkers`，而不是直接执行重逻辑；

### 与 CRI 的交互关键路径

- 依赖注入：`pkg/kubelet/kubelet.go`
  - `kubeDeps.RemoteRuntimeService = remote.NewRemoteRuntimeService(...)`
  - `kubeDeps.RemoteImageService = remote.NewRemoteImageService(...)`
- 调用点：
  - 容器与 Pod 沙箱生命周期：`pkg/kubelet/kuberuntime/` 组装 `RunPodSandbox/CreateContainer/StartContainer/StopContainer/Remove*` 等 CRI 调用；
  - 镜像：`ImageService.PullImage/ListImages`；

### 状态汇报与探针

- 状态管理器：`pkg/kubelet/status/status_manager.go`
  - `SetPodStatus(...)` 聚合并去重状态变化，批处理同步至 apiserver；
  - `SetContainerReadiness(...)` 维护容器级就绪并影响 Pod 条件；
- 探针：`pkg/kubelet/prober`（readiness/liveness/startup）
  - 探针结果通过 status manager 影响 `PodReady` 等条件，并可能触发容器重启；

### 卷与 Desired/Actual 状态机

- 卷管理器：`pkg/kubelet/volumemanager`
  - `desiredStateOfWorldPopulator` 根据 `PodManager` 的期望态填充目标挂载集合；
  - `reconciler` 对比 desired/actual，执行 attach/mount/unmount/detach；
  - 关键依赖：`PodManager`（`pkg/kubelet/pod`）提供当前 Pod 视图；

### 典型时序（创建 Pod）

1) Config/PLEG 事件触发 `syncLoopIteration` 提交给 `podWorkers`；
2) `podWorkers` 串行执行该 Pod 的 `syncPod`：
   - 卷准备（VolumeManager → CSI）→ 网络准备（CNI，由 runtime/插件完成）；
   - 计算容器变化并通过 CRI 创建/启动容器；
3) 探针开始工作，状态管理器汇报 PodStatus；
4) PLEG 后续事件促使状态与生命周期推进；

### 典型时序（优雅终止）

1) 接收删除/驱逐事件 → `podWorkers` 将目标 Pod 置于终止路径；
2) 发送 TERM，等待 `terminationGracePeriodSeconds`，必要时 KILL；
3) 卷与网络清理，PLEG 确认容器结束；
4) 状态管理器上报终止状态并最终移除。

### 关键文件速览（便于交叉定位）

- 主循环与入口：`cmd/kubelet/app/server.go`、`pkg/kubelet/kubelet.go`
- PLEG：`pkg/kubelet/pleg`（`GenericPLEG`、`Relist/Watch`）
- Pod 并发：`pkg/kubelet/pod_workers.go`
- 运行时抽象：`pkg/kubelet/kuberuntime/`、`pkg/kubelet/remote`（CRI 客户端）
- 状态：`pkg/kubelet/status/status_manager.go`
- 卷：`pkg/kubelet/volumemanager`、`pkg/volume`

