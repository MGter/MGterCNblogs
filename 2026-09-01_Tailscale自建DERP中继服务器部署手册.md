# Tailscale 自建 DERP 中继服务器部署手册

> 日期：2026-09-01
> 环境：阿里云 ECS（Ubuntu 22.04，无域名）→ 自签证书 + CertName 指纹校验
> 状态：✅ 已部署并验证通过（客户端 relay 已切换到自建节点）

---

## 1. 背景与原理

### 1.1 DERP 是什么

DERP（Designated Encrypted Relay for Packets）是 Tailscale 的中继服务器。当两台设备无法建立 P2P 直连（NAT 类型、防火墙、被墙）时，流量通过 DERP 中继；即使直连成功，DERP 也承担打洞前的初始连接和健康检查。

### 1.2 为什么自建

- 官方 DERP 节点大多在境外，国内网络常连不上（表现为 `tailscale status` 中 `health check` 报 "could not connect to ... relay server"）
- 自建节点放在国内云服务器，中继延迟低、稳定
- 数据经 DERP 转发是端到端加密的（Tailscale 是 WireGuard 封装），自建中继**看不到你的流量内容**，安全性和官方节点同级

### 1.3 架构拓扑

```
┌────────────┐   P2P 直连（首选）   ┌────────────┐
│  Windows   │ ◄──────────────────► │  其他设备   │
│  (mgter)   │                       │            │
└────────────┘                       └─────┬──────┘
        │                                 │
        │  直连失败时走 DERP 中继           │
        ▼                                 ▼
   ┌──────────────────────────────────────────┐
   │  阿里云 DERP (region 900, 39.107.92.91)  │
   │  443/TCP (HTTPS 中继) + 3478/UDP (STUN)  │
   └──────────────────────────────────────────┘
```

### 1.4 本次部署关键决策

| 决策点 | 选择 | 原因 |
|--------|------|------|
| 证书方案 | 自签 CA + `CertName: sha256-raw:<指纹>` | 无域名；新版 derper 支持按证书 SHA256 指纹校验，客户端无需导入 CA |
| DERP map 下发 | tailnet policy JSON（admin console） | **官方客户端不读本地 derp.json**，DERP map 由控制平面下发 |
| 部署方式 | WSL 交叉编译 derper 二进制 → scp 上传 | 服务器仅 1.6G 内存，避免编译 OOM |
| SSH 端口 | 2222（非标准） | 服务器原配置 |

---

## 2. 服务器端部署

### 2.1 前置准备

- 一台有公网 IP 的云服务器（本次：阿里云 39.107.92.91，Ubuntu 22.04，2C1.6G）
- 服务器开放 443/TCP 与 3478/UDP（阿里云安全组，见 §3）
- 本机有 Go 工具链（用于交叉编译 derper）

### 2.2 编译 derper（在本机/WSL 执行）

> 注意：最新 tailscale 要求 Go ≥ 1.26.6（使用 `go install tailscale.com/cmd/derper@latest` 时版本会自动对齐）

```bash
# 下载 Go 工具链（国内用阿里云镜像）
mkdir -p ~/go-toolchain && cd ~/go-toolchain
curl -fsSL -o go.tar.gz https://mirrors.aliyun.com/golang/go1.26.6.linux-amd64.tar.gz
tar xzf go.tar.gz -C /tmp/go2

# 编译 derper（GOPROXY 用国内代理加速）
export GOTOOLCHAIN=local GOPROXY=https://goproxy.cn,direct
/tmp/go2/go/bin/go install tailscale.com/cmd/derper@latest
# 产物：$(go env GOPATH)/bin/derper（约 25MB）
```

### 2.3 上传并安装

```bash
# 上传（服务器 SSH 端口为 2222，用别名或 -P）
scp -P 2222 ~/go/bin/derper aliyun:/tmp/derper

# 服务器侧：安装到 /opt/derper
ssh aliyun 'sudo mv /tmp/derper /opt/derper/derper && sudo chmod 755 /opt/derper/derper'
```

### 2.4 生成自签证书

**证书目录**：`/opt/derper/certs/`，包含：
- `ca.crt` / `ca.key` — 自签 CA（10 年有效期）
- `39.107.92.91.crt` / `39.107.92.91.key` — 服务器证书（SAN 含公网 IP，10 年）

> 证书文件名必须与 `--hostname` 一致（derper 的 manual 模式按 `<hostname>.crt` / `<hostname>.key` 查找）。

```bash
# 服务器需要 openssl（Ubuntu 自带）
# 1) 生成 CA
openssl req -x509 -newkey rsa:2048 -sha256 -days 3650 -nodes \
  -keyout /opt/derper/certs/ca.key -out /opt/derper/certs/ca.crt \
  -subj "/CN=MGTER-DERP-CA" \
  -addext "basicConstraints=critical,CA:TRUE"

# 2) 生成服务器密钥 + CSR
openssl req -newkey rsa:2048 -sha256 -nodes \
  -keyout /opt/derper/certs/server.key -out /opt/derper/certs/server.csr \
  -subj "/CN=39.107.92.91"

# 3) CA 签发（SAN 必须包含公网 IP）
openssl x509 -req -sha256 -days 3650 \
  -in /opt/derper/certs/server.csr \
  -CA /opt/derper/certs/ca.crt -CAkey /opt/derper/certs/ca.key -CAcreateserial \
  -out /opt/derper/certs/39.107.92.91.crt \
  -extfile <(printf "subjectAltName=IP:39.107.92.91\nbasicConstraints=CA:FALSE\nkeyUsage=digitalSignature,keyEncipherment\nextendedKeyUsage=serverAuth")

# derper manual 模式期望 <hostname>.crt 和 <hostname>.key
cp /opt/derper/certs/server.key /opt/derper/certs/39.107.92.91.key
chmod 600 /opt/derper/certs/*.key; chmod 644 /opt/derper/certs/*.crt
```

**记录证书指纹**（后面 policy 配置要用）：

```bash
openssl x509 -in /opt/derper/certs/39.107.92.91.crt -noout -fingerprint -sha256
# 输出形如: SHA256 Fingerprint=C4:5B:80:1C:...:38:9D
# 去掉冒号转小写 → c45b801c8435ab10b5a025a7fed8bbee6e5179a3daf5b9d29375c9488988389d
```

### 2.5 配置 systemd 服务

**`/etc/systemd/system/derper.service`**：

```ini
[Unit]
Description=Tailscale DERP relay server
After=network.target

[Service]
Type=simple
ExecStart=/opt/derper/derper \
  --hostname=39.107.92.91 \
  --certmode=manual \
  --certdir=/opt/derper/certs \
  --a=:443 \
  --stun-port=3478 \
  --http-port=-1 \
  --verify-clients=false
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

> 参数说明：
> - `--certmode=manual`：使用自签证书（不自签则用 letsencrypt）
> - `--http-port=-1`：禁用 80 端口 HTTP（只留 HTTPS 中继）
> - `--verify-clients=false`：不校验客户端（自建中继场景通常不需要，简化部署）

启动：

```bash
sudo cp derper.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now derper
sudo systemctl status derper        # 应为 active (running)
```

启动日志中会输出关键信息：

```
STUN server listening on [::]:3478
derper: serving on :443 with TLS
Using self-signed certificate for IP address "39.107.92.91". Configure it in DERPMap using:
  {"Name":"custom","RegionID":900,"HostName":"39.107.92.91","CertName":"sha256-raw:c45b80..."}
```

> 日志里那段 `CertName` 就是客户端要用的指纹校验字段，直接可用。

---

## 3. 阿里云安全组配置

在阿里云控制台 → ECS → 安全组 → 入方向规则，**必须放行**（本次踩坑点：只放 TCP 443 忘记放 UDP 3478，导致 STUN 不通）：

| 协议 | 端口 | 源 | 用途 |
|------|------|-----|------|
| TCP | 443 | 0.0.0.0/0 | DERP 中继（HTTPS/WSS） |
| UDP | 3478 | 0.0.0.0/0 | STUN 打洞探测 |
| TCP | 80 | 0.0.0.0/0 | （可选）captive portal 检查，可不开 |

**验证端口**（从外部机器）：

```bash
# TCP 443
timeout 5 bash -c "echo > /dev/tcp/39.107.92.91/443" && echo "443 OPEN"

# UDP 3478 —— 注意：普通裸 STUN 请求会被 derper 静默丢弃！
# 必须用 Tailscale 标准格式（带 SOFTWARE=tailnode + FINGERPRINT）才能收到响应
```

> ⚠️ **经验**：derper 的 STUN 服务只响应带 `SOFTWARE="tailnode"` 属性 + 有效 `FINGERPRINT`（CRC32 ^ 0x5354554E）的请求，普通 STUN 探针会超时——这不是故障，是防滥用设计。用官方 `tailscale debug derp` 或下述 §5 方式验证即可。

---

## 4. 客户端配置（tailnet policy）

### 4.1 关键认知

**官方 Tailscale 客户端不读取本地 `derp.json` 文件**（`C:\ProgramData\Tailscale\derp.json` 对官方客户端无效，那是 headscale 生态的约定）。DERP map 由**控制平面**（login.tailscale.com）通过 tailnet policy JSON 下发给所有设备。

### 4.2 admin console 配置

1. 打开 **https://login.tailscale.com/admin/acls**
2. 切换到 **JSON 编辑器**（图形编辑器只显示 ACL 规则，看不到 derpMap）
3. 在顶层添加 `derpMap` 字段（与 `"acls"` 平级）：

```json
"derpMap": {
  "OmitDefaultRegions": true,
  "Regions": {
    "900": {
      "RegionID":   900,
      "RegionCode": "aliyun",
      "RegionName": "Aliyun DERP",
      "Nodes": [
        {
          "Name":     "1",
          "RegionID": 900,
          "HostName": "39.107.92.91",
          "IPv4":     "39.107.92.91",
          "CertName": "sha256-raw:c45b801c8435ab10b5a025a7fed8bbee6e5179a3daf5b9d29375c9488988389d",
          "STUNPort": 3478,
          "DERPPort": 443
        }
      ]
    }
  }
}
```

4. 点 **Save**

### 4.3 字段说明

| 字段 | 说明 |
|------|------|
| `derpMap` | 控制平面下发的 DERP 地图，覆盖默认配置 |
| `OmitDefaultRegions` | `true` = 禁用官方 DERP 节点（只用自建）；`false` = 官方 + 自建共存兜底 |
| `Regions` | 自定义区域，**900–999 为保留区**，不会被官方占用 |
| `CertName: sha256-raw:<指纹>` | 按证书 SHA256 指纹校验自签证书（免导入 CA），替代 `InsecureForTests` 的更优方案 |
| `HostName` + `IPv4` | 中继地址；有 IPv4 时直连 IP 不走 DNS |
| `STUNPort` / `DERPPort` | 默认 3478 / 443，按实际填 |

> 多个自建节点：一个 region 只放一个节点；要冗余就加多个 region。

### 4.4 多设备分发

policy 保存后由控制平面自动推给**所有设备**，无需每台手动配置。各设备重启 Tailscale 服务（托盘 Exit → 重新打开，或 `net stop tailscale && net start tailscale`）即可立即生效。

---

## 5. 验证

### 5.1 客户端连通性测试（核心验证命令）

```powershell
tailscale debug derp 900
```

预期输出：

```
"Successfully established a DERP connection with node \"39.107.92.91\"",
"Node \"39.107.92.91\" returned IPv4 STUN response: 123.139.52.32:18896"
```

- `Info` 中两条都出现 = 中继 + STUN 全通 ✅
- `Warnings`（如 captive portal 的 port 80 检查失败）可忽略
- `Errors`（如 IPv6 无地址）可忽略（有 IPv4 即可）

### 5.2 确认设备实际中继路径

```powershell
tailscale status --json
```

看每个 peer 的 `Relay` 字段：

```
DESKTOP-K3GGCD7  100.95.235.125  online=True  relay=aliyun   ← 已走自建节点 ✅
```

### 5.3 服务器侧观察

```bash
sudo tail -f /var/log/journal/...   # 或 journalctl -u derper -f
curl -s --cacert /opt/derper/certs/ca.crt https://39.107.92.91/   # 应返回 DERP 欢迎页
```

---

## 6. 运维与故障排查

### 6.1 服务管理

```bash
sudo systemctl restart derper    # 重启
sudo systemctl status derper     # 状态
sudo journalctl -u derper -f     # 实时日志
```

### 6.2 常见问题

| 症状 | 原因 | 解决 |
|------|------|------|
| `tailscale debug derp 900` 报 no such region | policy 未保存/未下发 | 检查 admin console 保存；客户端重启；`tailscale status` 看是否在线 |
| 中继不通但 STUN 通 | 安全组只放了 UDP 3478 没放 TCP 443 | 开 TCP 443 |
| STUN 无响应 | ① 安全组没放 UDP 3478 ② 探针格式不对 | 放行 UDP 3478；用 `tailscale debug derp` 验证 |
| 健康检查报连不上 relay | derper 服务挂了 / 证书过期 | 检查服务状态；重签证书并更新 policy 指纹 |
| 证书到期 | 自签 10 年到期 | 重新生成 → `--certdir` 替换 → 更新 policy 中 `CertName` 指纹 |

### 6.3 证书指纹更新流程

```
1. 服务器重新生成证书（同 §2.4，或改 CN/有效期）
2. openssl 取新指纹
3. admin console policy 中替换 CertName 的 sha256-raw 值
4. 保存后客户端自动生效（无需重启 tailscaled）
```

### 6.4 诊断工具（本手册配套）

`derp-setup/` 目录下有自制的 STUN 探针（排查 UDP 通路用）：

- `stun_probe.py` — 裸 STUN 请求（只对官方节点有效，**对 derper 必超时**，用于对照）
- `stun_probe2.py` — 手工构造 tailnode+FINGERPRINT 请求
- `stun_probe3/` — 用官方 `tailscale.com/net/stun` 包编译的探针（最可靠）

---

## 7. 本次部署速查卡

| 项 | 值 |
|----|----|
| DERP 服务器 | 39.107.92.91（阿里云，Ubuntu 22.04） |
| 中继端口 | 443/TCP，STUN 3478/UDP，SSH 2222 |
| 服务路径 | `/opt/derper/derper`，证书 `/opt/derper/certs/` |
| systemd | `derper.service`（开机自启） |
| region | 900（aliyun），CertName 指纹 `sha256-raw:c45b...389d` |
| 客户端验证 | `tailscale debug derp 900` → DERP connection + STUN OK |
| 已生效 | DESKTOP-K3GGCD7 relay=aliyun ✅ |
| 登录方式 | WSL `ssh aliyun`（~/.ssh/config 已配别名） |

---

## 附录 A：为什么不用本地 derp.json

网上大量教程教"把 derp.json 放到 `C:\ProgramData\Tailscale\`"。**这是 headscale 生态的约定，官方 tailscale 客户端（v1.102.x）代码中不存在任何读取该文件的逻辑**（已通过源码确认：`grep -rn "derp.json" tailscale.com@v1.102.3` 无匹配）。官方客户端 DERP map 唯一来源是控制平面下发的 NetMap，自定义 DERP 的正规途径就是 admin console policy 的 `derpMap` 字段。

## 附录 B：无域名自签证书方案的取舍

- ✅ 优点：不依赖 DNS/域名；`CertName` 指纹校验免去每台设备导入 CA；证书 10 年超长有效期
- ⚠️ 注意：指纹随机器的证书而变化，换证书必须同步换 policy 指纹；相比 `InsecureForTests: true`（完全不校验证书）更安全
- 🔄 若有域名，更优方案是 Let's Encrypt（`--certmode=letsencrypt`），无需管理指纹，但国内申请/续期需注意网络

---

*手册由实际部署过程整理，所有命令均在本环境验证通过。*