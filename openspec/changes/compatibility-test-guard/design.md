# 兼容性看护测试套件 - 架构设计

## 1. 总体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                          触发方式                                    │
│    本地开发      │      CI 普通测试       │      nightly 回归        │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        兼容性测试套件                                │
│                                                                     │
│   第一层：静态算法/协议清单检查                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  compatibility-inventory.yaml                               │   │
│   │        │                                                    │   │
│   │        ▼                                                    │   │
│   │  InventoryScanner（JDK / Client / Server 算法支持性扫描）     │   │
│   │        │                                                    │   │
│   │        ▼                                                    │   │
│   │  清单比对报告（PASS/FAIL/WARN）                                │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   第二层：动态握手探测                                               │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│   │  Kafka Transport │  │ Kafka Kerberos  │  │    SFTP/SSH     │    │
│   │     Test Suite   │  │    Test Suite   │  │    Test Suite   │    │
│   └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │
│            │                    │                    │              │
│            └────────────────────┼────────────────────┘              │
│                                 ▼                                   │
│                    ┌─────────────────────┐                          │
│   ┌──────────────┐ │    探测辅助工具层    │                          │
│   │ HTTPS 入站   │ │  封装 Kafka/SFTP/   │                          │
│   │ Test Suite   │ │  HTTPS/Kerberos/    │                          │
│   └──────────────┘ │  JDK 探测           │                          │
│   ┌──────────────┐ │  统一输出结构化结果  │                          │
│   │ HTTPS 出站   │ └──────────┬──────────┘                          │
│   │ Test Suite   │            │                                     │
│   └──────────────┘            ▼                                     │
│                    ┌─────────────────────┐                          │
│                    │     基线加载器       │                          │
│                    │  加载三方期望结果    │                          │
│                    └─────────────────────┘                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          运行环境                                    │
│                                                                     │
│   默认：代表服务                                                     │
│   • Kafka 代表 broker（算法配置与三方对齐）                          │
│   • SFTP 代表服务（算法配置与三方对齐）                              │
│   • HTTPS 代表服务端 / 客户端（算法配置与三方对齐）                  │
│   • Kerberos 测试 KDC（可选）                                        │
│                                                                     │
│   扩展：三方真实环境                                                 │
│   • 通过标签控制，手动或 nightly 触发                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          输出                                        │
│   • 清单扫描结果                                                    │
│   • JUnit 风格测试结果                                              │
│   • 当前结果 vs 基线 diff                                            │
│   • 失败原因分析                                                     │
│   • 适配建议                                                         │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. 看护专题

### 2.1 Kafka Transport 专题

#### 清单字段

| 清单项 | 说明 |
|--------|------|
| `tlsProtocols` | 依赖的 TLS 版本 |
| `tlsCipherSuites` | 依赖的密码套件 |
| `saslMechanisms` | 依赖的 SASL 机制 |
| `securityProtocols` | 依赖的安全协议 |

#### 基线字段

| 基线项 | 说明 |
|--------|------|
| `tlsVersion` | 协商出的 TLS 版本 |
| `cipherSuite` | 协商出的密码套件 |
| `saslMechanism` | 协商出的 SASL 机制 |
| `serverCertificate.fingerprint` | 服务端证书 fingerprint |

#### 采集项

| 采集项 | 优先级 | 说明 | 常见问题 |
|--------|--------|------|---------|
| `tlsVersion` | P0 | 协商出的 TLS 版本 | JDK 升级后禁用旧版本 |
| `cipherSuite` | P0 | 协商出的密码套件 | Client/broker cipher 无交集 |
| `saslMechanism` | P0 | 认证机制 | GSSAPI / SCRAM / PLAIN 等 |
| `securityProtocol` | P0 | 安全协议 | SASL_SSL / SSL / PLAINTEXT |
| `enabledProtocols` | P0 | broker 支持的 TLS 版本列表 | broker 只支持 TLSv1.1 |
| `endpointIdentificationAlgorithm` | P0 | hostname 校验算法 | Kafka 3.x 默认行为变化 |
| `serverCertificateSubject` | P1 | 证书主题 | 证书轮换 |
| `serverCertificateSANs` | P1 | Subject Alternative Names | hostname verification 失败 |
| `serverCertificateValidFrom/To` | P1 | 有效期 | 证书过期 |
| `apiVersions` | P1 | 协商的 API 版本 | Client 与 broker 版本差距 |
| `connectionsMaxIdleMs` | P1 | 连接空闲超时 | 默认值变化导致断开 |

### 2.2 Kafka Kerberos 专题（高优先级）

#### 清单字段

| 清单项 | 说明 |
|--------|------|
| `kerberosEnctypes` | 依赖的 Kerberos 加密类型 |
| `jaasModule` | 依赖的 LoginModule |
| `jvmExports` | 是否需要 `--add-exports` |
| `jvmOpens` | 是否需要 `--add-opens` |

#### 基线字段

| 基线项 | 说明 |
|--------|------|
| `principal` | 客户端 principal |
| `negotiatedEType` | 协商出的 etype |
| `ticketLifetime` | 票证有效期 |
| `renewable` | 是否可续期 |
| `jvmExports` | 实际需要的 JVM exports |

#### 采集项

| 采集项 | 优先级 | 说明 |
|--------|--------|------|
| `loginModule` | P0 | LoginModule 类 |
| `useKeyTab` / `keyTab` / `principal` | P0 | JAAS 配置 |
| `clientSupportedEtTypes` | P0 | 客户端支持的 etype |
| `kdcSupportedEtTypes` | P0 | KDC 支持的 etype |
| `negotiatedEType` | P0 | 实际协商出的 etype |
| `permittedEnctypes` | P0 | `krb5.conf` 配置 |
| `ticketLifetime` / `renewable` | P1 | 票证生命周期 |
| `defaultRealm` / `kdcs` | P1 | KDC / realm 配置 |
| `sasl.kerberos.service.name` | P0 | broker 服务名 |
| `jvm.addExports` / `jvm.addOpens` | P0 | JDK 17+ 强封装需求 |

常见 etype 列表：

| etype | 强度 | JDK 默认支持 |
|-------|------|------------|
| `aes256-cts-hmac-sha1-96` | 强 | ✓ |
| `aes128-cts-hmac-sha1-96` | 中 | ✓ |
| `aes256-cts-hmac-sha384-192` | 强 | JDK 11+ |
| `aes128-cts-hmac-sha256-128` | 强 | JDK 11+ |
| `des3-cbc-sha1` | 弱 | JDK 17- 默认禁用 |
| `des-cbc-md5` | 弱 | JDK 17- 默认禁用 |
| `rc4-hmac` | 弱 | JDK 17- 默认禁用 |

### 2.3 SFTP/SSH 专题

#### 清单字段

| 清单项 | 说明 |
|--------|------|
| `kexAlgorithms` | 依赖的 KEX 算法 |
| `ciphers` | 依赖的 cipher 算法 |
| `macs` | 依赖的 MAC 算法 |
| `hostKeyAlgorithms` | 依赖的 host key 算法 |
| `authMethods` | 依赖的认证方式 |

#### 基线字段

| 基线项 | 说明 |
|--------|------|
| `kexAlgorithm` | 协商出的 KEX 算法 |
| `serverHostKeyAlgorithm` | 协商出的 host key 算法 |
| `serverHostKeyFingerprint` | host key fingerprint |
| `c2sCipher` / `s2cCipher` | 协商出的 cipher |
| `c2sMac` / `s2cMac` | 协商出的 MAC |
| `authMethod` | 实际使用的认证方式 |

#### 采集项

| 采集项 | 优先级 | 说明 | 常见取值 |
|--------|--------|------|---------|
| `kexAlgorithm` | P0 | 协商出的 KEX 算法 | curve25519-sha256 |
| `serverHostKeyAlgorithm` | P0 | 协商出的 host key 算法 | rsa-sha2-256 |
| `serverHostKeyFingerprint` | P0 | host key fingerprint | SHA256:... |
| `c2sCipher` / `s2cCipher` | P0 | 协商出的 cipher | aes256-ctr |
| `c2sMac` / `s2cMac` | P0 | 协商出的 MAC | hmac-sha2-256 |
| `authMethods` | P0 | 支持的认证方式 | publickey / password |
| `publicKeyAlgorithm` | P0 | 公钥算法 | rsa-sha2-256 |
| `sftpVersion` | P1 | SFTP 协议版本 | 3/4/5/6 |

### 2.4 HTTPS 专题

#### 清单字段

| 清单项 | 说明 |
|--------|------|
| `tlsProtocols` | 依赖的 TLS 版本 |
| `tlsCipherSuites` | 依赖的密码套件 |
| `certificateKeyTypes` | 服务端证书密钥类型 |
| `clientAuth` | 是否要求客户端证书 |
| `endpointIdentificationAlgorithm` | hostname 校验算法 |
| `truststoreType` | 信任库类型 |

#### 基线字段

| 基线项 | 说明 |
|--------|------|
| `tlsVersion` | 协商出的 TLS 版本 |
| `cipherSuite` | 协商出的密码套件 |
| `alpnProtocols` | ALPN 协商结果 |
| `serverCertificate.fingerprint` | 服务端证书 fingerprint |
| `clientAuth` | 客户端证书要求 |

#### 采集项

| 采集项 | 优先级 | 说明 | 常见问题 |
|--------|--------|------|---------|
| `tlsVersion` | P0 | 协商出的 TLS 版本 | JDK 升级后禁用旧版本 |
| `cipherSuite` | P0 | 协商出的密码套件 | Client/server cipher 无交集 |
| `alpnProtocols` | P0 | ALPN 协商结果 | HTTP/2 协商失败 |
| `endpointIdentificationAlgorithm` | P0 | hostname 校验算法 | 默认行为变化 |
| `certificateSubject` | P0 | 证书主题 | 证书轮换 |
| `certificateSANs` | P0 | Subject Alternative Names | hostname verification 失败 |
| `certificateValidFrom/To` | P0 | 有效期 | 证书过期 |
| `certificateSignatureAlgorithm` | P0 | 签名算法 | JDK 禁用 SHA1 签名 |
| `certificateKeyType` | P0 | 公钥算法类型 | RSA / ECDSA / Ed25519 |
| `clientAuth` | P0 | 是否要求客户端证书 | mTLS 配置变化 |
| `httpProtocolVersion` | P1 | HTTP 协议版本 | HTTP/1.1 vs HTTP/2 |

#### 入站采集方式

启动本地 HTTPS 服务端，用代表客户端（配置与三方客户端对齐）连接，读取服务端 `SSLSession`。

#### 出站采集方式

使用我方 HTTP Client 连接第三方 HTTPS 端点或代表服务端，读取客户端侧 `SSLSession`。

### 2.5 JDK Security 基线

#### 清单字段

| 清单项 | 说明 |
|--------|------|
| `tlsProtocols` | 依赖的 TLS 版本 |
| `tlsCipherSuites` | 依赖的密码套件 |
| `kerberosEnctypes` | 依赖的 Kerberos etype |
| `providers` | 依赖的安全 provider |

#### 采集项

| 采集项 | 优先级 | 说明 |
|--------|--------|------|
| `java.version` | P0 | JDK 版本 |
| `java.vendor` | P0 | JDK 厂商 |
| `jdk.tls.disabledAlgorithms` | P0 | 禁用的 TLS 算法 |
| `jdk.certpath.disabledAlgorithms` | P0 | 证书路径禁用算法 |
| `security.providers` | P1 | 已加载 provider |
| `permittedKerberosEnctypes` | P0 | 允许的 Kerberos etype |
| `java.security.file.path` | P1 | 实际加载的 java.security 路径 |

### 2.6 组件版本采集

| 组件 | 采集项 | 优先级 |
|------|--------|------|
| Kafka Client | 版本号 | P0 |
| Apache SSHD | 版本号 | P0 |
| JSch / sshj | 版本号 | P0（如使用） |
| HTTP Client | 实现与版本 | P0 |
| HTTP Server | 容器与版本 | P1 |
| Spring Boot | 管理的 Kafka/SSHD/HTTP 版本 | P1 |
| BouncyCastle | 版本号 | P1 |
| 国密 provider | 版本号 | P1（如使用 SM 系列算法） |

### 2.7 运行环境采集

#### 操作系统层（P1）

| 采集项 | 说明 | 典型问题 |
|--------|------|---------|
| `osCryptoPolicy` | RHEL/Fedora crypto-policies | 系统级禁用算法 |
| `timezone` | 时区 | Kerberos 时间校验 |
| `ntpSynced` | NTP 同步状态 | Kerberos 时钟偏差 |

#### 容器层（P1）

| 采集项 | 说明 |
|--------|------|
| `containerImageDigest` | 容器镜像摘要 |
| `containerBaseOs` | 基础系统 |
| `openSslVersion` | OpenSSL/LibreSSL 版本 |

## 3. 测试分层

```
                    ┌─────────────────┐
                    │    E2E 测试      │
                    │  连接三方真实环境 │
                    │  慢、真实、手动跑 │
                    └────────┬────────┘
                             │
            ┌────────────────┴────────────────┐
            │         集成测试（主力）          │
            │  使用代表服务验证 handshake       │
            │  稳定、可重复、CI 可跑           │
            └────────────────┬────────────────┘
                             │
            ┌────────────────┴────────────────┐
            │      清单检查（新增强）           │
            │  验证 JDK/Client/Server 算法支持性│
            │  快、无外部依赖                  │
            └─────────────────────────────────┘
```

| 层级 | 作用 | 速度 | 稳定性 | 外部依赖 |
|------|------|------|--------|---------|
| 清单检查 | 验证依赖算法/协议是否仍被支持 | 秒级 | 高 | 无 |
| 集成测试 | 验证 handshake 协商结果 | 分钟级 | 高 | Testcontainers 代表服务 |
| E2E 测试 | 验证三方真实环境 | 分钟级 | 中 | 三方测试环境 |

## 4. 组件职责

| 组件 | 职责 |
|------|------|
| 清单文件 | 记录系统依赖的算法/协议 |
| 清单扫描器 | 扫描当前 JDK/Client/Server 支持、禁用、默认启用的算法 |
| 测试用例 | 按专题组织，定义“测什么”、“期望什么” |
| 探测辅助工具 | 封装握手探测逻辑，输出结构化结果 |
| 基线加载器 | 读取各三方基线数据 |
| 基线文件 | 存储每个三方、每个环境的期望协商结果 |
| 代表服务 | 模拟三方算法配置，提供稳定测试目标 |
| 测试报告 | 展示清单结果、当前结果、基线、diff、适配建议 |

## 5. 兼容性清单

### 5.1 清单文件

`compatibility-inventory.yaml` 记录系统依赖的算法/协议，用于静态清单检查。

```yaml
inventoryVersion: "2026-06-01"
subject: "compatibility-inventory"

required:
  jdk:
    tlsProtocols: ["TLSv1.2"]
    tlsCipherSuites: ["TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]
    kerberosEnctypes: ["aes256-cts-hmac-sha1-96", "aes128-cts-hmac-sha1-96"]
    providers: ["SunJSSE", "SunJCE", "SunJGSS"]
    
  kafka:
    securityProtocols: ["SASL_SSL", "SSL"]
    saslMechanisms: ["GSSAPI", "SCRAM-SHA-512"]
    apiVersions: ["3.6"]
    
  sftp:
    kexAlgorithms: ["curve25519-sha256", "diffie-hellman-group14-sha256"]
    ciphers: ["aes256-ctr", "aes128-ctr"]
    macs: ["hmac-sha2-256"]
    hostKeyAlgorithms: ["rsa-sha2-256", "ssh-ed25519"]
    authMethods: ["publickey"]

  https:
    inbound:
      tlsProtocols: ["TLSv1.2"]
      tlsCipherSuites: ["TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]
      certificateKeyTypes: ["RSA"]
      clientAuth: "NONE"  # NONE / WANT / NEED
    outbound:
      tlsProtocols: ["TLSv1.2"]
      tlsCipherSuites: ["TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]
      endpointIdentificationAlgorithm: "HTTPS"
      truststoreType: "PKCS12"
```

### 5.2 清单扫描项

| 扫描项 | 来源 | 用途 |
|--------|------|------|
| `supportedTlsProtocols` | `SSLContext.getDefault().getSupportedSSLParameters()` | 检查 TLS 协议是否仍受支持 |
| `enabledTlsProtocols` | `SSLParameters.getProtocols()` | 检查 TLS 协议是否默认启用 |
| `disabledTlsAlgorithms` | `Security.getProperty("jdk.tls.disabledAlgorithms")` | 检查所需 cipher 是否被禁用 |
| `supportedTlsCipherSuites` | `SSLParameters.getCipherSuites()` | 检查 cipher 是否仍受支持 |
| `securityProviders` | `Security.getProviders()` | 检查所需 provider 是否加载 |
| `permittedKerberosEnctypes` | `Security.getProperty("permitted_enctypes")` | 检查 etype 是否被允许 |
| `defaultKrb5Etypes` | `krb5.conf` | 检查默认 etype 偏好 |

### 5.3 清单检查规则

| 规则 | 含义 | 示例 |
|------|------|------|
| REQUIRED_SUPPORTED | 依赖算法必须在支持列表中 | `TLSv1.2` 必须在 `supportedTlsProtocols` 中 |
| REQUIRED_ENABLED | 依赖算法最好在默认启用列表中 | `TLSv1.2` 在 `enabledTlsProtocols` 中 |
| NOT_DISABLED | 依赖算法不能在禁用列表中 | `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` 不在 `disabledTlsAlgorithms` 中 |
| PROVIDER_PRESENT | 依赖 provider 必须已加载 | `SunJSSE` 在 provider 列表中 |

### 5.4 清单来源

| 来源 | 优先级 | 说明 |
|------|--------|------|
| 现有基线文件 | 高 | 从 baseline 中使用的算法反推 |
| 业务代码/配置 | 高 | 直接读取 ssl/krb5/ssh/https 配置 |
| 三方对接文档 | 中 | 伙伴提供的算法要求 |
| 人工整理 | 兜底 | 特殊场景补充 |

### 5.5 清单维护

- 新接入三方时补充依赖算法
- 三方算法要求变更时更新
- 本方主动升级并验证后更新
- 定期审计，删除不再使用的依赖

## 6. 基线管理

### 6.1 基线内容

每个三方、每个环境保存一份期望的协商结果。基线文件只存储需要对比的核心字段：

| 专题 | 基线字段 |
|------|---------|
| Kafka Transport | TLS 版本、cipher suite、SASL 机制、证书 fingerprint |
| Kafka Kerberos | principal、etype、ticket 生命周期、renewable、JVM exports |
| SFTP/SSH | KEX、host key 算法与 fingerprint、cipher、MAC、认证方式 |
| HTTPS 入站 | TLS 版本、cipher suite、客户端证书要求、ALPN |
| HTTPS 出站 | TLS 版本、cipher suite、证书 chain、ALPN |

### 6.2 基线来源

| 来源 | 优先级 | 说明 |
|------|--------|------|
| 三方测试环境 | 最高 | 最真实 |
| 代表服务 | 常用 | 稳定可控 |
| 生产环境只读采样 | 兜底 | 需授权 |

### 6.3 基线维护

- 新三方接入时采集一次
- 三方配置变更后更新
- 本方主动升级并验证后更新
- 定期刷新（建议每季度）

## 7. 数据流

1. 定义 `compatibility-inventory.yaml` 依赖清单
2. 定义 `baseline/*.json` 期望协商结果
3. 测试启动
4. 清单扫描器扫描当前 JDK/Client/Server 支持情况
5. 与清单比对，输出清单检查结果
6. 基线加载器加载对应三方基线
7. 探测辅助工具连接代表服务或真实环境
8. 采集当前协商结果
9. 与基线进行对比
10. 输出测试结果与 diff
11. 失败时输出适配建议

## 8. CI 策略

| 场景 | 运行内容 | 失败影响 |
|------|---------|---------|
| PR CI | 清单检查 + 集成测试 | 不阻塞 PR，仅暴露问题 |
| 每日构建 | 清单检查 + 全量集成测试 | 告警，人工跟进 |
| 发布前 | 清单检查 + E2E + 全量矩阵测试 | 发布评审参考 |
| 本地开发 | 按需运行清单检查或相关专题测试 | 即时反馈 |

## 9. 安全考虑

- keytab、证书私钥等敏感凭证使用 CI secret manager 或本地安全存储
- 测试代码中不硬编码敏感信息
- 生产环境采样需获得授权，使用只读方式
- 基线文件中不包含私钥
- 清单文件只包含算法/协议名称，不包含环境地址或凭证

## 10. 统一输出示例

### 10.1 清单扫描输出示例

```json
{
  "scanType": "inventory",
  "scannedAt": "2026-06-01T10:00:00Z",
  "environment": {
    "jdk": "17.0.12",
    "kafkaClient": "3.6.2",
    "sshd": "2.15.0",
    "httpClient": "JDK HttpClient 17.0.12"
  },
  "checks": [
    {
      "subject": "jdk.tls.protocol",
      "required": "TLSv1.2",
      "actual": "supported",
      "status": "PASS"
    },
    {
      "subject": "jdk.tls.cipher",
      "required": "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
      "actual": "disabled",
      "status": "FAIL"
    }
  ]
}
```

### 10.2 握手探测输出示例

```json
{
  "baselineId": "partner-a-2026-06-01",
  "collectedAt": "2026-06-01T10:00:00Z",
  "environment": {
    "jdk": "17.0.12",
    "kafkaClient": "3.6.2",
    "sshd": "2.15.0",
    "httpClient": "JDK HttpClient 17.0.12",
    "containerImageDigest": "sha256:abcd..."
  },
  "jdkSecurity": {
    "disabledTlsAlgorithms": ["SSLv3", "RC4", "DES", "MD5withRSA", "3DES_EDE_CBC"],
    "disabledCertAlgorithms": ["MD2", "MD5", "RSA keySize < 1024"],
    "providers": ["SUN", "SunRsaSign", "SunEC", "SunJSSE", "SunJCE", "SunJGSS", "SunSASL"],
    "permittedKerberosEnctypes": ["aes256-cts-hmac-sha1-96", "aes128-cts-hmac-sha1-96"]
  },
  "kafka": {
    "target": "partner-a-kafka:9092",
    "securityProtocol": "SASL_SSL",
    "tlsVersion": "TLSv1.2",
    "cipherSuite": "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
    "endpointIdentificationAlgorithm": "",
    "saslMechanism": "GSSAPI",
    "apiVersion": "3.6",
    "connectionsMaxIdleMs": 540000,
    "serverCertificate": {
      "subject": "CN=kafka.partner-a.com",
      "sans": ["kafka.partner-a.com"],
      "issuer": "CN=Partner A Root CA",
      "validFrom": "2025-01-01",
      "validTo": "2026-01-01",
      "signatureAlgorithm": "SHA256withRSA"
    },
    "kerberos": {
      "principal": "kafka-client@EXAMPLE.COM",
      "serviceName": "kafka",
      "negotiatedEType": "aes256-cts-hmac-sha1-96",
      "ticketLifetime": "24h",
      "renewable": true,
      "jaasModule": "com.sun.security.auth.module.Krb5LoginModule",
      "jvmExports": ["--add-exports java.security.jgss/sun.security.jgss=ALL-UNNAMED"],
      "kdcSupportedEtTypes": ["aes256-cts-hmac-sha1-96", "aes128-cts-hmac-sha1-96", "rc4-hmac"]
    }
  },
  "sftp": {
    "target": "partner-a-sftp:22",
    "kex": "curve25519-sha256",
    "serverHostKey": "rsa-sha2-256",
    "serverHostKeyFingerprint": "SHA256:abcd...",
    "c2sCipher": "aes256-ctr",
    "s2cCipher": "aes256-ctr",
    "c2sMac": "hmac-sha2-256",
    "s2cMac": "hmac-sha2-256",
    "c2sCompression": "none",
    "s2cCompression": "none",
    "authMethod": "publickey",
    "publicKeyAlgorithm": "rsa-sha2-256",
    "sftpVersion": 3
  },
  "https": {
    "inbound": {
      "listenAddress": "0.0.0.0:8443",
      "tlsVersion": "TLSv1.2",
      "cipherSuite": "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
      "alpnProtocols": ["h2", "http/1.1"],
      "clientAuth": "NONE",
      "serverCertificate": {
        "subject": "CN=api.our-service.com",
        "sans": ["api.our-service.com"],
        "issuer": "CN=Our Root CA",
        "validTo": "2026-01-01",
        "signatureAlgorithm": "SHA256withRSA",
        "fingerprint": "SHA256:abcd..."
      }
    },
    "outbound": {
      "target": "https://partner-a-api.example.com",
      "tlsVersion": "TLSv1.2",
      "cipherSuite": "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
      "alpnProtocols": ["http/1.1"],
      "endpointIdentificationAlgorithm": "HTTPS",
      "serverCertificate": {
        "subject": "CN=partner-a-api.example.com",
        "sans": ["partner-a-api.example.com"],
        "issuer": "CN=Partner A Root CA",
        "validTo": "2026-01-01",
        "signatureAlgorithm": "SHA256withRSA",
        "fingerprint": "SHA256:efgh..."
      }
    }
  }
}
```

## 11. 看护触发条件

当以下任意项发生变化时，应触发兼容性看护：

1. JDK 主/次版本升级
2. JDK 小版本升级（可能伴随 security baseline 变化）
3. Kafka Client 版本升级
4. Apache SSHD / JSch / sshj 版本升级
5. HTTP Client / HTTP Server 版本升级
6. Spring Boot / 依赖管理平台升级
7. BouncyCastle / 国密 provider 升级
8. `java.security` 配置变更
9. `krb5.conf` / JAAS 配置变更
10. keytab / 证书轮换
11. 容器基础镜像升级
12. `compatibility-inventory.yaml` 清单变更
13. `baseline/*.json` 基线变更

## 12. 待补充场景

以下场景在后续分析中需进一步确认是否纳入：

- 国密 SM2/SM3/SM4 算法场景
- 双向 TLS（mTLS）客户端证书场景
- FIPS 140-2/140-3 模式场景
- 多云/多区域部署差异
- 动态 provider 加载（运行时 `Security.addProvider`）
- 自定义 TrustManager / HostnameVerifier
- HTTP/3 (QUIC) 场景
- WebSocket over TLS 场景

## 13. 相关文档

- 方案提案：`openspec/changes/compatibility-test-guard/proposal.md`
- 执行任务：`openspec/changes/compatibility-test-guard/tasks.md`
- 能力级采集规范：`openspec/specs/compatibility-guard/spec.md`
