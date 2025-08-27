## Kubernetes 排查问题指南

面向集群值班与平台运维的实战速查手册，按场景给出诊断路径与常用命令。

### 0. 快速定位清单（优先顺序）

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep -E 'CrashLoopBackOff|ImagePullBackOff|Pending|Error'
kubectl -n kube-system get pods -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

### 1. 节点 NotReady / ReadyFlapping

- 检查 kubelet/containerd：

```bash
sudo systemctl status kubelet | cat
sudo journalctl -u kubelet -b -n 200 | cat
sudo systemctl status containerd | cat
sudo journalctl -u containerd -b -n 200 | cat
```

- 核对 cgroup 驱动一致（systemd）：containerd `SystemdCgroup=true`；`kubelet --cgroup-driver=systemd`。
- 网络内核参数：`br_netfilter`、`ip_forward=1`。
- 证书过期：`kubeadm certs check-expiration`。

### 2. Pod Pending（调度失败）

```bash
kubectl describe pod <POD>
kubectl get nodes -o json | jq '.items[].status.allocatable'
```

- 检查资源不足（CPU/内存/存储/加速卡）。
- 亲和/反亲和、污点/容忍是否匹配。
- 存储卷未就绪（PVC Pending）。
- 节点可调度性：`kubectl describe node <NODE>`（条件与 taints）。

### 3. CrashLoopBackOff / OOMKilled

```bash
kubectl logs <POD> -c <CONTAINER> --previous
kubectl describe pod <POD>
```

- 启动探针失败/配置错误（环境变量、Secret 挂载）。
- OOM：设置合理 `requests/limits`；查看 `Last State` 与 `OOMKilled` 原因。
- 文件句柄/权限：检查 SecurityContext、PSA/PSS、PodSecurityPolicy（如仍在使用）。

### 4. 网络故障（DNS/Service/跨 Pod 互通）

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl run -it --rm --restart=Never --image=ghcr.io/curl/curl:8.7.1 curl -- sh
# 容器内测试：
nslookup kubernetes.default.svc.cluster.local
curl -I http://<SERVICE_NAME>.<NAMESPACE>.svc.cluster.local
```

- CNI 插件 Pod 是否就绪；`kubectl -n kube-system logs <CNI_POD>`。
- kube-proxy 是否工作（iptables/IPVS 规则是否写入）。
- NetworkPolicy 是否拒绝流量。

### 5. 镜像拉取失败（ImagePullBackOff）

```bash
kubectl describe pod <POD> | sed -n '/Events/,$p'
```

- 私有仓库认证：检查 Secret `imagePullSecrets` 与节点到仓库的连通性。
- 镜像标签错误或不可用；网络代理/防火墙限制。

### 6. 存储问题（PVC Pending / 挂载失败）

```bash
kubectl get pvc -A
kubectl describe pvc <PVC>
kubectl describe pv <PV>
```

- CSI 控制器/节点插件日志；StorageClass 是否配置动态供给。
- 访问模式（RWO/ROX/RWX）与底层后端兼容性。

### 7. RBAC/权限问题（Forbidden）

```bash
kubectl auth can-i get pods --as system:serviceaccount:<NS>:<SA> -n <NS>
kubectl auth reconcile -f <RBAC.yaml> --dry-run=client -o yaml | diff - <(kubectl get -f - -o yaml)
```

- 确认 Role/ClusterRole 与 RoleBinding/ClusterRoleBinding 对应关系。
- 准入控制器是否阻断（查看 apiserver 日志或 Audit 日志）。

### 8. 控制平面健康

```bash
kubectl get componentstatuses # 新版本可能废弃，使用探针与指标
kubectl -n kube-system get pods -o wide
sudo crictl ps -a | cat
```

- etcd 选举/延迟/磁盘 IO；证书有效期；数据目录空间。
- apiserver QPS 限流、长尾延迟；`watch` 堵塞。

### 9. 常用工具与日志位置

- `kubectl logs/describe/top`，`stern`/`kail` 聚合日志。
- `journalctl -u kubelet/containerd`；`crictl` 查看容器与镜像。
- 网络：`ip a`、`ip r`、`iptables -L -t nat`、`ipvsadm -Ln`、`ss -lntup`。

### 10. 故障复现与最小化案例

- 将可疑部署最小化（单容器、禁探针、默认网络/存储），逐步添加配置定位。
- 隔离环境变量与镜像版本，开启详细日志级别（`-v=6` 以上）。

### 附：值班排查流程（简版）

1) 观测整体：节点/系统命名空间健康 → 事件 → 告警。
2) 确认影响面：命名空间/应用清单 → 是否集中在某节点/可用区。
3) 从易到难：资源/网络/存储 → 权限 → 控制面。
4) 最小化复现与旁路修复（回滚/重启/扩容）。


