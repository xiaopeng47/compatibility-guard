# 兼容性看护采集规范

## 1. 目标

本文档定义服务对接第三方 Kafka、SFTP 场景下，需在 JDK、Kafka Client、Apache SSHD、Kerberos 等组件升级时采集的算法、协议及环境信息。采集结果用于建立兼容性基线，支撑日常版本迭代中的自动 diff 与告警。

## 2. 采集对象总览

```
┌─────────────────────────────────────────────────────────────────┐
│                         采集对象                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐  │
│   │    JDK      │   │   Kafka     │   │        SFTP         │  │
│   │   环境层     │   │  传输/认证层 │   │      SSH 层         │  │
│   └──────┬──────┘   └──────┬──────┘   └──────────┬──────────┘  │
│          │                 │                      │             │
│          │           ┌─────┴─────┐                │             │
│          │           │ Kerberos  │                │             │
│          │           │ 认证专题   │                │             │
│          │           └───────────┘                │             │
│          │                                         │             │
│          └─────────────────────────────────────────┘             │
│                          │                                      │
│                          ▼                                      │
│              统一采集 → 基线库存储 → diff → 告警                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 采集优先级

| 优先级 | 含义 | 触发时机 |
|--------|------|----------|
| P0 | 核心兼容性项，失败即阻断 | 每次 PR、每次 JDK/依赖升级 |
| P1 | 基线采集与失败排查项 | 初始基线、nightly、失败时 |
| P2 | 深度诊断与特殊场景项 | 疑难问题排查、安全审计 |

## 4. JDK 环境层采集

### 4.1 基础版本信息（P0）

| 采集项 | 来源 | 说明 |
|--------|------|------|
| `java.version` | `System.getProperty("java.version")` | 主/次/补丁版本 |
| `java.vendor` | `System.getProperty("java.vendor")` | OpenJDK/Oracle/其他 |
| `java.home` | `System.getProperty("java.home")` | JDK 安装路径 |

### 4.2 安全策略配置（P0）

| 采集项 | 来源 | 说明 |
|--------|------|------|
| `jdk.tls.disabledAlgorithms` | `java.security` | 禁用的 TLS 算法 |
| `jdk.tls.legacyAlgorithms` | `java.security` | 遗留算法 |
| `jdk.certpath.disabledAlgorithms` | `java.security` | 证书路径禁用算法 |
| `jdk.tls.client.protocols` | 系统属性 | 客户端默认 TLS 版本 |

### 4.3 Provider 与扩展配置（P1）

| 采集项 | 来源 | 说明 |
|--------|------|------|
| `security.providers` | `Security.getProviders()` | 已加载安全 provider |
| `java.security.file.path` | 运行时检测 | 实际加载的 java.security 路径 |
| `java.security.file.hash` | 文件哈希 | 判断是否被修改 |
| `BouncyCastle version` | 依赖扫描/运行时 | 第三方 provider 版本 |
| `JCE policy` | 运行时检测 | JDK 8/11 是否 Unlimited |

### 4.4 系统属性与环境变量（P1）

| 采集项 | 来源 | 说明 |
|--------|------|------|
| `JAVA_TOOL_OPTIONS` | 环境变量 | 全局 JVM 参数 |
| `JDK_JAVA_OPTIONS` | 环境变量 | JDK 特定参数 |
| `javax.net.ssl.*` | 系统属性 | 自定义 SSL 配置 |
| `jsse.enableSNIExtension` | 系统属性 | SNI 开关 |
| `https.protocols` / `https.cipherSuites` | 系统属性 | HttpsURLConnection 默认 |

### 4.5 高级配置（P2）

| 采集项 | 来源 | 说明 |
|--------|------|------|
| `com.sun.security.enableCRLDP` | 系统属性 | CRL 分发点 |
| `ocsp.enable` | 系统属性 | OCSP 校验 |
| TLS extensions | SSL debug | ALPN、SNI 等扩展 |
| FIPS mode | 运行时检测 | 是否运行在 FIPS 140-2/140-3 模式 |

## 5. Kafka 传输层采集

### 5.1 TLS 层（P0）

| 采集项 | 说明 | 常见问题 |
|--------|------|---------|
| `tlsVersion` | 协商出的 TLS 版本 | JDK 升级后禁用旧版本 |
| `cipherSuite` | 协商出的密码套件 | Client/broker cipher 无交集 |
| `enabledProtocols` | broker 支持的 TLS 版本列表 | broker 只支持 TLSv1.1 |
| `supportedCipherSuites` | 客户端支持的 cipher 列表 | 升级后列表变短 |
| `endpointIdentificationAlgorithm` | hostname 校验算法 | Kafka 3.x 默认行为变化 |
| `keystoreType` / `truststoreType` | JKS / PKCS12 | 默认偏好变化 |

### 5.2 证书信息（P1）

| 采集项 | 说明 | 常见问题 |
|--------|------|---------|
| `serverCertificateSubject` | 证书主题 | 证书轮换 |
| `serverCertificateSANs` | Subject Alternative Names | hostname verification 失败 |
| `serverCertificateIssuer` | 签发机构 | 根证书信任链变化 |
| `serverCertificateValidFrom/To` | 有效期 | 证书过期 |
| `serverCertificateSignatureAlgorithm` | 签名算法 | JDK 禁用 SHA1 签名 |
| `certificateChain` | 完整证书链 | 中间证书缺失 |
| `sniEnabled` / `sniServerName` | SNI 配置 | broker 要求 SNI |

### 5.3 SASL 层（P0）

| 采集项 | 说明 | 取值示例 |
|--------|------|---------|
| `saslMechanism` | 认证机制 | GSSAPI / SCRAM-SHA-256 / SCRAM-SHA-512 / PLAIN / OAUTHBEARER |
| `security.protocol` | 安全协议 | SASL_SSL / SASL_PLAINTEXT / SSL / PLAINTEXT |
| `sasl.jaas.config` | JAAS 配置内容 | - |

### 5.4 Kafka 协议与行为层（P1）

| 采集项 | 说明 | 常见问题 |
|--------|------|---------|
| `apiVersions` | 协商的 API 版本 | Client 与 broker 版本差距 |
| `clientSoftwareVersion` | 客户端版本声明 | 兼容性问题 |
| `interBrokerProtocolVersion` | broker 间协议版本 | 与 client 兼容 |
| `logMessageFormatVersion` | 消息格式版本 | 新 client 读旧格式 |
| `connectionsMaxIdleMs` | 连接空闲超时 | 默认值变化导致断开 |
| `metadataMaxAgeMs` | 元数据刷新间隔 | 默认值变化 |
| `requestTimeoutMs` | 请求超时 | 延迟变化 |

### 5.5 SASL/OAuth 扩展（P2）

| 采集项 | 说明 |
|--------|------|
| `sasl.login.callback.handler.class` | OAuth 回调处理器 |
| `sasl.oauthbearer.token.endpoint.url` | Token endpoint |
| `sasl.oauthbearer.scope` | OAuth scope |
| `sasl.oauthbearer.expected.audience` | Audience 校验 |
| `sasl.oauthbearer.expected.issuer` | Issuer 校验 |

## 6. Kafka Kerberos 认证专题采集

### 6.1 JAAS 配置（P0）

| 采集项 | 说明 | 示例 |
|--------|------|------|
| `loginModule` | LoginModule 类 | `com.sun.security.auth.module.Krb5LoginModule` |
| `useKeyTab` | 是否使用 keytab | `true` / `false` |
| `keyTab` | keytab 路径 | `/etc/security/keytabs/kafka.keytab` |
| `principal` | 客户端 principal | `kafka-client@EXAMPLE.COM` |
| `useTicketCache` | 是否使用 ticket cache | `true` / `false` |
| `renewTGT` | 是否续期 TGT | `true` / `false` |
| `storeKey` | 是否存储 key | `true` / `false` |
| `isInitiator` | 是否是发起方 | `true` / `false` |

### 6.2 Kerberos 加密类型（P0）

| 采集项 | 说明 | 常见取值 |
|--------|------|---------|
| `clientSupportedEtTypes` | 客户端支持的 etype | aes256-cts-hmac-sha1-96, aes128-cts-hmac-sha1-96 |
| `kdcSupportedEtTypes` | KDC 支持的 etype | 可能包含 rc4-hmac, des-cbc-md5 |
| `negotiatedEType` | 实际协商出的 etype | - |
| `permittedEnctypes` | `krb5.conf` 配置 | 客户端允许的 etype |
| `defaultTktEnctypes` | 默认 TGT etype | - |
| `defaultTgsEnctypes` | 默认 TGS etype | - |

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

### 6.3 票证生命周期（P1）

| 采集项 | 说明 |
|--------|------|
| `tgtStartTime` | TGT 生效时间 |
| `tgtEndTime` | TGT 过期时间 |
| `tgtRenewTill` | TGT 可续期截止时间 |
| `serviceTicketStartTime` | Service ticket 生效时间 |
| `serviceTicketEndTime` | Service ticket 过期时间 |
| `ticketLifetime` | 票证有效期配置 |
| `renewable` | 是否可续期 |
| `forwardable` | 是否可转发 |

### 6.4 KDC 与 realm 配置（P1）

| 采集项 | 说明 |
|--------|------|
| `defaultRealm` | 默认 realm |
| `kdcs` | KDC 地址列表 |
| `dnsLookupKdc` | 是否 DNS 解析 KDC |
| `dnsLookupRealm` | 是否 DNS 解析 realm |
| `udpPreferenceLimit` | UDP/TCP 偏好 |
| `kdcTimeout` | KDC 超时 |
| `crossRealm` | 是否跨 realm |

### 6.5 Kafka Client Kerberos 配置（P0）

| 采集项 | 说明 |
|--------|------|
| `sasl.kerberos.service.name` | broker 服务名 |
| `sasl.kerberos.kinit.cmd` | kinit 命令路径 |
| `sasl.kerberos.min.time.before.relogin` | 重登录间隔 |
| `sasl.kerberos.ticket.renew.jitter` | 续期抖动 |
| `sasl.kerberos.ticket.renew.window.factor` | 续期窗口因子 |

### 6.6 JDK 17+ 强封装相关（P0）

| 采集项 | 说明 |
|--------|------|
| `jvm.addExports` | 是否需 `--add-exports` |
| `jvm.addOpens` | 是否需 `--add-opens` |

常用参数：

```bash
--add-exports java.security.jgss/sun.security.jgss=ALL-UNNAMED
--add-exports java.security.jgss/sun.security.jgss.krb5=ALL-UNNAMED
--add-exports java.security.krb5/sun.security.krb5=ALL-UNNAMED
--add-opens java.base/sun.security.util=ALL-UNNAMED
```

### 6.7 Kerberos 高级协议细节（P2）

| 采集项 | 说明 |
|--------|------|
| `preauthType` / `paType` | 预认证类型 |
| `kvno` | Key version number |
| `saltType` | Salt 类型 |
| `kerberosFlags` | forwardable / renewable / proxiable |
| `canonicalize` | 名称规范化 |
| `spnegoMechanism` | GSS-API mechanism OID |

### 6.8 Kerberos 环境变量（P1）

| 采集项 | 说明 |
|--------|------|
| `KRB5_CONFIG` | krb5.conf 路径 |
| `KRB5CCNAME` | ticket cache 路径 |
| `KRB5_CLIENT_KTNAME` | client keytab 路径 |
| `KRB5_TRACE` | debug 输出路径 |

## 7. SFTP/SSH 层采集

### 7.1 密钥交换 KEX（P0）

| 采集项 | 说明 | 常见取值 |
|--------|------|---------|
| `kexAlgorithm` | 协商出的 KEX 算法 | curve25519-sha256 |
| `clientKexAlgorithms` | 客户端支持的 KEX | diffie-hellman-group14-sha256 |
| `serverKexAlgorithms` | 服务端支持的 KEX | ecdh-sha2-nistp256 |

常见 KEX 算法：

- `curve25519-sha256`
- `diffie-hellman-group14-sha256`
- `diffie-hellman-group16-sha512`
- `diffie-hellman-group-exchange-sha256`
- `ecdh-sha2-nistp256`
- `rsa2048-sha256`

### 7.2 Host Key（P0）

| 采集项 | 说明 | 常见取值 |
|--------|------|---------|
| `serverHostKeyAlgorithm` | 协商出的 host key 算法 | rsa-sha2-256 |
| `serverHostKeyFingerprint` | host key fingerprint | SHA256:... |
| `clientHostKeyAlgorithms` | 客户端支持的 host key | ssh-ed25519 |
| `serverHostKeyAlgorithms` | 服务端支持的 host key | ssh-rsa |

常见 Host Key 算法：

- `ssh-rsa`（JDK 17+ / SSHD 2.15+ 常出问题）
- `rsa-sha2-256`
- `rsa-sha2-512`
- `ssh-ed25519`
- `ecdsa-sha2-nistp256`

### 7.3 加密算法 Cipher（P0）

| 采集项 | 说明 | 常见取值 |
|--------|------|---------|
| `c2sCipher` | client-to-server cipher | aes256-ctr |
| `s2cCipher` | server-to-client cipher | aes128-ctr |
| `clientCiphers` | 客户端支持的 cipher | aes256-gcm@openssh.com |
| `serverCiphers` | 服务端支持的 cipher | 3des-cbc（旧算法） |

常见 cipher：

- `aes256-ctr`
- `aes128-ctr`
- `aes256-gcm@openssh.com`
- `aes128-gcm@openssh.com`
- `3des-cbc`（旧算法）

### 7.4 MAC（P0）

| 采集项 | 说明 | 常见取值 |
|--------|------|---------|
| `c2sMac` | client-to-server MAC | hmac-sha2-256 |
| `s2cMac` | server-to-client MAC | hmac-sha2-512 |
| `clientMacs` | 客户端支持的 MAC | hmac-sha1 |
| `serverMacs` | 服务端支持的 MAC | hmac-md5（旧算法） |

常见 MAC：

- `hmac-sha2-256`
- `hmac-sha2-512`
- `hmac-sha1`
- `hmac-md5`（旧算法）

### 7.5 压缩（P1）

| 采集项 | 说明 |
|--------|------|
| `c2sCompression` | client-to-server compression |
| `s2cCompression` | server-to-client compression |

### 7.6 认证层（P0）

| 采集项 | 说明 | 取值 |
|--------|------|------|
| `authMethods` | 支持的认证方式 | publickey / password / keyboard-interactive |
| `clientAuthMethod` | 实际使用的认证方式 | publickey |
| `publicKeyAlgorithm` | 公钥算法 | rsa-sha2-256 / ed25519 |
| `privateKeyFormat` | 私钥格式 | OpenSSH / PKCS#8 / PEM |
| `strictHostKeyChecking` | 是否严格校验 host key | yes / no / ask |
| `knownHostsFormat` | known_hosts 格式 | hashed / plain |
| `hostKeyVerifier` | 自定义 verifier | 类名 |

### 7.7 SFTP 协议层（P1）

| 采集项 | 说明 |
|--------|------|
| `sftpVersion` | SFTP 协议版本（3/4/5/6） |
| `sftpExtensions` | 服务端支持的扩展 |
| `sftpMaxPacketSize` | 最大包大小 |
| `sftpMaxReadSize` | 最大读取大小 |

### 7.8 SSH 会话层（P2）

| 采集项 | 说明 |
|--------|------|
| `sshBanner` | 服务端 banner |
| `serverVersion` | SSH 软件版本 |
| `clientVersion` | 客户端版本声明 |
| `sessionTimeout` | 会话超时 |
| `keepAliveInterval` | 心跳间隔 |

## 8. 组件版本采集

| 组件 | 采集项 | 优先级 |
|------|--------|--------|
| Kafka Client | 版本号 | P0 |
| Apache SSHD | 版本号 | P0 |
| JSch / sshj | 版本号 | P0（如使用） |
| Spring Boot | 管理的 Kafka/SSHD 版本 | P1 |
| BouncyCastle | 版本号 | P1 |
| 国密 provider | 版本号 | P1（如使用 SM 系列算法） |

## 9. 运行环境采集

### 9.1 操作系统层（P1）

| 采集项 | 说明 | 典型问题 |
|--------|------|---------|
| `osCryptoPolicy` | RHEL/Fedora crypto-policies | 系统级禁用算法 |
| `timezone` | 时区 | Kerberos 时间校验 |
| `ntpSynced` | NTP 同步状态 | Kerberos 时钟偏差 |
| `locale` | 区域设置 | 编码问题 |

### 9.2 容器层（P1）

| 采集项 | 说明 |
|--------|------|
| `containerImageDigest` | 容器镜像摘要 |
| `containerBaseOs` | 基础系统 |
| `openSslVersion` | OpenSSL/LibreSSL 版本 |

### 9.3 网络层（P2）

| 采集项 | 说明 |
|--------|------|
| `proxyConfig` | HTTP/SOCKS 代理 |
| `dnsResolution` | DNS 解析结果 |
| `tcpConnectTime` | TCP 连接时间 |
| `handshakeTime` | TLS/SSH handshake 耗时 |
| `firewallIdleTimeout` | 防火墙 idle 超时 |

## 10. 统一采集输出示例

```json
{
  "baselineId": "partner-a-2026-06-01",
  "collectedAt": "2026-06-01T10:00:00Z",
  "environment": {
    "jdk": "17.0.12",
    "kafkaClient": "3.6.2",
    "sshd": "2.15.0",
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
  }
}
```

## 11. 看护触发条件

当以下任意项发生变化时，应触发兼容性看护：

1. JDK 主/次版本升级
2. JDK 小版本升级（可能伴随 security baseline 变化）
3. Kafka Client 版本升级
4. Apache SSHD / JSch / sshj 版本升级
5. Spring Boot / 依赖管理平台升级
6. BouncyCastle / 国密 provider 升级
7. `java.security` 配置变更
8. `krb5.conf` / JAAS 配置变更
9. keytab / 证书轮换
10. 容器基础镜像升级

## 12. 待补充场景

以下场景在后续分析中需进一步确认是否纳入：

- 国密 SM2/SM3/SM4 算法场景
- 双向 TLS（mTLS）客户端证书场景
- FIPS 140-2/140-3 模式场景
- 多云/多区域部署差异
- 动态 provider 加载（运行时 `Security.addProvider`）
- 自定义 TrustManager / HostnameVerifier
