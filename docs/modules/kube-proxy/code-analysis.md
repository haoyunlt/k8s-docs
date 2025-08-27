## Kube-Proxy 代码剖析（中文）

### 组件概览

- 从 API Server watch Service/Endpoints(EndpointSlice)；
- 构建转发表并下发到内核（iptables/ipvs）。

### 关键路径

- 入口：`cmd/kube-proxy/app`；
- iptables 模式：`pkg/proxy/iptables`，基于链规则匹配并做 DNAT；
- ipvs 模式：`pkg/proxy/ipvs`，基于虚拟服务/真实服务器表项转发；
- Endpoints 管理：`pkg/proxy/endpoints`。

### 规则同步

- 统一入口：`syncProxyRules()`（不同实现内维护）根据最新的 Service/EndpointSlice 视图生成内核规则；
- iptables：构建 `KUBE-SVC`/`KUBE-SEP` 等链并插入/删除规则以匹配 ClusterIP/NodePort；
- ipvs：构建虚拟服务并维护 real servers，引入一致性与增量更新策略；
- Windows（可选）：`pkg/proxy/winkernel` 使用 HCN API 同步规则，测试中调用 `syncProxyRules()` 验证。


