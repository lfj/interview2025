# PromQL 与 Prometheus 学习文档

> 面向入门到实战，覆盖 Prometheus 架构与指标类型、PromQL 语法与函数、常用查询示例、记录规则与告警、性能与治理、部署与生态、练习清单。强调时间序列思维与正确用法 vs 误用陷阱。

## 目录
- [Prometheus 基础](#prometheus-基础)
- [指标类型与命名](#指标类型与命名)
- [采集与数据模型](#采集与数据模型)
- [PromQL 基础检索](#promql-基础检索)
- [区间与速率函数](#区间与速率函数)
- [聚合与分组](#聚合与分组)
- [二元运算与向量匹配](#二元运算与向量匹配)
- [常用函数](#常用函数)
- [直方图与分位数](#直方图与分位数)
- [记录规则与告警](#记录规则与告警)
- [实战查询示例](#实战查询示例)
- [Prometheus HTTP API](#prometheus-http-api)
- [误用与陷阱](#误用与陷阱)
- [性能与维护建议](#性能与维护建议)
- [部署与生态](#部署与生态)
- [学习路径与练习](#学习路径与练习)
- [速查清单](#速查清单)

---

## Prometheus 基础
- 组件：Prometheus Server（抓取/存储/查询）、Exporter（暴露指标）、Alertmanager（告警路由）、Pushgateway（短作业）、Grafana（可视化）。
- 拉取模型：Server 按 `scrape_config` 定期访问目标 `/<metrics_path>`（默认 `/metrics`）拉取样本，写入 TSDB。
- 查询入口：内置表达式浏览器、HTTP API、Grafana 数据源。

## 指标类型与命名
- 指标类型：
  - `counter` 累积计数，只增不减（重启会重置）；速率用 `rate/irate`，增量用 `increase`。
  - `gauge` 瞬时量，可增减（内存、并发数、队列长度）。
  - `histogram` 直方图，导出 `*_bucket/sum/count`，适合分布与分位数计算。
  - `summary` 摘要，客户端聚合分位数，跨实例聚合困难，优先推荐 `histogram`。
- 命名约定：`<namespace>_<subsystem>_<metric>_<unit>`，单位明确（`_seconds/_bytes/_total`）。
- 标签最佳实践：用标签描述维度（`job/instance/method/status/route`），避免高基数标签（如用户ID）。

## 采集与数据模型
- 采样频率：`scrape_interval` 决定采样密度；过低影响精度，过高增加存储与查询压力。
- Exporter 常见：`node_exporter`（主机）、`kube-state-metrics`（K8s 对象）、`cAdvisor`（容器）、应用自定义（OpenMetrics）。
- 遥测规范：指标稳定、标签一致、单位统一、避免混合类型；版本升级保持兼容。

## PromQL 基础检索
- 即时选择器（瞬时向量）：返回当前时间点的样本。
  - `up` 检查抓取目标是否存活。
  - `http_requests_total{method="GET",status=~"2.."}` 基于标签过滤。
- 区间选择器（范围向量）：附加 `[5m]` 表示取过去 5 分钟的样本序列，用于速率/增量等计算。

## 区间与速率函数
- `rate(counter[5m])` 计算窗口内平均每秒速率，平滑，适合图表。
- `irate(counter[1m])` 计算最后两个样本的瞬时速率，敏感，适合告警。
- `increase(counter[5m])` 计算窗口增量，适合总量增长评估。
- 窗口建议：趋势图用 5–10 分钟窗口；告警用 1–2 分钟配合 `for` 持续抖动控制。

## 聚合与分组
- 聚合算子：`sum/avg/min/max/count`。
- 分组保留：`sum by (label1, label2) ( ... )` 保留指定维度。
- TopK/BottomK：`topk(n, vector)`、`bottomk(n, vector)`。
- 示例：
```
sum by (instance) (rate(http_requests_total[5m]))
topk(5, sum by (route) (rate(http_requests_total{status=~"5.."}[5m])))
```

## 二元运算与向量匹配
- 运算符：`+ - * / % ^`；比较：`== != > < >= <=`。
- 向量匹配：
  - `on(label)` 依据指定标签对齐；
  - `group_left/group_right` 解决一对多匹配。
- 错误率示例（按服务）：
```
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (service) (rate(http_requests_total[5m]))
```

## 常用函数
- 计数类：`rate/irate/increase`（counter）、`delta/deriv`（gauge 变化量/导数）。
- 区间聚合：`sum_over_time/avg_over_time/min_over_time/max_over_time`。
- 分位数：`histogram_quantile(φ, ...)`（配合 `*_bucket`）；`quantile_over_time(φ, series[5m])`（单条时序）。
- 其他：`absent`（缺失检测）、`clamp_max/clamp_min`（阈值裁剪）、`label_replace`（标签重写）。

## 直方图与分位数
- 正确用法（按路由计算 p95 延迟）：
```
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)
)
```
- 维度选择：仅按需要的维度 `sum by (...)`；必须保留 `le` 标签。
- 反模式：对 `summary` 做跨实例分位数；对直方图丢失 `le`；在低样本窗口求分位数。

## 记录规则与告警
- Recording rules（预计算指标）：简化复杂表达式，提高查询性能。
```
# rules.yml
groups:
- name: app-rules
  rules:
  - record: job:http_requests:rate5m
    expr: sum by (job) (rate(http_requests_total[5m]))
  - record: route:http_request_duration_seconds:p95
    expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))
```
- Alerting rules（告警条件与持续时长）：
```
# alerts.yml
groups:
- name: app-alerts
  rules:
  - alert: HighErrorRate
    expr: (
      sum(rate(http_requests_total{status=~"5.."}[5m])) /
      sum(rate(http_requests_total[5m]))
    ) > 0.05
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "5xx 错误率超过 5%"
      description: "过去5分钟内错误率持续超过阈值"
```
- Alertmanager：去重/抑制/路由（邮件/IM/钉钉/Slack），值班策略与静默窗口。

## 实战查询示例
- 主机 CPU 使用率（node_exporter）：
```
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```
- 主机内存使用率：
```
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```
- 磁盘写入速率：
```
sum by (instance, device) (rate(node_disk_writes_completed_total[5m]))
```
- 网络接收吞吐：
```
sum by (instance) (rate(node_network_receive_bytes_total[5m]))
```
- K8s 容器 CPU（cAdvisor）：
```
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{image!=""}[5m]))
```
- Pod 重启率（kube-state-metrics）：
```
sum by (namespace, pod) (rate(kube_pod_container_status_restarts_total[5m]))
```
- HTTP p95 延迟（按路由）：
```
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))
```
- 错误率（按服务）：
```
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (service) (rate(http_requests_total[5m]))
```

## Prometheus HTTP API

Prometheus 提供了完整的 HTTP API，用于查询指标、管理配置、查看状态等。所有 API 端点都在 `/api/v1/` 路径下。

### 查询 API（Query API）

#### 1. 即时查询（Instant Query）

**端点：** `GET /api/v1/query`

**参数：**
- `query` (string, required): PromQL 查询表达式
- `time` (rfc3339 | unix_timestamp, optional): 查询时间点，默认当前时间
- `timeout` (duration, optional): 查询超时时间，默认由 `-query.timeout` 参数决定

**示例：**

```bash
# 查询当前 CPU 使用率
curl 'http://localhost:9090/api/v1/query?query=1-avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))'

# 查询指定时间点的指标
curl 'http://localhost:9090/api/v1/query?query=up&time=2024-01-15T10:00:00Z'

# 带超时的查询
curl 'http://localhost:9090/api/v1/query?query=up&timeout=10s'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "instance": "localhost:9100",
          "job": "node"
        },
        "value": [1705315200, "1"]
      }
    ]
  }
}
```

#### 2. 范围查询（Range Query）

**端点：** `GET /api/v1/query_range`

**参数：**
- `query` (string, required): PromQL 查询表达式
- `start` (rfc3339 | unix_timestamp, required): 开始时间
- `end` (rfc3339 | unix_timestamp, required): 结束时间
- `step` (duration, required): 查询步长（如 `15s`、`1m`、`5m`）
- `timeout` (duration, optional): 查询超时时间

**示例：**

```bash
# 查询过去 1 小时的 CPU 使用率，每 1 分钟一个数据点
curl 'http://localhost:9090/api/v1/query_range?query=1-avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))&start=2024-01-15T09:00:00Z&end=2024-01-15T10:00:00Z&step=1m'

# 使用 Unix 时间戳
curl 'http://localhost:9090/api/v1/query_range?query=up&start=1705315200&end=1705318800&step=15s'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "instance": "localhost:9100",
          "job": "node"
        },
        "values": [
          [1705315200, "1"],
          [1705315260, "1"],
          [1705315320, "1"]
        ]
      }
    ]
  }
}
```

#### 3. 查询元数据（Query Metadata）

**端点：** `GET /api/v1/query`

**说明：** 使用 `query` 参数为 `__name__` 可以查询所有指标名称

**示例：**

```bash
# 查询所有指标名称
curl 'http://localhost:9090/api/v1/query?query={__name__=~".+"}'
```

### 标签 API（Label API）

#### 1. 获取所有标签名称

**端点：** `GET /api/v1/labels`

**参数：**
- `match[]` (string, optional): 指标选择器，可重复多次
- `start` (rfc3339 | unix_timestamp, optional): 开始时间
- `end` (rfc3339 | unix_timestamp, optional): 结束时间

**示例：**

```bash
# 获取所有标签名称
curl 'http://localhost:9090/api/v1/labels'

# 获取特定指标的标签名称
curl 'http://localhost:9090/api/v1/labels?match[]=http_requests_total'
```

**响应格式：**

```json
{
  "status": "success",
  "data": [
    "instance",
    "job",
    "method",
    "status"
  ]
}
```

#### 2. 获取标签值

**端点：** `GET /api/v1/label/<label_name>/values`

**参数：**
- `match[]` (string, optional): 指标选择器
- `start` (rfc3339 | unix_timestamp, optional): 开始时间
- `end` (rfc3339 | unix_timestamp, optional): 结束时间

**示例：**

```bash
# 获取 job 标签的所有值
curl 'http://localhost:9090/api/v1/label/job/values'

# 获取特定指标的 status 标签值
curl 'http://localhost:9090/api/v1/label/status/values?match[]=http_requests_total'
```

**响应格式：**

```json
{
  "status": "success",
  "data": [
    "200",
    "404",
    "500"
  ]
}
```

### 序列 API（Series API）

#### 1. 查询序列

**端点：** `GET /api/v1/series`

**参数：**
- `match[]` (string, required): 指标选择器，可重复多次
- `start` (rfc3339 | unix_timestamp, required): 开始时间
- `end` (rfc3339 | unix_timestamp, required): 结束时间

**示例：**

```bash
# 查询所有 http_requests_total 的序列
curl 'http://localhost:9090/api/v1/series?match[]=http_requests_total&start=2024-01-15T09:00:00Z&end=2024-01-15T10:00:00Z'

# 查询多个指标
curl 'http://localhost:9090/api/v1/series?match[]=http_requests_total&match[]=up&start=2024-01-15T09:00:00Z&end=2024-01-15T10:00:00Z'
```

**响应格式：**

```json
{
  "status": "success",
  "data": [
    {
      "__name__": "http_requests_total",
      "instance": "localhost:8080",
      "job": "api",
      "method": "GET",
      "status": "200"
    },
    {
      "__name__": "http_requests_total",
      "instance": "localhost:8080",
      "job": "api",
      "method": "POST",
      "status": "200"
    }
  ]
}
```

### 目标 API（Target API）

#### 1. 获取所有目标

**端点：** `GET /api/v1/targets`

**参数：**
- `state` (string, optional): 过滤状态（`active`、`dropped`、`any`），默认 `any`

**示例：**

```bash
# 获取所有目标
curl 'http://localhost:9090/api/v1/targets'

# 只获取活跃目标
curl 'http://localhost:9090/api/v1/targets?state=active'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "activeTargets": [
      {
        "discoveredLabels": {
          "__address__": "localhost:9100",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "job": "node"
        },
        "labels": {
          "instance": "localhost:9100",
          "job": "node"
        },
        "scrapePool": "node",
        "scrapeUrl": "http://localhost:9100/metrics",
        "globalUrl": "http://localhost:9100/metrics",
        "health": "up",
        "lastError": "",
        "lastScrape": "2024-01-15T10:00:00Z",
        "lastScrapeDuration": 0.012,
        "scrapeInterval": "15s",
        "scrapeTimeout": "10s"
      }
    ],
    "droppedTargets": []
  }
}
```

#### 2. 获取目标元数据

**端点：** `GET /api/v1/targets/metadata`

**参数：**
- `match_target` (string, optional): 目标标签选择器
- `metric` (string, optional): 指标名称
- `limit` (integer, optional): 返回结果数量限制

**示例：**

```bash
# 获取所有目标的元数据
curl 'http://localhost:9090/api/v1/targets/metadata'

# 获取特定指标的元数据
curl 'http://localhost:9090/api/v1/targets/metadata?metric=http_requests_total'
```

### 规则 API（Rules API）

#### 1. 获取所有规则

**端点：** `GET /api/v1/rules`

**参数：**
- `type` (string, optional): 规则类型（`alert`、`record`），默认返回所有

**示例：**

```bash
# 获取所有规则
curl 'http://localhost:9090/api/v1/rules'

# 只获取告警规则
curl 'http://localhost:9090/api/v1/rules?type=alert'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "groups": [
      {
        "name": "app-alerts",
        "file": "/etc/prometheus/rules/alerts.yml",
        "interval": 60,
        "rules": [
          {
            "name": "HighErrorRate",
            "query": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) > 0.05",
            "duration": 120,
            "labels": {
              "severity": "warning"
            },
            "annotations": {
              "summary": "5xx 错误率超过 5%"
            },
            "alerts": [
              {
                "labels": {
                  "alertname": "HighErrorRate",
                  "severity": "warning"
                },
                "annotations": {
                  "summary": "5xx 错误率超过 5%"
                },
                "state": "firing",
                "activeAt": "2024-01-15T10:00:00Z",
                "value": "0.08"
              }
            ],
            "health": "ok",
            "evaluationTime": 0.001,
            "lastEvaluation": "2024-01-15T10:00:00Z"
          }
        ]
      }
    ]
  }
}
```

### 告警 API（Alerts API）

#### 1. 获取所有告警

**端点：** `GET /api/v1/alerts`

**示例：**

```bash
# 获取所有告警
curl 'http://localhost:9090/api/v1/alerts'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "alerts": [
      {
        "labels": {
          "alertname": "HighErrorRate",
          "severity": "warning"
        },
        "annotations": {
          "summary": "5xx 错误率超过 5%"
        },
        "state": "firing",
        "activeAt": "2024-01-15T10:00:00Z",
        "value": "0.08"
      }
    ]
  }
}
```

### 元数据 API（Metadata API）

#### 1. 获取指标元数据

**端点：** `GET /api/v1/metadata`

**参数：**
- `metric` (string, optional): 指标名称
- `limit` (integer, optional): 返回结果数量限制

**示例：**

```bash
# 获取所有指标的元数据
curl 'http://localhost:9090/api/v1/metadata'

# 获取特定指标的元数据
curl 'http://localhost:9090/api/v1/metadata?metric=http_requests_total'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "http_requests_total": [
      {
        "type": "counter",
        "help": "Total number of HTTP requests",
        "unit": ""
      }
    ]
  }
}
```

### 状态 API（Status API）

#### 1. 获取配置

**端点：** `GET /api/v1/status/config`

**示例：**

```bash
curl 'http://localhost:9090/api/v1/status/config'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "yaml": "<content of prometheus.yml>"
  }
}
```

#### 2. 获取标志（Flags）

**端点：** `GET /api/v1/status/flags`

**示例：**

```bash
curl 'http://localhost:9090/api/v1/status/flags'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "storage.tsdb.path": "/prometheus",
    "storage.tsdb.retention.time": "15d",
    "web.enable-lifecycle": "false"
  }
}
```

#### 3. 获取运行时信息

**端点：** `GET /api/v1/status/runtimeinfo`

**示例：**

```bash
curl 'http://localhost:9090/api/v1/status/runtimeinfo'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "startTime": "2024-01-15T09:00:00Z",
    "CWD": "/prometheus",
    "reloadConfigSuccess": true,
    "lastConfigTime": "2024-01-15T09:00:00Z",
    "chunkCount": 12345,
    "timeSeriesCount": 6789,
    "corruptionCount": 0,
    "goroutineCount": 45,
    "GOMAXPROCS": 4,
    "GOGC": "100",
    "GODEBUG": ""
  }
}

```

#### 4. 获取构建信息

**端点：** `GET /api/v1/status/buildinfo`

**示例：**

```bash
curl 'http://localhost:9090/api/v1/status/buildinfo'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "version": "2.45.0",
    "revision": "a1b2c3d4e5f6",
    "branch": "HEAD",
    "buildUser": "prometheus@build",
    "buildDate": "20240115-10:00:00",
    "goVersion": "go1.21.0"
  }
}
```

#### 5. 获取 TSDB 状态

**端点：** `GET /api/v1/status/tsdb`

**示例：**

```bash
curl 'http://localhost:9090/api/v1/status/tsdb'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "headStats": {
      "numSeries": 12345,
      "numLabelPairs": 67890,
      "chunkCount": 123456,
      "minTime": 1705315200000,
      "maxTime": 1705318800000
    },
    "seriesCountByMetricName": [
      {
        "name": "http_requests_total",
        "value": 1000
      }
    ],
    "labelValueCountByLabelName": [
      {
        "name": "job",
        "value": 10
      }
    ],
    "memoryInBytesByLabelName": [
      {
        "name": "job",
        "value": 1024
      }
    ],
    "seriesCountByLabelPair": [
      {
        "name": "job=api",
        "value": 500
      }
    ]
  }
}
```

### TSDB API

#### 1. 获取标签值基数

**端点：** `GET /api/v1/status/tsdb`

**说明：** 已在状态 API 中说明

#### 2. 清理 Tombstones

**端点：** `POST /api/v1/admin/tsdb/clean_tombstones`

**示例：**

```bash
curl -X POST 'http://localhost:9090/api/v1/admin/tsdb/clean_tombstones'
```

**响应格式：**

```json
{
  "status": "success"
}
```

#### 3. 删除序列

**端点：** `POST /api/v1/admin/tsdb/delete_series`

**参数：**
- `match[]` (string, required): 指标选择器
- `start` (rfc3339 | unix_timestamp, optional): 开始时间
- `end` (rfc3339 | unix_timestamp, optional): 结束时间

**示例：**

```bash
# 删除特定指标的序列
curl -X POST 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=http_requests_total{status="500"}'

# 删除时间范围内的序列
curl -X POST 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=http_requests_total&start=2024-01-15T09:00:00Z&end=2024-01-15T10:00:00Z'
```

**响应格式：**

```json
{
  "status": "success"
}
```

### Admin API

#### 1. 快照（Snapshot）

**端点：** `POST /api/v1/admin/tsdb/snapshot`

**参数：**
- `skip_head` (boolean, optional): 是否跳过 head block，默认 `false`

**示例：**

```bash
# 创建快照
curl -X POST 'http://localhost:9090/api/v1/admin/tsdb/snapshot'

# 跳过 head block 创建快照
curl -X POST 'http://localhost:9090/api/v1/admin/tsdb/snapshot?skip_head=true'
```

**响应格式：**

```json
{
  "status": "success",
  "data": {
    "name": "20240115T100000Z-abc123"
  }
}
```

#### 2. 重新加载配置

**端点：** `POST /-/reload`

**说明：** 需要启用 `--web.enable-lifecycle` 标志

**示例：**

```bash
curl -X POST 'http://localhost:9090/-/reload'
```

#### 3. 优雅关闭

**端点：** `POST /-/quit`

**说明：** 需要启用 `--web.enable-lifecycle` 标志

**示例：**

```bash
curl -X POST 'http://localhost:9090/-/quit'
```

### 常用 API 组合示例

#### 1. 监控面板数据获取

```bash
# 获取 CPU 使用率（过去 1 小时，每 1 分钟一个点）
curl 'http://localhost:9090/api/v1/query_range?query=1-avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))&start=2024-01-15T09:00:00Z&end=2024-01-15T10:00:00Z&step=1m'

# 获取内存使用率
curl 'http://localhost:9090/api/v1/query_range?query=1-(node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes)&start=2024-01-15T09:00:00Z&end=2024-01-15T10:00:00Z&step=1m'

# 获取错误率
curl 'http://localhost:9090/api/v1/query_range?query=sum(rate(http_requests_total{status=~"5.."}[5m]))/sum(rate(http_requests_total[5m]))&start=2024-01-15T09:00:00Z&end=2024-01-15T10:00:00Z&step=1m'
```

#### 2. 告警检查

```bash
# 检查所有告警
curl 'http://localhost:9090/api/v1/alerts'

# 检查特定告警规则
curl 'http://localhost:9090/api/v1/rules?type=alert'
```

#### 3. 目标健康检查

```bash
# 检查所有目标状态
curl 'http://localhost:9090/api/v1/targets?state=active'

# 检查特定目标
curl 'http://localhost:9090/api/v1/targets' | jq '.data.activeTargets[] | select(.labels.job=="node")'
```

#### 4. 指标发现

```bash
# 获取所有指标名称
curl 'http://localhost:9090/api/v1/label/__name__/values'

# 获取特定标签的所有值
curl 'http://localhost:9090/api/v1/label/job/values'
```

### API 错误处理

所有 API 在出错时返回以下格式：

```json
{
  "status": "error",
  "errorType": "<string>",
  "error": "<string>"
}
```

**常见错误类型：**
- `bad_data`: 查询表达式错误
- `timeout`: 查询超时
- `canceled`: 查询被取消
- `execution`: 执行错误
- `internal`: 内部服务器错误

**示例：**

```json
{
  "status": "error",
  "errorType": "bad_data",
  "error": "parse error at char 10: unexpected \"}\" in label matcher"
}
```

### API 认证

如果 Prometheus 启用了认证，需要在请求头中添加认证信息：

```bash
# Basic 认证
curl -u username:password 'http://localhost:9090/api/v1/query?query=up'

# Bearer Token
curl -H "Authorization: Bearer <token>" 'http://localhost:9090/api/v1/query?query=up'
```

### API 最佳实践

1. **使用适当的超时时间**
   ```bash
   curl 'http://localhost:9090/api/v1/query?query=up&timeout=10s'
   ```

2. **合理设置查询步长**
   - 短期查询（< 1 小时）：`step=15s` 或 `step=1m`
   - 中期查询（1-24 小时）：`step=5m` 或 `step=15m`
   - 长期查询（> 24 小时）：`step=1h` 或更大

3. **使用 Recording Rules 优化查询**
   - 复杂查询转换为 Recording Rules
   - API 直接查询预计算指标

4. **批量查询优化**
   - 使用 `match[]` 参数组合多个指标
   - 避免多次单独查询

5. **错误处理**
   - 检查 `status` 字段
   - 处理 `timeout` 和 `bad_data` 错误
   - 实现重试机制

## 误用与陷阱
- 对 `counter` 直接 `sum` 比较速率而不 `rate`。应使用 `rate/irate`。
- 对 `gauge` 使用 `increase`。`increase` 仅适用于 `counter`。
- 使用 `irate` 画趋势图。过于敏感，图表应使用 `rate`。
- 直方图分位数不保留 `le` 标签。导致 `histogram_quantile` 无法正确计算。
- 标签基数爆炸：在标签中使用动态 ID/UUID，查询与存储代价巨大。

## 性能与维护建议
- 控制窗口大小：告警短窗口 + `for` 持续；图表长窗口平滑。
- 使用 Recording rules 缓存热点查询，Grafana 直接用预计算指标。
- 合理聚合维度：在 `sum by (...)` 中仅保留必要标签，避免不必要的高基数组合。
- TSDB 管理：压缩策略与保留期、分片与远端存储（Thanos/Cortex/Mimir）。
- 联邦与远程写：跨集群聚合、长期存储与全局查询能力。

## 部署与生态
- Server 启动与配置：
  - `prometheus.yml` 配置 `scrape_configs` 指向 exporter（如 `node_exporter`、应用 `/metrics`）。
  - 配置 `rule_files` 引入记录与告警规则。
- Grafana：
  - 添加 Prometheus 数据源；面板中用 PromQL 构建图表与告警。
- Alertmanager：
  - 在 `prometheus.yml` 配置 `alerting`；在 `alertmanager.yml` 定义路由与接收人。
- 云原生扩展：
  - Thanos/Cortex/Mimir 实现水平扩展、长期存储与全局查询。
  - 在 K8s 中使用 ServiceMonitor/PodMonitor 进行抓取声明与自动发现。

## 学习路径与练习
- 入门：
  - 识别指标类型（counter/gauge/histogram）、单位与命名规范。
  - 编写基础选择器与 `rate/increase`，理解区间选择器 `[5m]`。
- 进阶：
  - 直方图分位数、向量匹配（`on`/`group_left/right`）、`label_replace`。
  - Recording/Alerting rules，Alertmanager 抑制与静默。
- 实战练习：
  - 为你的服务定义 `http_requests_total`、`http_request_duration_seconds_bucket`。
  - 写出错误率与 p95 延迟告警，用 Recording rules 缓存复杂查询。
  - 为节点与容器资源制作 Grafana 仪表板，并标注 SLO 阈值线。

## 速查清单
- 总请求速率：
```
sum(rate(http_requests_total[5m]))
```
- 错误率：
```
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```
- p95 延迟：
```
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```
- CPU 使用率：
```
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))
```
- 内存使用率：
```
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```
- Pod 重启率：
```
sum(rate(kube_pod_container_status_restarts_total[5m])) by (pod)
```

---

如需将此文档扩展为“带图示与完整规则示例”的进阶版，或根据你的技术栈（K8s/微服务/数据库/队列）生成专属查询手册，可继续告知偏好与深度。