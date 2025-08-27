## etcd 代码与架构要点（中文）

说明：etcd 不在本仓库内，以下为与 Kubernetes 结合的关键点。

- 存储模型：多版本 MVCC，revision 单调递增；
- Watch：从 compact 版本后需自愈重建；
- 事务：apiserver 使用乐观并发（`resourceVersion` 对应 etcd revision）。

调优方向：

- 磁盘：独立高性能 SSD，确保 wal fsync 稳定；
- 网络：低抖动、低延迟；
- 集群规模：3/5 成员奇数，避免过大拓扑导致选举复杂与写放大。


