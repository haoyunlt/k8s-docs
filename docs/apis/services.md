## 接口与业务逻辑：Services（中文）

### 关键 API

- CRUD：`/api/v1/namespaces/{ns}/services`
- Endpoints/EndpointSlice：自动维护后端实例集合

### 业务逻辑关联

- kube-proxy 监听 Service/EndpointSlice，生成数据面规则（iptables/ipvs）；
- CoreDNS 监听 Service，提供域名解析；
- 类型差异：ClusterIP、NodePort、LoadBalancer、ExternalName。


