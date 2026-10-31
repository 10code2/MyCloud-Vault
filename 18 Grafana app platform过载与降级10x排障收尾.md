# 18 Grafana app platform 过载与降级 10.x 排障收尾

> **服务器**：华为云 Flexus 2核2G · CentOS 7.9 · Docker 24.0.9
> **服务**：Grafana 监控（`https://monitor.cloud321.cloud`）
> **当前状态**：✅ **已彻底解决**（2026-10-31 12:07）。Grafana 从 11.x（latest）降级到 10.4.10 LTS，gpx_ 子进程从 13 个降到 0，grafana RSS 从 185M 降到 57M，启动就绪从 40 秒降到 5 秒。monitor.cloud321.cloud 正常访问，Node Exporter Full 面板数据正常显示。
> **后续**：事故闭环后对 9 容器做全业务只读排查，结论与 3 个隐患处置方案见 [[19 服务器全业务只读排查与隐患处置方案]]。
> **前序**：[[16 Grafana公网暴露触发IO饱和排障实录]]（事故前半：3000 公网暴露止血 + 插件清理）→ [[17 Grafana插件自动重装与止血收尾排障续录]]（事故中半：backgroundinstaller 禁用 + plugins 清空，22:12 阶段一闭合）
> **本篇解决**：阶段一闭合后，实际访问 monitor 发现负载**再次爆满**——根因不是插件、不是 preinstall，而是 **Grafana 11+ app platform 的 gpx_ 子进程架构在 2G 内存机器上结构性过载**。最终通过降级到 Grafana 10.4.10 彻底解决。
> **最后更新**：2026-10-31

---

## 一、基础概念：app platform、gpx_ 进程与页缓存颠簸

### 1.1 Grafana 11+ app platform 架构

Grafana 11 引入了 **app platform**（应用平台）架构：每个**带 backend 的数据源插件**（Prometheus、Loki、Elasticsearch、MySQL、PostgreSQL、Tempo 等 13 个）在 Grafana 启动时**都会 fork 一个独立的 Go 子进程**，不管你是否使用这个数据源。

```
Grafana 11+ 进程树：
grafana（主进程）
├── gpx_grafana-pro      ← Prometheus 数据源 backend
├── gpx_grafana-pyr      ← Pyroscope
├── gpx_grafana-ela      ← Elasticsearch
├── gpx_grafana-pos      ← PostgreSQL
├── gpx_grafana-lok      ← Loki
├── gpx_grafana-jae      ← Jaeger
├── gpx_grafana-mss      ← MSSQL
├── gpx_grafana-mys      ← MySQL
├── gpx_grafana-tem      ← Tempo
├── gpx_grafana-ope      ← OpenTSDB
├── gpx_grafana-inf      ← InfluxDB
├── gpx_grafana-zip      ← Zipkin
└── gpx_grafana_clo      ← CloudWatch
```

**关键认知**：这些 gpx_ 进程的代码在 Grafana **镜像内部**，不在数据卷的 plugins 目录里。清空 plugins 目录、禁用 preinstall **都不能阻止它们启动**。

### 1.2 gpx_ 进程名规律

进程名 `gpx_` 是 Grafana Plugin Executor 的缩写。命名规律：`gpx_` + 插件 ID 的缩写（取前 3 个字母）：

| 进程名 | 对应插件 | 说明 |
|---|---|---|
| `gpx_grafana-pro` | prometheus | Prometheus 数据源 |
| `gpx_grafana-pyr` | pyroscope | Pyroscope 性能分析 |
| `gpx_grafana-ela` | elasticsearch | ES 数据源 |
| `gpx_grafana-pos` | postgresql | PostgreSQL 数据源 |
| ... | ... | 共 13 个 |

### 1.3 页缓存颠簸（thrashing）——2G 内存的致命伤

Go 程序的二进制文件（50~100MB）通过 `mmap` 映射到进程地址空间。访问代码页时产生缺页，内核从磁盘读入。这部分映射页**不计入 RSS**（只有实际驻留的页才算 RSS），但**每次被换出后再访问就要冷读磁盘**。

13 个 gpx_ 子进程 = 13 个 Go 二进制，mmap 映射面合计约 **700MB~1.4GB**——远超 2G 机器的页缓存容量（当前 buff+cache 才 350~420M）。

**恶性循环链条**：
```
内存不足 → kswapd 回收页缓存 → gpx_ 代码页被换出
→ 下次访问触发缺页冷读 → IO 拥塞（wa 95%+）
→ 所有进程排队等 IO → 负载飙升 → 更多内存压力
→ 页缓存进一步缩小 → 更多冷读 → 恶性循环
```

**RSS 不是真正的杀手——mmap 冷读面才是。** gpx_ 进程总 RSS 只有 51.8MB（每个约 4M），看起来不大，但 mmap 映射面才是 IO 压力的真正来源。

### 1.4 Grafana 10.x vs 11.x 架构对比

| 维度 | Grafana 10.x | Grafana 11+ |
|---|---|---|
| 进程架构 | 单进程 | 主进程 + 13 个 gpx_ 子进程 |
| 数据源 backend | 主进程内执行 | 独立 Go 子进程 |
| app platform | 无 | 有（gpx_ + bleve + cleanup 后台任务） |
| 推荐内存 | 2G 可运行 | 官方推荐 4G+ |
| 启动就绪 | 5 秒 | 40 秒+（需 fork 13 个子进程） |

---

## 二、目标清单

| # | 目标 | 状态 |
|---|---|---|
| 1 | 定位阶段一闭合后负载再次爆满的根因 | ✅ gpx_ 子进程 + 页缓存颠簸 |
| 2 | 尝试禁用 gpx_ 进程（GF_PLUGINS_DISABLE_PLUGINS） | ✅ 13→2，但不够 |
| 3 | 确认禁用后实际访问 dashboard 的负载表现 | ✅ wa 95%+，判断"2 个可接受"是错的 |
| 4 | 决策：降级 10.x vs 升配 4G | ✅ 选降级（不花钱，用户授权直接操作） |
| 5 | SSH 直连服务器执行降级 | ✅ paramiko + SFTP，15 分钟完成 |
| 6 | 配置数据源 + 导入 dashboard | ✅ Prometheus + Node Exporter Full |
| 7 | 全套验证通过 | ✅ gpx_=0, RSS 57M, curl 0.001s |

---

## 三、实操步骤：降级 Grafana 10.4.10 全流程

### 步骤 0：止血（kill grafana 让服务器恢复）

- **目标**：服务器负载降到正常，为后续操作创造条件
- **操作**：`kill -9 $(pgrep -f grafana)`（绕过 docker stop，IO 拥塞下 docker stop 会卡死）
- **预期结果**：gpx_ 进程归零，负载 1 分钟内降到 1 以下
- **验证**：`uptime` + `ps -eo comm | grep -c gpx_`

### 步骤 1：备份

- **目标**：保留 11.x 的数据库和 compose，确保可回滚
- **操作**：
  ```bash
  cp /var/lib/docker/volumes/monitor_grafana-data/_data/grafana.db \
     /var/lib/docker/volumes/monitor_grafana-data/_data/grafana.db.bak-v11
  cp /opt/cloudlab/monitor/docker-compose.yml \
     /opt/cloudlab/monitor/docker-compose.yml.bak-before-downgrade
  ```
- **验证**：`ls -la` 确认两个备份文件存在

### 步骤 2：修改 compose

- **目标**：镜像从 `latest` 改为 `10.4.10`，删除 11.x 专属的环境变量
- **改动**：
  ```yaml
  grafana:
    image: grafana/grafana:10.4.10    # ← 从 latest 改为 10.4 LTS
    # 删除 environment 段（GF_PLUGINS_PREINSTALL_DISABLED + GF_PLUGINS_DISABLE_PLUGINS）
    #   10.x 没有 app platform，不需要这些
    container_name: grafana
    restart: unless-stopped
    ports:
      - '3000:3000'
    volumes:
      - grafana-data:/var/lib/grafana
  ```
- **验证**：`grep grafana /opt/cloudlab/monitor/docker-compose.yml` 确认镜像版本

### 步骤 3：删旧容器 + 删旧 db

- **目标**：11.x 的 grafana.db schema 与 10.x 不兼容，需删旧 db 让 10.x 创建新库
- **操作**：
  ```bash
  docker rm -f grafana
  rm -f /var/lib/docker/volumes/monitor_grafana-data/_data/grafana.db \
        /var/lib/docker/volumes/monitor_grafana-data/_data/grafana.db-journal
  ```
- **注意**：旧 db 已在步骤 1 备份为 `.bak-v11`，删除安全

### 步骤 4：拉取镜像并启动

- **目标**：docker compose 拉取 10.4.10 镜像并启动 grafana
- **操作**：
  ```bash
  cd /opt/cloudlab/monitor && docker compose up -d grafana
  ```
- **预期**：镜像约 108MB（比 11.x 小），拉取 + 启动约 2-3 分钟
- **验证**：`docker ps --filter name=grafana` 确认 `Up` 且镜像为 `grafana/grafana:10.4.10`

### 步骤 5：等待就绪

- **目标**：grafana 10.x HTTP 服务就绪
- **操作**：
  ```bash
  for i in $(seq 1 12); do
    sleep 5
    echo -n "第$((i*5))s: "
    curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" --max-time 5 http://127.0.0.1:3000/
  done
  ```
- **预期**：第 5 秒出现 `302 0.00xs`（10.x 启动很快，不像 11.x 要 40 秒）

### 步骤 6：配置 Prometheus 数据源

- **目标**：grafana 能查询 prometheus 的监控数据
- **操作**：
  ```bash
  curl -s -X POST -u "admin:admin" -H "Content-Type: application/json" \
    -d '{"name":"Prometheus","type":"prometheus","url":"http://prometheus:9090","access":"proxy","isDefault":true}' \
    http://127.0.0.1:3000/api/datasources
  ```
- **预期**：返回 JSON，包含 `"id":1,"name":"Prometheus","type":"prometheus"`
- **注意**：URL 用 `http://prometheus:9090`（容器名），因为 grafana 和 prometheus 在同一个 Docker 网络（monitor_default）

### 步骤 7：修改 admin 密码

- **操作**：
  ```bash
  curl -s -X PUT -u "admin:admin" -H "Content-Type: application/json" \
    -d '{"oldPassword":"admin","newPassword":"你的密码","confirmNew":"你的密码"}' \
    http://127.0.0.1:3000/api/user/password
  ```
- **预期**：返回 `{"message":"User password changed"}`

### 步骤 8：导入 Node Exporter 面板

- **操作**：浏览器登录 Grafana → Dashboards → Import → 输入 **1860** → Load → 选择 Prometheus 数据源 → Import
- **预期**：Node Exporter Full 面板显示 CPU/Memory/Disk/Network 监控图表

### 步骤 9：全套验证

| 验证项 | 命令 | 预期 |
|---|---|---|
| gpx_ 进程数 | `ps -eo comm \| grep -c gpx_` | **0**（10.x 没有 app platform） |
| grafana 版本 | `docker exec grafana grafana --version` | `10.4.10` |
| 容器状态 | `docker ps --filter name=grafana` | `Up` + `grafana/grafana:10.4.10` |
| curl 计时 | `curl -s -o /dev/null -w "%{http_code} %{time_total}s" http://127.0.0.1:3000/` | `302 0.001s` |
| grafana RSS | `ps -eo rss,comm \| grep grafana` | ~57M（单进程） |
| 负载 | `uptime` | < 1 |
| 数据源 | `curl -s -u "admin:密码" http://127.0.0.1:3000/api/datasources` | 包含 Prometheus |

### 回滚（仅当验证失败）

```bash
cp /opt/cloudlab/monitor/docker-compose.yml.bak-before-downgrade /opt/cloudlab/monitor/docker-compose.yml
cp /var/lib/docker/volumes/monitor_grafana-data/_data/grafana.db.bak-v11 \
   /var/lib/docker/volumes/monitor_grafana-data/_data/grafana.db
cd /opt/cloudlab/monitor && docker compose up -d grafana
```

---

## 四、根因链条回顾：三层根因的层层剥离

整个事故有三层根因，每解决一层才暴露下一层：

```
第一层（16号已解决）：3000 端口公网暴露 → 被境外扫描 → apiserver/cleanup/bleve
                      反复唤醒读插件文件 → IO 饱和
  解决：安全组 + iptables DROP 3000
  ↓ 解决后发现

第二层（17号已解决）：backgroundinstaller 自动重装插件 → plugins 膨胀到 429M
                      → grafana 启动时扫描插件目录 → IO 尖峰
  解决：GF_PLUGINS_PREINSTALL_DISABLED=true + 清空 plugins 目录
  ↓ 解决后发现

第三层（本篇解决）：Grafana 11+ app platform → 13 个 gpx_ 子进程
                    → mmap 冷读面 700MB+ → 2G 内存页缓存颠簸 → IO 拥塞
  解决：降级到 Grafana 10.4.10（单进程架构，无 gpx_）
```

**每一步在当时看都是合理的**——第一层和第二层的问题确实存在、确实在造成 IO 压力。但根因的第三层直到实际访问 dashboard 才暴露。

---

## 五、踩坑记录与关键经验

### 踩坑表

| # | 问题现象 | 根因 | 解决办法 |
|---|---|---|---|
| 1 | 阶段一闭合后访问 monitor，负载又爆（load 4.12, wa 83%） | 13 个 gpx_ 子进程的 mmap 冷读 | 降级 10.x |
| 2 | grafana 日志显示登录耗时 9-14 秒 | sqlite 在 IO 压力下查询极慢 | 降级后 IO 解除，查询秒回 |
| 3 | 日志显示 `identity.not-found` 被误读为"密码错误" | 用户名输入有误，非数据库问题 | 用户纠正用户名 |
| 4 | `GF_PLUGINS_DISABLE_PLUGINS` 禁用列表写了 `grafana-pyroscope-app` 但 gpx_grafana-pyr 仍在 | 进程名 `pyr` 对应的插件 ID 可能是 `pyroscope`（datasource）而非 `grafana-pyroscope-app`（app） | 禁用列表 ID 与实际进程对应的插件 ID 不匹配 |
| 5 | 禁用到 2 个 gpx_，判断"可接受"后实际访问 dashboard wa 95%+ | 基于启动后 5 分钟无人访问的乐观数据下结论，没有实际使用场景验证 | **稳态指标好 ≠ 负载下好，必须在实际使用场景验证** |
| 6 | 监控面板显示 grafana RSS 132.9M，命令行 `ps` 显示 51.8M | 132.9M 是 grafana 主进程，51.8M 是 13 个 gpx_ 子进程总和，测的是不同对象 | 真实总内存 = 主进程 + 子进程 |
| 7 | phase2 脚本 `set -euo pipefail` 下 grep 无匹配返回 1 终止脚本 | gpx_=0 时 `grep gpx_` 返回退出码 1，pipefail 传播，set -e 终止 | 去掉 set -e，grep/ps 管道加 `\|\| true` |
| 8 | docker compose up 在 IO 拥塞下被 canceled | dockerd→containerd 链路在 IO 拥塞下阻塞 | kill -9 绕过 docker stop，等负载恢复再操作 |

### 三条关键经验

#### 经验 1：先评估架构适配性，再深入配置排查

遇到资源类问题（IO 拥塞、内存不足），**第一步应该问"这个软件/版本在这台机器上跑合理吗"**，而不是直接深入到配置层面（删插件、禁用进程）做减法。

本次事故中，如果一开始就评估"Grafana 11+ 官方推荐 4G，这台机器只有 2G 且跑了 9 个容器"，可能更早得出降级结论，省掉中间几个小时的插件清理和 gpx_ 禁用尝试。

**判据**：当排障进入第三轮仍在同一层面打转时，跳到架构层重新审视。

#### 经验 2：稳态指标好 ≠ 负载下好

"启动后 5 分钟无人访问"测到的 wa 0%、curl 0.001s，不能代表"实际使用时没问题"。**必须在真实使用场景（打开 dashboard、查询数据）下验证**，才能确认问题解决。

**判据**：验证必须覆盖实际使用场景，不能只测空载状态。登录、打开面板、查询数据三个动作至少各做一次。

#### 经验 3：RSS 不等于磁盘读面（mmap 冷读面）

Go 程序的 RSS（实际驻留内存）远小于 mmap 映射面。13 个 gpx_ 进程 RSS 才 51.8M，但 mmap 映射面合计 700MB+，在 2G 机器上远超页缓存容量。

**判据**：评估进程的 IO 压力，不能只看 RSS，要看 mmap 映射的二进制大小 × 进程数。`pmap -x <pid> | tail -1` 可以看总映射面。

---

## 六、降级前后对比数据

| 指标 | Grafana 11.x（latest） | Grafana 10.4.10 | 改善 |
|---|---|---|---|
| gpx_ 子进程 | 13 个 | **0** | 砍掉全部 |
| grafana RSS | 185M（主 132.9M + 子 51.8M） | **57M** | -69% |
| 启动就绪 | 40 秒+ | **5 秒** | -87% |
| curl 计时（本机） | 0.97-1.0 秒（负载高时） | **0.001 秒** | -99.9% |
| available 内存 | 232M | **308M** | +33% |
| 负载（访问 dashboard 时） | 16+（wa 95%+） | **0.25** | 完全解决 |
| 镜像大小 | ~300MB | **108MB** | -64% |
| 后台任务 | cleanup + bleve + usage stats + plugins.update.checker | 仅基础后台 | 大幅减少 |

---

## 七、项目不足与后续扩展

### 7.1 当前局限

| 项 | 说明 | 紧急度 |
|---|---|---|
| iptables DROP 3000 未持久化 | 运行时规则，重启丢失；但安全组已封 3000，公网安全无影响 | 低 |
| ports 仍 0.0.0.0 直绑 | 3000 端口仍对 0.0.0.0 绑定；iptables 拦截公网但内部仍可达 | 低 |
| 旧 grafana:latest 镜像未清理 | 占磁盘空间（~300MB） | 低 |

### 7.2 后续可做

1. **iptables 持久化**：`iptables-save > /etc/sysconfig/iptables`，确保重启不丢规则
2. **ports→expose 改造**：取消 3000 的 0.0.0.0 直绑，NPM 共享 Docker 网络 + 反代改容器名（阶段二端口收口）
3. **清理旧镜像**：`docker rmi grafana/grafana:latest` 释放磁盘
4. **清理 plugins.bak-manual**：确认稳定后删除 429M 的旧插件备份
5. **公安备案**：截止 2026-11-28

### 7.3 架构层面的反思

这次事故暴露了一个更深层的问题：**2核2G 机器上跑了 9 个容器（npm、portal、vaultwarden、alist、halo、grafana、prometheus、node-exporter、uptime-kuma），内存本就紧张**。Grafana 11+ 的 app platform 架构只是压垮骆驼的最后一根稻草。

长期来看，如果服务数量继续增长，需要考虑：
- 升级服务器到 4G（最彻底）
- 合并轻量服务（如 node-exporter + prometheus 合并到一个容器）
- 对内存大户（如 Halo）设置更严格的内存限制

---

## 八、面试话术

### 1 分钟口述版本

"我做了一个云服务器监控平台，用 Grafana + Prometheus + node-exporter。上线后遇到一个严重的 IO 拥塞问题：Grafana 3000 端口被公网扫描，触发 IO 饱和。止血后深入排查发现三层根因：第一层是端口暴露，第二层是插件自动重装，第三层是 Grafana 11+ 的 app platform 架构——13 个 gpx_ 子进程的 mmap 冷读面在 2G 内存机器上导致页缓存颠簸。最终通过降级到 Grafana 10.4.10 LTS 彻底解决，gpx_ 进程从 13 降到 0，RSS 从 185M 降到 57M。"

### 高频问答

**Q：为什么不用最新版 Grafana？**
A：Grafana 11+ 引入了 app platform 架构，每个带 backend 的数据源插件会 fork 一个独立 Go 子进程，13 个子进程的 mmap 映射面合计 700MB+，远超 2G 机器的页缓存容量。10.4 LTS 是单进程架构，资源占用更低，在这台机器上是正确的版本选择。

**Q：怎么定位到 gpx_ 进程是根因的？**
A：通过 `ps -eo pid,stat,wchan,comm` 发现 13 个 `gpx_grafana-*` 子进程。搜索 Grafana 文档确认这是 app platform 的插件执行器。然后看 vmstat 发现即使不访问，wa 也周期性飙到 90%+，`cache` 列在持续缩小——典型的页缓存颠簸。关键是认识到 RSS 51.8M 不大，但 mmap 冷读面才是 IO 压力的真正来源。

**Q：为什么不用 GF_PLUGINS_DISABLE_PLUGINS 禁用？**
A：试过了，从 13 个降到 2 个。但实际访问 dashboard 时 wa 仍然 95%+——2 个 gpx_ 的 mmap 冷读面 + grafana 主进程 + 后台任务，在 2G 机器上还是会导致页缓存颠簸。禁用是治标，降级才是治本。

**Q：最大的教训是什么？**
A：遇到资源类问题，第一步应该评估"这个软件版本在这台机器上跑合理吗"，而不是直接深入到配置层面做减法。如果一开始就评估 Grafana 11+ 需要 4G 内存，而这台机器只有 2G，可能更早得出降级结论，省掉中间几个小时的弯路。

---

## 九、历史备查

### 9.1 时间线

| 时间 | 事件 |
|---|---|
| 10-30 22:12 | 阶段一闭合（17 号结尾）：plugins 清空 + preinstall 禁用，grafana 秒回 302 |
| 10-30 22:38 | 实际访问 monitor，负载又爆（load 4.12, wa 83%） |
| 10-30 22:49 | 诊断：发现 13 个 gpx_ 进程，grafana 日志显示登录请求 9-14 秒 |
| 10-30 22:59 | 搜索确认 Grafana 11+ app platform 机制（ComputingForGeeks 实测数据） |
| 10-30 23:00 | phase2 脚本：禁用 12 个 gpx_（GF_PLUGINS_DISABLE_PLUGINS） |
| 10-30 23:01 | phase2 执行被 IO 拥塞中断（docker compose up canceled） |
| 10-31 10:06 | 重试 phase2，成功但 gpx_ 降到 2（pyroscope ID 不匹配） |
| 10-31 10:28 | 复测：wa 0%，curl 0.001s → 判断"2 个可接受" |
| 10-31 10:36 | 实际访问 dashboard → wa 95%+ → **判断错误** |
| 10-31 10:42 | 确认根因：2G 内存页缓存颠簸，架构选型与资源不匹配 |
| 10-31 10:47 | 用户提供 SSH 凭据，授权直接操作 |
| 10-31 11:08 | kill grafana 止血 |
| 10-31 11:55 | 降级 Grafana 10.4.10 全流程开始 |
| 10-31 11:57 | 全套验证通过：gpx_=0, RSS 57M, curl 0.001s |
| 10-31 12:07 | 用户确认 monitor.cloud321.cloud 正常访问，Node Exporter Full 面板数据正常 |

### 9.2 SSH 直连操作方式

本次降级首次采用了"助手 SSH 直连服务器"的操作模式（之前是用户手动执行 + 贴输出）。使用 Python paramiko 库通过 SSH 连接服务器，SFTP 上传 compose 文件，exec_command 执行命令。全程约 15 分钟完成降级，对比之前手动方式（每个诊断轮次 5-10 分钟），效率提升显著。

### 9.3 关键搜索来源

- ComputingForGeeks：Grafana 内置插件 backend 进程的实测数据（RSS 692M→382M，禁用后 gpx_ 从 13 降到 1）
- Grafana 官方文档：`GF_PLUGINS_DISABLE_PLUGINS` 配置项
- Grafana 社区：app platform 架构说明、gpx_ 进程名规律

### 9.4 备份文件清单

| 文件 | 内容 | 位置 |
|---|---|---|
| `grafana.db.bak-v11` | Grafana 11.x 的数据库（2.0M，已无用，保留备查） | 数据卷 `_data/` |
| `docker-compose.yml.bak-before-downgrade` | 降级前的 compose（含 11.x 环境变量） | `/opt/cloudlab/monitor/` |
| `plugins.bak-manual` | 429M 旧插件目录（来自 17 号手动 mv） | 数据卷 `_data/` |
| `docker-compose.yml.bak-*`（3 个） | 历次 compose 修改的备份 | `/opt/cloudlab/monitor/` |
