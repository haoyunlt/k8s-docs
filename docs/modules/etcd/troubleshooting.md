## etcd 排查指南（中文）

### 快速体征

- 成员健康：`etcdctl --endpoints=<e1,e2,e3> endpoint health`
- 成员列表：`etcdctl --endpoints=<...> member list`
- 指标：抓取 `/metrics` 关注 wal/fsync、raft propose/apply 延迟。

### 常见问题

- 选举频繁：网络抖动或磁盘延迟过高；
- 写入慢：磁盘 IO 瓶颈（wal 同步）、快照/压缩滞后；
- 存储膨胀：长时间大量小写或 watch 未压缩，需定期 `compact` 与 `defrag`。

### 维护操作

- 压缩：`etcdctl compact <revision>`；
- 整理：`etcdctl defrag`；
- 备份/还原：使用 `snapshot save` 与 `snapshot restore`。

### 与 apiserver 的联动

- apiserver 超时/429 时检查 etcd 指标是否异常；
- 调整 apiserver 存储 QPS 与缓存大小，避免热点写放大。


