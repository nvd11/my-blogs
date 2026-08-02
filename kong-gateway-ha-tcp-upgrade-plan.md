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

> **改动文件总览（先看这里）**

| 目标 | 仓库 | 文件 | 改动内容 |
|------|------|------|---------|
| 目标 1 | `nvd11/my-argocd-manifests` | `argocd-apps/kong-controller-app.yaml` | `helm.values` 加 `deployment.daemonset: true`（首选，无需硬编码副本数） |
| 目标 2 | `nvd11/my-argocd-manifests` | `argocd-apps/kong-controller-app.yaml` | `helm.values` 加 `proxy.externalTrafficPolicy: Local` |
| 目标 3 | `nvd11/my-argocd-manifests` | `argocd-apps/kong-controller-app.yaml` | `helm.values` 加 `proxy.stream` + `nginx_stream_listen`（**本次**） |
| 目标 3 | `nvd11/redis-deployment` | `k8s/redis.yaml`（追加）或新建 `k8s/tcpingress.yaml` | 新增 `TCPIngress` 资源（**下次**，Redis 接入时） |

**为什么都在 `kong-controller-app.yaml`？** 因为 Kong 是 Helm chart 部署的，所有配置（副本数、亲和性、网络策略、stream）都是 chart 的 values，集中在 ArgoCD Application 的 `helm.values` 字段。改这一个文件 → push → ArgoCD 自动同步 → 生效。

### 目标 1：让 Kong Controller 覆盖 3 节点

**文件**：`my-argocd-manifests/argocd-apps/kong-controller-app.yaml`

**首选方案：DaemonSet 模式（推荐，摆脱硬编码副本数）**

Kong chart 原生支持 `deployment.daemonset: true`，把 controller 从 Deployment 渲染成 **DaemonSet**——K8s 的 DaemonSet 天生就是"每个节点一个 pod"，**不需要写 `replicas`，节点数变多少就有多少 pod**，加节点自动扩容、删节点自动收缩。

**改动**：在现有 `helm.values` 里追加（只需一行开关）：

```yaml
    helm:
      values: |
        ingressController:
          installCRDs: false # 保留
        env:
          database: "off"    # 保留
        gateway:
          enabled: true      # 保留
        deployment:                                  # ← 新增
          daemonset: true                            # ← 新增: DaemonSet 模式
```

**为什么这是正解**：
- 不硬编码 `3`，集群加节点 → 自动多一个 controller pod，天然"一个 node 一个 pod"
- DaemonSet 模式模板自动省略 `replicas` 字段（已从 chart 源码确认）
- 与 svclb 的机制完全一致（svclb 也是 DaemonSet，每节点一个）

**备选方案：Deployment + replicas（如果不想用 DaemonSet）**

需要硬编码副本数，节点数变化时要手动改：

```yaml
        controller:
          replicas: 3
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchLabels:
                      app.kubernetes.io/name: kong
                  topologyKey: kubernetes.io/hostname   # 按节点维度排他
```

**"一个 node 一个 pod"的两种实现方式**（仅针对 Deployment 备选，效果相同、机制不同，二选一）：

**方式 A：podAntiAffinity（硬性排他，推荐）** —— 语义就是"同一 hostname 不许有两个 kong pod"，配合 `replicas: 3` 自然分散到 3 节点。上面备选方案就是方式 A。

**方式 B：topologySpreadConstraints（拓扑分布）** —— K8s 1.19+ 官方推荐的现代做法，`maxSkew: 1` 表示节点间 pod 数差异不超过 1：

```yaml
        controller:
          replicas: 3
          topologySpreadConstraints:                    # ← 替代 affinity 字段
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: DoNotSchedule
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: kong
```

**选择建议**：首选 DaemonSet 模式（无需管副本数）；如果必须用 Deployment，3 节点 3 副本的"精确对等"场景用方式 A（硬性、直白、无中间态）；节点数多于副本数的弹性场景用方式 B。

**预期效果（DaemonSet 模式）**：

```
vm-0-2-debian:  kong-controller pod (DaemonSet 自动)
free-arm-vm:    kong-controller pod (DaemonSet 自动)
nuc:            kong-controller pod (DaemonSet 自动)
新增节点:        kong-controller pod 自动出现 ← 无需改配置！
```

### 目标 2：让 3 门卫只找自己节点的 Kong Controller

**文件**：`my-argocd-manifests/argocd-apps/kong-controller-app.yaml`

**改动**：在 `helm.values` 追加：

```yaml
        proxy:                                        # ← 新增
          externalTrafficPolicy: Local                # ← 新增
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

> **本次范围**：只做 Kong 侧的 TCP 能力（文件 A）——让 Kong 能监听并转发 TCP 端口。
> **下次范围**：Redis 接入（文件 B + Redis 部署）——定义具体路由规则，作为独立阶段再做。

**文件 A（本次）**：`my-argocd-manifests/argocd-apps/kong-controller-app.yaml`（开 stream）

**改动**：在 `helm.values` 追加：

```yaml
        proxy:
          externalTrafficPolicy: Local                # 目标2加的，保留
          stream:                                     # ← 新增
            - containerPort: 6379                     # ← 新增 (示例端口, 可改)
              servicePort: 6379                       # ← 新增
        env:
          database: "off"                             # 保留
          nginx_stream_listen: "[::]:6379"            # ← 新增
```

**本次验收**：Kong 的 stream 模式开启、6379 端口被监听（`ss -tlnp | grep 6379` 可见）。**不需要** TCPIngress——没有路由规则时流量进得来但没处去，这没关系，本次只验证"能力就绪"。

**文件 B（下次，Redis 接入时再做）**：`redis-deployment/k8s/redis.yaml`（追加 TCPIngress）或新建 `redis-deployment/k8s/tcpingress.yaml`

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

**文件**：`my-argocd-manifests/argocd-apps/kong-controller-app.yaml`

```bash
# 1. 编辑文件
cd ~/my-argocd-manifests
vim argocd-apps/kong-controller-app.yaml    # helm.values 加 deployment.daemonset: true

# 2. 推送 (ArgoCD 自动同步, ≤3分钟)
git add argocd-apps/kong-controller-app.yaml
git commit -m "feat(kong): deploy controller as DaemonSet (one pod per node)"
git push origin main
```

```bash
# 3. 验证: 3 个 controller 分布 3 节点
kubectl get pods -n kong-system -o wide
# 4. 验证: HTTP 路由依然正常
curl http://100.77.64.95:31850/svc2
curl http://100.105.130.0:31850/svc2
curl http://100.104.150.19:31850/svc2
```

### 阶段 C：目标 2 —— 切换 Local 策略

**文件**：`my-argocd-manifests/argocd-apps/kong-controller-app.yaml`

```bash
# 1. 编辑文件 (在目标1的基础上追加)
vim argocd-apps/kong-controller-app.yaml    # helm.values 加 proxy.externalTrafficPolicy: Local
git add argocd-apps/kong-controller-app.yaml
git commit -m "feat(kong): set externalTrafficPolicy Local"
git push origin main
```

```bash
# 2. 验证: 三个节点入口全部 200
curl http://100.77.64.95:31850/svc2
curl http://100.105.130.0:31850/svc2
curl http://100.104.150.19:31850/svc2
# 3. 确认 Service 策略已生效
kubectl get svc kong-ingress-controller-kong-proxy -n kong-system -o jsonpath="{.spec.externalTrafficPolicy}"
```

### 阶段 D：目标 3 —— 开启 Kong TCP 能力（本次范围）

**文件**：`my-argocd-manifests/argocd-apps/kong-controller-app.yaml`（开 stream）

```bash
# 1. 编辑文件 (在目标2的基础上追加 proxy.stream + nginx_stream_listen)
vim argocd-apps/kong-controller-app.yaml
git add argocd-apps/kong-controller-app.yaml
git commit -m "feat(kong): enable TCP stream proxy on 6379"
git push origin main
```

```bash
# 2. 验证: Kong 的 stream 监听已就绪 (任选一个节点)
kubectl exec -n kong-system deploy/kong-ingress-controller-kong -- ss -tlnp 2>/dev/null | grep 6379
# 或在节点上看: ss -tlnp | grep 6379
# 3. 确认: Kong pod 正常, HTTP 路由不受影响
kubectl get pods -n kong-system
curl http://100.77.64.95:31850/svc2
```

> ✅ 本次到此为止：Kong 已具备 TCP 转发能力（stream 模式开启、端口监听就绪）。
> 没有 TCPIngress 时流量进得来但无路由可去——这是预期状态，不影响任何现有功能。

### 阶段 D2（下次，Redis 接入时再做）

**文件**：`redis-deployment/k8s/redis.yaml`（追加 TCPIngress）或新建 `redis-deployment/k8s/tcpingress.yaml`

```bash
# 1. 编辑文件 (新增 TCPIngress 资源)
cd ~/redis-deployment
vim k8s/tcpingress.yaml    # 或追加到 k8s/redis.yaml
git add k8s/
git commit -m "feat(k8s): add TCPIngress for redis"
git push origin main
```

```bash
# 2. 部署 Redis (如果还没有)
# 3. 验证: Redis 经 Kong TCP 转发可达
redis-cli -h 100.77.64.95 -p 6379 -a <PASSWORD> ping   # PONG
redis-cli -h 100.105.130.0 -p 6379 -a <PASSWORD> ping  # PONG
redis-cli -h 100.104.150.19 -p 6379 -a <PASSWORD> ping # PONG
```

### 阶段 E：回归测试

**验证命令**（无文件改动）：

```bash
# fastapi-svc / quarkus-svc 经三个节点入口全部 200
curl -s -o /dev/null -w "%{http_code}\n" http://100.77.64.95:31850/svc2
curl -s -o /dev/null -w "%{http_code}\n" http://100.105.130.0:31850/svc2
curl -s -o /dev/null -w "%{http_code}\n" http://100.104.150.19:31850/svc2
curl -s -o /dev/null -w "%{http_code}\n" http://100.77.64.95:31850/svc1
curl -s -o /dev/null -w "%{http_code}\n" http://100.105.130.0:31850/svc1
curl -s -o /dev/null -w "%{http_code}\n" http://100.104.150.19:31850/svc1

# TCP 转发 (Redis ping) 经三个节点入口可达
# 集群无异常
kubectl get pods -A | grep -v Completed
```

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

**本次改造（Kong 能力就绪）**：
1. **目标 1**：`kubectl get pods -n kong-system` 每个节点 1 个 controller（DaemonSet 模式，`kubectl get ds -n kong-system` 可见）
2. **目标 2**：`kubectl get svc kong-proxy -o yaml` 的 `externalTrafficPolicy: Local`；三个入口 curl 全部 200
3. **目标 3（本次）**：Kong stream 模式开启，`ss -tlnp | grep 6379` 可见监听；HTTP 路由零回归

**下次（Redis 接入）**：
4. **目标 3（下次）**：TCPIngress 生效，`redis-cli -h <任意节点> -p 6379 ping` 返回 PONG

## 7. 后续讨论（暂不实施）

- **Redis 是否真的走 Kong TCP 转发？** 实验验证后由业务需求决定：
  - 走 Kong：统一入口、统一认证/审计（适合"全能网关"诉求）
  - 直连 hostNetwork：延迟最低（Redis 作为数据面服务，直连 `100.105.130.0:6379` 更简单）
- 生产化时考虑：stream 模式的 TLS 终止、UDP 代理（需 `udpProxy`）、多副本 leader 选举监控

## 8. 参考资料

- 相关排障记录：`oci-k3s-node-kong-lb-flannel-tailscale.md`（同仓库）
- Kong Helm chart values：`proxy.stream` / `controller.affinity` / `proxy.externalTrafficPolicy`
- K3s svclb 机制：DaemonSet 每节点一个，`Local` 策略需每节点有后端
