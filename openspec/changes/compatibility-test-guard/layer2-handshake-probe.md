# 兼容性看护第二层整体方案

## 1. 方案定位

第二层是独立于第一层的生产级兼容性巡检能力。

- **核心目标**：在生产环境上，采集 Java 进程与第三方系统对接时实际使用的算法、协议，判断当前是否仍能成功握手；若失败，定位是哪些算法/协议不兼容。
- **触发方式**：按需执行。
- **输出形式**：本地 Markdown + JSON 报告，人工阅读比对。
- **聚焦范围**：仅算法、协议、握手协商结果。
- **前提假设**：网络、证书、密码、权限等已打通，不考虑这些外部因素导致的失败。

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         兼容性看护第二层                             │
│                         （生产算法/协议巡检）                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌───────────────┐                                                 │
│   │   触发层       │  人工执行 / 按需调用                             │
│   │   (Trigger)   │                                                 │
│   └───────┬───────┘                                                 │
│           │                                                         │
│           ▼                                                         │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    目标发现 (Discovery)                      │   │
│   │  • 服务名 / PID 列表                                         │   │
│   │  • 配置文件路径（Kafka / HTTPS / SFTP / Kerberos）           │   │
│   │  • 三方 endpoint 列表                                        │   │
│   └───────────────────────────┬─────────────────────────────────┘   │
│                               │                                     │
│           ┌───────────────────┼───────────────────┐                 │
│           ▼                   ▼                   ▼                 │
│   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐        │
│   │  被动抓取器    │   │  主动探测器    │   │  配置解析器    │        │
│   │  (Capturer)   │   │  (Prober)     │   │  (Parser)     │        │
│   │               │   │               │   │               │        │
│   │ Java Agent    │   │ 独立 CLI      │   │ 读取 ssl/krb5 │        │
│   │ attach JVM    │   │ 用生产配置    │   │ /jaas/ssh     │        │
│   │ 读已有连接    │   │ 连三方        │   │ 等配置        │        │
│   └───────┬───────┘   └───────┬───────┘   └───────┬───────┘        │
│           │                   │                   │                 │
│           └───────────────────┼───────────────────┘                 │
│                               ▼                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    结果汇聚 (Collector)                      │   │
│   │  • 去重（同一 endpoint 多次连接）                            │   │
│   │  • 标注来源（passive / active）                              │   │
│   │  • 按服务 / endpoint / 专题聚合                              │   │
│   └───────────────────────────┬─────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    兼容性分析 (Analyzer)                     │   │
│   │  • 与历史基线 diff                                           │   │
│   │  • 与已弃用算法清单比对                                      │   │
│   │  • 握手失败归因                                              │   │
│   └───────────────────────────┬─────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    报告生成 (Reporter)                       │   │
│   │  • Markdown 报告（人工阅读）                                 │   │
│   │  • JSON 报告（后续工具化）                                   │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心组件职责

| 组件 | 职责 |
|------|------|
| **Trigger** | 接收用户输入（PID、配置文件、输出路径），启动巡检 |
| **Discovery** | 识别要采集的目标：JVM、endpoint、配置文件 |
| **Capturer** | 通过 Java Agent attach 到 JVM，被动读取已有连接的协商结果 |
| **Prober** | 在生产主机上独立运行，主动连接三方 endpoint 完成握手 |
| **Parser** | 解析各类配置文件：kafka client props、JAAS、krb5、ssh config、SSL config |
| **Collector** | 合并 Capturer 和 Prober 的结果，去重、聚合 |
| **Analyzer** | 对比历史基线和弃用清单，识别异常并归因 |
| **Reporter** | 将分析结果写入本地 Markdown 和 JSON 文件 |
| **Baseline Store** | 本地基线文件，记录历史巡检结果 |

---

## 4. 两种采集方式详细说明

### 4.1 被动抓取（Passive Capture）

**原理**：将 Java Agent attach 到目标 JVM，在内存中拦截或读取已建立连接的协商结果。

**采集对象**：

| 对象 | 可采集内容 |
|------|-----------|
| TLS 连接 | `SSLSession.getProtocol()`、`getCipherSuite()`、服务端证书 |
| Kafka Client | TLS 协商结果 + `security.protocol` + `sasl.mechanism` |
| HTTPS Client | TLS 协商结果 + ALPN 协议 |
| SSH/SFTP Client | KEX、cipher、MAC、host key（依赖客户端库 API） |
| Kerberos | Ticket etype、生命周期、renewable 标志 |

**优点**：

- 观测真实业务流量，结果最准确
- 不需要发起新连接
- 能发现“已有旧连接仍在用旧算法，新建连接已变化”的场景

**限制**：

- 只能抓当前有流量的连接
- 不同客户端库版本需要适配
- JDK 17+ 强封装可能限制内部 API 访问
- Agent attach 需要生产环境权限

### 4.2 主动探测（Active Probe）

**原理**：在生产主机上运行独立进程，使用生产环境的配置和凭证，主动连接三方 endpoint 并完成握手。

**探测对象**：

| 对象 | 探测方式 |
|------|---------|
| Kafka broker | TLS Socket 握手 + Kafka AdminClient 验证 SASL |
| HTTPS endpoint | TLS Socket 握手 或 HttpClient 完整请求 |
| SFTP/SSH server | SSH client 连接并读取协商结果 |
| Kerberos KDC | `Krb5LoginModule` 登录并读取 ticket |

**优点**：

- 不依赖当前是否有业务流量
- 可以覆盖所有配置的 endpoint
- 能验证“现在新建连接是否还能成功”

**限制**：

- 会发起真实网络连接
- 可能触发三方审计、限流、告警
- 对于 SASL_SSL 等场景，TLS 和认证层需要分开验证
- 需要安全获取生产凭证

### 4.3 组合策略

一次巡检中按以下顺序执行：

```
1. Discovery 识别所有目标 JVM 和 endpoint
2. 对每个目标 JVM 执行 Capturer（被动抓取）
3. 从配置解析出所有 endpoint，减去已被 Capturer 覆盖的
4. 对剩余 endpoint 执行 Prober（主动探测）
5. Collector 合并结果
6. Analyzer 分析异常
7. Reporter 输出报告
```

### 4.4 运行约束

第二层直接运行在生产环境，需满足以下约束：

| 操作 | 权限要求 | 风险控制 |
|------|---------|---------|
| Java Agent attach | 与目标 JVM 同用户或 root | 只在运维窗口执行，提供 `--dry-run` |
| 读取 Kafka Client 配置 | 读取生产配置权限 | 不输出敏感字段到报告 |
| 主动探测连接三方 | 出站网络权限 + 生产凭证 | 只握手不发送业务请求 |
| 报告文件存储 | 普通文件权限 | 报告不含密码、keytab、私钥 |

---

## 5. 专题抽象设计

第二层不局限于某个具体协议，采用专题化设计。每个专题定义自己的采集策略、结果结构、比对规则。

### 5.1 专题列表

| 专题 | 说明 | 优先级 |
|------|------|--------|
| `kafka-transport` | Kafka 连接的 TLS/SASL 算法协议 | P0（第一刀） |
| `kafka-kerberos` | Kerberos etype、ticket 生命周期 | P1 |
| `https-outbound` | 我方 HTTP Client 出站的 TLS/ALPN | P1 |
| `https-inbound` | 我方 HTTPS Server 入站的 TLS/ALPN | P1 |
| `sftp-ssh` | SFTP/SSH 的 KEX、cipher、MAC、host key | P1 |

### 5.2 专题接口抽象

每个专题实现统一的采集接口：

```java
public interface CompatibilityTopic {
    // 专题标识
    String getTopicName();

    // 从配置解析目标 endpoint 列表
    List<Endpoint> discoverEndpoints(ServiceConfig config);

    // 被动抓取
    List<HandshakeResult> capture(ServiceTarget target);

    // 主动探测
    List<HandshakeResult> probe(ServiceConfig config, Endpoint endpoint);

    // 结果字段定义
    ResultSchema getResultSchema();

    // 异常判断规则
    List<CompatibilityRule> getRules();
}
```

### 5.3 专题结果结构

每个专题的结果包含通用字段 + 专题字段：

```json
{
  "topic": "kafka-transport",
  "endpoint": "kafka.partner-a.com:9093",
  "source": "passive",
  "service": "order-service",
  "pid": 12345,
  "collectedAt": "2026-06-17T10:00:00Z",
  "common": {
    "reachable": true,
    "tlsVersion": "TLSv1.2",
    "cipherSuite": "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
  },
  "topicSpecific": {
    "securityProtocol": "SASL_SSL",
    "saslMechanism": "GSSAPI"
  }
}
```

---

## 6. 兼容性分析逻辑

### 6.1 分析维度

| 维度 | 说明 |
|------|------|
| **历史基线 diff** | 当前结果与上次巡检结果对比，识别变化 |
| **弃用算法检测** | 当前算法是否属于已知弃用/弱算法清单 |
| **握手失败归因** | 失败时根据错误信息判断是 TLS、SASL、SSH KEX 等哪一层 |

**基线文件**：每次巡检输出 `baseline.json`，仅包含可比对字段（如 endpoint、TLS 版本、cipher suite、SASL mechanism），排除时间戳、PID 等运行时变量。下次巡检加载该基线做 diff。

**弃用清单**：独立维护 `deprecated-algorithms.json`，记录已知不安全或即将禁用的算法/协议，用于主动预警。

### 6.2 异常分级

| 级别 | 含义 | 示例 |
|------|------|------|
| **ERROR** | 握手失败或使用了已禁用的算法/协议 | TLSv1.0 已禁用 |
| **WARNING** | 使用了即将弃用或弱安全的算法/协议 | AES_128、SHA1 签名 |
| **INFO** | 结果发生变化但仍在兼容范围内 | TLSv1.2 → TLSv1.3 |
| **OK** | 正常 | 无变化且算法安全 |

### 6.3 失败归因规则

根据错误关键字或异常类型判断：

| 错误特征 | 归因 |
|----------|------|
| `No cipher suites in common` / `handshake_failure` with cipher mismatch | TLS Cipher 不兼容 |
| `No appropriate protocol` / `SSLHandshakeException: Protocol version` | TLS 版本不兼容 |
| `SASL authentication failed` / `GSS initiate failed` | SASL / Kerberos 不兼容 |
| `No supported KEX algorithm` / `Algorithm negotiation fail` | SSH KEX 不兼容 |
| `Host key verification failed` / `Changed host key` | Host key 不兼容 |

---

## 7. 输出报告设计

### 7.1 文件结构

一次巡检输出一个目录：

```
compatibility-patrol-20260617-100000/
├── report.md          # 人工阅读
├── report.json        # 机器解析
├── baseline.json      # 本次结果作为下次基线
└── raw/
    ├── passive-capture.json
    └── active-probe.json
```

`baseline.json` 仅保留可比对字段，例如：

```json
{
  "topic": "kafka-transport",
  "endpoint": "kafka.partner-a.com:9093",
  "common": {
    "tlsVersion": "TLSv1.2",
    "cipherSuite": "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
  },
  "topicSpecific": {
    "securityProtocol": "SASL_SSL",
    "saslMechanism": "GSSAPI"
  }
}
```

### 7.2 Markdown 报告结构

```markdown
# 兼容性巡检报告

## 1. 概览

- 巡检时间: 2026-06-17 10:00:00
- 目标服务: order-service
- 目标 PID: 12345
- 专题数: 1
- Endpoint 总数: 5
- 异常数: 1 ERROR, 1 WARNING

## 2. 按专题汇总

| 专题 | Endpoint 数 | OK | WARNING | ERROR |
|------|------------|----|---------|-------|
| kafka-transport | 5 | 3 | 1 | 1 |

## 3. 详细结果

### kafka-transport

| Endpoint | Source | TLS | Cipher | SASL | Status |
|----------|--------|-----|--------|------|--------|
| kafka.partner-a.com:9093 | passive | TLSv1.2 | AES_256_GCM_SHA384 | GSSAPI | OK |
| kafka.partner-b.com:9093 | passive | TLSv1.2 | AES_128_GCM_SHA256 | SCRAM-SHA-512 | WARNING |

## 4. 异常说明与建议

### ERROR: kafka.partner-c.com:9093

- 问题：主动探测握手失败
- 错误：`No cipher suites in common`
- 归因：TLS Cipher 不兼容
- 建议：检查 JDK 升级后是否禁用了对方所需的 cipher suite

## 5. 原始数据

详见 `raw/passive-capture.json` 和 `raw/active-probe.json`
```

### 7.3 JSON 报告结构

```json
{
  "patrolId": "patrol-20260617-100000",
  "patrolTime": "2026-06-17T10:00:00Z",
  "target": {
    "service": "order-service",
    "pid": 12345
  },
  "summary": {
    "totalEndpoints": 5,
    "ok": 3,
    "warning": 1,
    "error": 1
  },
  "results": [
    {
      "topic": "kafka-transport",
      "endpoint": "kafka.partner-a.com:9093",
      "source": "passive",
      "status": "OK",
      "common": {},
      "topicSpecific": {}
    }
  ],
  "anomalies": [
    {
      "endpoint": "kafka.partner-c.com:9093",
      "level": "ERROR",
      "category": "tls-cipher-incompatible",
      "message": "No cipher suites in common",
      "suggestion": "检查 JDK 升级后是否禁用了对方所需的 cipher suite"
    }
  ]
}
```
