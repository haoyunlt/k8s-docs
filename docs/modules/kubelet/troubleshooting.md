## Kubelet 排查指南（中文）

### 快速体征

- 服务与日志：`systemctl status kubelet`，`journalctl -u kubelet -f`
- 节点状态：`kubectl get node -o wide`，`kubectl describe node <name>`
- Pod 状态与事件：`kubectl get pods -A -o wide`，`kubectl describe pod <pod> -n <ns>`
- 探针与容器运行时：检查 CRI（containerd/cri-o）与 CNI/CSI 插件日志。

### 常见问题

- 节点 NotReady：
  - 网络/CNI 启动失败；
  - 证书失效或 kubelet 无法与 apiserver 通信；
  - 系统资源不足（磁盘压力、内存压力）。
- Pod 创建失败：
  - 镜像拉取失败（私有仓库凭据/网络）；
  - 挂载失败（CSI 驱动或权限问题）；
  - Init 容器/探针失败导致主容器未就绪。

### 定位要点

- 配置文件：`/var/lib/kubelet/config.yaml` 与启动参数（kubeadm 在 `/var/lib/kubelet/kubeadm-flags.env`）；
- 运行时：`crictl ps -a`、`crictl logs <cid>`，检查 sandbox 与容器状态；
- 网络：`ip a`、`ip route`、`iptables-save` 或 `ipvsadm -Ln`（取决于 kube-proxy 模式）。

### 指标与健康检查

- `10255/10250` 端口（受安全配置影响）；
- `/metrics` 指标：pod 启动延迟、镜像拉取、probe 结果统计。

### 源码位置

- 入口：`cmd/kubelet/`
- 主要逻辑：`pkg/kubelet/...`（pod 管理、镜像拉取、probe、volume、status 等）。


