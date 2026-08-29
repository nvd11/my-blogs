# 深度拆解 Kubernetes 网关核心：KIC 与 Kong 的本质边界、分布式自治控制面权衡与 Gateway API 解耦实战

在把业务集群推向跨云与混合云（腾讯云 + OCI + 本地 NUC 家宽）的过程中，API 网关是整套系统流量进出的咽喉要道。最近我们在深入排查网关路由与梳理 ArgoCD GitOps 拓扑时，集中探讨了几个在云原生网关演进中非常容易混淆、但直接决定系统稳定性与网络延迟的核心问题：

1. **名字套娃背后的真相**：`kong-ingress-controller-kong` 到底谁是谁？KIC 和 Kong 核心究竟是什么关系？
2. **多控制面之争**：为什么我们的 KIC（控制面）有 3 个实例，还非要和 Kong 数据面同居在一个 Pod 里？“单控制面 + 多数据面”在异构跨云环境下为什么反而容易踩坑？
3. **Gateway API 的职责解耦**：`kong-gateway-infra` 与 `kong-ingress-controller` 到底谁管开端口，谁管挂路由？
4. **网关鉴权选型**：在开源版 Kong 缺乏商业 OIDC 插件的情况下，为什么自写 Lua 插件和 `OAuth2-Proxy` 各有其战场？

本文将这几次关键的架构推演与实战结论完整沉淀下来，作为混合云网关设计的底层参考。

---

## 一、 KIC 与 Kong 的本质边界与 Sidecar 协同

很多人在初接触 Kong Ingress Controller 时，容易把 KIC 和 Kong 混为一谈。在我们的集群里，ArgoCD 上甚至出现了一个看似套娃的 Pod 名字：`kong-ingress-controller-kong-xxxxx`。

### 1. 概念解耦：翻译官（Go） vs 守门人（Nginx/Lua）

* **Kong（数据面 / Data Plane）**：
  本质是一个基于 OpenResty（Nginx + LuaJIT）的高性能反向代理引擎。它**完全不认识任何 Kubernetes 概念**（不知道什么是 `HTTPRoute`、`Service`、`Namespace`）。它的工作非常纯粹：监听网络端口（HTTP 8000、HTTPS 8443、TCP Stream 6379），执行内存中的插件，并以微秒级的速度转发物理流量。它通过本地环回接口暴露了一个 Admin API（`http://127.0.0.1:8444`）。
* **KIC（控制面 / Control Plane）**：
  是一个专门用 Go 编写的 Kubernetes 控制器。它的唯一职责是**充当 Kubernetes API 与 Kong 之间的“实时翻译官”**。KIC 通过长连接 Watch 集群里的 K8s 资源变更（Gateway、HTTPRoute、Service、Secret、KongPlugin），一旦发现开发者提交了新配置，它就把这些声明式 YAML 翻译成 Kong 理解的 JSON 格式（Routes、Services、Plugins、Upstreams），再调用 Kong 的 Admin API 下发。

```
           [ Kubernetes API Server ]
         (HTTPRoute / Service / Secret)
                    │
                    ▼ Watch 变更
    ┌────────────────────────────────────────┐
    │ 🧠 KIC (Kong Ingress Controller)       │ ➡️ 【控制面】(Go 编写)
    │    负责把 K8s 资源转译为 Kong JSON 规则  │
    └──────────────────┬─────────────────────┘
                       │
                       │ 走 127.0.0.1:8444 本地环回调用 Admin API
                       ▼
    ┌────────────────────────────────────────┐
    │ 💪 Kong (Proxy 核心引擎)               │ ➡️ 【数据面】(Nginx + LuaJIT)
    │    负责接收真实流量、执行插件并高速转发   │
    └────────────────────────────────────────┘
                       │
                       ▼ 真实物理流量发往后端 Pod (Quarkus / LiteLLM / DbGate)
```

### 2. 套娃名字的由来与 Pod 内部结构

`kong-ingress-controller-kong` 并不是单选题，它是通过 Helm 命名公式（`ReleaseName + ChartName`）拼接出的 Pod 名称：
- Release Name: `kong-ingress-controller`
- Chart Name: `kong`

在我们的 DaemonSet 架构中，进入任意节点查看 Pod 的内部容器：
```bash
$ kubectl get pods -n kong-system kong-ingress-controller-kong-vrnrf -o jsonpath='{range .spec.containers[*]}{.name}{": "}{.image}{"\n"}{end}'
ingress-controller: kong/kubernetes-ingress-controller:3.1
proxy: kong:3.6
```

**结论**：每个 Pod 内部都是经典的 **Sidecar 双容器紧密同居**——`proxy` 容器就是 Kong 的核心数据面，`ingress-controller` 就是 KIC 控制面。它们共享同一个网络命名空间（`127.0.0.1`），KIC 无论多高频地下发路由，走的都是内存级的 Localhost 环回，完全没有跨网络的 RPC 开销。

---

## 二、 控制面架构之争：全自治 DaemonSet vs 单控制面解耦

在传统的企业内网架构中，通常提倡“控制面与数据面解耦”：即只部署 1 个独立的 KIC 控制面（Deployment），而在各个工作节点上只部署纯净的 Kong Proxy 数据面（DaemonSet）。

但在我们这种**公有云（腾讯云）+ 海外算力（OCI 新加坡）+ 家宽边缘（广州移动 NUC）**的异构跨广域网拓扑下，我们为什么坚持选择了**全自治的 DaemonSet 模式（每个节点都是独立的 KIC + Kong 双容器）**？

### 1. 两种架构形态对比

```
【方案 A：当前全自治 DaemonSet】          【方案 B：传统单控制面解耦】

       [ K8s API Server ]                      [ K8s API Server ]
      ┌────────┴────────┐                              │
Watch │                 │ Watch                        ▼
      ▼                 ▼                     [ 集中式 KIC 实例 ]
┌───────────┐     ┌───────────┐                        │
│ 节点 A     │     │ 节点 B     │           跨广域网 Admin API / mTLS
│ KIC+Kong  │     │ KIC+Kong  │            ┌───────────┴───────────┐
└───────────┘     └───────────┘            ▼                       ▼
 (每个节点自给自足，零外部依赖)          ┌───────────┐           ┌───────────┐
                                        │ 节点 A     │           │ 节点 B     │
                                        │ 纯 Proxy  │           │ 纯 Proxy  │
                                        └───────────┘           └───────────┘
```

### 2. 深度权衡与实际网络收益

| 评估维度 | 方案 A：全自治 DaemonSet (当前) | 方案 B：单控制面 + 各节点数据面 |
| :--- | :--- | :--- |
| **K8s API Server 压力** | 每个节点 1 个 Watch 连接（节点少时完全无感） | 仅 1 个 Watch 连接，节点数超 50+ 时有优势 |
| **跨网络通信依赖** | **零依赖**（配置下发在本地 127.0.0.1 完成） | **强依赖跨云长连接**（需暴露并维护 Admin API / mTLS） |
| **家宽网络抖动容忍** | **极高**（家宽断网，本地网关自洽运行） | **较弱**（网络丢包时导致配置下发重试或状态不一致） |
| **算力就近闭环** | **本地直接路由**（如 OCI 节点直接消化 LiteLLM） | 路由生效延迟依赖跨云下发速度 |

### 3. 杀手级收益：海外流量就地闭环，绝对避免跨国绕路

在我们集群中，`litellm-svc` 部署在 OCI 新加坡节点（`free-arm-vm`）上。
当海外客户端（如 GCP 或海外调用方）直接访问 OCI 节点的公网/Tailscale IP（`100.105.130.0`）时：

1. 流量打入 OCI 节点，被本地 `svclb` 结合 `externalTrafficPolicy: Local` 直接送入 **OCI 本地的 Kong Proxy**；
2. OCI 本地的 KIC 早就通过 K8s API 在本地转译好了全量路由，识别到 `/litellm` 的后端 Pod（`10.42.2.62`）就在本机的容器网络中；
3. **整个请求在 OCI 物理机内部完成 100% 内存级转发闭环，往返耗时仅 ~166ms，国内腾讯云节点连 1 个字节的流量都不会经过！**

如果采用单控制面中转或集中式网关，海外客户端的请求将不得不跨大半个地球先打进国内腾讯云 IDC（~477ms），再由腾讯云跨海转发到 OCI，不仅网络延迟暴增到 600ms+，还平白消耗国内云服务器珍贵的出网带宽。

因此，**全自治 DaemonSet 用每个节点多出的几十兆内存，换取了跨网络环境下最顶级的自愈能力与零绕路性能**。

---

## 三、 Gateway API 时代的分层解耦：Infra vs Controller vs Route

在我们的 `my-argocd-manifests` 仓库中，关于网关拆分出了两个独立的应用：`kong-ingress-controller` 和 `kong-gateway-infra`。这两者的职责边界非常具有代表性。

```
1️⃣ 【kong-ingress-controller】(Helm Chart 应用)
    └── 运行 Kong 软件本体与后台 DaemonSet（打通物理 80/443/6379 端口）
              │
              ▼ 接管并驱动
2️⃣ 【kong-gateway-infra】(Gateway.yaml / GatewayClass.yaml)
    └── 声明公共逻辑大门: Gateway (kong-main-gateway) 监听 :80
              ▲
              │ 声明绑定 (parentRef: kong-main-gateway)
              ├────────────────────────┬────────────────────────┐
              │                        │                        │
3️⃣ 【quarkus-svc-route】      3️⃣ 【litellm-svc-route】    3️⃣ 【dbgate-route】
    (路径: /svc1)                (路径: /litellm)            (路径: /dbgate)
```

### 1. 物理开端口 vs 逻辑挂门牌

* **`kong-ingress-controller`（物理层）**：
  负责安装 DaemonSet 和 Service。它在宿主机和 iptables 层面开辟物理通路（绑定 80、443、NodePort 31850 和 TCP Stream 6379）。
* **`kong-gateway-infra`（逻辑层）**：
  声明 `GatewayClass: kong` 和 `Gateway: kong-main-gateway`。它负责定义这扇门叫什么、属于什么协议（HTTP/TCP）、允许哪些命名空间（`allowedRoutes.namespaces.from: All`）的应用来挂载路由。
* **业务服务（应用层）**：
  各个微服务（Quarkus、FastAPI、DbGate）编写自己的 `HTTPRoute`，通过 `parentRef: kong-main-gateway` 认领大门，声明自己的匹配路径（如 `/dbgate`）和安全插件。

这种拆分使得基础设施运维人员与业务微服务开发人员的权限完全隔离，业务方上线服务只需要关注自己的 `HTTPRoute`，无需触碰网关底层配置。

---

## 四、 网关鉴权实践：开源版 OIDC 的现实与最佳解法

在为新部署的数据库管理服务（DbGate）配置安全入口时，公网暴露必须增加身份认证。我们评估了自写 Lua 插件与云原生方案的优劣。

### 1. 开源版 Kong 的插件现状

实地进入 `kong:3.6.1` 容器目录 `/usr/local/share/lua/5.1/kong/plugins` 确认：
- 官方内置了 `basic-auth`、`key-auth`、`jwt`、`hmac-auth`、`ldap-auth` 等基础鉴权插件。
- 官方的 **`openid-connect` (OIDC) 插件是企业收费版（Kong Enterprise）独占的**，开源版镜像中未包含。

### 2. 为什么对接 Auth0 不建议自写 Lua 插件？

写一个检查自定义 Header 的 Lua 插件非常简单（20 行代码即可），但 **OAuth2 / OIDC 网页登录是一个庞大且精密的安全状态机**：
1. 生成加密 state / nonce 并防止 CSRF 攻击；
2. 拦截未登录请求并执行 302 授权码重定向；
3. 处理 `/callback` 路由，异步请求 Auth0 换取 Token；
4. 拉取 Auth0 JWKS 公钥并完成 RS256 签名校验与轮换缓存；
5. 加密 Session 并写入 HttpOnly Cookie。

自己手写 Lua 实现上述流程需要维护数千行底层密码学与 HTTP 逻辑，极易引入安全漏洞。

### 3. 最佳实践推荐

* **对接 Auth0 / Google SSO 网页单点登录**：
  采用 CNCF 开源的 **`OAuth2-Proxy`** 容器配合 Kong（或作为 Ingress 鉴权中间件）。Go 语言静态编译，常驻内存仅 ~20MB，一行代码不用写即可获得完整的企业级 SSO 登录体验。
* **无状态 API 接口调用**：
  直接使用 Kong 自带的开源 **`jwt`** 插件，配置 Auth0 的公钥证书，由 Kong 在网关层无状态秒级验签。
* **私有定制化安全逻辑**：
  利用 Kong 提供的 `kong.log`、`schema.lua` 和 `handler.lua` 编写轻量 Lua 插件，通过 K8s `ConfigMap` 声明式挂载，实现免编译容器镜像的热插拔扩展。

---

## 五、 总结

通过对 KIC 与 Kong 的底层解耦、异构多节点全自治架构的落地、以及 Gateway API 的分层实践，我们构建了一套在跨云与家宽混合环境下具备**高容灾、零绕路、声明式自愈**的现代网关基础设施。

在复杂网络拓扑下，不要盲目套用同机房的集中式控制面假设；让数据面与控制面在边缘自包含、自自治，往往是应对广域网不确定性最坚固的工程选择。
