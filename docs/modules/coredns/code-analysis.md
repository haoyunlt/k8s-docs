## CoreDNS 代码与插件要点（中文）

说明：CoreDNS 不在本仓库内，以下为与 Kubernetes 集成的核心点。

- 与 kube-dns Service 集成：通过 `cluster.local` 域与 `kubeapi`/`kubernetes` 插件解析；
- 常用插件：`kubernetes`、`forward`、`cache`、`health`、`ready`、`metrics`；
- 性能关键：缓存、并发、上游可用性与网络延迟。


