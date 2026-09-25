# Linux 云主机监控与告警系统（运维实验项目）

基于 Prometheus + Grafana + Alertmanager 的云主机监控告警实验环境：采集 CPU、内存、磁盘等主机指标与 Nginx 业务指标，配置告警规则，并通过模拟服务中断完成一次完整的「告警触发 → 通知 → 恢复」故障处理演练。

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

| 服务 | 端口 | 说明 |
| --- | --- | --- |
| Grafana | <http://localhost:3000> | 可视化面板（账号见 `.env`） |
| Prometheus | <http://localhost:9090> | 指标采集与告警规则评估 |
| Alertmanager | <http://localhost:9093> | 告警接收、分组、抑制 |
| Node Exporter | <http://localhost:9100> | 主机 CPU/内存/磁盘指标 |
| Nginx 演示站点 | <http://localhost:8080> | 被监控的模拟业务服务（宿主机端口可用 `.env` 中 `NGINX_PORT` 修改） |
| nginx-exporter | <http://localhost:9113> | Nginx 连接数/请求量指标 |

## 目录结构

```
cloud-monitor-alert/
├── docker-compose.yml          # 六个容器的编排定义
├── .env.example                # Grafana 口令模板（复制为 .env 使用）
├── prometheus/
│   ├── prometheus.yml          # 抓取周期/告警规则/目标列表
│   └── alerts.yml              # 告警规则（CPU/内存/磁盘/目标失联/Nginx）
├── alertmanager/
│   └── alertmanager.yml        # 告警路由/分组/抑制规则
├── grafana/provisioning/       # 数据源自动注册（免手工添加）
├── nginx/nginx.conf            # 演示站点配置（含 stub_status 指标端点）
└── html-app/index.html         # 演示站点页面
```

## 快速开始

前提：已安装 Docker（Windows 用 Docker Desktop，Linux 用 Docker Engine + Compose 插件）。

```bash
cp .env.example .env      # 修改里面的 Grafana 管理密码
docker compose up -d
docker compose ps         # 六个容器应为 running
```

国内镜像拉取慢时，可在 Docker 配置中添加镜像加速地址后重试。

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

### 第二步：Grafana 出图

1. 登录 <http://localhost:3000>（账号密码在 `.env`）
2. 数据源 `Prometheus` 已通过 provisioning 自动创建，无需手工添加
3. 左侧菜单 → Dashboards → Import → 输入官方模板 ID **1860**（Node Exporter Full）→ 选择 Prometheus 数据源 → Import
4. 即可看到 CPU、内存、磁盘、网络的完整主机面板，截图保存

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

## 相对参考仓库的修改点

结构参考 [durrello/prometheus-grafana-docker](https://github.com/durrello/prometheus-grafana-docker)（MIT License），本项目为学习目的的复刻与改造，主要改动：

1. **修正 Node Exporter 宿主机挂载**：原仓库未挂载 `/`、`/proc`、`/sys`，磁盘等宿主机指标采集不完整；已补齐挂载并增加 `--path.rootfs` 参数（注意：Windows Docker Desktop 下采集到的是 Docker 虚拟机指标，Linux 服务器上才是真实宿主机数据）
2. **修正 NginxDown 告警表达式**：实测发现只停 Nginx 进程时 `up` 仍为 1（exporter 存活），原逻辑无法发现业务故障；改为基于 `nginx_up` 判断业务可用性
3. **新增 Grafana 数据源自动 provisioning**：启动即用，免手工配置
4. **新增 Nginx 业务告警组**（NginxDown / NginxHighActiveConnections），把"目标失联"翻译成业务语言
5. **开启 `--web.enable-lifecycle`**：修改规则文件后可热重载，不用重启容器
6. **Nginx 宿主机端口可配置**（`NGINX_PORT`，默认 8080），避免与宿主机上已有服务冲突
7. **告警注释中文化**、`for` 时长按实验场景调优，并标注了生产环境应如何取值

## 常见问题

- **端口冲突**：确认宿主机 3000/8080/9090/9093/9100/9113 未被占用，或修改 compose 端口映射
- **磁盘指标看不到真实数据**：Docker Desktop（Windows/macOS）下 node-exporter 采集的是虚拟机指标，属预期现象；部署到 Linux 云主机即为真实数据
- **告警一直 inactive**：阈值未达到属正常；想快速看到效果可临时调低阈值、缩短 `for` 时长后热重载

## 简历写法参考

> **Linux 云主机监控与告警系统（运维实验项目）**：基于 Docker Compose 编排 Prometheus / Grafana / Alertmanager / Node Exporter 构建监控环境，采集主机与 Nginx 业务指标；编写 CPU/内存/磁盘/服务可用性告警规则并接入 Alertmanager 完成分组与抑制；通过模拟业务容器中断完成告警触发、确认与恢复的全流程故障处理演练，输出监控面板与告警记录。

如实区分：环境部署、规则编写与调整、故障演练是本人完成；组件本身为开源软件，参考仓库见上方链接（MIT License，感谢原作者 durrello）。
