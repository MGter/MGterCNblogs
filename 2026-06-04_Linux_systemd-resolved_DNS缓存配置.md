---
created: 2026-06-04
tags: [Linux, DNS, systemd, 网络配置, 教程]
---

# Linux systemd-resolved DNS 缓存配置手册

> systemd-networkd 环境下的 DNS 本地缓存配置指南，减少上游 DNS 并发压力。

---

## 适用场景

- 系统使用 `systemd-networkd` 管理网络
- 需要开启本地 DNS 缓存，减少对上游 DNS 服务器的并发压力
- 希望重启后配置自动生效

---

## 操作步骤

### 1. 确认 systemd-resolved 已安装并启用

```bash
# 安装（Debian/Ubuntu）
sudo apt update && sudo apt install systemd-resolved

# 启用并启动服务
sudo systemctl enable --now systemd-resolved
```

### 2. 修复 /etc/resolv.conf 为正确的软链接

`systemd-resolved` 要求 `/etc/resolv.conf` 指向其 stub 解析器：

```bash
sudo mv /etc/resolv.conf /etc/resolv.conf.backup
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

验证：
```bash
ls -l /etc/resolv.conf
# 应显示 /etc/resolv.conf -> /run/systemd/resolve/stub-resolv.conf

cat /etc/resolv.conf
# 应显示 nameserver 127.0.0.53
```

### 3. 配置 DNS 缓存参数

编辑 `/etc/systemd/resolved.conf`，在 `[Resolve]` 段落设置：

```ini
[Resolve]
# 推荐：只缓存成功结果，不缓存失败结果
Cache=no-negative

# 可选：手动指定上游 DNS（多个用空格分隔）
DNS=114.114.114.114 223.5.5.5
```

**参数说明：**

| 参数 | 说明 |
|------|------|
| `Cache=yes` | 同时缓存成功和失败结果（默认） |
| `Cache=no-negative` | 只缓存成功结果（推荐高并发场景） |
| `DNS=` | 上游 DNS，不写则用 DHCP 下发的 DNS |

### 4. 重启服务并验证

```bash
sudo systemctl restart systemd-resolved
sudo systemctl status systemd-resolved
resolvectl status
```

### 5. 验证缓存是否生效

```bash
# 查看缓存统计
resolvectl statistics

# 第一次查询（Cache Miss）
resolvectl query www.baidu.com

# 第二次查询同一域名（应命中缓存）
resolvectl query www.baidu.com

# 再次查看统计
resolvectl statistics
```

期望输出：
```
Cache
  Current Cache Size: 3
          Cache Hits: 1
        Cache Misses: 6
         Cache Hit Rate: 14.3%
```

### 6. 确认重启后持久化

- `systemd-resolved` 默认开机自启
- 软链接 `/etc/resolv.conf` 重启后依然存在
- 配置 `/etc/systemd/resolved.conf` 保持不变

可重启后重复第 6 步验证。

---

## 常见问题

| 问题 | 解决方法 |
|------|----------|
| `resolvectl status` 显示 `resolv.conf mode: missing` | 重新执行第 2 步创建软链接 |
| 缓存命中率始终为 0 | 确认 `Cache=` 参数未被注释；重复查询同一域名 |
| 服务启动失败 | 检查 `resolved.conf` 语法，`journalctl -u systemd-resolved` 查看日志 |
| 想用 dnsmasq 作为更强大的缓存 | 停止 systemd-resolved，安装 dnsmasq，修改 `/etc/resolv.conf` 指向 `127.0.0.1` |

---

## 与 DNS 并发问题的关系

- 缓存启用后，多个程序同时请求同一域名 → 只有第一次走上游，后续命中本地缓存
- 高并发场景（容器、微服务）可显著减少上游 DNS 压力
- 每秒上千个不同域名的极大规模 → 建议改用 `dnsmasq` 并增大 `cache-size`
