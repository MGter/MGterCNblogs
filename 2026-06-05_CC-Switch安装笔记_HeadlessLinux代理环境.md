# CC-Switch 安装笔记 — Headless Linux（代理环境）

> **CC-Switch 是什么**：AI CLI 工具的 provider/proxy 管理器——切换 AI 后端 provider、管理本地多应用代理路由。
> **部署场景**：内网无图形界面的 Linux 服务器，需通过 HTTP 代理访问 GitHub。
> **⚠️ 实测补充**：装完 Web 界面看不到 cost/usage 是常见问题，见第七节。

---

## 一、前置检查

```bash
cat /etc/os-release | head -3 && uname -m  # OS + 架构
echo "DISPLAY=$DISPLAY"                     # 空 = 无 GUI，不要装 AppImage
df -h ~ | tail -1                           # 至少 50MB 可用
curl -x <PROXY_HOST>:<PROXY_PORT> -sI https://github.com | head -1  # 代理通吗
```

---

## 二、安装步骤

### Step 1：装 CLI 核心

```bash
export http_proxy=http://<代理IP>:<代理端口>
export https_proxy=http://<代理IP>:<代理端口>

curl -x <代理IP>:<代理端口> -fsSL \
  -o /tmp/install-ccswitch.sh \
  "https://github.com/SaladDay/cc-switch-cli/releases/latest/download/install.sh"

chmod +x /tmp/install-ccswitch.sh && bash /tmp/install-ccswitch.sh && rm /tmp/install-ccswitch.sh

# 验证
~/.local/bin/cc-switch --version
```

如果 `~/.local/bin` 不在 PATH：

```bash
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

### Step 2：装 Web 控制台

```bash
# 2a. 克隆仓库（临时配 git 代理）
git config --global http.proxy http://<代理IP>:<代理端口>
git config --global https.proxy http://<代理IP>:<代理端口>
git clone https://github.com/Laliet/cc-switch-web.git ~/cc-switch-web
git config --global --unset http.proxy
git config --global --unset https.proxy

# 2b. 下载预编译 server 二进制
curl -x <代理IP>:<代理端口> -fsSL \
  -o ~/cc-switch-web/cc-switch-server \
  "https://github.com/Laliet/cc-switch-web/releases/latest/download/cc-switch-server-linux-x86_64"
chmod +x ~/cc-switch-web/cc-switch-server

# 2c. 清理冗余（可选）
rm -rf ~/cc-switch-web/.git
```

### Step 3：启动 Web Server

```bash
cd ~/cc-switch-web

# 后台启动，指定端口 + 允许 HTTP（内网可用）
HOST=0.0.0.0 PORT=18888 ALLOW_HTTP_BASIC_OVER_HTTP=1 \
  nohup ./cc-switch-server > ~/cc-switch-web/server.log 2>&1 &

echo "PID=$!"
```

验证：

```bash
sleep 2
ss -tlnp | grep 18888
cat ~/cc-switch-web/server.log
# 期望输出: Starting web server on http://0.0.0.0:18888
```

### Step 4：获取登录凭证

```bash
cat ~/.cc-switch/web_password
```

---

## 三、访问信息

| 项目 | 值 |
|:---|:---|
| Web URL | `http://<服务器IP>:18888` |
| 用户名 | `admin` |
| 密码 | `~/.cc-switch/web_password` 文件内容 |

---

## 四、踩坑记录

| 现象 | 原因 | 解决 |
|:---|:---|:---|
| `cc-switch` 启动后 GTK 报错崩溃 | 无 GUI 环境但运行了 AppImage | 装 CLI 版，不要装 GUI |
| 端口 3000/18888 被占用 | 之前 `--help` 调用实际上启动了 server（`--help` 被忽略） | `ss -tlnp \| grep <端口>` → `kill <PID>` |
| `Refusing to start without ALLOW_HTTP_BASIC_OVER_HTTP` | `HOST=0.0.0.0` 模式下强制要求 TLS | 内网可信环境：加 `ALLOW_HTTP_BASIC_OVER_HTTP=1` |
| `curl: Proxy CONNECT aborted` 大文件下载失败 | 代理限制长时间 CONNECT 隧道 | 直接下载二进制（Step 2b），而非走 deploy 脚本。还不行就别的机器下载然后 scp |
| 命令静默失败 | `/` 磁盘写满（`df -h /` 确认） | `find /tmp -size +100M` 清理旧 AppImage / Claude 临时文件 |
| Web 能启动但局域网其他机器访问不了 | `HOST` 设成了 `127.0.0.1`（默认） | 必须 `HOST=0.0.0.0`；同时检查防火墙 |

---

## 五、CLI 常用命令

```bash
cc-switch provider list           # 列出已配置的 AI provider
cc-switch provider switch <name>  # 切换活跃 provider
cc-switch proxy --help            # 管理本地多应用代理路由
cc-switch daemon start            # 启动守护进程
cc-switch update                  # 升级到最新版
cc-switch interactive             # 交互式 TUI 界面
```

---

## 六、安装后文件结构

```
~/.local/bin/cc-switch            # CLI 二进制 (15MB)
~/cc-switch-web/                   # Web 服务目录
~/cc-switch-web/cc-switch-server   # Web server 二进制 (12.7MB)
~/cc-switch-web/server.log         # 服务日志
~/.cc-switch/                      # 配置和数据
~/.cc-switch/cc-switch.db          # SQLite 数据库
~/.cc-switch/web_password          # Web 控制台密码
~/.cc-switch/web_env               # CSRF token 等环境变量
```

---

## 七、常见问题：装好了但看不到 cost/usage

> 最常见 bug，涉及 3 个层面，按顺序排查。

### 问题 1：数据库 schema 不匹配

**症状**：`config show` / `proxy show` 报错 `table proxy_config has no column named circuit_success_threshold`

**原因**：CC-Switch 二进制升了级（v5.7.0），但数据库还是旧版（v3.16.1），新增字段不存在。

**修复：**

```bash
# 备份旧数据库
cp ~/.cc-switch/cc-switch.db ~/.cc-switch/cc-switch.db.bak

# 删库重建（配置会丢失，需重新设置）
rm ~/.cc-switch/cc-switch.db

# 验证
~/.local/bin/cc-switch config show
# 不再报错 → schema 已重建
```

也可尝试 `cc-switch update` 后自动迁移，无效就删库重建。

### 问题 2：Daemon 没运行

**症状**：`cc-switch daemon status` 返回 `daemon not reachable`

**原因**：Web server 只负责展示界面，daemon 才是真正拦截 API 流量、记录 cost/usage 的组件。

**修复：**

```bash
~/.local/bin/cc-switch daemon start

# 验证
~/.local/bin/cc-switch daemon status
# 期望输出: daemon running
```

### 问题 3：Claude Code 流量绕过了 CC-Switch

**症状**：daemon 和 Web server 都正常，但 cost/usage 始终为零。

**原因**：`~/.claude/settings.json` 中 `ANTHROPIC_BASE_URL` 直接指向 provider API（如 `https://api.deepseek.com/anthropic`），没走 CC-Switch。

**修复：**

```bash
# 查看当前配置
cat ~/.claude/settings.json | grep ANTHROPIC_BASE_URL

# 改为走 cc-switch 代理（daemon 默认监听 127.0.0.1:8080）
# settings.json 中改为：
# "ANTHROPIC_BASE_URL": "http://127.0.0.1:8080/anthropic"
```

**链路**：Claude Code → `127.0.0.1:8080`（cc-switch daemon）→ 实际 provider API，daemon 在此过程中记录每条请求的 cost。

---

### 快速诊断命令

```bash
# 1. 版本一致性
~/.local/bin/cc-switch --version

# 2. 数据库健康
~/.local/bin/cc-switch config show 2>&1

# 3. daemon 状态
~/.local/bin/cc-switch daemon status 2>&1

# 4. 流量走向
cat ~/.claude/settings.json | grep -i "base_url"

# 5. Web server 状态
ss -tlnp | grep 18888
```

---

*整理于 2026-06-05，原文为 CC-Switch 官方 install 文档 + 实机 bug 排查经验*
