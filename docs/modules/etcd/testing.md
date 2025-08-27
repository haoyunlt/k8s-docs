## etcd 测试指南（中文）

### 健康与性能

- 压测：并发 put/get/watch，观察 P99 与磁盘 IO；
- 可靠性：网络分区/成员故障恢复测试。

### 与 Kubernetes 联调

- 在测试集群中进行大规模对象创建与更新，评估 apiserver→etcd 延迟；
- 校验 compact/defrag 对集群稳定性的影响与窗口选择。


