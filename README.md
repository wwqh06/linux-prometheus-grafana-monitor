# 基于 Prometheus + Grafana 的 Linux 主机监控与故障告警实践

本项目基于 VMware 虚拟化与 Docker Compose 容器编排技术，快速搭建了一套面向企业级 Linux 服务器的轻量级主机性能监控系统。项目实现了主机底层指标采集、时序存储与动态可视化监控，并通过注入 CPU 突发负载完成了系统动态感知能力的闭环验证。

---

## 目录
- [项目架构](#项目架构)
- [技术栈](#技术栈)
- [核心配置文件](#核心配置文件)
- [快速部署与运行](#快速部署与运行)
- [效果验证与成果展示](#效果验证与成果展示)
- [故障排查与运维实战记录](#故障排查与运维实战记录)
- [项目收益与总结](#项目收益与总结)

---

## 项目架构

本系统由数据采集、时序存储、可视化展示三大核心组件构成：

```text
+-------------------------------------------------------------------------------+
|                      Windows 宿主机 (物理 PC / 浏览器终端)                     |
|                                                                               |
|   浏览器访问 Grafana (3000)                浏览器访问 Prometheus (9090)       |
+-------------------------------------------------------------------------------+
                                      | (VMnet8 虚拟网络 / 局域网端口转发)
                                      v
+-------------------------------------------------------------------------------+
|                     VMware Linux 虚拟机 (CentOS Stream 9)                     |
|                     IP: 192.168.17.128 | 防火墙放行: 9090, 3000, 9100         |
|                                                                               |
|  +-------------------------------------------------------------------------+  |
|  |                           Docker Compose 容器集群                        |  |
|  |                                                                         |  |
|  |  +-------------------+      拉取指标      +--------------------------+  |  |
|  |  |    Prometheus     | <----------------- |      node_exporter       |  |  |
|  |  |    (端口 9090)    |  (每15秒定时抓取)   |       (端口 9100)        |  |  |
|  |  +-------------------+                    +--------------------------+  |  |
|  |            |                                            |               |  |
|  |            | 提供数据查询                               | 挂载读取      |  |
|  |            v                                            v               |  |
|  |  +-------------------+                    +--------------------------+  |  |
|  |  |      Grafana      |                    |  Linux 宿主机内核与系统  |  |  |
|  |  |    (端口 3000)    |                    |  /proc, /sys, /rootfs    |  |  |
|  |  +-------------------+                    +--------------------------+  |  |
|  +-------------------------------------------------------------------------+  |
+-------------------------------------------------------------------------------+
```

### 组件分工与交互
1. **Node Exporter (数据采集员)**：以只读方式挂载 Linux 宿主机的 `/proc`、`/sys` 内核目录，采集并暴露真实的物理与操作系统指标（端口：`9100`）。
2. **Prometheus (时序数据库)**：基于 Pull 模型，每隔 15 秒向 Node Exporter 发起 HTTP GET 请求抓取最新时序指标并入库（端口：`9090`）。
3. **Grafana (可视化大盘)**：接入 Prometheus 作为默认数据源，执行 PromQL 查询，将时序数据渲染为多维动态监控图表（端口：`3000`）。

---

## 技术栈

| 领域 | 选用技术 / 工具 | 说明 |
| :--- | :--- | :--- |
| **基础环境** | CentOS Stream 9 (Kernel 5.14) | 运行于 VMware Workstation (NAT 模式) |
| **容器运行时** | Docker CE 27.x | 容器引擎与隔离环境 |
| **服务编排** | Docker Compose v2.x | 基础设施即代码 (IaC) 一键拉起多容器栈 |
| **监控套件** | Prometheus + Node Exporter | 开源时序监控指标收集生态 |
| **可视化展示** | Grafana 11.x | 仪表盘渲染引擎 (模板 ID: 1860) |

---

## 核心配置文件

### 1. `prometheus.yml`
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "linux_host"
    static_configs:
      - targets: ["node_exporter:9100"]
```

### 2. `docker-compose.yml`
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:z
      - prometheus_data:/prometheus

  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    restart: always
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  prometheus_data:
  grafana_data:
```

---

## 快速部署与运行

### 1. 准备目录与配置
```bash
mkdir -p /opt/monitor
cd /opt/monitor
# 将上述 prometheus.yml 与 docker-compose.yml 保存至该目录下
```

### 2. 启动监控集群
```bash
docker compose up -d
```

### 3. 放行宿主机防火墙端口
```bash
firewall-cmd --zone=public --add-port={9090,3000,9100}/tcp --permanent && firewall-cmd --reload
```

### 4. 访问服务
- **Prometheus Targets 状态**：`http://<虚拟机IP>:9090/targets`
- **Grafana 仪表盘**：`http://<虚拟机IP>:3000` (初始账号: `admin` / 密码: `admin`)
  - 数据源 URL: `http://prometheus:9090`
  - 导入仪表盘模板 ID: `1860` (Node Exporter Full)

---

## 效果验证与成果展示

### 1. Prometheus 采集链路验证 (Targets UP)
Prometheus 成功与 Node Exporter 握手，所有抓取目标均处于健康的 `UP` 状态：


### 2. Grafana 主机性能监控大盘 (Node Exporter Full)
全方位实时监测主机 CPU 核心数、可用物理内存、磁盘利用率及网卡收发吞吐量：


### 3. CPU 压力模拟与动态恢复验证
通过启动多重循环脚本模拟主机突发高负载场景，监控大盘精准捕获 CPU 从 5% 飙升至 100% 的陡峭曲线，并在终止压测后平稳复原，验证了监控系统秒级感知的有效性与真实性：


---

## 故障排查与运维实战记录

在实际部署与调优过程中，攻克了以下关键网络与权限问题，形成了可复用的排障方案：

1. **虚拟机网卡 DHCP 无法获取 IP**
   - **问题现象**：网卡显示“正在连接”，终端执行 ping 提示“网络不可达”。
   - **原因分析**：虚拟机原为桥接模式，个人 Wi-Fi 路由环境限制多 MAC 地址请求或未桥接到正确物理网卡。
   - **解决方案**：切换为 VMware NAT 模式，复用宿主机网络通道，并通过 `sudo nmcli connection up ens33` 重新激活网卡获取内网 IP。
2. **普通用户权限受限 (sudoers 缺失)**
   - **问题现象**：`user1 未出现在 sudoers 文件中`。
   - **解决方案**：使用 root 用户执行 `usermod -aG wheel user1`，将普通账号加入 CentOS 默认管理员用户组 `wheel`，实现标准规范的运维提权。
3. **跨系统代理穿透与 Docker 镜像拉取加速**
   - **问题现象**：国内公共镜像站偶发 502 Bad Gateway，官方镜像源跨国拉取被限速至 30KB/s。
   - **原因分析**：Windows 宿主机代理工具默认未接管 VMware 虚拟网卡（VMnet8）流量，虚拟机无法享受主机代理链路。
   - **解决方案**：在 Windows 宿主机代理软件中开启 TUN 虚拟网卡模式，实现网络层对虚拟机流量的全局接管，拉取速率提升至 20MB/s 以上。
4. **单核 CPU 满载对服务响应的影响**
   - **问题现象**：压测期间 Grafana 右上角出现持续旋转的 `Cancel` 状态。
   - **原因分析**：测试机仅分配了 1 核 vCPU，压测进程占满全部计算资源，导致 Prometheus 查询 API 响应延迟。执行 `pkill -f cpu_test.sh` 释放资源后系统立即恢复秒级响应。

---

## 项目收益与总结

- 掌握了轻量级可复用的 **Docker Compose 基础设施即代码 (IaC)** 交付流程。
- 深入理解了 **Pull 监控模型**、`/proc` 内核文件系统映射机制及时序数据可视化链路。
- 积累了真实的 Linux 网络排障、防火墙配置及宿主机-虚拟机代理穿透运维经验。
