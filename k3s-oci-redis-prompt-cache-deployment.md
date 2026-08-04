# 跨云 K3s 实战：定向部署 Redis 到 OCI 4C24G ARM 节点做 LiteLLM 缓存

在高频调用大语言模型（LLM）的场景下，为了降低 API 费用并提升响应速度，通常会引入 LiteLLM 配合 Redis 实现 **Prompt 缓存（Prompt Caching）** 与响应复用。

由于 LLM 缓存与 Embedding 向量数据需要占用较大内存，且属于需要长期积累的“有状态资产”，如果部署在随时可能被回收的 GCP 临时/抢占式节点上，一旦节点离线，搭建好的缓存就会瞬间蒸发。

本文记录一次真实的跨云架构落地：我们将一台位于 OCI（Oracle Cloud Infrastructure）新加坡区域的 4C24G 永久免费 ARM 节点（`free-arm-vm`）加入现有 K3s 集群，并通过 Kubernetes 的 `nodeSelector` / `nodeAffinity` 与 K3s 原生 `local-path` 持久化存储，将 Redis 服务精准锚定在该节点上，作为 LiteLLM 的高可靠缓存基座。

同时，针对 K3s 多节点集群中的网络转发损耗，本文重点阐述了**直连 OCI VM 的安装思路**，并附上了实测跨节点 LB 流量路径的延迟对比数据。

---

## 1. 整体架构设计与直连 OCI 节点思路

### 1.1 为什么选择“直连 OCI VM”设计思路

在 K3s 集群中，默认的 Service LoadBalancer (`svclb-k3s`) 会以 DaemonSet 形式在所有节点上运行入口 Pod。如果客户端（GCP 上的 LiteLLM）随机选择一个 K3s 节点 IP 作为 Redis 入口，Kubernetes 的 `kube-proxy` / `flannel` 可能会将请求跨网二次转发给真正的 Pod 所在节点。

*   **传统多跳路由灾难**：如果流量从 GCP 发往腾讯云节点（Control Plane），再由 `kube-proxy` 转发给 OCI 节点上的 Redis Pod，数据包将在跨云叠加链路上进行多次折返（GCP ➔ 腾讯云 ➔ OCI ➔ 腾讯云 ➔ GCP），导致延迟成倍暴涨。
*   **直连（Direct Path）思路**：
    1. 通过 `nodeAffinity` 将 Redis Pod 强绑定在 OCI ARM 节点 (`free-arm-vm`)。
    2. 配置 LiteLLM 客户端直接指定 OCI 节点的 Tailscale 内网 IP（`100.105.130.0`）及 NodePort / Direct ClusterIP。
    3. 流量直接由 GCP 通过 Tailscale 加密隧道一跳直达 OCI 节点本地 Pod，将跨网交互限制为**有且仅有 1 次**必要的物理跨云 RTT。

### 1.2 跨云拓扑与流量路径

```mermaid
flowchart TB
    subgraph GCP["☁️ GCP 节点 (London / US)"]
        LiteLLM["LiteLLM Gateway<br/>(Prompt Cache Client)"]
    end

    subgraph TailscaleNet["🔒 Tailscale Mesh Overlay (100.x.x.x)"]
        TSGCP["GCP Tailscale Node<br/>(100.94.13.17)"]
        TSOCI["OCI Tailscale Node<br/>(100.105.130.0)"]
        TSTencent["Tencent Tailscale Node<br/>(100.77.64.95)"]
        TSNUC["NUC Tailscale Node<br/>(100.104.150.19)"]
        
        TSGCP ==>|"✅ 直连路径: 166ms RTT (推荐)"| TSOCI
        TSGCP -.-|"❌ 绕路跳跃: 708ms RTT"| TSTencent
        TSGCP -.-|"❌ 折返跳跃: 324ms RTT"| TSNUC
    end

    subgraph OCI["☁️ OCI 新加坡节点 (free-arm-vm, 4C24G ARM)"]
        K3sAgent["K3s Agent Service"]
        
        subgraph K3sPodSpace["K3s Pod Space"]
            RedisPod["Redis Pod<br/>(Targeted to OCI Node)"]
            RedisSvc["Service: redis-service<br/>(NodePort / ClusterIP: 6379)"]
        end
        
        LocalStorage["Local Path Provisioner<br/>(/var/lib/rancher/k3s/storage/<pvc-id>)"]
    end

    LiteLLM --> TSGCP
    TSOCI --> RedisSvc --> RedisPod
    RedisPod -->|"RDB/AOF 落盘持久化"| LocalStorage
    TSTencent -.->|"Kube-Proxy 跨节点转发"| TSOCI
    TSNUC -.->|"Kube-Proxy 跨节点转发"| TSOCI
```

### 1.3 跨地域网络延迟与收益计算

*   **跨网 RTT**：GCP 与 OCI 新加坡之间的 Tailscale 隧道物理 RTT 稳定在 **165ms ~ 166ms** 左右。
*   **收益对比**：
    *   **Cache Miss（未命中）**：额外增加 166ms 查 Redis 开销，但在 LLM 动辄 2000ms ~ 8000ms 的推理首字延迟面前，166ms 几乎完全感知不到。
    *   **Cache Hit（命中）**：直接返回 Redis 里的 Prompt 结果，耗时仅 166ms，瞬间节省 **2~10 秒**的推理等待时间，并实现 **100% Token 零计费**。
*   **内存与持久化断层优势**：OCI 提供的 24GB 内存为 Redis 提供了充裕的 LRU 缓存空间，搭配 `local-path` 将持久化数据写入 OCI 本地 NVMe/SSD，规避了云盘卷跨云挂载的复杂性。

---

## 2. 三节点 LB 入口跨网延操对比实测

为了验证“直连 OCI VM”与“经由其他 K3s 节点转发”对延迟的具体影响，我们从 GCP 节点（`Alice`）向集群内 3 个节点的 LB / Redis 端口发起实时 TCP 与 Redis RESP 协议 PING 压测，实测数据如下：

### 2.1 延时对比数据汇总

| 流量接入点 (LB Node Entry) | 物理节点类型 | Tailscale 内网 IP | TCP 6379 建连延时 | Redis PING 响应 (RTT) | 相对直连额外损耗 | 链路路由特征 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **直连 OCI 节点 (推荐)** | OCI 4C24G ARM | `100.105.130.0` | **165.79 ms** | **166.34 ms** | **0 ms (基准)** | **单次物理跨云**，本地 Pod 响应，极低抖动 |
| **经由 本地 NUC 节点** | Home NUC Worker | `100.104.150.19` | 218.85 ms | **323.97 ms** | **+157.63 ms** | GCP ➔ 家庭宽带 ➔ Flannel 转发 ➔ OCI |
| **经由 腾讯云 节点** | Tencent Control Plane | `100.77.64.95` | 248.46 ms | **708.14 ms** | **+541.80 ms** | GCP ➔ 腾讯云 ➔ 双重 Overlay 转发 ➔ OCI |

### 2.2 数据分析与结论

1. **直连 OCI 节点 (`166.34 ms`)**：请求到达 OCI 节点后，通过本地 Linux bridge/iptables 直接送达本地 Pod，波动标准差 `< 0.5ms`，是理论上的最低延时界限。
2. **经 NUC 节点转发 (`323.97 ms`)**：产生了二次折返。流量先跨国连入家庭局域网 NUC，再通过集群 CNI 转发到 OCI，叠加了家庭宽带穿透与中继开销。
3. **经 腾讯云 节点转发 (`708.14 ms`)**：延时暴涨近 **540ms**！原因在于跨国公网路由叠加了多重 `kube-proxy` 封包解包开销，如果高频 Prompt 缓存走此入口，缓存收益将被严重侵蚀。

---

## 3. GitOps 配置代码目录分布

为符合基础设施即代码（IaC）与 GitOps 规范，配置文件在代码仓库中按照 Base / Overlay 的结构进行拆分：

```text
k8s-manifests/
└── apps/
    └── redis-llm-cache/
        ├── base/
        │   ├── kustomization.yaml
        │   ├── redis-deployment.yaml
        │   ├── redis-service.yaml
        │   ├── redis-pvc.yaml
        │   └── redis-configmap.yaml
        └── overlays/
            └── production/
                ├── kustomization.yaml
                ├── node-affinity-patch.yaml
                └── resources-patch.yaml
```

*   `base/`：包含 Redis 通用的部署逻辑（ConfigMap、PVC、Deployment、Service）。
*   `overlays/production/`：生产环境配置，负责注入 OCI 节点的精准调度策略 (`nodeAffinity`) 以及生产级内存限额。

---

## 4. 核心配置文件与配置精解

### 4.1 Base 层配置

#### ConfigMap (`base/redis-configmap.yaml`)
为 Redis 定制内存淘汰策略与持久化机制：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: redis-system
data:
  redis.conf: |
    # 绑定所有网卡，接受 Pod 内部与 Tailscale 网格请求
    bind 0.0.0.0
    protected-mode no
    port 6379
    tcp-backlog 511
    timeout 0
    tcp-keepalive 300
    
    # 内存管理：OCI ARM 给 24G 内存，分配 16GB 给 Redis
    maxmemory 16gb
    maxmemory-policy allkeys-lru
    
    # 持久化策略：RDB + AOF 双重保险
    save 900 1
    save 300 10
    save 60 10000
    rdbcompression yes
    dbfilename dump.rdb
    dir /data
    
    appendonly yes
    appendfilename "appendonly.aof"
    appendfsync everysec
    no-appendfsync-on-rewrite yes
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
```

#### PVC (`base/redis-pvc.yaml`)
申请使用 K3s 默认的 `local-path` 存储类：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: redis-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 50Gi
```

#### Deployment (`base/redis-deployment.yaml`)
定义基础 Redis Pod，配置存储挂载与健康检查探针：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
  namespace: redis-system
  labels:
    app.kubernetes.io/name: redis-cache
    app.kubernetes.io/part-of: litellm-infra
spec:
  replicas: 1
  strategy:
    type: Recreate # 确保单节点 local-path 卷平滑卸载与挂载
  selector:
    matchLabels:
      app: redis-cache
  template:
    metadata:
      labels:
        app: redis-cache
    spec:
      containers:
        - name: redis
          image: redis:7.2-alpine
          command:
            - redis-server
            - /usr/local/etc/redis/redis.conf
          ports:
            - containerPort: 6379
              name: redis
          resources:
            requests:
              cpu: "500m"
              memory: "2Gi"
            limits:
              cpu: "3500m"
              memory: "18Gi"
          livenessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 5
            periodSeconds: 5
          volumeMounts:
            - name: redis-data
              mountPath: /data
            - name: redis-config
              mountPath: /usr/local/etc/redis
      volumes:
        - name: redis-data
          persistentVolumeClaim:
            claimName: redis-pvc
        - name: redis-config
          configMap:
            name: redis-config
```

#### Service (`base/redis-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: redis-system
  labels:
    app: redis-cache
spec:
  type: ClusterIP
  ports:
    - port: 6379
      targetPort: 6379
      name: redis
  selector:
    app: redis-cache
```

---

### 4.2 Production Overlay 层配置

#### 节点定向补丁 (`overlays/production/node-affinity-patch.yaml`)
通过 `nodeAffinity` 结合自定义 Node Label，硬性约束 Redis Pod 只能调度至 OCI 节点 `free-arm-vm`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
  namespace: redis-system
spec:
  template:
    spec:
      # 使用 nodeAffinity 进行节点精准锚定
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                      - free-arm-vm
                  - key: topology.kubernetes.io/node-type
                    operator: In
                    values:
                      - oci-arm
      # 针对 OCI ARM 节点的架构兼容约束
      tolerations:
        - key: "arch"
          operator: "Equal"
          value: "arm64"
          effect: "NoSchedule"
```

#### Overlay Kustomization (`overlays/production/kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: redis-system

resources:
  - ../../base

patches:
  - path: node-affinity-patch.yaml
```

---

## 5. 部署落地与实测验证

### Step 1: 给 OCI Node 标记特征标签

在集群 Control Plane 执行命令，打上标记：

```bash
# 检查现有节点状态
kubectl get nodes -o wide

# 为 OCI 节点添加节点类型标签
kubectl label node free-arm-vm topology.kubernetes.io/node-type=oci-arm --overwrite

# 验证标签应用情况
kubectl get node free-arm-vm --show-labels
```

### Step 2: 应用 Kustomize 配置

```bash
# 创建命名空间
kubectl create namespace redis-system --dry-run=client -o yaml | kubectl apply -f -

# 部署生产配置
kubectl apply -k k8s-manifests/apps/redis-llm-cache/overlays/production
```

### Step 3: 验证 Pod 调度与存储绑定

检查 Pod 是否准确落盘在 OCI 节点，并且 PVC 成功绑定：

```bash
# 查看 Pod 所在节点
kubectl get pods -n redis-system -o wide
# 输出预期包含: NODE: free-arm-vm

# 检查 PV 与 Local-Path 绑定情况
kubectl get pvc -n redis-system
kubectl describe pv -n redis-system
```

在 OCI 节点物理机上直接核实持久化路径：

```bash
# 登录 OCI 节点查看 local-path 创建的物理目录
ssh free-arm-vm "ls -la /var/lib/rancher/k3s/storage/pvc-*"
```

### Step 4: 跨网络读写与性能测试

在 GCP 节点的 LiteLLM 所在环境进行跨节点 Redis 读写测试：

```bash
# 获取 Redis Service ClusterIP 或在网格内直接测试
REDIS_IP=$(kubectl get svc redis-service -n redis-system -o jsonpath='{.spec.clusterIP}')

# 写入大型测试 Key
kubectl run redis-test --rm -i --tty --image=redis:7.2-alpine -- redis-cli -h $REDIS_IP PING

# 模拟 LiteLLM Prompt Cache 写入与读取
kubectl run redis-test --rm -i --tty --image=redis:7.2-alpine -- redis-cli -h $REDIS_IP SET litellm:cache:prompt_hash_001 "response_data_chunk_sample"
kubectl run redis-test --rm -i --tty --image=redis:7.2-alpine -- redis-cli -h $REDIS_IP GET litellm:cache:prompt_hash_001
```

---

## 6. 运维踩坑与性能调优总结

1. **内存使用防 OOM 机制**：
   * OCI 机器虽然有 24G 内存，但必须为 Linux OS、K3s Agent 以及 Tailscale 进程预留 4G~6G 缓冲区。
   * Redis `maxmemory` 设为 `16gb`，同时设置容器 `limits.memory: 18Gi`，留出 2G 缓冲，防止 Redis 内存暴涨触发 Linux kernel 的 OOM Killer 杀死 `k3s-agent`。

2. **`local-path` 与 Pod 调度强绑定**：
   * `local-path` 属于 Local Volume，数据的物理路径绑定在 `free-arm-vm` 本地。
   * 必须在 Deployment 中配置 `strategy.type: Recreate`。如果是 RollingUpdate，K8s 会试图先启动新 Pod 再关旧 Pod，导致新旧 Pod 抢占同一个本地文件锁或引发调度挂起。

3. **AOF Rewrite 的 CPU 抖动防护**：
   * 在 ARM 节点上，AOF 重写（`BGREWRITEAOF`）会 fork 子进程并短暂消耗 CPU。设置 `no-appendfsync-on-rewrite yes` 可以避免在重写期间阻塞 Redis 主线程，保障 LiteLLM 高并发请求下的低延迟响应。
