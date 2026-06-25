# 🌐 NAS IPv6 → 办公室 IPv4 访问方案  
Lucky + Cloudflare 完整图文教程

### 📋 场景描述

  * **NAS** ：飞牛 OS，只有 IPv6 公网地址
  * **办公室** ：只能访问 IPv4，无法直接访问 IPv6
  * **需求** ：在办公室通过域名访问家里的 NAS 服务



### 🚫 硬性约束

  * ❌ 办公室电脑不安装任何软件（无 VPN 客户端、无代理、无插件）
  * ✅ 办公室只用浏览器输入域名就能访问 NAS
  * ✅ 所有配置都在 NAS 端完成



## 💡 核心原理：Cloudflare 充当 IPv4↔IPv6 桥梁
    
    
    ┌──────────────┐         ┌─────────────────┐         ┌──────────────┐
    │  办公室电脑   │ ─IPv4→  │  Cloudflare CDN  │ ─IPv6→  │  家里的 NAS   │
    │  (只有 IPv4)  │         │  (双栈: v4+v6)   │         │  (只有 IPv6)  │
    └──────────────┘         └─────────────────┘         └──────────────┘
                                      ↑
                               Lucky 定时更新
                               AAAA 记录 (DDNS)
      

**Cloudflare 的 CDN 节点同时支持 IPv4 和 IPv6** ，当你开启代理（橙色云朵）后：

  1. 办公室用 IPv4 访问 `nas.example.com` → DNS 解析到 Cloudflare 的 IPv4 节点
  2. Cloudflare 节点用 IPv6 连接到你 NAS 的真实 IPv6 地址
  3. 数据在 Cloudflare 节点中转 → 完美桥接 IPv4 和 IPv6



## 📊 三种方案对比

方案| 原理| 难度| 稳定性| 端口限制| 推荐  
---|---|---|---|---|---  
Cloudflare DNS 代理 | AAAA 记录 + 橙色云朵，Cloudflare 自动 v4↔v6 桥接 | ⭐⭐ 简单 | ⭐⭐⭐⭐⭐ | 仅 HTTP/HTTPS 端口 | ⭐ 首选  
Cloudflare Tunnel | NAS 安装 cloudflared，建出站隧道 | ⭐⭐⭐ 中等 | ⭐⭐⭐⭐⭐ | 支持任意 TCP | 备选  
frp / 内网穿透 | 需一台 IPv4 公网 VPS 中转 | ⭐⭐⭐⭐ | ⭐⭐⭐ | 灵活 | 成本高  
  
## 🔬 各方案深度解析：原理、优缺点

### 方案 A：Cloudflare DNS 代理 + Lucky DDNS

#### 🧠 实现原理
    
    
    Step 1  Lucky 定时检测 NAS 的 IPv6 地址，通过 Cloudflare API 更新 AAAA 记录
    Step 2  DNS 记录开启 Proxy（橙色云朵），Cloudflare 不返回真实 IPv6，返回自己的 IPv4 地址
    Step 3  办公室浏览器 → DNS 查 nas.example.com → 得到 104.21.xx.xx (Cloudflare IPv4)
    Step 4  浏览器 → TLS → 104.21.xx.xx:8443 → Cloudflare 边缘节点
    Step 5  Cloudflare 边缘 → IPv6 → [你的 NAS IPv6]:8443 → Lucky 反向代理
    Step 6  Lucky → 127.0.0.1:5666 → 飞牛 OS
    

**关键点：** Cloudflare 的 Anycast 网络在全球每个节点都是双栈的。它对外用 IPv4 跟你的办公室通信，对内用 IPv6 跟你的 NAS 通信。橙色云朵就是告诉 Cloudflare "帮我做这个翻译"。

#### ✅ 优点

  * **配置极简：** Lucky 填几个参数即可，不用在 NAS 上装额外软件（Lucky 你本来就有）
  * **零成本：** Cloudflare 免费计划完全够用，不需要买 VPS
  * **自动 CDN 加速：** 静态资源会被缓存在 Cloudflare 全球边缘节点
  * **自带 DDoS 防护：** Cloudflare 免费挡常见攻击
  * **SSL 证书自动化：** Cloudflare 自动签发和续期边缘证书
  * **IPv6 变动无感：** Lucky 自动 DDNS，运营商换 IP 不影响使用



#### ❌ 缺点

  * **端口严重受限：** Cloudflare 免费版只代理特定 HTTP/HTTPS 端口（80,443,8080,8443 等），非标端口不行
  * **Lucky 反向代理多一层：** NAS 端口不是支持列表时，需要 Lucky 做端口映射（如 5666→8443）
  * **依赖 Lucky 持续运行：** Lucky 挂了 = DDNS 不更新 = IPv6 换了就连不上
  * **Cloudflare 能看到明文：** 如果选 Flexible 模式，CF↔NAS 段不加密（选 Full 模式可解决）
  * **国内访问可能绕路：** Cloudflare 回源走国际出口，非直连，延迟略高（通常 200-400ms）



#### 🎯 适用场景

只访问 NAS 的 Web 管理界面、Jellyfin、文件管理器等 **HTTP/HTTPS 服务** ，且能接受 200-400ms 延迟。这是 80% 用户的最佳选择。

### 方案 B：Cloudflare Tunnel (cloudflared)

#### 🧠 实现原理
    
    
    Step 1  在 NAS 上安装 cloudflared，运行后主动向 Cloudflare 发起一条出站 TCP 长连接
    Step 2  这条连接是 QUIC 协议（基于 UDP），对 NAT 极其友好，不需要任何入站端口
    Step 3  Cloudflare 在控制面创建一条"隧道"，并自动创建 DNS CNAME 记录指向隧道
    Step 4  用户访问 nas.example.com → DNS → Cloudflare 边缘 → 隧道 → cloudflared → NAS 本地服务
    
      关键对比：
      ┌──────────────┐         ┌──────────────┐
      │  方案 A       │         │  方案 B       │
      │  Cloudflare   │         │  Cloudflare   │
      │  对外 IPv4    │         │  对外 IPv4    │
      │  对内 IPv6 ──→ NAS       │  隧道内 ────→ cloudflared → 127.0.0.1
      │  (需要 IPv6)  │         │  (不需要任何公网 IP!)
      └──────────────┘         └──────────────┘
    

**关键区别：** 方案 A 是 Cloudflare "主动找你"（需要 IPv6 可入站），方案 B 是 cloudflared "主动找 Cloudflare"（纯出站连接，防火墙只管出不管入）。

#### ✅ 优点

  * **无需任何公网 IP：** 连 IPv6 都不需要，cloudflared 主动连出去。这对移动宽带/校园网/大内网是救命方案
  * **无视运营商防火墙：** 出站连接几乎不会被拦截（QUIC over UDP 443）
  * **支持任意 TCP 端口：** SSH(22)、FTP(21)、数据库(3306) 都能通过 Tunnel 暴露
  * **不用 Lucky DDNS：** Tunnel 自己维持连接，IP 变动自动重连，零感知
  * **加密到 NAS：** 整条链路 TLS 加密，Cloudflare 边缘到 NAS 全程 QUIC 加密
  * **支持私有网络：** 可以配合 Cloudflare Access 做零信任认证（Google/GitHub 登录后才能访问）



#### ❌ 缺点

  * **要在 NAS 上装 cloudflared：** 比方案 A 多一步，需要 SSH 进 NAS 操作
  * **依赖 cloudflared 进程存活：** 进程挂了就断了（可以配 systemd 自动重启）
  * **每个服务要单独配置 ingress 规则：** 配置文件比方案 A 略复杂
  * **国内访问同样绕路：** 和方案 A 一样走 Cloudflare 国际网络
  * **Tunnel 是 Cloudflare 闭源服务：** 锁定在 Cloudflare 生态，不能换 CDN



#### 🎯 适用场景

需要暴露**非 HTTP 端口** （SSH、数据库等），或**连 IPv6 公网都没有** 的场景。如果你家宽带是移动大内网连 IPv6 都不给，Tunnel 是唯一解。

### 方案 C：frp / 内网穿透（自建 VPS 中转）

#### 🧠 实现原理
    
    
    
      ┌──────────┐      ┌──────────────────┐      ┌──────────┐
      │  办公室   │ IPv4 │  VPS (双栈公网)   │ IPv4 │   NAS    │
      │  IPv4    │ ───→ │  frps (服务端)    │ ←─── │  frpc    │
      │          │      │  有 IPv4 + IPv6   │      │  客户端   │
      └──────────┘      └──────────────────┘      └──────────┘
                             ↑
                        需要租用 VPS
                        (约 ¥30-100/月)
    
      流程：
      1. 租一台有 IPv4 公网 IP 的 VPS（国内推荐阿里云/腾讯云轻量）
      2. VPS 运行 frps（服务端），监听公网端口
      3. NAS 运行 frpc（客户端），主动连接 frps，建立隧道
      4. 办公室访问 VPS_IP:端口 → frps 转发 → frpc 隧道 → NAS
    

#### ✅ 优点

  * **完全自主可控：** 数据走自己的 VPS，不经过第三方 CDN
  * **国内延迟极低：** VPS 选国内机房，延迟通常 <10ms
  * **端口随心所欲：** 想映射什么端口映射什么端口
  * **带宽可控：** VPS 带宽是你的，不像 Cloudflare 免费版限速



#### ❌ 缺点

  * **要花钱：** VPS 月付 ¥30-100，一年 ¥360-1200
  * **需要运维 VPS：** 系统更新、安全加固、frps 维护
  * **单点故障：** VPS 挂了全部服务中断
  * **安全责任在你：** VPS 暴露公网，被攻击你自己扛
  * **带宽瓶颈：** VPS 带宽通常在 3-10Mbps，大文件传输慢
  * **要备案：** 国内 VPS 建站需要 ICP 备案（个人使用可以不做，但有风险）



#### 🎯 适用场景

对**延迟极其敏感** （如远程桌面、在线剪辑），或需要**大带宽、非标协议** ，且有预算。普通人不需要这个方案。

### 方案 D：Tailscale / ZeroTier（组网方案）

#### 🧠 实现原理
    
    
      Tailscale 基于 WireGuard，在所有设备间建立 P2P 加密 mesh 网络。
      即使 NAS 和办公室在不同 NAT 后面，它也通过 DERP 中继服务器完成打洞。
      
      办公室 ──WireGuard 隧道── Tailscale 协调服务器 ──WireGuard 隧道── NAS
               (打洞成功则 P2P 直连，失败则走 DERP 中继)
    

#### ✅ 优点

  * 配置极简：装客户端 → 登录 → 自动组网
  * 免费版支持 3 用户 100 设备
  * P2P 直连延迟极低
  * 全端口可达，跟局域网一样



#### ❌ 缺点

  * 办公室电脑也要装客户端（不是纯浏览器访问）
  * 打洞失败时走中继，速度和 Cloudflare 方案差不多
  * 不适合分享给他人（每人要装客户端）



#### 🎯 适用场景

你自己多设备之间的互访（手机、笔记本、NAS），而不是对外分享服务。

⚠️ 不满足本方案约束：办公室需要安装 Tailscale 客户端，不是纯浏览器方案。

## 📊 四种方案一图总结

方案| 原理| 公网要求| 成本| 端口| 延迟| 难度| 推荐场景  
---|---|---|---|---|---|---|---  
A. CF 代理 | 橙色云朵 v4↔v6 | 需 IPv6 | 免费 | HTTP only | 200-400ms | ⭐⭐ | Web 访问首选  
B. CF Tunnel | 出站隧道 | 不需要！ | 免费 | 全部 TCP | 200-400ms | ⭐⭐⭐ | 无公网首选  
C. frp/VPS | 自建中转 | 不需要 | ¥30-100/月 | 全部 | <10ms | ⭐⭐⭐⭐ | 低延迟需求  
D. Tailscale | WireGuard 组网 | 不需要 | 免费 | 全部 | 10-100ms | ⭐ | 个人多设备互访  
  
## 🛤️ 推荐决策路径
    
    
                        ┌── 是 ──→ **方案 A: CF 代理 ✅**
      NAS 有 IPv6 公网？──┤
                        └── 否 ──→ **方案 B: CF Tunnel ✅**
    
      ❌ 方案 C (frp)：满足约束但月付 ¥30+，无必要
      ❌ 方案 D (Tailscale)：违反约束 — 办公室需装客户端
    
      **你的场景：NAS 有 IPv6 + 办公室零软件 → 方案 A，一秒都不要犹豫。**
    

## 📦 前提条件

1| 拥有一个**域名** ，并将 DNS 托管到 Cloudflare  
---|---  
2| NAS 上已安装 **Lucky** （飞牛 OS 应用商店可装）  
3| NAS 的 IPv6 地址能正常访问（手机 5G 网络测试 `http://[IPv6地址]:端口`）  
4| Cloudflare 账号（免费即可）  
  
## 方案一：Cloudflare DNS 代理（推荐 ⭐）

### Step 1：将域名 DNS 托管到 Cloudflare

1注册/登录 [Cloudflare 控制台](https://dash.cloudflare.com) → 添加站点 → 输入你的域名

2Cloudflare 会给你两个 NS 地址（如 `abby.ns.cloudflare.com`），去你的域名注册商处修改 NS 记录

3等待 NS 生效（通常 10 分钟 ~ 2 小时），Cloudflare 显示 "Active" 即完成

### Step 2：获取 Cloudflare API Token

1Cloudflare 控制台 → 右上角头像 → **My Profile** → **API Tokens**

2点击 **Create Token** → 选择 **"Edit zone DNS"** 模板

3Zone Resources 选择你的域名 → 点击 **Continue to summary** → **Create Token**

4⚠️ 立即复制保存 Token！页面关闭后无法再次查看。

**Cloudflare API Token 格式** （供后续 Lucky 配置用）：
    
    
    Zone ID:   在 Cloudflare 域名概览页右侧可复制
    API Token: 刚才创建的 token (如 aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890)

### Step 3：在 Lucky 中配置 DDNS

1打开 NAS 的 Lucky 管理页面（默认端口 `16601`）→ 左侧菜单 **动态域名**

2点击 **添加** ，填写以下信息：

配置项| 填写内容| 说明  
---|---|---  
DDNS 服务商| `Cloudflare`| 下拉选择  
域名| `nas.example.com`| 要访问的子域名  
记录类型| AAAA (IPv6)| NAS 只有 IPv6，必须选 AAAA  
获取 IP 方式| 网卡获取 / 接口获取| 选 NAS 实际使用的网卡  
Proxy (代理)| ✅ 开启| 这就是橙色云朵，关键开关！  
TTL| 120 (自动)| 保持默认  
API Token| 粘贴刚才创建的 Token|   
Zone ID| 粘贴 Zone ID|   
  
### 🔑 Proxy（橙色云朵）是关键！

开启 Proxy 后，DNS 解析返回的是 Cloudflare 的 IPv4 地址，而不是你 NAS 的 IPv6 地址。

Cloudflare 收到请求后，通过 IPv6 回源到你的 NAS → 自动完成 v4↔v6 转换。
    
    
      DNS 查询 nas.example.com:
      
      Proxy OFF (灰色云朵):
        → 返回 240e:xxx:xxx:xxx  (你的 IPv6)
        → 办公室 IPv4 无法连接 ❌
      
      Proxy ON (橙色云朵): ✅
        → 返回 104.21.xx.xx  (Cloudflare IPv4)
        → 办公室 IPv4 正常访问 ✅
        → Cloudflare 内部用 IPv6 回源到你的 NAS ✅

### Step 4：配置 Cloudflare 防火墙规则（端口放行）

1Cloudflare 控制台 → 你的域名 → **DNS** → **Records**

2确认 Lucky 自动创建的 AAAA 记录存在，且为 橙色云朵 (Proxied)

3Cloudflare 免费版默认只代理以下端口：

协议| 支持的端口  
---|---  
HTTP| 80, 8080, 8880, 2052, 2082, 2086, 2095  
HTTPS| 443, 2053, 2083, 2087, 2096, 8443  
  
⚠️ 重要：你的 NAS 服务端口必须落在上表中。比如飞牛 OS 默认端口是 `5666/5667`，不在支持列表中。解决方案： 

  1. **方法 A（推荐）** ：在 Lucky 中设置**反向代理** ，把 5666 映射到 443/8443
  2. **方法 B** ：使用 Lucky 的 Web 服务功能做转发
  3. **方法 C** ：直接用 Cloudflare Tunnel（方案二），无端口限制



### Step 5：Lucky 反向代理配置（端口映射）

1Lucky → **Web 服务** → **添加 Web 服务规则**

2配置如下：

配置项| 值| 说明  
---|---|---  
监听端口| `8443`| Cloudflare 支持的 HTTPS 端口  
启用 TLS| ✅ 开启| 使用 Cloudflare 的免费证书或自签  
反向代理目标| `http://127.0.0.1:5666`| 飞牛 OS 的本地地址  
前端域名| `nas.example.com`| 与 DDNS 域名一致  
  
3Cloudflare → SSL/TLS → 加密模式选择 **Full** 或 **Flexible**

### Step 6：Cloudflare SSL/TLS 配置

模式| 流量加密| 适用场景  
---|---|---  
Flexible| 用户↔CF 加密, CF↔NAS 明文| NAS 无 HTTPS, 最简单  
Full| 全程加密| NAS 有自签证书, 推荐  
Full (Strict)| 全程加密 + 证书验证| NAS 有受信证书  
  
推荐使用 Full 模式。Lucky 自带证书管理，可以配合使用。

### Step 7：验证

  1. 办公室电脑打开浏览器 → `https://nas.example.com:8443`
  2. 手机用 5G（IPv6）→ 访问 `http://[NAS_IPv6]:5666` → 确认 NAS 本身可达
  3. Cloudflare 控制台 → Analytics → 看到流量经过代理



## 方案二：Cloudflare Tunnel（备选，无端口限制）

如果方案一的端口限制无法满足需求（比如需要访问非标准端口），可以用 Cloudflare Tunnel。

### 原理
    
    
      NAS (cloudflared) ──出站隧道──→ Cloudflare 边缘 ──→ 用户 (IPv4/IPv6)
      (无需任何公网 IP)
      

### Step 1：安装 cloudflared

1SSH 连接到飞牛 OS（或通过 Lucky 的终端功能）

2下载安装 cloudflared：
    
    
    # 下载 cloudflared (amd64)
    curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
    chmod +x /usr/local/bin/cloudflared
    
    # 验证
    cloudflared --version

### Step 2：创建 Tunnel
    
    
    # 登录 Cloudflare
    cloudflared tunnel login
    # 浏览器会弹出授权页面，选择你的域名
    
    # 创建隧道
    cloudflared tunnel create nas-tunnel
    # 记下返回的 Tunnel ID (如 12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
    
    # 配置隧道
    mkdir -p ~/.cloudflared
    cat > ~/.cloudflared/config.yml << 'EOF'
    tunnel: 你的Tunnel-ID
    credentials-file: /root/.cloudflared/你的Tunnel-ID.json
    
    ingress:
      # 飞牛 OS 主界面
      - hostname: nas.example.com
        service: http://localhost:5666
      
      # 可以添加更多服务
      - hostname: jellyfin.example.com
        service: http://localhost:8096
      
      # 默认规则（必须放在最后）
      - service: http_status:404
    EOF

### Step 3：添加 DNS 记录 + 启动隧道
    
    
    # cloudflared 会自动创建 DNS CNAME 记录
    cloudflared tunnel route dns nas-tunnel nas.example.com
    
    # 安装为系统服务，开机自启
    cloudflared service install
    
    # 启动
    systemctl start cloudflared
    systemctl status cloudflared

**Tunnel 方案优点：**

  * ✅ 无需 Lucky DDNS（Tunnel 自动保持连接）
  * ✅ 无需公网 IP（连 IPv6 都不需要）
  * ✅ 无端口限制（可代理任意 TCP 服务）
  * ✅ Cloudflare 自带 DDoS 防护 + CDN 加速



## 方案三：Lucky 反向代理 + Cloudflare（进阶）

将 Lucky 的 "Web 服务" 和 Cloudflare 配合，可以实现更灵活的代理：
    
    
      用户 (IPv4)
        │
        ▼
      Cloudflare CDN (端口 443)
        │  IPv6 回源
        ▼
      Lucky 反向代理 (端口 8443)
        ├── nas.example.com       → http://127.0.0.1:5666   (飞牛 OS)
        ├── jellyfin.example.com  → http://127.0.0.1:8096   (Jellyfin)
        └── file.example.com      → http://127.0.0.1:8080   (文件管理)
    

所有子域名的 AAAA 记录（橙色云朵）都指向 NAS IPv6，Lucky 根据 `Host` 头分发到不同后端服务。

## 📋 Lucky 核心配置汇总

### 1\. 动态域名 (DDNS)

服务商| Cloudflare  
---|---  
记录类型| AAAA (IPv6)  
Proxy| ✅ 开启 (橙色云朵)  
更新间隔| 300 秒 (5 分钟)  
  
### 2\. Web 服务 (反向代理)

监听端口| 8443 (Cloudflare 支持的 HTTPS 端口)  
---|---  
目标地址| http://127.0.0.1:5666  
TLS| ✅ 开启  
  
## ❓ 常见问题

问题| 原因| 解决  
---|---|---  
办公室能 ping 通域名但无法访问 | 端口不在 Cloudflare 代理范围内 | 改用 8443/2053 等支持的端口  
提示 521/522 错误 | Cloudflare 无法连接到 NAS 的 IPv6 | 检查 NAS 防火墙、IPv6 是否可达、Lucky 代理端口是否正常  
DDNS 更新了但访问的还是旧 IP | TTL 缓存或 Lucky 未正确更新 | 降低 TTL 到 120, 查看 Lucky 日志确认更新成功  
IPv6 地址经常变化 | 运营商动态分配 | Lucky 的自动 DDNS 更新可解决, 缩短间隔到 120s  
Cloudflare 代理速度慢 | 回源走国际线路 | 正常现象, 几百 ms 延迟可接受; 如需低延迟考虑 Tunnel  
飞牛 OS 应用商店没有 Lucky | 镜像源未收录 | SSH 进 NAS 用 Docker 安装: `docker run -d --net=host --restart=always --name lucky gdy666/lucky`  
  
## 🍀 飞牛 OS 特别说明

  * 飞牛 OS 基于 Debian，支持 Docker 和 apt 安装软件
  * 默认 Web 管理端口：`5666` (HTTP) / `5667` (HTTPS)
  * Lucky 默认端口：`16601`
  * 如果飞牛 OS 应用商店有 Lucky，直接在商店安装最方便
  * SSH 登录：`ssh your-user@NAS_IPv6地址`
  * 如果 IPv6 防火墙挡住了入站流量：飞牛 OS → 设置 → 防火墙 → 放行需要的端口



* * *

📐 文档版本: v1.0 | 📅 2026-06-24

NAS IPv6 Only → Lucky DDNS + Cloudflare Proxy → 办公室 IPv4 无障碍访问
