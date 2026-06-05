# CC-Switch 安装笔记 — Headless Linux（代理环境）

> **CC-Switch 是什么**：AI CLI 工具的 provider/proxy 管理器——切换 AI 后端 provider、管理本地多应用代理路由。
> **部署场景**：内网无图形界面的 Linux 服务器，需通过 HTTP 代理访问 GitHub。

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

*整理于 2026-06-05，原文为 CC-Switch 官方 install 文档*
