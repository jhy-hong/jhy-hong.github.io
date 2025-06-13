Prometheus 是云原生生态中的核心监控工具，其设计理念与实现方式与传统监控系统有很大不同。下面从 **概念、原理、优化策略、查询优化** 等维度进行全面介绍。

------

## 一、Prometheus 相关概念

| 概念                        | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| **时间序列（Time Series）** | 每一条以时间为索引的数据记录，如某台机器的 CPU 使用率。      |
| **指标（Metric）**          | Prometheus 收集的基础单位，如 `http_requests_total`、`node_cpu_seconds_total`。 |
| **标签（Label）**           | 用于区分同一指标的不同维度，如 `method="GET"`、`status="200"`。 |
| **样本（Sample）**          | 某个时间点上的具体值，如时间戳 + 值。                        |
| **Job / Instance**          | `Job` 表示一个逻辑服务，`Instance` 表示某个具体采集地址（主机 + 端口）。 |
| **Exporter**                | 用于暴露 Prometheus 可采集指标的组件，如 Node Exporter、MySQL Exporter。 |
| **Scrape**                  | Prometheus 向目标拉取指标数据的过程。                        |
| **Pushgateway**             | 用于暂存短生命周期任务的指标，推送给 Prometheus。            |
| **PromQL**                  | Prometheus 查询语言，功能强大、用于分析和聚合指标。          |



------

## 二、核心原理

### 1. **拉模式采集（Pull-based）**

Prometheus 默认通过 HTTP GET 请求周期性地“拉取”被监控目标的数据（通过 `/metrics` 接口），不依赖被动接收。

### 2. **数据模型**

Prometheus 的数据模型基于多维度时间序列，每个时间序列由：

```
arduino


复制编辑
<metric_name>{label1="value1", label2="value2", ...}
```

组成，支持丰富的聚合和过滤。

### 3. **本地时序数据库（TSDB）**

- 数据以 block 存储，分段压缩为 chunk
- 存储结构：时间戳 + 值（每个时间序列）
- 持久化存储目录：`/var/lib/prometheus`

### 4. **服务发现（Service Discovery）**

- 支持静态配置、Kubernetes、Consul、EC2 等动态发现目标
- 自动注册 scrape 目标，提高可维护性

------

## 三、优化策略

### 1. **采集优化**

| 策略         | 说明                                               |
| ------------ | -------------------------------------------------- |
| 降低采集频率 | 例如：默认 15s，非关键指标可设置为 60s，减少数据量 |
| 精简指标数量 | 配置 `metric_relabel_configs` 过滤不必要的指标     |
| 限制标签基数 | 避免 high-cardinality label，如 `user_id`、`uuid`  |
| 细分 job     | 不同 job 设置不同 scrape 配置，提高调度效率        |



### 2. **存储优化**

| 策略                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| TSDB retention 时间控制      | 如：保留 15d，而不是默认的永久                               |
| 启用远程存储（Remote Write） | 接入 Thanos、VictoriaMetrics、Cortex 等，实现高可用/长期存储 |
| 使用压缩格式                 | Prometheus 使用 Gorilla 压缩算法，默认已启用，不建议修改     |



### 3. **部署优化**

| 项目                   | 建议                                                 |
| ---------------------- | ---------------------------------------------------- |
| 分片采集               | 一个 Prometheus 实例负责一部分 job，避免单点压力过大 |
| 联邦集群（Federation） | 通过上层 Prometheus 汇聚下游数据                     |
| 异步采集与导入         | 通过 Pushgateway、remote write 缓解采集压力          |



------

## 四、PromQL 查询优化

### 1. **查询基本形式**

```
promql复制编辑http_requests_total{job="web", method="GET"}  # 精确匹配标签
rate(http_requests_total[5m])                 # 计算速率
sum by (method) (rate(http_requests_total[1m])) # 按标签聚合
```

### 2. **查询优化策略**

| 优化点                         | 示例 / 说明                                                  |
| ------------------------------ | ------------------------------------------------------------ |
| 避免不必要的正则匹配           | `{job=~"web.*"}` 尽量用 `{job="web"}` 替代                   |
| 控制时间窗口长度               | `rate(metric[1h])` 比 `rate(metric[5m])` 资源消耗大          |
| 聚合后再做计算                 | 优化表达式顺序，先 `sum` 后 `rate` 通常性能更好              |
| 限制 `label_replace()` 使用    | 该函数开销大，避免频繁使用                                   |
| 使用 `subquery` 控制样本量     | 如：`avg_over_time(rate(metric[1m])[10m:1m])`                |
| 使用 `offset` 避免读取最新数据 | Prometheus TSDB 对“当前时间”的样本查询开销大，可使用 `offset 5s` 延迟处理 |

---

## 六、prmetheus告警的架构

![prometheus告警架构图](https://github.com/user-attachments/assets/c3ac59f6-47da-47ec-af1d-eca96b806ccf)

### 相关组件详解：

-  Prometheus Server：Prometheus 生态最重要的组件，主要用于抓取和存储时间序

列数据，同时提供数据的查询和告警策略的配置管理

-  Alertmanager：Prometheus 生态用于告警的组件，Prometheus Server 会将告警发

送给 Alertmanager，Alertmanager 根据路由配置，将告警信息发送给指定的人或

组。Alertmanager 支持邮件、Webhook、微信、钉钉、短信等媒介进行告警通知；

- Grafana：用于展示数据，便于数据的查询和观测；

- Push Gateway：Prometheus 本身是通过 Pull 的方式拉取数据，但是有些监控数据

可能是短期的，如果没有采集数据可能会出现丢失。Push Gateway 可以用来解决

此类问题，它可以用来接收数据，也就是客户端可以通过 Push 的方式将数据推送

到 Push Gateway，之后 Prometheus 可以通过 Pull 拉取该数据；

- Exporter：主要用来采集监控数据，比如主机的监控数据可以通过 node_exporter

采集，MySQL 的监控数据可以通过 mysql_exporter 采集，之后 Exporter 暴露一个

接口，比如/metrics，Prometheus 可以通过该接口采集到数据；

- PromQL：PromQL 其实不算 Prometheus 的组件，它是用来查询数据的一种语法，

比如查询数据库的数据，可以通过 SQL 语句，查询 Loki 的数据，可以通过 LogQL，

查询 Prometheus 数据的叫做 PromQL；

- Service Discovery：用来发现监控目标的自动发现，常用的有基于 Kubernetes、

Consul、Eureka、文件的自动发现等。