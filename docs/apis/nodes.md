## 接口与业务逻辑：Nodes（中文）

### 关键 API

- CRUD：`/api/v1/nodes`
- 状态：`/api/v1/nodes/{name}/status`（kubelet 上报）

### 业务逻辑关联

- Node 生命周期控制器：NotReady/驱逐/污点与容忍度；
- 资源与可分配：capacity/allocatable 影响调度；
- 节点升级与排空（drain/cordon/uncordon）。


