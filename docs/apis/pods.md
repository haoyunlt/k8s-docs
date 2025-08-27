## 接口与业务逻辑：Pods（中文）

### 关键 API

- CRUD：`/api/v1/namespaces/{ns}/pods`
- 绑定：`/api/v1/namespaces/{ns}/pods/{name}/binding`（由 scheduler 发起）
- 日志/exec：通过 kubelet 代理通道转发（认证后）

### 业务逻辑关联

- 创建：由控制器/用户提交 → 调度 → 节点创建 → 状态回报；
- 删除：优雅终止 → 容器退出 → 卷卸载 → 状态终止；
- 事件：调度失败、探针失败、镜像拉取失败等通过 Event 暴露。


