# Linux 云主机监控与告警系统（运维实验项目）

基于 Prometheus + Grafana + Alertmanager 的云主机监控告警实验环境：采集 CPU、内存、磁盘等主机指标与 Nginx 业务指标，配置告警规则，并通过模拟服务中断完成一次完整的「告警触发 → Alertmanager 接收确认 → 告警恢复」故障处理演练（Alertmanager 当前为本地空操作接收器，未接入邮件等外部通知渠道）。

> 对应岗位：云资源日常维护 / 日常监控 / 告警处理 / 故障处理 / 运行质量分析

## 架构

```
                    ┌──────────────────────────────────────────┐
                    │              Docker 网络 monitor-net       │
                    │                                          │
  主机指标 ──────►  │  node-exporter:9100 ──┐                  │
  业务指标 ──────►  │  nginxexporter:9113 ──┼──► prometheus:9090 ──► alertmanager:9093
                    │                       │    │  抓取+规则评估    │  告警分组/抑制
                    │  nginx:80(8080) ◄─────┘    ▼                │
                    │  （模拟业务站点）        grafana:3000          │
                    └──────────────────────────────────────────┘
                                                    │
                                                    ▼
                                            浏览器访问面板/告警页
```

| 服务 | 地址 | 说明 |
| --- | --- | --- |
| Grafana | <http://localhost:3000> | 可视化面板（账号见 `.env`） |
| Prometheus | <http://localhost:9090> | 指标采集与告警规则评估 |
| Alertmanager | <http://localhost:9093> | 告警接收、分组、抑制 |
| Nginx 演示站点 | <http://localhost:8080> | 被监控的模拟业务服务（宿主机端口可用 `.env` 中 `NGINX_PORT` 修改） |
| Node Exporter | 容器网络内部 :9100 | 主机 CPU/内存/磁盘指标，仅供 Prometheus 抓取，不映射宿主机端口 |
| nginx-exporter | 容器网络内部 :9113 | Nginx 连接数/请求量指标，仅供 Prometheus 抓取，不映射宿主机端口 |

## 目录结构

```
cloud-monitor-alert/
├── docker-compose.yml          # 六个容器的编排定义
├── .env.example                # Grafana 口令模板（复制为 .env 使用）
├── LICENSE                     # MIT（含原作者版权与本项目修改声明）
├── prometheus/
│   ├── prometheus.yml          # 抓取周期/告警规则/目标列表
│   └── alerts.yml              # 告警规则（CPU/内存/磁盘/目标失联/Nginx/指标缺失）
├── alertmanager/
│   └── alertmanager.yml        # 告警路由/分组/抑制规则
├── grafana/provisioning/
│   ├── datasources/            # Prometheus 数据源自动注册
│   └── dashboards/             # 监控面板自动加载（免手工导入）
├── nginx/nginx.conf            # 演示站点配置（含 stub_status 指标端点）
├── html-app/index.html         # 演示站点页面
└── docs/evidence/              # 实际运行截图与告警 API 证据（本机实测采集）
```

## 快速开始

前提：已安装 Docker（Windows 用 Docker Desktop，Linux 用 Docker Engine + Compose 插件）。

```bash
cp .env.example .env      # 修改里面的 Grafana 管理密码
docker compose up -d
docker compose ps         # 六个容器应为 running
```

国内镜像拉取慢时，可在 Docker 配置中添加镜像加速地址后重试。

### 安全默认配置（本地演示）

- 所有页面端口仅绑定 `127.0.0.1`，局域网其他机器无法访问；需要局域网演示时去掉 compose 端口映射中的 `127.0.0.1:` 前缀（务必同时设置强管理密码）
- node-exporter 与 nginx-exporter 不映射宿主机端口，Prometheus 通过容器网络抓取
- Grafana 匿名访问默认关闭（未登录访问 API 返回 401）；本地想免登录看面板或截图时，在 `.env` 中设 `GF_ANONYMOUS=true`（只读 Viewer 角色）
- Grafana 管理口令通过 `.env` 的 `GF_ADMIN_PASSWORD` 设置；不创建 `.env` 时为 admin/admin，仅限本机演示使用
- 镜像全部固定具体版本（Prometheus v3.15.0 / Grafana 13.2.2 / Alertmanager v0.34.1 / Node Exporter v1.12.1 / Nginx 1.29.8 / nginx-prometheus-exporter 1.5.0），避免 latest 拉到不兼容的新版本
- **部署到公网/云服务器前必做**：设置强管理密码（不依赖 admin/admin 回退值）、恢复端口绑定检查（仅开放必要端口并配合防火墙/安全组）、为 Alertmanager 接入真实通知渠道

### 第一步：确认指标采集正常

打开 <http://localhost:9090/targets>，四个 job（prometheus / node_exporter / nginx / alertmanager）状态都应为 `UP`。

常用验证查询（Prometheus 页面执行）：

```promql
# CPU 使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 根分区剩余空间百分比
node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100

# Nginx 活跃连接数
nginx_connections_active{job="nginx"}
```

### 第二步：Grafana 自动出图（免手工导入）

`grafana/provisioning/` 已同时配置数据源自动注册与 **Dashboard 自动加载**：

1. 打开 <http://localhost:3000>，无需手工添加数据源或导入面板
2. 首页即可看到自动加载的「云主机监控面板」，包含 8 个面板：
   CPU 使用率、内存使用率、磁盘使用率（各分区）、网络流量、
   Nginx 活跃连接数、Nginx 请求速率、采集目标在线数、Nginx 业务状态
3. Nginx 业务状态面板会随服务启停实时切换「运行中 / 已停止」

如需更详细的主机指标，可另行导入官方模板 1860（Node Exporter Full），本项目的自定义面板已覆盖日常巡检所需的核心指标。

## 告警规则说明

规则文件：`prometheus/alerts.yml`

| 告警名 | 触发条件 | 级别 |
| --- | --- | --- |
| HighCPUUsage | CPU 使用率 > 80% 持续 2 分钟 | warning |
| HighMemoryUsage | 可用内存 < 15% 持续 2 分钟 | warning |
| LowDiskSpace | 根分区剩余 < 20% 持续 5 分钟 | warning |
| TargetDown | 任一抓取目标失联 > 1 分钟（`up == 0`） | critical |
| NginxHighActiveConnections | Nginx 活跃连接 > 500 持续 1 分钟 | warning |
| NginxDown | Nginx 业务不可用（`nginx_up == 0`）持续 1 分钟 | critical |
| NginxMetricsMissing | `nginx_up` 指标断流 > 2 分钟（`absent()`） | warning |

> **实测踩坑记录（面试可讲）**：`up` 只代表"采集链路"存活。只停掉 Nginx 进程、保留 exporter 时，`up{job="nginx"}` 仍然是 1——此时必须看 exporter 探测真实业务后暴露的 `nginx_up` 指标。所以 TargetDown 用 `up` 判断（抓取目标失联），NginxDown 用 `nginx_up` 判断（业务进程不可用），两者互补。

修改规则后不用重启容器，热加载即可：

```bash
curl -X POST http://localhost:9090/-/reload
```

告警状态流转可在 <http://localhost:9090/alerts>（Prometheus 侧）和 <http://localhost:9093>（Alertmanager 侧）查看：`inactive → pending → firing`，恢复后转为 resolved。

> 说明：Alertmanager 默认接收器为本地空操作，仅演示「Prometheus → Alertmanager」的告警流转与状态管理；接入邮件/企业微信/Slack 通知只需在 `alertmanager/alertmanager.yml` 的 receivers 中添加对应配置。

## 故障处理实验（重点）

模拟一次「业务中断 → 告警触发 → 确认 → 恢复 → 告警解除」的完整流程：

```bash
# 1. 制造故障：停止 Nginx 业务容器（exporter 保留运行）
docker compose stop nginx

# 2. nginx_up 变为 0，约 1 分钟后 NginxDown 进入 pending → firing
#    9090/alerts 与 9093 页面均可见（截图：告警 firing + 页面异常）
#    注意：此时 up{job="nginx"} 仍为 1，TargetDown 不会触发——这正是 up 与 nginx_up 的区别

# 3. 若想触发 TargetDown，可改为停止导出器本身：
#    docker compose stop nginxexporter

# 4. 恢复服务
docker compose start nginx

# 5. 约一分钟后告警自动 resolved（截图：恢复后的告警状态与 Grafana 曲线回稳）
```

本仓库已完成一轮完整实测：`stop nginx` → `NginxDown pending → firing → Alertmanager active` → `start nginx` → `nginx_up=1 → 告警解除`，全链路验证通过。

完整跑一遍并保留：**Grafana 面板截图、firing 告警截图、resolved 恢复截图**。

## 实际运行证据

`docs/evidence/` 保存了本机（Windows 11 + Docker Desktop + WSL2）完整实测的材料，全部来自真实运行，非手工绘制：

| 文件 | 内容 |
| --- | --- |
| `prometheus-targets-all-up.png` | 四个抓取目标全部 UP |
| `grafana-dashboard-all-normal.png` | 自动加载的 8 面板监控面板（Nginx 运行中） |
| `prometheus-alerts-nginxdown-firing.png` | 场景1：停止 Nginx 后 NginxDown 进入 FIRING |
| `alertmanager-nginxdown-active.png` | 场景1：Alertmanager 收到告警（active） |
| `api-alerts-nginxdown-firing.json` / `api-alertmanager-nginxdown.json` | 告警 API 原始返回 |
| `prometheus-alerts-resolved.png` | 恢复 Nginx 后告警全部解除 |
| `prometheus-targets-nginxexporter-down.png` | 场景2：停止导出器后 nginx 目标 DOWN |
| `prometheus-alerts-targetdown-metricsmissing.png` | 场景2：TargetDown 与 NginxMetricsMissing 同时 FIRING |
| `api-alerts-targetdown-firing.json` | 场景2 告警 API 原始返回 |
| `grafana-dashboard-nginx-stopped.png` | 故障期间面板（Nginx 指标断流） |

三场景实测结论：

1. **停止 Nginx 业务容器** → `nginx_up=0` → NginxDown 约 1 分钟后 pending → firing，Alertmanager 同步接收；此时 `up{job="nginx"}` 仍为 1，TargetDown 不触发（符合设计）
2. **停止 Nginx Exporter** → `up=0` 触发 TargetDown；同时 `nginx_up` 序列消失，验证了 `nginx_up == 0` 对缺失指标不生效的盲区，`absent(nginx_up)` 触发 NginxMetricsMissing 补位
3. **恢复服务** → 两组场景的告警均在约 1 分钟内自动解除，Prometheus 与 Alertmanager 告警数归零

> 说明：Alertmanager 当前为本地空操作接收器，仅验证「Prometheus → Alertmanager」的告警流转与状态管理，未接入真实邮件/Webhook 通知。

## 相对参考仓库的修改点

结构参考 [durrello/prometheus-grafana-docker](https://github.com/durrello/prometheus-grafana-docker)（MIT License），本项目为学习目的的复刻与改造，主要改动：

1. **修正 Node Exporter 宿主机挂载**：原仓库未挂载 `/`、`/proc`、`/sys`，磁盘等宿主机指标采集不完整；已补齐挂载并增加 `--path.rootfs` 参数（注意：Windows Docker Desktop 下采集到的是 Docker 虚拟机指标，Linux 服务器上才是真实宿主机数据）
2. **修正 NginxDown 告警表达式**：实测发现只停 Nginx 进程时 `up` 仍为 1（exporter 存活），原逻辑无法发现业务故障；改为基于 `nginx_up` 判断业务可用性
3. **新增指标缺失盲区告警**：`NginxMetricsMissing`（`absent(nginx_up)`），覆盖 exporter 进程挂掉、指标序列整体消失而 `== 0` 类规则无法触发的情况（已实测触发）
4. **新增 Grafana 全自动 provisioning**：数据源自动注册（显式 uid）+ 自定义 8 面板 Dashboard 自动加载，clone 后 `docker compose up -d` 即出图，无需手工导入
5. **新增 Nginx 业务告警组**（NginxDown / NginxHighActiveConnections），把"目标失联"翻译成业务语言
6. **开启 `--web.enable-lifecycle`**：修改规则文件后可热重载，不用重启容器
7. **Nginx 宿主机端口可配置**（`NGINX_PORT`，默认 8080），避免与宿主机上已有服务冲突
8. **告警注释中文化**、`for` 时长按实验场景调优，并标注了生产环境应如何取值
9. **安全默认配置**：页面端口仅绑定 `127.0.0.1`、两个 exporter 不映射宿主机端口、Grafana 匿名访问默认关闭（`GF_ANONYMOUS` 可开）、管理口令经 `.env` 注入
10. **镜像版本固定**：六个组件全部固定具体版本，避免 latest 意外升级引入不兼容变更

## 常见问题

- **端口冲突**：确认宿主机 3000/8080/9090/9093 未被占用，或修改 compose 端口映射（Nginx 端口可用 `.env` 的 `NGINX_PORT`）
- **磁盘指标看不到真实数据**：Docker Desktop（Windows/macOS）下 node-exporter 采集的是虚拟机指标，属预期现象；部署到 Linux 云主机即为真实数据
- **告警一直 inactive**：阈值未达到属正常；想快速看到效果可临时调低阈值、缩短 `for` 时长后热重载
- **Grafana 启动后不断重启（Datasource provisioning error）**：旧版本 grafana-data 卷中残留的自动 uid 数据源与新的 `uid: prometheus` 冲突；执行 `docker compose down -v` 清空演示数据卷后重新 `up -d` 即可（会丢失历史指标，实验环境无影响）
- **磁盘面板在 Docker Desktop 下显示多个挂载点**：VM 内根文件系统挂载点与 Linux 服务器不同（如 `/mnt`），面板已按文件系统类型过滤展示全部真实分区；在真实服务器上即显示 `/` 等标准挂载点
- **`LowDiskSpace` 告警在 Docker Desktop 下不触发**：规则限定 `mountpoint="/"`（生产语义），VM 内无该挂载点；如需在实验环境演示磁盘告警，可将 `mountpoint` 改为 `/mnt` 后热重载

## 开源来源与许可证

本项目结构参考 [durrello/prometheus-grafana-docker](https://github.com/durrello/prometheus-grafana-docker)（MIT License，Copyright (c) 2026 Durrell Gemuh）复刻并修改，原作者版权声明与许可证全文保留于 [LICENSE](LICENSE)，本项目的修改部分以独立声明标注，同样以 MIT 协议分发。组件本身（Prometheus、Grafana、Alertmanager、Node Exporter、Nginx）为各自官方开源镜像，非本人开发。

## 简历写法参考

> **Linux 云主机监控与告警系统（运维实验项目）**：基于 Docker Compose 编排 Prometheus / Grafana / Alertmanager / Node Exporter 构建监控环境，采集主机与 Nginx 业务指标；编写 CPU/内存/磁盘/服务可用性告警规则并接入 Alertmanager 完成分组与抑制；通过模拟业务容器中断完成告警触发、确认与恢复的全流程故障处理演练，输出监控面板与告警记录。

如实区分：环境部署、规则编写与调整、故障演练是本人完成；组件本身为开源软件，参考仓库见上方链接（MIT License，感谢原作者 durrello）。
