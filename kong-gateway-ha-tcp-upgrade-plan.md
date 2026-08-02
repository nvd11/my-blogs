# Kong API 网关改造计划：三节点覆盖 + 本地转发 + TCP 流代理

> 状态：计划中（实验性质）
> 日期：2026-08-02
> 集群：腾讯云 K3s（vm-0-2-debian 控制面 / OCI free-arm-vm / 本地 NUC）
> 管理方式：ArgoCD（App-of-Apps）→ Helm chart `kong-2.38.0`

## 1. 背景与动机

当前集群的 Kong 网关存在三个架构上的短板，本次改造要一并解决：

| 短板 | 现状 | 改造后 |
|------|------|--------|
| 控制面单点 | Kong Controller 只有 1 副本，落在腾讯云节点 | 3 副本，每节点 1 个 |
| 入口转发绕路 | svclb 收流量后可能跨节点转发（`externalTrafficPolicy: Cluster`） | 只找本节点 controller（`Local`） |
| 只支持 HTTP | Kong 只监听 80/443，不处理 TCP 流量 | 支持 TCP 流代理（Redis 等） |

这是一个**实验性改造**，目标是把 Kong 从"单点 HTTP 网关"升级为"高可用全能网关"。改造期间不影响现有 fastapi-svc / quarkus-svc 的 HTTP 路由（需要分步验证）。

## 2. 现状盘点（已实地核实）

### 2.1 Kong 部署形态

| 项目 | 现状 |
|------|------|
| 管理方式 | ArgoCD Application `kong-ingress-controller`（Helm chart `kong-2.38.0`） |
| Kong 版本 | KIC 3.1 / Kong 3.6（`ingress-controller` + `proxy` 双容器） |
| 控制面 | Deployment 1 副本，在 `vm-0-2-debian`（腾讯云） |
| 数据面入口 | `svclb` DaemonSet，**3 个节点各 1 个**（K3s 自动部署） |
| 入口端口 | `kong-proxy` Service：80:31850 / 443:31324（LoadBalancer） |
| externalTrafficPolicy | `Cluster`（可能跨节点转发） |
| TCPIngress CRD | ✅ 已存在（Kong chart 自带 `tcpingresses.configuration.konghq.com`） |

### 2.2 相关基础设施

- K3s 版本：server v1.35.5+k3s1 / agent v1.36.2+k3s1
- 节点 flannel 已统一走 Tailscale（`--flannel-iface tailscale0`，MTU 1230）
- 三个节点互连正常（之前排障已修复）
- 备份已存：`~/kong-backup-2026-08-02/`（deployment / svc / configmap / ingressclass）

### 2.3 ArgoCD 现有配置（`kong-ingress-controller` Application）

```yaml
spec:
  source:
    chart: kong
    repoURL: https://charts.konghq.com
    targetRevision: 2.38.0
    helm:
      values: |
        ingressController:
          installCRDs: false
        env:
          database: "off"
        gateway:
          enabled: true
  destination:
    name: tencent-dp1-cluster
    namespace: kong-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 3. 改造目标与实施方案

### 目标 1：让 Kong Controller 覆盖 3 节点

**思路**：`replicas: 3` + 反亲和，让每个节点恰好 1 个 controller pod。

**方案**（修改 ArgoCD values）：

```yaml
controller:
  replicas: 3
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: kong
          topologyKey: kubernetes.io/hostname
```

- `replicas: 3`：3 副本
- `podAntiAffinity` + `topologyKey: kubernetes.io/hostname`：保证 3 个 pod 分布在不同节点，不会挤在一起
- 注意：KIC 多副本时只有 leader 实际 watch K8s 写配置，其余 standby；配置同步后所有 proxy 数据面都生效

**预期效果**：

```
vm-0-2-debian:  kong-controller pod #1 (leader 或 standby)
free-arm-vm:    kong-controller pod #2
nuc:            kong-controller pod #3
```

### 目标 2：让 3 门卫只找自己节点的 Kong Controller

**思路**：`externalTrafficPolicy: Local`，svclb 只转发到本节点的 controller pod。

**方案**（修改 ArgoCD values）：

```yaml
proxy:
  externalTrafficPolicy: Local
```

**⚠️ 关键前提**：`Local` 策略要求**每个节点都有对应后端 pod**，否则该节点门卫收到的流量会直接丢弃。所以目标 1 必须先完成（3 节点各 1 个 controller），再切 `Local`——**顺序不能反**。

**预期效果**：

```
客户端 → 100.77.64.95:31850 → 腾讯云 svclb → 本节点 controller #1
客户端 → 100.105.130.0:31850 → OCI svclb    → 本节点 controller #2
客户端 → 100.104.150.19:31850 → NUC svclb    → 本节点 controller #3
```

零跨节点转发，延迟最低，且保留客户端真实 IP（`Local` 的附加收益）。

### 目标 3：让 Kong Controller 支持 TCP 流量转发

**思路**：开启 Kong 的 stream 模式（TCP 代理），用 `TCPIngress` 声明 TCP 路由。

**方案**（修改 ArgoCD values）：

```yaml
proxy:
  stream:
    - containerPort: 6379      # Redis 端口（示例）
      servicePort: 6379
env:
  nginx_stream_listen: "[::]:6379"
```

创建 TCPIngress 资源（加到 redis-deployment 仓库或独立 manifest）：

```yaml
apiVersion: configuration.konghq.com/v1beta1
kind: TCPIngress
metadata:
  name: redis-tcp
  namespace: redis
spec:
  rules:
    - port: 6379
      backend:
        serviceName: redis
        servicePort: 6379
```

**注意**：
- Kong 的 stream 模式是**透传 TCP**，不解析协议内容，适合 Redis/MySQL/自定义 TCP 服务
- 需要在 `kong-proxy` Service 上暴露对应端口（Helm 的 `proxy.stream` 会自动处理）
- UDP 需要单独的 `udpProxy` 配置，本次不做

## 4. 实施步骤（分步，每步验证）

### 阶段 A：准备（已完成 ✅）

- [x] 侦察部署形态（Helm + ArgoCD）
- [x] 备份现状（`~/kong-backup-2026-08-02/`）
- [x] 确认 TCPIngress CRD 存在

### 阶段 B：目标 1 —— Controller 三副本

1. 修改 `my-argocd-manifests/argocd-apps/kong-ingress-controller-app.yaml` 的 values
2. 等待 ArgoCD 自动同步（≤3 分钟）
3. 验证：`kubectl get pods -n kong-system -o wide` 看到 3 个 controller 分布 3 节点
4. 验证：fastapi-svc / quarkus-svc HTTP 路由依然正常（`curl http://<任意节点>:31850/svc2`）

### 阶段 C：目标 2 —— 切换 Local 策略

1. 确认阶段 B 的 3 个 controller pod 已就位
2. 修改 values 加 `externalTrafficPolicy: Local`
3. 验证：三个节点入口分别测试，全部 200
4. 验证：从三个节点分别 curl，确认流量走本节点（可看 controller 日志）

### 阶段 D：目标 3 —— TCP 流代理

1. 修改 values 加 `proxy.stream` + `nginx_stream_listen`
2. 部署 Redis（如果还没有）或准备一个测试 TCP 服务
3. 创建 `TCPIngress` 资源
4. 验证：`redis-cli -h <任意节点> -p 6379 ping` 经 Kong 转发可达

### 阶段 E：回归测试

- [ ] fastapi-svc `/svc2` 经三个节点入口全部 200
- [ ] quarkus-svc `/svc1` 经三个节点入口全部 200
- [ ] TCP 转发（Redis ping）经三个节点入口可达
- [ ] `kubectl get pods -A` 无异常重启

## 5. 风险与回滚

| 风险 | 影响 | 缓解 |
|------|------|------|
| 改 proxy 配置 | HTTP 路由短暂中断（几十秒） | 滚动更新，避开业务高峰 |
| `Local` 策略但节点无 pod | 该节点入口丢流量 | 严格按顺序：先目标 1 后目标 2 |
| stream 配置错误 | Kong 起不来 | 备份已存，回滚 = 恢复旧 values |
| 反亲和调度失败 | pod 分布不均 | 检查节点 label 与可用资源 |

**回滚路径**：
```bash
# 1. 恢复 ArgoCD Application 的旧 values（从备份或 Git 历史）
# 2. 等 ArgoCD 自动回滚
# 3. 极端情况直接 kubectl apply 备份文件:
kubectl apply -f ~/kong-backup-2026-08-02/deployment.yaml
kubectl apply -f ~/kong-backup-2026-08-02/proxy-svc.yaml
```

## 6. 验收标准

1. **目标 1**：`kubectl get pods -n kong-system` 3 个 controller，`-o wide` 显示分布在 3 个不同节点
2. **目标 2**：`kubectl get svc kong-proxy -o yaml` 的 `externalTrafficPolicy: Local`；三个入口 curl 全部 200
3. **目标 3**：TCP 流量经 Kong 转发可达（Redis ping PONG），HTTP 路由零回归

## 7. 后续讨论（暂不实施）

- **Redis 是否真的走 Kong TCP 转发？** 实验验证后由业务需求决定：
  - 走 Kong：统一入口、统一认证/审计（适合"全能网关"诉求）
  - 直连 hostNetwork：延迟最低（Redis 作为数据面服务，直连 `100.105.130.0:6379` 更简单）
- 生产化时考虑：stream 模式的 TLS 终止、UDP 代理（需 `udpProxy`）、多副本 leader 选举监控

## 8. 参考资料

- 相关排障记录：`oci-k3s-node-kong-lb-flannel-tailscale.md`（同仓库）
- Kong Helm chart values：`proxy.stream` / `controller.affinity` / `proxy.externalTrafficPolicy`
- K3s svclb 机制：DaemonSet 每节点一个，`Local` 策略需每节点有后端
