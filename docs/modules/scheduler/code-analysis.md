## Scheduler 代码剖析（中文）

### 启动与框架

- 入口：`cmd/kube-scheduler/app`
- 通过 `framework` 注册插件，阶段包括：QueueSort、PreFilter、Filter、PostFilter、PreScore、Score、Reserve、Permit、PreBind、Bind、PostBind；
- 多队列与优先级：`pkg/scheduler/internal/queue`。

### 关键路径

1) 调度主循环：`pkg/scheduler/scheduler.go:Run()` 启动 `go wait.UntilWithContext(ctx, sched.ScheduleOne, 0)`；
2) 单个 Pod 调度：`pkg/scheduler/schedule_one.go:ScheduleOne(ctx)`
   - 从队列取出 `NextPod`（`SchedulingQueue.Pop`）；
   - 预选：运行 `PreFilter`、`Filter`，得到可行节点集；
   - 优选：运行 `PreScore`、`Score`，聚合打分并选择候选；
   - 预留与许可：`Reserve`/`Permit` 插件可拦截或异步等待；
   - 绑定：`PreBind` → `Bind` → `PostBind`；
   - 缓存与指标：更新调度缓存、记录 `schedulingLatency` 等；
3) 队列与快照：`pkg/scheduler/internal/queue` 维护待调度队列与 `PodNominator`；
4) 抢占（Preemption）：当不可调度且存在 `PostFilter` 插件时，尝试选择牺牲者集合：
   - 见 `schedule_one.go` 关于 `Unschedulable` 分支，`diagnosis.NodeToStatus.SetAbsentNodesStatus(s)` 辅助效率；
   - 工具函数：`pkg/scheduler/util/utils.go:GetEarliestPodStartTime` 等辅助受害者排序；
   - 支持 Extender 参与：`testing/framework/fake_extender.go:ProcessPreemption` 展示接口形态。

### 常用插件

- NodeResourcesFit、TaintToleration、NodeAffinity、InterPodAffinity、TopologySpread、DefaultBinder 等。

### 关键文件速览

- 核心：`pkg/scheduler/scheduler.go`（Run）、`pkg/scheduler/schedule_one.go`（ScheduleOne）
- 队列：`pkg/scheduler/internal/queue`（优先级、Nominator、Async API Calls）
- 框架：`pkg/scheduler/framework`（插件接口与生命周期）
- 抢占：`pkg/scheduler/util`（受害者选择与排序），`schedule_one.go` 的 Unschedulable 路径


