

> 项目：CloudLab 云平台 · 服务 2（第一个真实业务服务）
> 环境：华为云 Flexus L 实例 · 2核2G · CentOS 7.9
> 访问地址：https://vault.cloud321.cloud
> 日期：2026-10-18

---

## 一、项目介绍：Vaultwarden 是什么

Vaultwarden 是一个**自托管密码管理器**，是 Bitwarden 官方服务端的 Rust 重写版本，兼容所有 Bitwarden 官方客户端（浏览器扩展、手机 App、桌面端）。

### 它解决什么问题

密码太多记不住，记在纸上/浏览器里不安全。密码管理器可以：
- 把所有网站密码加密存储
- 浏览器自动填充
- 生成随机强密码
- 多设备同步

### 核心安全机制：零知识架构

这是 Vaultwarden 最关键的设计——**服务器永远看不到明文密码**：

```
手机/电脑客户端
    ↓ 输入主密码
    ↓ 本地用 AES-256 加密所有密码
    ↓ 只把加密后的密文发给服务器
Vaultwarden 服务器
    ↓ 只存密文，不知道主密码，解不开任何数据
```

即使服务器被入侵，攻击者拿到的也只是加密数据，没有主密码根本解不开。

### Vaultwarden vs Bitwarden 官方

| 对比 | Vaultwarden | Bitwarden 官方 |
|---|---|---|
| 数据存储 | 存在自己服务器 | 存在 Bitwarden 云 |
| 资源占用 | 极轻（~10MB 内存） | 官方服务端较重 |
| 客户端 | 兼容官方客户端 | 官方客户端 |
| 费用 | 服务器费用 | 基础免费 |

**为什么自托管**：数据完全自己掌控，同时部署过程是很好的运维项目。

---

## 二、要做什么（任务清单）

| 步骤 | 任务 |
|---|---|
| 1 | 创建 Vaultwarden 目录和配置文件 |
| 2 | 启动容器 |
| 3 | 放行 8080 端口 |
| 4 | 解决 HTTPS 安全上下文问题（购买域名 + 解析） |
| 5 | 在 NPM 中配置反代和 HTTPS 证书 |
| 6 | 注册自己的账号 |
| 7 | 关闭公开注册（安全加固） |

**最终效果**：
- 通过 `https://vault.cloud321.cloud` 访问密码管理
- 浏览器扩展/手机 App 可连接同步

---

## 三、可能遇到的问题（预先了解）

| 问题 | 原因 | 解决 |
|---|---|---|
| `no configuration file` | 在错误目录执行 docker-compose | cd 到含 yml 的目录 |
| curl 返回 000 | 容器没启动 | 检查配置文件并启动 |
| 8080 访问超时 | 端口未放行 | iptables + 安全组放行 |
| 提示非安全上下文 | HTTP 下 Web Crypto 受限 | 配置域名 + HTTPS |
| ping 域名失败 | DNS 缓存/解析未生效 | nslookup 验证，flushdns |

---

## 四、实际部署过程

### 4.1 第一次部署：在错误目录执行

最开始没有先创建配置文件，直接在 `/etc/yum.repos.d` 目录执行：
```bash
docker-compose ps
```

**报错**：
```
no configuration file provided: not found
```

**根因**：`docker-compose` 必须在**包含 docker-compose.yml 的目录**下运行，它默认在当前目录找配置文件。

用 curl 测试端口：
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
# 000
```

**000 表示连接失败**（不是 HTTP 状态码，也不是重定向），因为容器根本没启动、8080 没有服务在监听。

### 4.2 重新创建配置并启动

```bash
# 创建目录并进入
mkdir -p /opt/cloudlab/vaultwarden
cd /opt/cloudlab/vaultwarden

# 创建配置文件
cat > docker-compose.yml << 'EOF'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    ports:
      - '8080:80'
    volumes:
      - ./data:/data
    environment:
      - SIGNUPS_ALLOWED=true
      - WEBSOCKET_ENABLED=true
EOF

# 启动
docker-compose up -d
```

**启动结果**：
```
[+] Running 6/6
 ✔ vaultwarden Pulled (5.9s)
 ✔ Container vaultwarden Started
```

验证：
```bash
docker-compose ps
```

**结果**：
```
NAME          IMAGE                       STATUS                 PORTS
vaultwarden   vaultwarden/server:latest   Up (health: starting)  0.0.0.0:8080->80/tcp
```

### 配置逐项说明

| 配置 | 说明 |
|---|---|
| `ports: 8080:80` | 容器内 80 端口映射到宿主机 8080 |
| `volumes: ./data:/data` | 密码数据库持久化 |
| `SIGNUPS_ALLOWED=true` | 先开放注册，注册完账号后改成 false |
| `WEBSOCKET_ENABLED=true` | 启用实时同步（反代需配 WebSocket） |

### 4.3 8080 端口访问超时

**现象**：浏览器访问 `http://ipv4:8080` 提示响应时间过长。

**根因**：8080 端口没有在 iptables 和华为云安全组放行。

**解决**：
```bash
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
service iptables save
```
华为云安全组入方向同时添加 8080 端口。

### 4.4 HTTPS 安全上下文问题

端口放行后能打开网页，但出现提示：
```
没使用安全上下文，而 API 的正常运行需要使用 HTTPS
```

**根因**：Vaultwarden 使用浏览器的 **Web Crypto API** 做客户端加密，现代浏览器要求安全上下文（HTTPS 或 localhost），纯 HTTP 下加密 API 被限制。

**要解决必须配置 HTTPS**，而 HTTPS 证书需要一个域名，所以接下来要购买域名并配置反代。

---

### 4.5 购买域名

**个人持有 vs 企业持有**：
- 个人持有：身份证认证，可做个人备案，适合个人项目
- 企业持有：营业执照认证，可做经营性网站

选择**个人持有**（个人运维项目完全够用，以后可过户）。

**后缀选择**（首年/续费）：
- .top：14 / 39（性价比高）
- .cn：29-38 / 35
- .com：35-76 / 70-80（最通用）
- 注意避坑：.site 首年 12 但续费 125

最终选择域名：**cloud321.cloud**

> 优惠口令主要用于续费，首年注册用不上，直接在活动页下单即可。

### 4.6 域名解析

在阿里云域名控制台配置：

**泛解析**（所有子域名生效）：

| 记录类型 | 主机记录 | 记录值 |
|---|---|---|
| A | `*` | ipv4 |

**根域名**：

| 记录类型 | 主机记录 | 记录值 |
|---|---|---|
| A | `@` | ipv4 |

泛解析让 `vault.`、`status.`、`blog.` 等所有子域名自动指向服务器。

**验证**：
```bash
nslookup vault.cloud321.cloud 223.5.5.5
# Address: ipv4
```
> 直接查阿里 DNS（223.5.5.5）绕过本地缓存。本地 ping 不到可执行 `ipconfig /flushdns` 刷新。

---

### 4.7 在 NPM 中配置反代 + HTTPS

登录 NPM 管理后台（`http://ipv4:81`）→ Proxy Hosts → Add Proxy Host。

**Details 标签页**：

| 配置 | 值 |
|---|---|
| Domain Names | `vault.cloud321.cloud` |
| Scheme | `http` |
| Forward Hostname/IP | `172.17.0.1` |
| Forward Port | `8080` |
| Block Common Exploits | 勾选 |

> `172.17.0.1` 是 Docker 网桥网关，NPM 容器通过它访问宿主机上的 8080 端口。

**SSL 标签页**：

| 配置 | 值 |
|---|---|
| SSL Certificate | Request a new SSL Certificate |
| Force SSL | 勾选 |
| HTTP/2 Support | 勾选 |
| Websockets Support | **勾选**（实时同步需要） |
| Email | 填邮箱 |
| I Agree | 勾选 |

保存后 NPM 自动向 Let's Encrypt 申请证书并完成配置。

### 4.8 验证 HTTPS 并注册账号

浏览器访问 `https://vault.cloud321.cloud`：
- 首次加载稍慢（HTTPS 握手）
- 出现 Vaultwarden 登录页
- 地址栏显示小锁

点"创建账号"，输入邮箱 + 强主密码（12 位以上，大小写+数字+符号），注册后登录。

### 4.9 关闭公开注册

自己注册完成后，为防止陌生人注册，修改配置：
```bash
cd /opt/cloudlab/vaultwarden
sed -i 's/SIGNUPS_ALLOWED=true/SIGNUPS_ALLOWED=false/' docker-compose.yml
docker-compose up -d
```

现在只有你的账号能登录。

---

## 五、客户端配置（后续完成）

Vaultwarden 兼容 Bitwarden 官方客户端：
- 浏览器扩展（Chrome/Edge/Firefox 搜 Bitwarden）
- 手机 App
- 桌面端

**关键**：登录前在设置里把服务器地址改成 `https://vault.cloud321.cloud`，否则会连到官方服务器。

---

## 六、验证清单

```bash
# 1. 容器运行（Up）
cd /opt/cloudlab/vaultwarden
docker-compose ps

# 2. 确认已关闭注册
grep SIGNUPS_ALLOWED docker-compose.yml
# SIGNUPS_ALLOWED=false

# 3. 浏览器访问
# https://vault.cloud321.cloud（地址栏小锁）
```

---

## 七、踩坑记录

| 问题 | 报错 | 根因 | 解决 |
|---|---|---|---|
| 找不到配置 | no configuration file | 在错误目录执行 | cd 到项目目录 |
| 端口连不上 | curl 000 | 容器没启动 | 创建配置并启动 |
| 8080 超时 | 无响应 | 端口未放行 | iptables+安全组 |
| 非安全上下文 | HTTPS 警告 | HTTP 下 Web Crypto 受限 | 配域名+HTTPS |
| WebSocket 不工作 | 同步失败 | 反代没开 WebSocket | NPM 勾选 Websockets |
| ping 域名失败 | 找不到主机 | 本地 DNS 缓存 | nslookup验证，flushdns |

---

## 八、下一步

Vaultwarden 密码服务完成，接下来部署 **Uptime Kuma 服务状态监控**（公开状态页，给面试官看），流程类似：容器启动 → 放行端口 → NPM 反代 + HTTPS。

> **备案收尾（2026-10-29 备案通过后）**：`vault.cloud321.cloud` 的 NPM 反代与证书已在 4.7 配置，收尾阶段核对规则生效、Force SSL / WebSocket 正常；客户端服务器地址切到 `https://vault.cloud321.cloud`。通用清单见 [[07 网站ICP备案]] §7.2。Vaultwarden 为自用后台，不做公开导航入口、保持关闭公开注册。

---

## 九、面试话术

> "我在云平台上部署了 Vaultwarden 自托管密码管理器，它采用零知识架构，密码在客户端用 AES-256 加密，服务器只存密文，即使被入侵也无法解密。我通过 docker-compose 部署，配置了数据卷持久化和 WebSocket 实时同步；为了让浏览器的 Web Crypto API 正常工作，我购买了域名并通过 Nginx Proxy Manager 配置了 Let's Encrypt HTTPS 证书；注册账号后关闭了公开注册做安全加固。过程中解决了目录错误、端口放行、安全上下文、WebSocket 代理等问题。"
