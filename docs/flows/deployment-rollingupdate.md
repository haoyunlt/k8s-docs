## 业务流程：Deployment 滚动更新（中文）

描述 Deployment → ReplicaSet → Pod 的级联与滚更策略、进度与回滚。

```mermaid
flowchart TD
  D[Deployment] -->|reconcile| RSnew[ReplicaSet(new)]
  D -->|scale down| RSold[ReplicaSet(old)]
  RSnew --> PodsNew
  RSold --> PodsOld
```

### 关键逻辑

- 控制器：`pkg/controller/deployment` 协调新旧 RS；
- 进度跟踪：`ProgressDeadlineSeconds`、`MaxUnavailable/MaxSurge`；
- 终止副本：在特性门控打开时统计 `terminatingReplicas` 辅助进度判断。


