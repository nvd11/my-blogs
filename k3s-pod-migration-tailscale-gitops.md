# 跨云与 Homelab 混合集群实战：基于 Tailscale 与 GitOps 实现 K3s Pod 零停机漂移

在多云与混合云架构中，经常会遇到云端 VPS 资源受限（如 2C2G 节点内存紧俏），而本地 Homelab 拥有空闲高性能算力（如 16G 内存的 Intel NUC）的情况。

本文记录一次真实的混合集群改造实践：将一台位于家庭局域网内的 NUC 主机通过 Tailscale 跨网连入腾讯云上的 K3s 集群作为 Worker 节点，并利用 GitOps 流水线将 Java (Quarkus) 微服务容器从云端平滑迁移至本地 NUC，同时保持云端 Kong API 网关的正常路由。

---

## 1. 架构设计与网络打通

### 1.1 Tailscale + Flannel 跨网穿透
云端 VPS 与本地 Homelab 之间没有固定公网 IP，直接暴露端口存在安全隐患。我们采用 Tailscale 组建 Mesh 虚拟局域网，将所有通信收拢在加密通道内。

- **控制面 (Master)**：腾讯云 Debian 2C2G，Tailscale IP 为 `100.77.64.95`
- **工作节点 (Worker)**：本地 NUC Ubuntu 24.04，Tailscale IP 为 `100.104.150.19`

在 K3s 层面，核心是将容器 Overlay 网络（Flannel VXLAN）绑定到 Tailscale 网卡上。

**Master 节点配置 (`/etc/systemd/system/k3s.service`)**：
```bash
ExecStart=/usr/local/bin/k3s \
    server \
    --tls-san 43.139.214.231 \
    --tls-san 100.77.64.95 \
    --flannel-iface tailscale0 \
    --node-ip 100.77.64.95
```

**NUC 节点接入**：
```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  sudo INSTALL_K3S_MIRROR=cn \
  K3S_URL="https://100.77.64.95:6443" \
  K3S_TOKEN="<NODE_TOKEN>" \
  sh -s - agent --flannel-iface=tailscale0 --node-ip=100.104.150.19
```

### 1.2 MTU 与国内镜像源避坑
1. **Flannel MTU 自动适配**：Tailscale 网卡 MTU 为 1280，Flannel 扣除 50 字节 VXLAN 报头后，自动将容器网络 `flannel.1` 的 MTU 调整为 1230，避免了跨网传输包分片导致的 TCP 连接挂起。
2. **容器镜像加速配置**：在 NUC 的 `/etc/rancher/k3s/registries.yaml` 中配置国内镜像源（如 DaoCloud），解决国内网络环境下拉取 Docker Hub 及 klipper-lb 等基础镜像超时（`i/o timeout`）的问题：

```yaml
mirrors:
  "docker.io":
    endpoint:
      - "https://docker.m.daocloud.io"
  "registry.k8s.io":
    endpoint:
      - "https://k8s.m.daocloud.io"
```

---

## 2. 基于 GitOps 的无感知 Pod 漂移

根据“CI 与代码同行，CD 与代码隔离”原则，迁移 Pod 物理节点**不需要修改任何业务代码**，仅需在基础设施模具与 CD 配置库中申明。

```
+--------------------------+        +---------------------------+
| my-shared-helm-charts    |        | my-argocd-manifests       |
| (通用 Helm 模具)          |        | (ArgoCD 部署图纸)         |
| 新增 nodeSelector 语法   |        | 注入 nodeSelector 指定 nuc|
+------------+-------------+        +-------------+-------------+
             |                                    |
             +------------------+-----------------+
                                |
                                v
                   +--------------------------+
                   | ArgoCD (Aliyun CP1)      |
                   | 自动监听 Git 变更并 Sync |
                   +------------+-------------+
                                |
                                v
                   +--------------------------+
                   | K3s Cluster (Tencent DP1)|
                   | 销毁旧 Pod，在 NUC 启动  |
                   +--------------------------+
```

### 2.1 修改通用 Helm 模板库 (`my-shared-helm-charts`)
在通用的 `deployment.yaml` 模版中，追加 `nodeSelector` 的渲染逻辑：

```yaml
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

打上 Git Tag `v1.1.3` 并推送到 GitHub。

### 2.2 修改 CD 部署申明库 (`my-argocd-manifests`)
在应用定义文件 `argocd-apps/quarkus-svc-app.yaml` 中，将 Helm 模具版本提升至 `v1.1.3`，并在 `values` 中注入节点选择规则：

```yaml
source:
  repoURL: 'https://github.com/nvd11/my-shared-helm-charts.git'
  path: charts/generic-web-service
  targetRevision: 'v1.1.3'
  helm:
    values: |
      replicaCount: 1
      containerPort: 8080
      nodeSelector:
        kubernetes.io/hostname: "nuc"
```

### 2.3 零停机滚动更新流程
代码提交后，阿里云上的 ArgoCD 控制面自动拉取图纸更新，驱动腾讯云 K3s 执行 Pod 漂移：

1. K8s 在 NUC 节点拉起新的 `quarkus-svc` Pod。
2. 就绪探针（Readiness Probe）检测通过后，K3s 将 EndpointSlice 自动更新为 NUC 上的 Pod IP (`10.42.1.x`)。
3. 运行在腾讯云 Master 上的 Kong Ingress Controller 实时感知 Endpoint 变化，将上游路由指向新 Pod。
4. 优雅下线并销毁运行在云端 VPS 上的旧 Pod。

---

## 3. 效果验证与性能测量

完成迁移后，查看节点与 Pod 状态：

```bash
$ kubectl get nodes -o wide
NAME            STATUS   ROLES           AGE   INTERNAL-IP      OS-IMAGE
nuc             Ready    worker          10m   100.104.150.19   Ubuntu 24.04.4 LTS
vm-0-2-debian   Ready    control-plane   57d   100.77.64.95     Debian GNU/Linux 12

$ kubectl get pods -l app=quarkus-svc -o wide
NAME                           READY   STATUS    IP           NODE
quarkus-svc-69bf4fcdfc-sddgw   1/1     Running   10.42.1.13   nuc
```

对云端 Kong 网关入口发起实测：

```bash
$ curl -i http://43.139.214.231/svc1
HTTP/1.1 200 OK
Via: kong/3.6.1
X-Kong-Upstream-Latency: 9
X-Kong-Proxy-Latency: 0

ok
```

### 延迟对比与分析

- **网络 RTT**：腾讯云 VPS 与本地 NUC 之间的 Tailscale P2P 直连延迟为 **6.45 ms**。
- **网关代理开销 (`X-Kong-Proxy-Latency`)**：Kong 运行在云端 VPS，匹配路由与代理计算耗时维持在 **0 ~ 1 ms**。
- **网关上游处理延迟 (`X-Kong-Upstream-Latency`)**：相比纯云端同节点运行时的 **5 ms**，跨网漂移到 NUC 后增加至 **9 ~ 10 ms**。

增加的 4~5 ms 延时主要来自跨公网的 Tailscale 传输开销。对于非极高频交易类业务，毫秒级增加基本不可感知，却成功释放了云端 VPS 珍贵的内存资源，实现了 Homelab 算力的无缝拓展。
