# 告别端口尾巴与自签证书：用 Cloudflare + OCI Socat 穿透暴露阿里云 K3s ArgoCD

## 背景与痛点

我的 GitOps 管控集群（`aliyun-k3s`）跑在一台 2C2G（2GB 内存）的阿里云 ECS 上（`8.148.149.80`）。上面跑着完整的 ArgoCD 全家桶（`application-controller`、`repo-server`、`server`、`redis` 等）。

默认情况下，ArgoCD Server 是通过 K3s 的 NodePort 暴露出来的：
* **HTTP 端口**：`31589`（内部强制 307 重定向到 HTTPS）
* **HTTPS 端口**：`30080`（自带自签证书）

日常访问有几个明显的痛点：
1. **必须带端口尾巴**：每次登录都要输入 `https://8.148.149.80:30080`，非常繁琐。
2. **浏览器自签证书警告**：每次打开都会弹红色的“您的连接不是私密连接”，得手动点高级继续。
3. **阿里云节点内存吃紧**：2GB 内存跑完 K3s 控制面和 ArgoCD 后，物理内存已经占用了近 1.6GB（80% 以上）。如果为了暴露 443 端口强行在阿里云上装一套 Kong Ingress Controller 或者 Envoy，常驻内存会飙升 300MB~500MB，遇到突发流量极容易触发 Linux OOM Killer 把 `argocd-application-controller` 干掉。

我手头刚在阿里云买了一个新域名 `jppwl.asia`，并把 DNS 托管到了 Cloudflare。目标很明确：**实现标准 `https://argo.jppwl.asia`（443 端口 + 权威免费 SSL 绿锁）直接访问 ArgoCD，同时不在阿里云节点增加任何重型组件和内存负担。**

---

## 架构选型与踩坑实录

### 尝试方案一：Cloudflare Origin Rules（直接重写回源端口）

Cloudflare 免费版提供了一个叫 **Origin Rules（回源规则）** 的功能，理论上支持在边缘节点接收到 443 请求后，自动将回源目标端口重写为指定的端口（如 `30080`）。

配置看起来非常美好：
1. `argo.jppwl.asia` 解析到阿里云 IP `8.148.149.80`，开启 Cloudflare 代理（小黄云）。
2. 配置 Origin Rule：当 `Hostname == argo.jppwl.asia` 时，回源端口重写至 `30080`。
3. Cloudflare SSL/TLS 模式设置为 `Full`。

#### 踩坑现场：525 SSL Handshake Failed 与中国大陆未备案拦截

规则部署完成后，用 curl 测试：

```bash
curl -s -i "https://argo.jppwl.asia"
```

返回直接报错：
```http
HTTP/2 525 
server: cloudflare
error code: 525 (SSL handshake failed)
```

为什么会 525？我用 Python 写了个脚本，直接模拟不同的 SNI 向阿里云机器的 30080 端口发起 TLS 握手测试：

```python
import ssl, socket

hostname = '8.148.149.80'
port = 30080

for sni in [None, 'localhost', '8.148.149.80', 'argo.jppwl.asia']:
    try:
        ctx = ssl.create_default_context()
        ctx.check_hostname = False
        ctx.verify_mode = ssl.CERT_NONE
        with socket.create_connection((hostname, port), timeout=3) as sock:
            with ctx.wrap_socket(sock, server_hostname=sni) as ssock:
                print(f"SNI={sni} -> SUCCESS: TLS={ssock.version()}")
    except Exception as e:
        print(f"SNI={sni} -> FAILED: {e}")
```

输出真相大白：

```text
SNI=None           -> SUCCESS: TLS=TLSv1.3
SNI=localhost      -> SUCCESS: TLS=TLSv1.3
SNI=8.148.149.80   -> SUCCESS: TLS=TLSv1.3
SNI=argo.jppwl.asia -> FAILED: [Errno 104] Connection reset by peer
```

**原因分析**：
* 阿里云 `8.148.149.80` 属于中国大陆地域（国内机房）。
* 虽然域名 `jppwl.asia` 在阿里云完成了**域名实名认证**，但尚未办理**网站工信部 ICP 备案**。
* Cloudflare 在向源站建立 TLS 连接时，数据包中必须携带客户端请求的 `SNI: argo.jppwl.asia`。
* 阿里云机房的 DPI 硬件防火墙在检测到未备案域名的 TLS Client Hello 时，直接在物理链路层强行发送 TCP RST 重置了连接，导致 Cloudflare 报 525。

也就是说：**只要公网入口直接打在国内大陆 IP 上，未备案域名就无法直接走标准 80/443/NodePort 暴露。**

---

### 方案二：利用 OCI 海外节点作为代理，走 Tailscale 隧道安全穿透

既然国内公网有 ICP 检测，但海外节点没有，而且我的所有节点都在同一个 Tailscale 虚拟局域网中（WireGuard 隧道加密）。

我在 OCI 新加坡有一台闲置的 Always Free 机器 `free-amd-vm2`（`161.118.240.218`，Tailscale IP `100.122.84.84`），它本身就是 `aliyun-k3s` 集群的 Worker 节点。

整体流量拓扑设计如下：

```mermaid
graph LR
    Browser(["💻 浏览器访问<br/>https://argo.jppwl.asia"]) -->|1. 标准 443 + 权威 SSL| CF["Cloudflare 边缘节点<br/>(开启 Universal SSL)"]
    
    CF -->|2. TLS 443 回源<br/>(海外直连，无备案拦截)| OCI["OCI 新加坡 free-amd-vm2<br/>161.118.240.218:443<br/>(socat 进程 · 内存仅 1.1MB)"]
    
    OCI -->|3. Tailscale WireGuard 隧道<br/>100.114.103.101:30080| AliMaster["阿里云 Master 节点<br/>8.148.149.80<br/>ArgoCD Server Pod"]
```

#### 为什么这个方案最合理？
1. **海外节点免备案**：Cloudflare 回源到新加坡 OCI 节点，没有任何拦截，握手秒过。
2. **Tailscale 隧道安全穿透**：OCI 到阿里云走的纯内网 WireGuard 加密流量，国内机房 DPI 只看到 UDP 密文流量，无法窥探内容和域名。
3. **极低资源开销**：在 OCI 节点上用 `socat` 做纯 TCP 层的透明字节流转发，**内存占用仅 1.1MB**，不占 CPU，也不需要配置任何 Nginx/Caddy 证书。
4. **阿里云端零改动**：阿里云节点不需要装任何网关，不消耗额外内存。

---

## 落地实施步骤

### 1. 在 OCI AMD 节点上配置 socat 服务

登录 `free-amd-vm2`（`100.122.84.84`），创建 systemd 服务：

```bash
cat << "EOF" | sudo tee /etc/systemd/system/argocd-socat-proxy.service
[Unit]
Description=ArgoCD Port 443 Forwarder to Aliyun NodePort 30080
After=network.target tailscaled.service
Wants=tailscaled.service

[Service]
Type=simple
ExecStart=/usr/bin/socat TCP-LISTEN:443,fork,reuseaddr TCP:100.114.103.101:30080
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF
```

启动并设置开机自启：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now argocd-socat-proxy.service
```

放行防火墙端口：

```bash
sudo iptables -I INPUT 1 -p tcp --dport 443 -j ACCEPT
```

检查服务运行状态：

```bash
$ sudo systemctl status argocd-socat-proxy.service --no-pager

● argocd-socat-proxy.service - ArgoCD Port 443 Forwarder to Aliyun NodePort 30080
     Loaded: loaded (/etc/systemd/system/argocd-socat-proxy.service; enabled)
     Active: active (running)
     Memory: 1.1M
     CGroup: /system.slice/argocd-socat-proxy.service
             └─2245846 /usr/bin/socat TCP-LISTEN:443,fork,reuseaddr TCP:100.114.103.101:30080
```

---

### 2. 配置 Cloudflare DNS 与 SSL

在 Cloudflare 控制台（或通过 API）完成以下三步：

#### ① 修改 DNS 记录
把 `argo.jppwl.asia` 和 `argocd.jppwl.asia` 指向 OCI 节点的公网 IP，并开启代理（小黄云）：

```text
Type: A
Name: argo
IPv4 address: 161.118.240.218
Proxy status: Proxied (开启小黄云)
```

#### ② 设置 SSL 模式为 Full
进入 **SSL/TLS** 设置：
* 加密模式选择 **Full**（Cloudflare 对外提供自签 Universal SSL 证书，对源站发起 TLS 连接）。
* 因为 `socat` 在 443 端口直接将 TLS 握手流透传给后端的 ArgoCD `30080`，ArgoCD 自带自签名证书，`Full` 模式下 Cloudflare 接受自签证书，握手完美闭环。

---

## 最终验证

全部生效后，在终端发起验证：

```bash
$ curl -s -i "https://argo.jppwl.asia"
```

输出结果：

```http
HTTP/2 200 
date: Sun, 30 Aug 2026 09:47:53 GMT
content-type: text/html; charset=utf-8
content-security-policy: frame-ancestors 'self';
vary: Accept-Encoding
x-frame-options: sameorigin
x-xss-protection: 1
cf-cache-status: DYNAMIC
server: cloudflare

<!doctype html><html lang="en"><head><meta charset="UTF-8"><title>Argo CD</title><base href="/"><meta name="viewport" content="width=device-width,initial-scale=1"><link rel="icon" type="image/png" href="assets/favicon/favicon-32x32.png" sizes="32x32"/><link rel="icon" type="image/png" href="assets/favicon/favicon-16x16.png" sizes="16x16"/><link href="assets/fonts.css" rel="stylesheet"><script defer="defer" src="main.2c48e1df88f1dd48beac.js"></script></head><body><noscript><p>Your browser does not support JavaScript. Please enable JavaScript to view the site. Alternatively, Argo CD can be used with the <a href="https://argoproj.github.io/argo-cd/cli_installation/">Argo CD CLI</a>.</p></noscript><div id="app"></div></body><script defer="defer" src="extensions.js"></script></html>
```

在浏览器直接打开 `https://argo.jppwl.asia`：
* 地址栏显示干净的 `https://argo.jppwl.asia`，没有 `:30080` 后缀。
* 拥有完整的 Cloudflare 权威 SSL 绿色安全锁，零拦截、零自签警告。
* 内存开销：阿里云端 0 额外增加，OCI 端仅消耗 1.1MB。

---

## 总结与经验

1. **域名实名 != 网站备案**：国内云厂商的硬件防火墙针对 80/443/NodePort 的 Host Header 和 TLS SNI 做阻断。如果域名未在工信部备案，直接解析到国内 IP 会被 TCP RST。
2. **多云拓扑的优势在于借力**：遇到国内网络限制时，不需要推翻已有架构，让海外免费节点（OCI）当跳板，配合 Tailscale 私网穿透，可以用极低成本绕过链路限制。
3. **不要滥用重型网关**：对于纯粹的内网管理面板暴露，单行 `socat` TCP 代理在性能和资源占用上（1.1MB 内存）远胜额外拉起一套复杂的 Ingress/Nginx。
