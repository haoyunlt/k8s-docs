## 接口与业务逻辑：Deployments（中文）

### 关键 API

- CRUD：`/apis/apps/v1/namespaces/{ns}/deployments`
- 伸缩：`/apis/apps/v1/namespaces/{ns}/deployments/{name}/scale`

### 业务逻辑关联

- Deployment 控制器协调 ReplicaSet，实施滚动更新与回滚；
- 进度与健康：根据可用副本、终止副本、探针成功率衡量；
- 与 HPA/PodDisruptionBudget 的联动。


