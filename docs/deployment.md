## Kubernetes 部署指南

本指南覆盖从本地开发到生产高可用的常见部署路径：kind/minikube、kubeadm（单控/高可用）、托管云（EKS/GKE/AKS）。

### 1. 前提条件

- 目标主机：Linux x86_64（控制平面与工作节点）。
- 配置建议：2C4G+（控制平面）、2C4G+（工作节点）；生产视工作负载增加。
- 网络要求：节点间互通，不阻断 Pod/Service 端口；时间同步；禁用 swap。
- 组件版本：建议同一大版本（如 v1.30.x）。

### 2. 快速本地环境（可选）

- **kind**（Docker in Docker）：

```bash
brew install kind kubectl
kind create cluster --name demo
kubectl get nodes
```

- **minikube**：

```bash
brew install minikube kubectl
minikube start --driver=docker
kubectl get nodes
```

### 3. 使用 kubeadm 部署（生产推荐）

#### 3.1 系统准备（所有节点）

```bash
sudo swapoff -a
sudo sed -i.bak '/\sswap\s/d' /etc/fstab
sudo modprobe br_netfilter
echo 'net.bridge.bridge-nf-call-iptables = 1' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-kubernetes-cri.conf
sudo sysctl --system
```

#### 3.2 安装 containerd（所有节点）

```bash
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo systemctl enable --now containerd
```

确保 cgroup 驱动与 kubelet 一致（systemd）：在 `config.toml` 中设置：

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

重启 containerd：

```bash
sudo systemctl restart containerd
```

#### 3.3 安装 kubeadm/kubelet/kubectl（所有节点）

以 Debian/Ubuntu 为例：

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

#### 3.4 初始化控制平面（第一个控制平面节点）

准备 `kubeadm-config.yaml`：

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.30.0
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
controllerManager:
  extraArgs:
    node-cidr-mask-size: "24"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
```

初始化：

```bash
sudo kubeadm init --config kubeadm-config.yaml
```

按输出指引配置 `kubectl`：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 3.5 安装 CNI（Calico 示例）

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
```

#### 3.6 加入工作节点

在每个工作节点执行控制平面输出的 join 命令，例如：

```bash
sudo kubeadm join <CONTROL_PLANE_LB_OR_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

#### 3.7 基础组件与可选项

- Metrics Server：

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

- Kubernetes Dashboard（可选，生产谨慎）：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

#### 3.8 高可用（HA）部署要点

- 负载均衡：为多副本 apiserver 提供前端 VIP/LB。
- etcd：奇数副本（3/5）；可选 external etcd（推荐）或 stacked etcd。
- 其他控制平面节点加入：

```bash
sudo kubeadm join <LB_OR_FIRST_CP>:6443 --control-plane \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### 4. 验证与健康检查

```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
kubectl get svc -A
```

部署示例应用：

```bash
kubectl create deployment nginx --image=nginx:1.25
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl run curl --image=ghcr.io/curl/curl:8.7.1 -it --rm --restart=Never -- sh -lc 'curl -I nginx'
```

### 5. 回滚与清理

```bash
sudo kubeadm reset -f
sudo systemctl stop kubelet containerd
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F
sudo ipvsadm --clear || true
sudo rm -rf /etc/cni/net.d /var/lib/cni /var/lib/kubelet /etc/kubernetes ~/.kube
```

### 6. 常见参数与建议

- kube-proxy IPVS 模式：

```bash
kubectl -n kube-system edit configmap kube-proxy
# mode: "ipvs"
```

- containerd 镜像加速与私有仓库：配置 `/etc/containerd/config.toml` 的 `registry.mirrors`。
- 控制面监控：apiserver/etcd 指标与审计日志；集成 Prometheus/Grafana。


