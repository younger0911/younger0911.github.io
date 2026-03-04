---
title: openclaw可观测初探
date: 2026-03-02 20:58:08
tags: 
  - AI
  - openclaw
categories: 
  - AI
  - Observability
---

本文档记录了如何启用和配置 OpenClaw 的 `diagnostics-otel` 模块，将 trace、metrics 和 logs 通过 OpenTelemetry OTLP/HTTP 协议导出到 Jaeger 进行可视化分析。

<!-- more -->

## 1. 启用diagnostics-otel模块

openclaw.json，一般在/root/.openclaw目录

```json
{
  "plugins": {
    "allow": ["diagnostics-otel"],
    "entries": {
      "diagnostics-otel": {
        "enabled": true
      }
    }
  },
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://localhost:4318",
      "protocol": "http/protobuf",
      "serviceName": "openclaw",
      "traces": true,
      "metrics": true,
      "logs": true,
      "sampleRate": 1.0,
      "flushIntervalMs": 5000
    }
  }
}
```

## 2. 安装 OpenTelemetry 依赖

进入 diagnostics-otel 插件目录并安装依赖：

```bash
cd /usr/lib/node_modules/openclaw/extensions/diagnostics-otel
npm install
```

**OpenClaw 版本：** 2026.2.24  
**Jaeger 版本：** v1.76.0

**安装的依赖包包括（按照实际版本可能有变化）：**

- `@opentelemetry/api` (^1.9.0)
- `@opentelemetry/api-logs` (^0.212.0)
- `@opentelemetry/exporter-logs-otlp-proto` (^0.212.0)
- `@opentelemetry/exporter-metrics-otlp-proto` (^0.212.0)
- `@opentelemetry/exporter-trace-otlp-proto` (^0.212.0)
- `@opentelemetry/resources` (^2.5.1)
- `@opentelemetry/sdk-logs` (^0.212.0)
- `@opentelemetry/sdk-metrics` (^2.5.1)
- `@opentelemetry/sdk-node` (^0.212.0)
- `@opentelemetry/sdk-trace-base` (^2.5.1)
- `@opentelemetry/semantic-conventions` (^1.39.0)

## 3. 启动 Jaeger（OTLP 接收端）

使用 Docker 启动 Jaeger All-in-One：

```bash
# 首次启动
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest

# 后续启动（如果容器已存在）
docker start jaeger
```

**端口说明：**

- `16686`: Jaeger UI 访问端口
- `4318`: OTLP/HTTP 接收端口（OpenClaw 发送数据到此端口）

## 4. 调整刷新间隔

编辑 `~/.openclaw/openclaw.json`，修改 `flushIntervalMs`：

```json
{
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://localhost:4318",
      "protocol": "http/protobuf",
      "serviceName": "openclaw",
      "traces": true,
      "metrics": true,
      "logs": true,
      "sampleRate": 1.0,
      "flushIntervalMs": 5000
    }
  }
}
```

**关键修改：**

- `flushIntervalMs`: 从 60000ms 改为 5000ms（5秒）
  - 生产环境建议使用 30000-60000ms
  - 开发/测试环境可以使用 5000ms 快速查看数据

### 步骤 4：重启 Gateway

```bash
openclaw gateway restart
```

---

## 

## 验证方法

### 1. 检查 Gateway 日志

```bash
openclaw logs --follow | grep -i "diagnostics-otel"
```

**期望输出：**

```
diagnostics-otel: logs exporter enabled (OTLP/Protobuf)
```

### 2. 检查 Jaeger 服务列表

```bash
curl -s http://localhost:16686/api/services
```

**期望输出：**

```json
{"data":["jaeger-all-in-one","openclaw"],"total":2,...}
```

### 3. 查看 Trace 数据

```bash
curl -s "http://localhost:16686/api/traces?service=openclaw&limit=5"
```

### 4. 访问 Jaeger UI

打开浏览器访问：**http://localhost:16686**

- Service: 选择 `openclaw`
- Operation: 可查看 `openclaw.model.usage`, `openclaw.session.stuck` 等

![image-20260302210932739](/images/image-20260302210932739.png)

![image-20260302210859556](/images/image-20260302210859556.png)



---

## 导出的数据类型

### Traces（调用链）

| Span 名称                    | 说明                          |
| ---------------------------- | ----------------------------- |
| `openclaw.model.usage`       | 模型调用（token、成本、时长） |
| `openclaw.webhook.processed` | Webhook 处理                  |
| `openclaw.webhook.error`     | Webhook 错误                  |
| `openclaw.message.processed` | 消息处理                      |
| `openclaw.session.stuck`     | Session 卡住事件              |

### Metrics（指标）

| 指标名                      | 类型      | 说明                   |
| --------------------------- | --------- | ---------------------- |
| `openclaw.tokens`           | Counter   | Token 使用量（按类型） |
| `openclaw.cost.usd`         | Counter   | 模型调用成本（美元）   |
| `openclaw.run.duration_ms`  | Histogram | 运行时长               |
| `openclaw.context.tokens`   | Histogram | 上下文窗口使用         |
| `openclaw.webhook.received` | Counter   | 收到的 webhook 数      |
| `openclaw.webhook.error`    | Counter   | Webhook 错误数         |
| `openclaw.queue.depth`      | Histogram | 队列深度               |
| `openclaw.session.stuck`    | Counter   | 卡住 session 数        |

### Logs（日志）

- 通过 OTLP 导出到 `http://localhost:4318/v1/logs`
- 包含所有 OpenClaw 日志的结构化数据

---

## 故障排除

### 问题 1：Jaeger 中没有数据

**排查步骤：**

1. 检查 Jaeger 是否运行：`docker ps | grep jaeger`
2. 检查 OTLP 端口是否可达：`curl http://localhost:4318/v1/traces`
3. 检查 Gateway 日志是否有 exporter 启动信息
4. 确认 `flushIntervalMs` 不是太长

### 问题 2：Gateway 启动报错 `Cannot find module '@opentelemetry/api'`

**解决：** 执行步骤 1 安装 npm 依赖

### 问题 3：只有 logs 没有 traces

**原因：** traces 和 metrics 需要 SDK 初始化成功

**检查：**

- 确认 `traces: true` 和 `metrics: true` 已配置
- 查看 Gateway 启动日志是否有 SDK 错误

---

## 生产环境建议

### 1. 采样率调整

高流量环境建议降低采样率：

```json
"sampleRate": 0.1  // 只采集 10% 的 trace
```

### 2. 刷新间隔

```json
"flushIntervalMs": 60000  // 60秒，减少网络开销
```

### 3. 使用 OpenTelemetry Collector

生产环境建议使用 Collector 而非直接发送到 Jaeger：

```yaml
# collector-config.yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [jaeger]
```

### 4. 环境变量配置

可以通过环境变量覆盖配置：

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
export OTEL_SERVICE_NAME=openclaw-production
```

---

## 参考链接

- [OpenClaw Diagnostics 文档](https://docs.openclaw.ai/logging)
- [Jaeger 文档](https://www.jaegertracing.io/docs/)
- [OpenTelemetry 文档](https://opentelemetry.io/docs/)

---

## 附录：快速检查清单

```bash
# 1. 检查依赖是否安装
ls /usr/lib/node_modules/openclaw/extensions/diagnostics-otel/node_modules/@opentelemetry/

# 2. 检查 Jaeger 是否运行
docker ps | grep jaeger

# 3. 检查 OTLP 端口
curl http://localhost:4318/v1/traces

# 4. 检查 Gateway 是否发送数据
curl http://localhost:16686/api/services | grep openclaw

# 5. 查看最近 trace
curl "http://localhost:16686/api/traces?service=openclaw&limit=1"
```

