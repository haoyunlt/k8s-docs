## Kubernetes 测试与验证指南

面向平台与应用团队的分层验证手册，覆盖集群可用性、网络/存储、工作负载与扩展性测试。

### 1. 分层测试矩阵

- **基础可用性**: 节点、系统组件、API 基线能力。
- **网络**: Pod 互通、Service 转发、DNS、NetworkPolicy。
- **存储**: PVC 动态供给、读写模式、快照/扩容。
- **工作负载**: 部署/回滚、探针、弹性伸缩（HPA/VPA）。
- **弹性与恢复**: 节点宕机、重启、驱逐、控制面故障演练。
- **一致性与兼容**: K8s Conformance（Sonobuoy）。

### 2. 快速健康检查

```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
kubectl auth can-i list pods --all-namespaces
```

### 3. 网络测试

```bash
kubectl create deployment demo --image=nginx:1.25
kubectl expose deployment demo --port=80 --type=ClusterIP
kubectl run netshoot --image=nicolaka/netshoot -it --rm --restart=Never -- sh -lc "curl -I http://demo.default.svc.cluster.local"
```

- DNS：在容器内执行 `nslookup kubernetes.default`。
- NetworkPolicy：创建 deny-all，再按需放通，验证连通矩阵。

### 4. 存储测试

创建 `StorageClass`（如云盘/本地 provisioner），然后：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 5Gi
```

挂载并写入验证：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: io-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: io-test
  template:
    metadata:
      labels:
        app: io-test
    spec:
      containers:
      - name: busybox
        image: busybox:1.36
        command: ["sh","-lc","dd if=/dev/urandom of=/data/test.bin bs=1M count=100 && ls -lh /data && sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data-pvc
```

### 5. 工作负载与弹性伸缩

部署 HPA 测试：

```bash
kubectl create deployment hpa-demo --image=polinux/stress
kubectl set resources deploy hpa-demo --requests=cpu=100m --limits=cpu=200m
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=5
kubectl run loader --image=busybox:1.36 -it --rm -- sh -lc "while true; do wget -q -O- http://hpa-demo; done"
kubectl get hpa -w | cat
```

### 6. 故障演练与恢复

- 驱逐与重调度：`kubectl drain <NODE> --ignore-daemonsets --delete-emptydir-data`，观察工作负载迁移。
- 节点失联：停止 kubelet/containerd，验证控制面与调度恢复。
- 控制面 Pod 异常：在可控环境验证高可用与告警触发。

### 7. Conformance 测试（Sonobuoy）

```bash
SONO=$(curl -s https://api.github.com/repos/vmware-tanzu/sonobuoy/releases/latest | jq -r '.assets[] | select(.name | test("linux_amd64.tar.gz")) | .browser_download_url')
curl -L "$SONO" -o sonobuoy.tar.gz
tar -xzf sonobuoy.tar.gz
sudo mv sonobuoy /usr/local/bin/
sonobuoy run --mode=certified-conformance
sonobuoy status
sonobuoy retrieve .
sonobuoy results $(ls | grep tar.gz | head -n1)
```

### 8. 观测与基线

- 指标：apiserver/etcd/kubelet/kube-proxy；组件 SLO 与告警阈值。
- 日志：系统命名空间关键组件采集与留存；审计日志闭环。

### 9. CI 集成建议

- 在 CI 中执行 `kubectl` 校验清单的 `--dry-run=server`、`kubeconform`/`kubeval` 校验 Schema。
- 使用 kind 启动临时集群执行集成测试，用完销毁。


