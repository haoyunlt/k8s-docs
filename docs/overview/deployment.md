## Kubernetes 部署指南（中文）

本文提供从零到一的部署指导与要点，覆盖单机实验与高可用生产集群。建议优先使用 kubeadm；开发/测试可选 kind 或 minikube。

### 前置条件

- OS：Linux（生产建议），macOS/Windows 可通过 VM/容器化工具体验；
- 硬件：最低 2 CPU / 4 GiB（单机），生产按规模与 SLO 规划；
- 网络：允许节点互通与必要端口放行；
- 运行时：containerd/CRI-O；
- 模块：选择 CNI（如 Calico/Cilium）与存储（CSI）。

### 方案一：kubeadm（推荐）

1) 安装基础组件：kubeadm、kubelet、kubectl、容器运行时（containerd）。
2) 初始化控制平面（单控）：
   - `kubeadm init --pod-network-cidr=<CIDR> --kubernetes-version=<vX.Y.Z>`
   - 使用输出的 kubeconfig 配置 `~/.kube/config`。
3) 部署 CNI 插件（以 Calico 为例）：
   - `kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml`
4) 加入工作节点：
   - 在 Node 上执行 `kubeadm join <CONTROL_PLANE_IP>:6443 --token ... --discovery-token-ca-cert-hash ...`
5) 验证：
   - `kubectl get nodes -o wide`，`kubectl get pods -A`

高可用（多控）要点：

- etcd：内置栈（stacked etcd）或外部 etcd；推荐 3/5 节点奇数规模；
- 负载均衡：前置 TCP LB 暴露 6443 至多控；
- 证书与 kubeadm 配置：使用 ClusterConfiguration 指定 controlPlaneEndpoint；
- 升级：`kubeadm upgrade plan/apply`，分批漂移控制面与节点。

### 方案二：kind（开发/CI）

- 使用 Docker 在本机快速拉起多节点集群：
  - `kind create cluster --config kind.yaml`
  - 支持自定义端口映射、CNI、存储等；非常适合 CI/E2E。

### 方案三：minikube（本地体验）

- 一键体验：`minikube start --driver=docker`；
- 适合单节点实验与教学；
- 注意与生产的差异（LB/存储/网络）。

### 运维与安全建议

- 证书与密钥：启用外部 KMS，定期轮换；
- RBAC 最小权限；审计日志开启并集中收集；
- 资源配额与限制（LimitRange/ResourceQuota）；
- 节点安全基线与升级窗口管理；
- 监控告警：Prometheus/Grafana/Loki/Alertmanager；
- 备份：etcd 定期快照，关键组件与业务清单（YAML）纳入灾备。

附：英文部署文档位于 `docs/deployment.md`，本文为中文详解版本。


