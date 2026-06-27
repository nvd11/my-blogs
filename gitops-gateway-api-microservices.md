# 基于 GitOps 与 Gateway API 的现代微服务架构实践：从流量路由到自动化交付

## 1. 背景与架构愿景 (Introduction & Architecture Vision)

### 1.1 痛点分析
在传统的微服务交付模式中，开发与运维的边界往往模糊不清。对于多语言、多框架的异构微服务体系（例如同时并存的 Java Quarkus 和 Python FastAPI 服务），手动打包、人工修改 Kubernetes 部署图纸、手工 `kubectl apply` 触发部署不仅效率低下，且极易引入人为错误。此外，如何在这种混合架构下提供一个统一的流量入口，实现一致的路由策略和安全控制，同样是基础设施面临的一大挑战。

### 1.2 目标愿景
为了彻底摆脱“人肉运维”的泥沼，我们的核心愿景是构建一套 **100% 自动化、安全合规、高度解耦** 的云原生交付流水线。当开发者完成业务代码并执行 `git push` 的那一刻起，后续的打包构建、镜像推送、图纸更新以及目标集群的热部署、流量路由接管，都必须由系统无缝自动流转，实现真正的“代码即基础设施 (IaC)”。

### 1.3 核心技术栈选型
为了将上述愿景落地，我们组合了当前云原生领域的领先技术：
- **网关层 (Traffic Routing)**：采用 **Kong KIC** (Kubernetes Ingress Controller) 结合新一代标准的 **K8s Gateway API**。它不仅提供了比传统 Ingress 更强大的表达能力，还实现了基础设施提供者与应用开发者的角色分离。
- **发布层 (Continuous Deployment)**：引入 **ArgoCD** 作为 CD 大脑，践行 GitOps 理念。它持续监听 Git 仓库，确保集群运行状态与 Git 声明式图纸保持绝对一致。
- **交付层 (Continuous Integration)**：利用 **GitHub Actions** 构建轻量级的 CI 流水线，产出 OCI 标准镜像并托管至 **GHCR** (GitHub Container Registry)。

### 1.4 架构解耦：CI 与 CD 的物理隔离
本套架构的一大亮点在于**严格解耦 CI（应用代码库）与 CD（部署图纸库）**。
我们将业务代码（`kong-gitops-experiment`）与 Kubernetes 部署清单（`my-argocd-manifests`）拆分到两个物理隔离的 Git 仓库中。应用研发人员只关注业务逻辑；而所有的环境拓扑、路由规则和版本 Tag 均由 CD 仓库纳管。这种隔离彻底切断了“图纸修改误触发代码构建”的反模式，并为环境变更提供了不可篡改的审计追踪能力。
