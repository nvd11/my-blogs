# 基于 ArgoCD 优雅落地 K8s Gateway API 与 Kong 控制器

在现代云原生微服务架构中，Kubernetes Gateway API 正在逐步取代传统的 Ingress 成为新一代的标准流量入口。如何通过 GitOps（ArgoCD）优雅、解耦地安装和管理 Gateway API 的底层资源、Kong 控制器以及网关基础设施？本文将以真实的工程实践为例，详细拆解整个配置过程。

## 一、 架构设计思路：控制面与数据面配置解耦

在我们的配置仓库中，采用了纯正的 **App of Apps** 模式，并将配置文件严格划分为两个不同的目录，实现了调度层与基建实体的解耦：

1. **`argocd-apps/`（调度指挥部）**：存放所有的 `kind: Application` 文件。这些文件相当于“快递单”，它们负责告诉 ArgoCD 去哪里拉取配置（比如远端 Helm 仓库或其他 Git 目录），并空投到哪个目标集群。
2. **`infrastructure/kong-gateway/`（基建实体）**：存放目标集群真正需要落地的纯 K8s 资源清单（比如 `GatewayClass`、`Gateway`）。

### 1.1 全景架构图

```mermaid
graph TD
    subgraph GitRepo ["Git 仓库 (my-argocd-manifests)"]
        direction TB
        App["argocd-apps/ <br/> (ArgoCD 快递单)"]
        Infra["infrastructure/kong-gateway/ <br/> (K8s 基建实体)"]
    end

    subgraph ControlPlane ["控制面: 阿里云集群"]
        ArgoCD["ArgoCD 控制器"]
        App1["kong-controller-app"]
        App2["kong-infra-app"]
        ArgoCD -->|读取并同步| App1
        ArgoCD -->|读取并同步| App2
    end

    subgraph DataPlane ["数据面: 腾讯云集群"]
        KIC["Kong 控制器程序 <br/> (konghq.com/kic-gateway-controller)"]
        GC["GatewayClass <br/> (name: kong)"]
        GW["Gateway <br/> (name: kong-main-gateway)"]
    end

    App -.->|Git 同步| ArgoCD
    
    App1 ==>|派送: Helm 安装| KIC
    App2 ==>|派送: 投递 YAML| Infra
    Infra -.->|落地为| GC
    Infra -.->|落地为| GW
    
    GC -.->|认领大门| KIC
    GW -.->|关联类别| GC
```

## 二、 核心部署流程解析

部署整个网关体系存在严格的先后依赖关系：必须先有 CRD（字典），才能启动控制器（解析程序），最后才能创建网关实体。我们通过 ArgoCD 的 **Sync Waves（同步波次）** 完美解决了这个问题。

### 步骤 1：安装 Gateway API CRD (Wave 1)

首先，我们需要在集群中注册 Gateway API 的官方 CRD（如 `GatewayClass`, `Gateway`, `HTTPRoute` 等）。

创建 `argocd-apps/gateway-api-crds-app.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-api-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1" # 🎯 波次 1：最底层的基础设施字典
spec:
  project: default
  source:
    repoURL: 'https://github.com/kubernetes-sigs/gateway-api.git'
    targetRevision: v1.1.0
    path: config/crd/standard
  destination:
    name: 'tencent-dp1-cluster'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true # CRD 往往很大，必须使用 ServerSideApply 避免报错
```

### 步骤 2：安装 Kong Ingress Controller (KIC) (Wave 2)

CRD 就绪后，接下来安装 Kong 的控制器程序（KIC）。我们直接引用 Kong 官方的 Helm Chart。

创建 `argocd-apps/kong-controller-app.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kong-ingress-controller
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2" # 🎯 波次 2：等 CRD 装好后再安装控制器
spec:
  project: default
  source:
    repoURL: 'https://charts.konghq.com'
    chart: kong
    targetRevision: 2.38.0
    helm:
      values: |
        ingressController:
          installCRDs: false # 防止模板重复安装 CRD，产生冲突
        env:
          database: "off"
        gateway:
          enabled: true      # 核心：开启 Kong 对 Gateway API 的监听能力
  destination:
    name: 'tencent-dp1-cluster'
    namespace: kong-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

> **避坑指南**：我们在这里设置了 `ingressController.installCRDs: false`，目的是阻止 Helm 在模板渲染阶段重复处理 CRD（容易导致 ArgoCD sync 失败）。至于 Kong 自身的 CRD（如 `KongPlugin`），由于 Helm 自身的机制，会在第一次部署时通过 chart 内的 `crds/` 目录自动安装。

### 步骤 3：定义网关基础设施 (GatewayClass & Gateway)

有了控制器后，我们需要在集群中真正开辟一扇“大门”。这些属于基础设施层的实体资源，存放在 `infrastructure/kong-gateway/` 目录下。

**1. 定义 GatewayClass** (`infrastructure/kong-gateway/GatewayClass.yaml`)：
这相当于在集群中注册一个网关类别，并与刚才安装的 KIC 进行“暗号配对”。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: kong
  annotations:
    konghq.com/gatewayclass-unmanaged: "true" # 告诉 Kong 这是一个 Unmanaged 模式
spec:
  # 这是一句关键的配对声明：所有 kong 类别的大门，都交给这个名字的控制器处理。
  # Kong KIC 程序内部硬编码了自己就叫这个名字。
  controllerName: konghq.com/kic-gateway-controller 
```

**2. 定义 Gateway** (`infrastructure/kong-gateway/Gateway.yaml`)：
真正承接流量的入口。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kong-main-gateway
  namespace: default
spec:
  gatewayClassName: kong # 引用上方的 GatewayClass
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same # 允许同命名空间下的业务应用创建 HTTPRoute 来绑定
```

**3. 基础设施的 ArgoCD 调度器** (`argocd-apps/kong-infra-app.yaml`)：
最后，通过一个 Application 将上述基础设施资源空投到目标集群。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kong-gateway-infra
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/nvd11/my-argocd-manifests.git'
    targetRevision: HEAD
    path: infrastructure/kong-gateway
  destination:
    name: 'tencent-dp1-cluster'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 三、 总结

通过以上配置，我们实现了一个极度优雅的 GitOps 闭环：
1. **职责单一**：`argocd-apps` 只负责派发指令，`infrastructure` 只负责落地实体。
2. **时序可控**：Sync Waves 确保了 `CRD -> 控制器 -> 自定义资源` 的正确启动顺序。
3. **架构解耦**：利用标准的 Gateway API 和 `GatewayClass` 机制，业务应用（创建 `HTTPRoute` 时）只需认准 `kong-main-gateway`，完全不需要关心底层跑的是 Kong、Envoy 还是 Nginx。
