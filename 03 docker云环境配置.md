# 03 Docker 云环境配置

> 定位：从裸机到能跑 Docker 容器的完整初始化流程
> 环境：华为云 Flexus L 实例 · 2核2G · CentOS 7.9 · 公网 IP ipv4
> 日期：2026-10-18

---

## 一、整体目标

| 维度        | 说明                                                            |
| --------- | ------------------------------------------------------------- |
| **What**  | 把新买的云服务器从裸机初始化为 Docker 运行环境                                   |
| **Why**   | 后面所有服务（博客/密码管理/监控）都跑在 Docker 容器里，这一步是地基                       |
| **Where** | 华为云 ECS，系统 CentOS 7.9，操作目录 `/etc/yum.repos.d/`、`/etc/docker/` |
| **How**   | 换源 → 防火墙 → SELinux → 装 Docker → 配加速 → Compose                 |

---

## 二、换阿里云 yum 源

### 为什么要换源

CentOS 7 已于 **2024 年 6 月 30 日停止维护（EOL）**，官方源 `mirror.centos.org` 已失效，不换源的话 `yum install` 会报 404。

### 操作步骤

```bash
cd /etc/yum.repos.d/
mkdir -p backup
mv *.repo backup/

# 下载阿里云源 + EPEL 扩展源
curl -o CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
curl -o epel.repo https://mirrors.aliyun.com/repo/epel-7.repo

# 重建缓存
yum clean all && yum makecache
```

### 验证

```bash
yum repolist
# 应显示 base / epel / extras / updates，共约 30000+ 个包
```

### 安装基础工具

```bash
yum install -y vim wget curl net-tools lsof bash-completion
```

---

## 三、防火墙：firewalld → iptables

### 为什么不用 firewalld

Docker 和 firewalld 在 CentOS 7 上有**兼容性问题**：firewalld 会接管 iptables 规则，干扰 Docker 的容器网络（bridge、端口映射），导致容器网络异常。iptables 更底层、更稳定。

### 操作步骤

```bash
# 关闭并禁用 firewalld
systemctl stop firewalld
systemctl disable firewalld

# 安装并启动 iptables
yum install -y iptables-services
systemctl start iptables
systemctl enable iptables

# 放行关键端口
iptables -I INPUT -p tcp --dport 22 -j ACCEPT   # SSH
iptables -I INPUT -p tcp --dport 80 -j ACCEPT   # HTTP
iptables -I INPUT -p tcp --dport 443 -j ACCEPT  # HTTPS

# 持久化（否则重启丢失）
service iptables save
```

### 验证

```bash
iptables -L -n | head -10
```

> [!note]
> 华为云的**安全组**已经在云平台层面做了一层端口过滤，服务器内部的 iptables 是第二层防护。学习环境保持默认即可，生产环境建议精细化配置。

---

## 四、关闭 SELinux

### 为什么要关

SELinux（Security-Enhanced Linux）是 CentOS 的强制访问控制（MAC）机制，安全但配置复杂。它会**阻止 Docker 容器读写挂载的宿主机目录**，报 `permission denied`。学习环境先关闭减少排障成本。

### 操作步骤

```bash
# 临时关闭（立即生效，重启后恢复）
setenforce 0

# 永久关闭（重启后生效）
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

### 验证

```bash
getenforce
# 应显示 Permissive（临时关闭）或 Disabled（永久关闭）
```

> [!tip] 面试加分
> 可以补一句："生产环境不建议直接关闭 SELinux，应该通过 `chcon` 或 `semanage` 配置正确的安全上下文，或者给 Docker 挂载目录加 `:z` 标签。"——显得你知道权衡，不是只会一刀切。

---

## 五、安装 Docker CE

### 为什么用指定版本 24.0.9

Docker 官方从 2024 年起不再支持 CentOS 7，最新版装不上。**24.0.9 是最后一个支持 CentOS 7 的版本**。

### 操作步骤

```bash
# 1. 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 2. 添加阿里云 Docker CE 源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 3. 安装指定版本
yum install -y docker-ce-24.0.9 docker-ce-cli-24.0.9 containerd.io

# 4. 启动 + 开机自启
systemctl start docker
systemctl enable docker
```

---

## 六、配置镜像加速 + 日志轮转

### 为什么要配

1. **镜像加速**：国内直连 Docker Hub 很慢甚至超时，必须用国内镜像站
2. **日志轮转**：Docker 容器默认日志无大小限制，跑久了会把磁盘写满

### 第一次配置（公开加速地址，后来失败）

最开始配置了 DaoCloud 和 dockerproxy 两个公开加速地址：

```bash
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

systemctl daemon-reload
systemctl restart docker
```

### 部署服务时拉取镜像报错

后来部署第一个服务（NPM）时，拉取镜像超时：

```
Get "https://registry-1.docker.io/v2/": net/http: request canceled
while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

**根因**：配置的公开加速地址（daocloud、dockerproxy）在华为云网络环境下不可用，无法代理到 Docker Hub，请求最终还是走到了官方地址 `registry-1.docker.io` 然后超时。

**尝试过的其他公开加速地址**（效果都不稳定）：
- https://docker.1panel.live
- https://hub.rat.dev
- https://docker.anyhub.us.kg
- https://dockerhub.icu

### 最终方案：华为云 SWR 专属镜像加速器

**关键思路**：用哪家云的服务器，就优先用哪家云的镜像服务。华为云服务器访问华为云 SWR 走**内网**，不占用公网带宽，机房直连，速度最快最稳。

**获取地址**：
1. 登录华为云控制台 → 容器镜像服务 SWR → 镜像加速器
2. 复制专属地址（每个账号一个）：
   ```
   https://38b2244086554648b2278911c98a11fd.mirror.swr.myhuaweicloud.com
   ```

**最终配置**（华为云加速器放第一位，公开地址作为备选）：

```bash
cat > /etc/docker/daemon.json << 'EOF'
{
  "registry-mirrors": [
    "https://38b2244086554648b2278911c98a11fd.mirror.swr.myhuaweicloud.com",
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

systemctl daemon-reload
systemctl restart docker
```

**效果**：拉取速度从原来的 2M 飙到 **460M**，走内网完全起飞。

**配置后必须验证**——写错一个字符，Docker 不会报错，只会**静默跳过该镜像源直连官方源**：

```bash
docker info | grep -A3 "Registry Mirrors"   # 必须出现你的 SWR 地址
docker pull nginx:1.27-alpine               # 实测拉一个小镜像
```

> ⚠️ 踩过的坑：SWR 地址里 `myhuaweicloud.com` 前面那段是 **32 位十六进制**，手工抄写极易重复或漏字符（本笔记早期版本就写成了 48 位）。地址非法时的报错是 `Get "https://registry-1.docker.io/v2/": context deadline exceeded`——**错误信息指向的是官方源，很容易误判成"加速器失效"，实际是地址写错了**。判断方法：看 `docker info` 里 Registry Mirrors 是否有你配的那一条。

### 配置说明

| 配置项 | 作用 |
|---|---|
| `registry-mirrors` | 镜像加速地址，按顺序尝试 |
| `log-driver: json-file` | 日志格式（默认） |
| `max-size: 10m` | 单个日志文件最大 10MB |
| `max-file: 3` | 最多保留 3 个日志文件，轮转覆盖 |

> [!tip] 经验
> 1. 云服务器优先用同厂商的镜像服务，内网最快
> 2. 公开加速地址容易失效，只能做备选
> 3. 拉取到一半中断没关系，已下载的镜像层会缓存，重新拉取断点续传

### 验证

```bash
docker --version
# Docker version 24.0.9

docker info | grep -A5 "Registry Mirrors"
# 确认华为云加速器在列

docker run --rm hello-world
# 看到 "Hello from Docker!" 即成功
```

---

## 七、Docker Compose

### 为什么用 Compose

后面要同时跑 5+ 个容器，手动 `docker run` 太麻烦。Compose 用一个 YAML 文件定义所有服务（镜像、端口、数据卷、依赖关系），一条命令启动全部。

### 安装方式

Docker 24.0.9 **已内置 Compose v2 插件**，无需单独安装：

```bash
docker compose version
# Docker Compose version v2.27.1
```

使用方式：`docker compose up -d`（无横杠）

### 兼容旧写法

为了兼容网上教程的 `docker-compose`（有横杠）写法，做软链接：

```bash
ln -s /usr/libexec/docker/cli-plugins/docker-compose /usr/local/bin/docker-compose
docker-compose --version
```

> [!warning] 踩坑记录
> 尝试通过 `curl` 从 GitHub 下载 docker-compose 二进制失败（加速地址 `ghfast.top` 不可用），最终发现 Docker 已内置 Compose 插件，无需单独下载。**优先检查内置插件，不要重复造轮子。**

---

## 八、验证清单

初始化完成后，逐项确认：

```bash
# 1. 系统版本
cat /etc/redhat-release
# CentOS Linux release 7.9.2009 (Core)

# 2. yum 源
yum repolist | tail -1
# repolist: 30562

# 3. 防火墙
iptables -L -n | grep -E 'dpt:(22|80|443)'

# 4. SELinux
getenforce
# Permissive

# 5. Docker
docker --version && docker run --rm hello-world

# 6. Compose
docker compose version
```

全部通过 = 环境就绪，可以开始部署应用服务。

---

## 九、踩坑记录

| 问题 | 原因 | 解决 |
|---|---|---|
| `yum install` 报 404 | CentOS 7 官方源已 EOL | 换阿里云源 |
| Docker 最新版装不上 | Docker 官方已停止支持 CentOS 7 | 指定安装 24.0.9 |
| docker-compose 下载失败 | GitHub 加速地址 ghfast.top 不可用 | 用 Docker 内置的 Compose 插件 |
| 镜像拉取超时 request canceled | 公开加速地址不可用，回退到官方源 | 换华为云 SWR 内网加速器 |
| 报错指向 `registry-1.docker.io` 且 `context deadline exceeded` | **加速器地址写错**（如 ID 重复成 48 位）或加速器未生效，Docker 静默回退官方源 | `docker info \| grep -A3 "Registry Mirrors"` 核对；地址与华为云控制台逐字符比对 |
| 容器挂载目录 permission denied | SELinux 阻止 | `setenforce 0` 或挂载加 `:z` |
| 容器日志把磁盘写满 | 默认无日志大小限制 | `daemon.json` 配 `max-size`/`max-file` |

---

## 十、面试话术

> "我在华为云 CentOS 7.9 服务器上完成了 Docker 运行环境的初始化：针对 CentOS 7 EOL 后官方源失效的问题，替换为阿里云镜像源并引入 EPEL 扩展源；考虑到 Docker 与 firewalld 的网络兼容性，切换为 iptables 管理防火墙并配置了 22/80/443 端口规则；安装 Docker CE 24.0.9（最后一个支持 CentOS 7 的版本），通过 daemon.json 配置了 DaoCloud 镜像加速器和容器日志轮转（10MB×3）防止磁盘占满；使用 Docker Compose v2 进行多容器编排。"
