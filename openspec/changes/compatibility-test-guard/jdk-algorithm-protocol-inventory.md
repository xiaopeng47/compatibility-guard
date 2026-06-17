# JDK 运行环境下各对接场景算法/协议清单

本文档梳理在 JDK 运行环境下，Kafka、SFTP、Kerberos、HTTPS 入站、HTTPS 出站五类对接场景涉及的算法、协议、证书及配置项。用于生成 `compatibility-inventory.yaml` 依赖清单。

## 1. Kafka Client 对接

Kafka Client 通过 `security.protocol` 决定传输层和认证层组合：

| 协议 | TLS | SASL | 适用场景 |
|------|-----|------|---------|
| `PLAINTEXT` | ❌ | ❌ | 内网非敏感 |
| `SSL` | ✅ | ❌ | 单向 TLS |
| `SASL_PLAINTEXT` | ❌ | ✅ | 内网认证 |
| `SASL_SSL` | ✅ | ✅ | 生产最常见 |

### 1.1 TLS 层算法/协议

| 类别 | 常见取值 | 说明 |
|------|---------|------|
| TLS 协议版本 | `TLSv1.2`, `TLSv1.3` | JDK 8 默认 TLSv1.2；JDK 11+ 支持 TLSv1.3 |
| 密码套件 | `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`, `TLS_AES_256_GCM_SHA384` | 前者 TLSv1.2，后者 TLSv1.3 |
| 证书类型 | RSA, ECDSA | 服务端证书公钥算法 |
| 签名算法 | `SHA256withRSA`, `SHA384withRSA` | 证书签名算法 |
| Truststore 类型 | `JKS`, `PKCS12` | PKCS12 为现代首选 |
| Keystore 类型 | `JKS`, `PKCS12` | 客户端证书场景 |
| Hostname 校验 | `HTTPS`, `""` | Kafka 3.x 默认 `HTTPS` |

### 1.2 SASL 认证机制

| 机制 | 说明 | JDK/依赖要求 |
|------|------|-------------|
| `GSSAPI` | Kerberos 认证 | 需 JGSS，JDK 17+ 可能需 add-exports |
| `PLAIN` | 明文用户名密码 | 简单，无特殊 JDK 依赖 |
| `SCRAM-SHA-256` | 加盐挑战响应 | JDK 内置 SASL 支持 |
| `SCRAM-SHA-512` | 更强哈希 | JDK 内置 SASL 支持 |
| `OAUTHBEARER` | OAuth token | 需自定义回调处理器 |

### 1.3 JDK 相关注意点

| JDK 版本 | 影响 |
|---------|------|
| JDK 8 | 默认 TLSv1.2，需确认 JCE unlimited strength |
| JDK 11 | 支持 TLSv1.3，默认 cipher 列表变化 |
| JDK 17 | 默认禁用部分旧 cipher，强封装影响 Kerberos |

---

## 2. SFTP/SSH 对接

SFTP/SSH 握手协商分为：KEX、Host Key、Cipher、MAC、Compression、认证。

### 2.1 密钥交换 KEX

| 算法 | 安全性 | 兼容性 |
|------|--------|--------|
| `curve25519-sha256` | 强 | 现代系统标配 |
| `diffie-hellman-group14-sha256` | 强 | 兼容性最好 |
| `diffie-hellman-group16-sha512` | 强 | 安全但较慢 |
| `diffie-hellman-group-exchange-sha256` | 强 | 灵活 |
| `ecdh-sha2-nistp256` | 强 | 常见 |
| `rsa2048-sha256` | 中 | 已不推荐使用 |

### 2.2 Host Key 算法

| 算法 | 状态 | 注意 |
|------|------|------|
| `ssh-rsa` | 已废弃 | JDK 17+ / SSHD 2.15+ 常出问题 |
| `rsa-sha2-256` | 推荐 | RSA + SHA-2 |
| `rsa-sha2-512` | 推荐 | RSA + SHA-2 |
| `ssh-ed25519` | 强 | 现代首选 |
| `ecdsa-sha2-nistp256` | 中 | 部分旧系统支持 |

### 2.3 加密算法 Cipher

| 算法 | 模式 | 状态 |
|------|------|------|
| `aes256-ctr` | 流密码 | 推荐 |
| `aes128-ctr` | 流密码 | 推荐 |
| `aes256-gcm@openssh.com` | AEAD | 强，OpenSSH 6.2+ |
| `aes128-gcm@openssh.com` | AEAD | 强 |
| `3des-cbc` | CBC | 已弱，应避免 |

### 2.4 MAC 算法

| 算法 | 状态 |
|------|------|
| `hmac-sha2-256` | 推荐 |
| `hmac-sha2-512` | 推荐 |
| `hmac-sha1` | 可用，但不如 sha2 |
| `hmac-md5` | 已弱，应避免 |

### 2.5 认证方式

| 方式 | 说明 |
|------|------|
| `publickey` | 公钥认证，最常见 |
| `password` | 密码认证 |
| `keyboard-interactive` | 交互式 |

### 2.6 客户端库差异

| 客户端 | 默认算法列表 | JDK 强封装影响 |
|--------|-------------|---------------|
| Apache SSHD Client | 较新，默认禁用 ssh-rsa | 较小 |
| JSch | 较老，可能默认 ssh-rsa | 可能需要 add-exports |
| sshj | 中等 | 一般无强封装问题 |

---

## 3. Kerberos 对接

Kerberos 主要用于 Kafka GSSAPI 认证，也可能用于 SSH/SFTP。

### 3.1 加密类型 etype

| etype | 强度 | JDK 默认支持 |
|-------|------|------------|
| `aes256-cts-hmac-sha1-96` | 强 | ✓ |
| `aes128-cts-hmac-sha1-96` | 中 | ✓ |
| `aes256-cts-hmac-sha384-192` | 强 | JDK 11+ |
| `aes128-cts-hmac-sha256-128` | 强 | JDK 11+ |
| `des3-cbc-sha1` | 弱 | JDK 17- 默认禁用 |
| `des-cbc-md5` | 弱 | JDK 17- 默认禁用 |
| `rc4-hmac` | 弱 | JDK 17- 默认禁用 |

### 3.2 配置项

| 配置 | 来源 | 说明 |
|------|------|------|
| `permitted_enctypes` | `java.security` / `krb5.conf` | 客户端允许的 etype |
| `default_tkt_enctypes` | `krb5.conf` | 默认 TGT etype |
| `default_tgs_enctypes` | `krb5.conf` | 默认 TGS etype |
| `default_realm` | `krb5.conf` | 默认 realm |
| `kdcs` | `krb5.conf` | KDC 地址列表 |

### 3.3 JDK 17+ 强封装

读取 Kerberos ticket 等内部类需要 JVM 参数：

```bash
--add-exports java.security.jgss/sun.security.jgss=ALL-UNNAMED
--add-exports java.security.jgss/sun.security.jgss.krb5=ALL-UNNAMED
--add-exports java.security.krb5/sun.security.krb5=ALL-UNNAMED
--add-opens java.base/sun.security.util=ALL-UNNAMED
```

---

## 4. HTTPS 入站（我方作为服务端）

第三方客户端调用我方 HTTPS 接口。

### 4.1 TLS 服务端配置

| 类别 | 常见取值 | 说明 |
|------|---------|------|
| TLS 协议版本 | `TLSv1.2`, `TLSv1.3` | 服务端提供的版本 |
| 密码套件 | `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` 等 | 服务端提供的 cipher |
| 证书密钥类型 | RSA, ECDSA | 服务端证书 |
| 证书签名算法 | `SHA256withRSA` | 证书链签名 |
| 客户端证书要求 | `NONE`, `WANT`, `NEED` | mTLS 场景用 `NEED` |
| ALPN | `h2`, `http/1.1` | 协商 HTTP/2 |

### 4.2 容器差异

| 容器 | 配置项 | 注意 |
|------|--------|------|
| Tomcat | `sslEnabledProtocols`, `ciphers` | 配置在 Connector |
| Jetty | `SslContextFactory` | 包含 `IncludeProtocols`, `IncludeCipherSuites` |
| Undertow | `OPENSSL` / `JDK` provider | 算法列表配置方式不同 |
| Netty | `SslContextBuilder` | 需显式配置 cipher suites |

### 4.3 第三方客户端兼容性

入站兼容性取决于调用方的客户端能力：
- 旧版 Java 客户端可能不支持 TLSv1.3
- 某些客户端只支持特定 cipher
- 部分客户端需要 SNI

---

## 5. HTTPS 出站（我方作为客户端）

我方调用第三方 HTTPS 接口。

### 5.1 TLS 客户端配置

| 类别 | 常见取值 | 说明 |
|------|---------|------|
| TLS 协议版本 | `TLSv1.2`, `TLSv1.3` | 客户端支持的版本 |
| 密码套件 | `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` 等 | 客户端支持的 cipher |
| Truststore 类型 | `JKS`, `PKCS12` | 信任第三方证书 |
| Hostname 校验 | `HTTPS`, `""` | 默认通常为 `HTTPS` |
| ALPN | `h2`, `http/1.1` | 协商 HTTP/2 |

### 5.2 HTTP Client 实现差异

| 客户端 | TLS 配置方式 | 默认行为 |
|--------|-------------|---------|
| `HttpURLConnection` | 系统属性 / SSLContext | 较老，默认可能较保守 |
| JDK 11+ `HttpClient` | SSLContext / SSLParameters | 现代，支持 HTTP/2 |
| Apache HttpClient 5 | SSLContextBuilder | 灵活 |
| OkHttp | `ConnectionSpec` | 现代默认，禁用旧算法 |

### 5.3 mTLS 场景

| 配置 | 说明 |
|------|------|
| Keystore | 我方客户端证书 |
| Key manager algorithm | `SunX509` / `NewSunX509` |
| 证书链 | 需包含完整客户端证书链 |

---

## 6. 跨场景 JDK 安全配置

这些配置影响所有 TLS/Kerberos 场景。

### 6.1 java.security 关键属性

| 属性 | 影响 |
|------|------|
| `jdk.tls.disabledAlgorithms` | 禁用 TLS 协议/算法 |
| `jdk.certpath.disabledAlgorithms` | 禁用证书路径算法 |
| `jdk.tls.legacyAlgorithms` | 遗留算法标记 |
| `permitted_enctypes` | Kerberos 允许 etype |

### 6.2 Security Provider

| Provider | 作用 |
|----------|------|
| `SunJSSE` | TLS/SSL |
| `SunJCE` | 加密算法 |
| `SunJGSS` | Kerberos/GSS-API |
| `SunSASL` | SASL |
| `BouncyCastle` | 扩展算法/国密 |

### 6.3 环境相关

| 因素 | 影响 |
|------|------|
| 容器基础 OS | RHEL crypto-policies 可系统级禁用算法 |
| `JAVA_TOOL_OPTIONS` | 全局 JVM 参数 |
| `KRB5_CONFIG` | Kerberos 配置路径 |
| FIPS 模式 | 限制可用算法 |

---

## 7. 清单检查项汇总

基于以上分析，`compatibility-inventory.yaml` 建议包含以下检查项：

```yaml
required:
  jdk:
    tlsProtocols: ["TLSv1.2"]
    tlsCipherSuites:
      - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
      - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
    providers: ["SunJSSE", "SunJCE", "SunJGSS"]
    kerberosEnctypes:
      - "aes256-cts-hmac-sha1-96"
      - "aes128-cts-hmac-sha1-96"
    
  kafka:
    securityProtocols: ["SASL_SSL"]
    saslMechanisms: ["GSSAPI", "SCRAM-SHA-512"]
    tlsProtocols: ["TLSv1.2"]
    tlsCipherSuites:
      - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
    
  sftp:
    kexAlgorithms:
      - "curve25519-sha256"
      - "diffie-hellman-group14-sha256"
    ciphers:
      - "aes256-ctr"
      - "aes128-ctr"
    macs:
      - "hmac-sha2-256"
      - "hmac-sha2-512"
    hostKeyAlgorithms:
      - "rsa-sha2-256"
      - "rsa-sha2-512"
      - "ssh-ed25519"
    authMethods: ["publickey"]
    
  https:
    inbound:
      tlsProtocols: ["TLSv1.2"]
      tlsCipherSuites:
        - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
      certificateKeyTypes: ["RSA"]
      clientAuth: "NONE"
      alpnProtocols: ["http/1.1", "h2"]
    outbound:
      tlsProtocols: ["TLSv1.2"]
      tlsCipherSuites:
        - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
      endpointIdentificationAlgorithm: "HTTPS"
      alpnProtocols: ["http/1.1", "h2"]
```

---

## 8. 常见升级风险点

| 场景 | 风险 |
|------|------|
| JDK 8 → 11 | TLSv1.3 引入，默认 cipher 列表变化 |
| JDK 11 → 17 | 强封装影响 Kerberos；部分旧 cipher 默认禁用 |
| Kafka Client 升级 | hostname verification 默认行为变化 |
| Apache SSHD 升级 | 默认禁用 `ssh-rsa` |
| HTTP Client 升级 | ALPN / HTTP/2 默认行为变化 |
| 容器镜像升级 | OS crypto policy 系统级限制算法 |

---

## 相关文档

- 第一层实施计划：`openspec/changes/compatibility-test-guard/layer1-inventory-check.md`
- 架构设计：`openspec/changes/compatibility-test-guard/design.md`
- 方案提案：`openspec/changes/compatibility-test-guard/proposal.md`
- 执行任务：`openspec/changes/compatibility-test-guard/tasks.md`
- 能力级规范：`openspec/specs/compatibility-guard/spec.md`
