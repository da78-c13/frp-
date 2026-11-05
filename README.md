# pubrelay

**让没有公网 IPv4 的机器，也能被公网 IPv4 访问。**

一套开箱即用的 frp 反向代理中继方案，外加一个网页版端口管理面板。
不申请公网 IP、不改家里路由器、不备案，三条命令跑起来。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![frp](https://img.shields.io/badge/frp-v0.71.x-blue)](https://github.com/fatedier/frp)
[![Python](https://img.shields.io/badge/Python-3.6%2B%20%C2%B7%20%E9%9B%B6%E4%BE%9D%E8%B5%96-green)](panel/app.py)

---

## 要解决的问题

家里的 NAS / 软路由 / 树莓派，运营商只给 **公网 IPv6**，或者干脆只有内网 IP。
学校、公司、网吧的电脑，只有 **公网 IPv4**。
两边网络不通 —— `IPv4 → 没有 IPv4`，死结。

这套方案找一台**有公网 IPv4 的机器当中转**，让内网设备主动连出去，把服务"挂"在中转机的公网端口上。

| 场景 | 结果 |
|---|---|
| 手机流量打开家里的 NAS 管理页 | `http://你的域名:5667` |
| 公司电脑 SSH 家里的服务器 | `ssh -p 15022 user@你的域名` |
| 朋友连你自建的 Minecraft 服务器 | `域名:25565` |
| 随时增删要开放的端口 | 网页点一下，不用改配置文件 |

---

## 原理（一张图）

```
        公网 IPv4 世界
   手机 / 公司电脑 / 朋友
        │  http://你的域名:5667
        ▼
  ┌────────────────────────────┐
  │ 中转机（有公网 IPv4）       │
  │  frps 监听 7000 ← 控制通道  │
  │       监听 5667 / 15022 ... │
  └────────────────────────────┘
        ▲
        │  frpc 主动连出（NAT 允许出、不允许入）
        │
  ┌────────────────────────────┐
  │ 内网设备（家里的 NAS）      │
  │ 本地跑着 5667 / 22 等服务   │
  │ 只有内网 IP 或只有 IPv6     │
  └────────────────────────────┘
```

关键点：**连接方向是反的**。不是外面的人连进来，而是家里的设备先连出去；
隧道建立后，外面的访问被"灌"进这条已存在的连接里。

所以：家里路由器**不用**做端口映射（UPnP 都不用开），CGNAT / 大内网 / 纯 IPv6 都无所谓，
中转机只要求一个能用的公网 IPv4。

> 详细原理（NAT 为什么只堵一个方向、一次访问的完整数据流、IPv6 双栈怎么配合）见
> [docs/01-how-it-works.md](docs/01-how-it-works.md)。

---

## 三分钟部署

### 需要准备

1. **一台有公网 IPv4 的机器** —— 云服务器 / 公司电脑 / 学校实验室的机器都行，下文叫**中转机**。
   1 核 512M 足够，frps 基本不吃资源。
2. **一台要暴露出去的内网设备** —— 群晖、fnOS、OpenWrt、树莓派、Windows 小主机，下文叫**客户端**。
3. 一个域名 —— **可选**，但强烈建议。没有域名就直接用中转机的 IP 访问。

### 第 1 步：中转机跑 frps

```bash
# Linux（root）
git clone https://github.com/你的用户名/pubrelay.git
cd pubrelay
sudo bash server/deploy-frps-linux.sh
```

```powershell
# Windows（管理员 PowerShell）
powershell -ExecutionPolicy Bypass -File server\deploy-frps-windows.ps1
```

脚本做三件事：下载 frp → 写配置 → 注册成开机自启。

**跑之前先改配置**，两个地方必须改：

```toml
# examples/frps.toml
auth.token = "换成一串够长的随机字符"   # 客户端必须一模一样
allowPorts = [                        # 允许客户端占用的公网端口
  { start = 5667,  end = 5667 },
  { start = 15000, end = 15100 }
]
```

生成随机 token：

```bash
# Linux / macOS
head -c 24 /dev/urandom | base64 | tr -d '/+=' | head -c 32; echo
```

```powershell
# Windows
-join ((48..57) + (65..90) + (97..122) | Get-Random -Count 32 | % {[char]$_})
```

### 第 2 步：路由器放行端口

登录中转机所在网络的路由器，找「端口转发 / 虚拟服务器 / NAT」，把要用的端口转给中转机：

| 外部端口 | 内部 IP | 内部端口 | 协议 |
|---|---|---|---|
| 7000 | 中转机内网 IP | 7000 | TCP |
| 5667 | 中转机内网 IP | 5667 | TCP |

三个要点：

1. 中转机的内网 IP 要**固定**（静态 IP 或 DHCP 保留），否则重启后转发失效。
2. UDP 服务（Minecraft 基岩版等）要**单独加 UDP 规则**，TCP 和 UDP 是两套端口空间。
3. **别写「TCP 1-65535 全转发」** —— 除非你同时在 frps 里配了 `allowPorts`。

如果中转机本身就是云服务器（有独立公网 IP），跳过这一步，改云厂商的**安全组**即可。

### 第 3 步：客户端跑 frpc

```bash
# 先改 examples/frpc.toml：填中转机 IP 和 token
sudo bash client/deploy-frpc-linux.sh
```

```toml
# examples/frpc.toml
serverAddr = "1.2.3.4"          # 强烈建议写 IP，不要写域名
serverPort = 7000
auth.token = "和中转机完全一致"

[[proxies]]
name = "nas-web"
type = "tcp"
localIP = "127.0.0.1"
localPort = 5667                # NAS 上服务真实端口
remotePort = 5667               # 公网访问用的端口

[[proxies]]
name = "nas-ssh"
type = "tcp"
localPort = 22
remotePort = 15022              # 不占公网 22，避免和中转机自己的 SSH 打架
```

> `serverAddr` 写 IP 不写域名，是为了避开 DNS 没生效、以及机器上跑着 Clash / V2Ray
> 时域名被解析成 `198.18.0.x` fake-ip 导致连不上的坑。

Docker 部署（群晖 / fnOS 常见做法）：

```yaml
services:
  frpc:
    image: snowdreamtech/frpc:latest
    container_name: frpc
    restart: unless-stopped
    network_mode: host          # 必须，否则 localIP=127.0.0.1 指向容器自己
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml:ro
```

### 第 4 步：验证顺序

**从里往外一层层查，不要跳步**：本机服务 → 客户端 frpc → 隧道 → frps → 路由器 → 公网。

```bash
# ① 本机服务在监听吗
ss -tlnp | grep 5667

# ② 隧道建立了吗（客户端侧）
sudo journalctl -u frpc -n 20 --no-pager
#    [nas-web] start proxy success   ← 这条出现就是通了

# ③ 中转机侧端口在监听吗
ss -tlnp | grep 5667

# ④ 换个网络（手机热点）从外部打
curl -v http://你的中转机IP:5667
```

---

## 端口管理面板：日常真正用的是它

改 frpc 配置有多难受：SSH 上去 → `vim frpc.toml` → 照格式抄一段 → 重启 → **隧道断了，SSH 也断了**。

所以带了个面板：**单文件、零依赖、只用 Python 标准库**。

界面示意见 [docs/preview.txt](docs/preview.txt)。

### 部署

```bash
cd panel
sudo PANEL_PASSWORD='你的强口令' bash deploy.sh
```

装完访问 `http://<客户端内网IP>:2285`。脚本会自动装到 `/opt/portpanel/`、
注册 systemd 服务、并把面板自己映射到公网（幂等，不会重复加）。

| 环境变量 | 默认 | 说明 |
|---|---|---|
| `PANEL_PORT` | 2285 | 面板监听端口 |
| `PANEL_PASSWORD` | 随机生成并打到日志 | 登录口令 |
| `PANEL_FRPC_CONF` | `/etc/frp/frpc.toml` | 要管理的配置文件 |
| `PANEL_SERVICE` | `frpc` | 要重启的服务名 |
| `SKIP_FRP_MAP` | 0 | 设 1 则不给面板加公网映射 |

### 能干什么

| 功能 | 说明 |
|---|---|
| 列出全部映射 | 已启用显示端口 / 协议 / 名称；停用的显示为「已停用」，可一键恢复 |
| 新增映射 | 填本地端口 → 点一下生效，不用手写 TOML |
| 删除映射 | 每条后面的 ✕ |
| 停用 / 启用 | 停用是**注释掉整块配置**，不是删除，随时可恢复 |
| 端口扫描 | 实时列出本机监听端口 + 进程名，未映射的直接点 `+` |
| MC 一键开启 | Java 版 TCP 25565/25566 + 基岩版 UDP 19132/19133 |

### 三个安全设计

- **高危端口黑名单**：21 / 23 / 445 / 3306 / 6379 / 7890 等直接拒绝，连配置都不写。
- **自身通道保护**：面板自己那条映射不允许删，防止手滑把自己关在外面。
- **自动备份**：每次改配置前备份 `frpc.toml.bak.<时间戳>`，保留最近 20 份，改坏了随时回滚。

> 已知现象：每次增删映射，frpc 会重启，**公网短暂断开 1~3 秒**。这是 frp 的机制
> （改配置必须重启进程），面板在前端延迟 2 秒刷新让用户感知不到。正在用公网传大文件时避开这个时间点。

---

## 常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| `connect: connection refused` | 客户端连到了错误地址 | 检查 `serverAddr`，别让 DDNS 或代理软件写错 IP |
| `token in login doesn't match` | 两边 token 不一致 | 逐字符比对，注意首尾空格 |
| `port already used` | 公网端口被占了 | 换 `remotePort`，或查是否有重复配置 |
| `port not allowed` | 端口不在 `allowPorts` 内 | 改 `allowPorts`，或让客户端换端口 |
| 客户端 `login success` 但端口连不上 | 路由器没转发 / 防火墙拦了 | 查路由器规则和中转机防火墙 |
| SSH 改完配置就断，隧道也没起来 | `systemctl restart frpc` 把自己踢了 | 用 `systemd-run --no-block` 异步重启，或用面板 |
| 延迟忽高忽低 | 中转机带宽被打满 | 限速或换机器 |

异步重启（替代 `systemctl restart`）：

```bash
sudo systemd-run --no-block --unit=frpc-reload-$(date +%s) systemctl restart frpc
```

---

## 安全：六条硬要求

1. **token 必须强随机。** 泄露了任何人都能用你的中转机开端口。别用 `123456`。
2. **别把 frps 管理面板（默认 7500）暴露到公网。** 它给的是完整控制权，只监听 `127.0.0.1`。
3. **配 `allowPorts`。** 这是唯一挡住"拿到 token 就能开任意端口"的那道锁。
4. **数据库永不上公网。** 3306 / 5432 / 6379 / 27017 / 445 一个都不行，不是"密码够强就没事"。
5. **SSH 改高位端口 + 只允许密钥登录。** 公网 22 每天都在被爆破。
6. **DDNS 可能把你的代理出口 IP 写进 A 记录。** 开着 Clash 全局模式跑 DDNS 脚本，
   拿回来的"公网 IP"可能是代理机的，域名会指向一个毫不相关的地方。用国内回显源，或直接写死 IP。

完整清单见 [docs/05-security.md](docs/05-security.md)。

---

## 目录结构

```
pubrelay/
├── README.md                     ← 你正在读的（部署 + 使用全流程）
├── docs/
│   ├── 01-how-it-works.md        原理：为什么"反向"就能通
│   ├── 02-deploy-server.md       中转机部署（Windows / Linux / 保活 / 排错）
│   ├── 03-deploy-client.md       客户端部署（群晖 / fnOS / Docker / Windows）
│   ├── 04-port-panel.md          Web 端口管理面板
│   └── 05-security.md            安全清单
├── examples/
│   ├── frps.toml                 服务端配置样例
│   ├── frpc.toml                 客户端最小样例
│   └── frpc-all.toml             客户端全量样例（多端口 + UDP）
├── server/
│   ├── deploy-frps-linux.sh      Linux 一键部署
│   ├── deploy-frps-windows.ps1   Windows 一键部署（计划任务 + 看门狗）
│   └── frps-watchdog.ps1         崩溃自动拉起
├── client/
│   ├── deploy-frpc-linux.sh      Linux 客户端一键部署
│   └── frpc.service              systemd unit
└── panel/
    ├── app.py                    Web 端口管理面板（纯标准库，零依赖）
    ├── deploy.sh                 一键部署
    └── portpanel.service         systemd unit
```

---

## 兼容性

| 组件 | 说明 |
|---|---|
| frp | 0.52+ 支持 TOML 配置；样例基于 **v0.71.1** |
| 中转机 | Windows 10/11、Windows Server、任意 Linux、macOS |
| 客户端 | 任何能跑二进制的地方：群晖、威联通、fnOS、OpenWrt、树莓派、Docker |
| 面板 | Python 3.6+，无第三方依赖 |

## 这套方案的代价（说实话）

| 代价 | 说明 |
|---|---|
| 依赖中转机 | 它挂了就全挂，所以 `server/` 里配了看门狗和定时重启 |
| 多一跳 | 延迟 +10~40ms，带宽受中转机上行限制 |
| 安全边界变大 | 服务真的暴露在公网了，密码必须强 |

替代方案：Cloudflare Tunnel（不用公网 IP，但依赖 CF、国内速度看运气）、
Tailscale / ZeroTier（组虚拟局域网，只适合自己用，不方便给朋友访问）、
直接找运营商要公网 IP（部分地区只给 IPv6）。

---

## 参与贡献

欢迎 Issue 和 PR。提问题时请带上：

- 中转机和客户端的系统、frp 版本
- `frpc` / `frps` 的相关日志（**记得打码 token 和公网 IP**）
- 已经排除过哪些环节

改脚本、加平台支持、补文档都算贡献。不用问，直接开 PR。

## 协议

MIT —— 随便用、随便改、随便商用，出问题别找我。

如果这套东西帮你省了几天时间，给个 star 就够了。
