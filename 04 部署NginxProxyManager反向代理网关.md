
> 项目：CloudLab 云平台 · 服务 1
> 环境：华为云 Flexus L 实例 · 2核2G · CentOS 7.9 · 公网 IP ipv4
> 日期：2026-10-18

---

## 一、项目介绍：NPM 是什么

Nginx Proxy Manager（简称 NPM）是一个**可视化的 Nginx 反向代理管理面板**。它底层就是 Nginx，但把配置反向代理、申请 HTTPS 证书、管理多站点等操作都做成了网页界面，不用手写 Nginx 配置文件。

### 它解决什么问题

一台服务器只有一个 80 端口和一个 443 端口，但我们要在上面跑**多个服务**（密码管理、博客、监控、云盘……）。不可能让多个服务同时占用 80/443。

NPM 统一占用 80/443，作为所有请求的**总入口**，然后根据请求的**域名**，把流量转发到不同后端服务：

```
用户访问 vault.cloud321.cloud
    ↓ 请求到达 NPM（443端口）
    ↓ NPM 看域名是 vault，转发到 Vaultwarden 容器
用户访问 blog.cloud321.cloud
    ↓ NPM 看域名是 blog，转发到博客容器
```

### 在 CloudLab 中的位置

NPM 是整个平台的**第一层（接入层）**，后面所有服务都在它的后面，通过子域名对外暴露：

```
                    ┌─────────────────────┐
   用户请求 ──────→  │ NPM 反向代理网关     │
                    │ (80/443，统一入口)  │
                    └──────────┬──────────┘
           ┌────────┬──────────┼──────────┬────────┐
           ↓        ↓          ↓          ↓        ↓
       Vaultwarden  Uptime    Grafana    Halo    Alist
                   Kuma
```

---

## 二、要做什么（任务清单）

| 步骤 | 任务 |
|---|---|
| 1 | 创建 NPM 项目目录 |
| 2 | 编写 `docker-compose.yml`（镜像、端口、数据卷） |
| 3 | 拉取镜像并启动容器 |
| 4 | 放行管理后台 81 端口 |
| 5 | 登录管理后台，修改默认密码 |
| 6 | 验证容器运行状态 |

**最终效果**：
- 管理后台能通过 `http://ipv4:81` 访问
- 后续可以在面板里点鼠标配置其他服务的反代和 HTTPS

---

## 三、可能遇到的问题（预先了解）

| 问题 | 原因 | 解决方法 |
|---|---|---|
| 镜像拉取超时 | 国内访问 Docker Hub 慢，加速地址没配对 | 配置华为云 SWR 镜像加速器（见 03 笔记） |
| 管理后台打不开 | 81 端口未在安全组/iptables 放行 | 放行 81 端口 |
| 不知道默认密码 | NPM 有默认账号 | `admin@example.com` / `changeme` |
| 提示 `version is obsolete` | Compose v2 不要求 version 字段 | 警告可忽略，或删掉 version 行 |

---

## 四、实际部署过程

### 4.1 创建项目目录

在 `/opt/cloudlab/` 下为每个服务创建独立目录（NPM 是第一个）：

```bash
mkdir -p /opt/cloudlab/{npm,uptime-kuma,vaultwarden,grafana,prometheus,halo,alist,backup}
cd /opt/cloudlab/npm
```

### 4.2 编写 docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
EOF
```

### 配置逐项说明

| 配置 | 说明 |
|---|---|
| `image: jc21/nginx-proxy-manager` | NPM 官方镜像 |
| `container_name: npm` | 容器名 |
| `restart: unless-stopped` | 崩溃自动重启（除非手动停止） |
| `ports: 80:80` | HTTP 入口（网站访问） |
| `ports: 81:81` | **管理后台**（配置用） |
| `ports: 443:443` | HTTPS 入口 |
| `volumes: ./data:/data` | NPM 配置、数据库持久化 |
| `volumes: ./letsencrypt:/etc/letsencrypt` | HTTPS 证书持久化 |

> 💡 **为什么要挂载数据卷**：容器一旦删除，里面的数据就没了。挂载到宿主机目录后，配置、数据库、证书都存在宿主机，即使容器重建也不丢失。

### 4.3 第一次启动：镜像拉取失败

执行启动命令：
```bash
docker-compose up -d
```

**报错**：
```
✘ app Error Get "https://registry-1.docker.io/v2/":
  net/http: request canceled while waiting for connection (30.4s)
Error response from daemon: Get "https://registry-1.docker.io/v2/":
  net/http: request canceled while awaiting headers
```

**排查过程**：
1. 最初配置的加速地址是 daocloud 和 dockerproxy，但请求最终还是走到了官方地址 `registry-1.docker.io` 超时——说明这些公开加速地址在华为云环境下不可用
2. 尝试换成 1panel.live、hub.rat.dev 等公开地址，效果都不稳定
3. 最终意识到应该用**华为云自己的 SWR 镜像加速器**（走内网）

**解决**：在华为云控制台获取专属 SWR 加速器地址，配置到 `/etc/docker/daemon.json`（完整过程见 03 笔记第六章），重启 Docker 后重新拉取。

> 这个问题的详细排查和解决过程属于"Docker 环境配置"，已经记录在 03 笔记中。

### 4.4 第二次启动：成功

镜像加速配置好后重新启动：
```bash
docker-compose up -d
```

**拉取结果**：
```
[+] Running 6/6
 ✔ Pull complete
 ✔ Container npm Started
```

验证容器状态：
```bash
docker-compose ps
```

**结果**：
```
NAME  IMAGE                             STATUS  PORTS
npm   jc21/nginx-proxy-manager:latest   Up      0.0.0.0:80-81->80-81/tcp,
                                                0.0.0.0:443->443/tcp
```

容器状态 `Up`，80/81/443 三个端口都正确映射。

> 警告 `version is obsolete`：Compose v2 已经不要求 version 字段，不影响运行，可以忽略。如果想消除警告，删掉配置文件里的 `version: '3.8'` 行即可。

### 4.5 管理后台无法访问

**现象**：浏览器访问 `http://ipv4:81`，浏览器提示响应超时、转圈。

**排查思路**：
1. 先确认容器在跑（`docker-compose ps` 显示 Up）——容器没问题
2. 外部访问超时，大概率是**端口没放行**

**根因**：之前初始化时只在 iptables 和华为云安全组放行了 22/80/443，没有放行 **81** 端口。

**解决**：

iptables 放行：
```bash
iptables -I INPUT -p tcp --dport 81 -j ACCEPT
service iptables save
```

华为云安全组放行：
1. 控制台 → 服务器 → 安全组 → 入方向规则
2. 添加规则：TCP，端口 81，源 0.0.0.0/0

### 4.6 登录并修改默认密码

放行后访问 `http://ipv4:81`，使用默认账号登录：

- **邮箱**：`admin@example.com`
- **密码**：`changeme`

**修改密码**：登录后没有自动弹出修改提示，手动点击右上角头像 → 编辑详情：
- 把邮箱改成自己的（如 QQ@qq.com）
- 密码改成自己的强密码

修改后用新账号重新登录。

---

## 五、验证清单

```bash
# 1. 容器状态（应为 Up）
cd /opt/cloudlab/npm
docker-compose ps

# 2. 端口映射（80/81/443）
docker port npm

# 3. 容器日志（应有启动成功信息）
docker logs npm --tail 10

# 4. 浏览器访问管理后台
# http://ipv4:81
```

---

## 六、踩坑记录

| 问题 | 报错 | 根因 | 解决 |
|---|---|---|---|
| 镜像拉取失败 | request canceled awaiting headers | 公开加速地址不可用 | 配华为云 SWR 内网加速器（见03） |
| 管理后台打不开 | 响应超时 | 81 端口未放行 | iptables + 安全组放行 81 |
| 不知道登录密码 | 登录失败 | NPM 默认账号 | admin@example.com / changeme |
| 提示 version obsolete | 警告 | Compose v2 不要求 version | 忽略或删 version 行 |
| 后端端口直绑公网被扫描 | 整机 wa 飙高、IO 拥塞 | 后端 `ports: '端口:端口'` 默认绑 `0.0.0.0`，绕过 NPM 直接暴露；2026-10-30 Grafana 3000 即因此被境外直连打满 | 生产态后端改 `expose`、接入 NPM 共享网络用容器名直连；完整排障见 [[16 Grafana公网暴露触发IO饱和排障实录]]，端口收敛理念见 [[02 NPM反向代理网关扩展]] §3.6 |

---

## 七、下一步

NPM 网关就绪后，接下来部署**第一个真实业务服务 Vaultwarden 密码管理**，然后在 NPM 面板里为它配置反代和 HTTPS（见 05 笔记）。

---

## 八、面试话术

> "我在云服务器上部署了 Nginx Proxy Manager 作为统一的反向代理网关。因为一台服务器只有一个 80/443 端口，需要通过 NPM 基于子域名把请求分发到不同的后端服务。我用 docker-compose 编排容器，把配置和 Let's Encrypt 证书目录挂载到宿主机做持久化。部署时遇到了两个问题：一是镜像拉取超时，最终通过配置华为云内网镜像加速器解决；二是管理后台 81 端口访问不通，排查发现是安全组和 iptables 没放行。后续所有服务都通过 NPM 配置反代和自动 HTTPS。"
