# 清单扫描器实现分析

基于 `jdk-algorithm-protocol-inventory.md` 中梳理的算法/协议依赖，本节详细分析如何实现清单扫描器（InventoryScanner）。

## 核心原则

1. **先扫描 JDK，后扫描库**：JDK 层面的 TLS / Provider / Kerberos 是公共基础，且 API 稳定。
2. **能扫则扫，不能扫则标记**：Kafka / SSHD / HTTP Client 的默认算法列表不一定能通过标准 API 拿到，这时应诚实标记为 `UNKNOWN` 或跳过。
3. **区分“支持”和“默认启用”**：算法在 JDK 里存在 ≠ 默认会被协商使用。
4. **不连外部服务**：扫描器只做本地静态检查。

## 扫描器分层设计

```
┌─────────────────────────────────────────┐
│         InventoryScanner                │
│         （聚合入口）                     │
└──────────────────┬──────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     │             │             │
     ▼             ▼             ▼
┌─────────┐  ┌──────────┐  ┌──────────┐
│JdkScanner│  │KafkaScanner│  │SshScanner│
│(必选)    │  │(可选)     │  │(可选)    │
└─────────┘  └──────────┘  └──────────┘
     │
     ▼
┌──────────┐  ┌──────────┐
│HttpsScanner│  │其他...   │
│(可选)     │  │          │
└──────────┘  └──────────┘
```

第一阶段建议只实现 `JdkScanner`，其他库级扫描器后续按需扩展。

---

## 一、JdkScanner 扫描项

### 1.1 JDK 基础信息

| 扫描项 | API | 示例输出 |
|--------|-----|---------|
| `java.version` | `System.getProperty("java.version")` | `17.0.12` |
| `java.vendor` | `System.getProperty("java.vendor")` | `Ubuntu` |
| `java.home` | `System.getProperty("java.home")` | `/usr/lib/jvm/...` |

### 1.2 TLS 协议

```java
SSLContext sslContext = SSLContext.getDefault();
SSLParameters supportedParams = sslContext.getSupportedSSLParameters();
SSLParameters defaultParams = sslContext.getDefaultSSLParameters();

String[] supportedProtocols = supportedParams.getProtocols();  // 所有支持
String[] enabledProtocols = defaultParams.getProtocols();      // 默认启用
```

| 概念 | 含义 | 清单检查对应 |
|------|------|-------------|
| `supportedProtocols` | JDK 支持的所有 TLS 版本 | `REQUIRED_SUPPORTED` |
| `enabledProtocols` | 默认连接会使用的 TLS 版本 | `REQUIRED_ENABLED` |

### 1.3 TLS Cipher Suite

```java
String[] supportedCipherSuites = supportedParams.getCipherSuites();
String[] enabledCipherSuites = defaultParams.getCipherSuites();
```

注意：
- `supportedCipherSuites` 包含所有实现支持的 cipher
- `enabledCipherSuites` 是默认启用的子集
- 某些 cipher 可能因 `java.security` 配置而被禁用

### 1.4 禁用的算法

```java
String disabledTlsAlgorithms = Security.getProperty("jdk.tls.disabledAlgorithms");
String legacyTlsAlgorithms = Security.getProperty("jdk.tls.legacyAlgorithms");
String disabledCertAlgorithms = Security.getProperty("jdk.certpath.disabledAlgorithms");
```

返回的是逗号分隔字符串，可能包含条件：

```
SSLv3, RC4, DES, MD5withRSA, DH keySize < 1024, EC keySize < 224, 3DES_EDE_CBC
```

**解析难点**：
- 简单值：`SSLv3`, `RC4`
- 条件值：`DH keySize < 1024`
- 需要写一个解析器，判断某个具体算法是否被禁用

简化处理：先按字符串包含匹配，后续再完善条件解析。

### 1.5 Security Provider

```java
Provider[] providers = Security.getProviders();
List<String> providerNames = Arrays.stream(providers)
    .map(Provider::getName)
    .toList();
```

常见 provider：
- `SUN`
- `SunRsaSign`
- `SunEC`
- `SunJSSE`
- `SunJCE`
- `SunJGSS`
- `SunSASL`
- `BouncyCastle`

### 1.6 Kerberos Etype

Kerberos 允许的 etype 来源有多处，按优先级扫描：

```java
// 1. 系统属性
String enctypes = System.getProperty("sun.security.krb5.enctypes");

// 2. Security 属性
String permitted = Security.getProperty("permitted_enctypes");

// 3. krb5.conf（JDK 内部 API，JDK 17+ 需 add-exports）
// sun.security.krb5.Config.getInstance().get("libdefaults", "permitted_enctypes");
```

**JDK 17+ 强封装问题**：
- `sun.security.krb5.Config` 是内部 API
- 不 add-exports 时无法访问
- 处理策略：
  - 先尝试反射读取
  - 失败时标记为 `REQUIRES_ADD_OPENS` 或 `UNKNOWN`
  - 同时给出建议的 JVM 参数

### 1.7 java.security 文件路径

没有标准 API 能拿到实际加载的 `java.security` 路径。常用推断：

```java
String javaHome = System.getProperty("java.home");
Path defaultPath = Paths.get(javaHome, "conf", "security", "java.security");
// JDK 8: jre/lib/security/java.security
```

可以通过文件存在性和最后修改时间辅助判断，但不能 100% 确定是否被覆盖。

---

## 二、KafkaScanner（可选扩展）

Kafka Client 本身没有公开 API 返回“支持的 cipher 列表”或“支持的 SASL 机制”。原因：
- Kafka TLS 基于 JDK `SSLEngine`
- 支持的算法最终取决于 JDK + `java.security` 配置

### 能扫描什么

| 扫描项 | 方式 | 说明 |
|--------|------|------|
| Kafka Client 版本 | `AppInfoParser.getVersion()` | 版本号 |
| SASL 机制是否可用 | 检查 `Sasl.createSaslClient(...)` | 仅验证 JDK 是否支持该机制 |
| TLS 支持 | 复用 JdkScanner 结果 | Kafka 依赖 JDK TLS |

### 不能扫描什么

- Kafka Client 默认 cipher 偏好
- Kafka Client 默认 TLS 协议偏好
- 这些实际上和 JDK 默认一致

**结论**：Kafka 相关清单检查很大程度上可以复用 JdkScanner，KafkaScanner 只负责版本等额外信息。

---

## 三、SshScanner（可选扩展）

SSH 算法列表严重依赖客户端库：

| 客户端库 | 能否扫描默认算法 | 方式 |
|----------|----------------|------|
| Apache SSHD Client | 可以 | 读取 `CoreModuleProperties.PREFERRED_KEX_ALGORITHMS` 等 |
| JSch | 部分可以 | 读取 `Session.getConfig(...)` |
| sshj | 可以 | 读取 `DefaultConfig` |

### Apache SSHD Client 示例

```java
// SSHD 客户端默认 factory
ClientFactoryManager client = SshClient.setUpDefaultClient();
List<NamedFactory<KeyExchange>> kexFactories = client.getKeyExchangeFactories();
List<NamedFactory<Cipher>> cipherFactories = client.getCipherFactories();
List<NamedFactory<Mac>> macFactories = client.getMacFactories();
List<NamedFactory<Signature>> signatureFactories = client.getSignatureFactories();
```

### 处理策略

- 如果项目使用 Apache SSHD Client，实现 `SshdScanner`
- 如果使用 JSch/sshj，分别实现对应 Scanner 或跳过
- 第一版可以先不实现，依赖 JdkScanner 做 TLS 层面检查

---

## 四、HttpsScanner（可选扩展）

### 出站扫描

我方作为客户端时，最终 TLS 能力来自 JDK：

```java
SSLContext sslContext = SSLContext.getDefault();
// 复用 JdkScanner 的扫描逻辑
```

但不同 HTTP Client 可能有额外限制：
- OkHttp 的 `ConnectionSpec` 会过滤 cipher
- Apache HttpClient 5 有 `TlsConfig`

这些需要库级扫描。

### 入站扫描

我方作为服务端时，TLS 能力取决于 HTTP Server：
- Tomcat Connector 配置
- Jetty `SslContextFactory`
- Undertow `Builder`
- Netty `SslContextBuilder`

这些大多通过配置而非运行时 API 决定。扫描方式：
1. 如果能读取到运行中的 Server 配置，直接读取
2. 否则退化为 JdkScanner + 配置文件解析

**第一版建议**：HTTPS 清单检查复用 JdkScanner，重点检查 TLS 协议/cipher/provider。

---

## 五、核心类设计

```java
public interface InventoryScanner {
    InventoryScanResult scan();
}

public class JdkInventoryScanner implements InventoryScanner {
    @Override
    public InventoryScanResult scan() {
        return InventoryScanResult.builder()
            .javaVersion(System.getProperty("java.version"))
            .javaVendor(System.getProperty("java.vendor"))
            .supportedTlsProtocols(scanSupportedTlsProtocols())
            .enabledTlsProtocols(scanEnabledTlsProtocols())
            .supportedTlsCipherSuites(scanSupportedTlsCipherSuites())
            .enabledTlsCipherSuites(scanEnabledTlsCipherSuites())
            .disabledTlsAlgorithms(parseDisabledAlgorithms("jdk.tls.disabledAlgorithms"))
            .disabledCertAlgorithms(parseDisabledAlgorithms("jdk.certpath.disabledAlgorithms"))
            .securityProviders(scanProviders())
            .permittedKerberosEnctypes(scanKerberosEnctypes())
            .build();
    }
}

public class InventoryScanResult {
    private String javaVersion;
    private String javaVendor;
    private List<String> supportedTlsProtocols;
    private List<String> enabledTlsProtocols;
    private List<String> supportedTlsCipherSuites;
    private List<String> enabledTlsCipherSuites;
    private List<String> disabledTlsAlgorithms;
    private List<String> disabledCertAlgorithms;
    private List<String> securityProviders;
    private List<String> permittedKerberosEnctypes;
    // getters...
}
```

---

## 六、扫描结果示例

```json
{
  "scanType": "jdk-inventory",
  "scannedAt": "2026-06-17T10:00:00Z",
  "environment": {
    "javaVersion": "17.0.12",
    "javaVendor": "Ubuntu",
    "javaHome": "/usr/lib/jvm/java-17-openjdk-amd64"
  },
  "supportedTlsProtocols": ["TLSv1.2", "TLSv1.3"],
  "enabledTlsProtocols": ["TLSv1.2", "TLSv1.3"],
  "supportedTlsCipherSuites": [
    "TLS_AES_256_GCM_SHA384",
    "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
    "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
  ],
  "enabledTlsCipherSuites": [
    "TLS_AES_256_GCM_SHA384",
    "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
  ],
  "disabledTlsAlgorithms": [
    "SSLv3",
    "RC4",
    "DES",
    "MD5withRSA",
    "DH keySize < 1024",
    "EC keySize < 224",
    "3DES_EDE_CBC",
    "DESede"
  ],
  "disabledCertAlgorithms": [
    "MD2",
    "MD5",
    "RSA keySize < 1024",
    "DSA keySize < 1024"
  ],
  "securityProviders": [
    "SUN",
    "SunRsaSign",
    "SunEC",
    "SunJSSE",
    "SunJCE",
    "SunJGSS",
    "SunSASL"
  ],
  "permittedKerberosEnctypes": [
    "aes256-cts-hmac-sha1-96",
    "aes128-cts-hmac-sha1-96"
  ]
}
```

---

## 七、实现难点与应对

| 难点 | 应对 |
|------|------|
| JDK 17+ 访问 Kerberos 内部 API 失败 | 捕获异常，标记 `UNKNOWN`，提示需 add-exports |
| `disabledAlgorithms` 格式复杂 | 先字符串包含匹配，后续用规则引擎解析 |
| 不同 JDK 厂商默认值不同 | 扫描时记录 `java.vendor`，报告里展示 |
| 库级算法偏好拿不到 | 第一版跳过，依赖 JdkScanner；后续按库实现扩展 Scanner |
| OS crypto policy 影响 | 扫描 `os.name` / `os.version`，RHEL 系列额外提示 |

---

## 八、第一阶段实现建议

只实现 `JdkInventoryScanner`，覆盖：

1. JDK 版本信息
2. TLS 协议支持/启用
3. TLS cipher 支持/启用
4. TLS 禁用算法
5. 证书路径禁用算法
6. Security provider
7. Kerberos etype（ best-effort ）

暂时不实现：
- KafkaScanner
- SshScanner
- HttpsScanner（除复用 JdkScanner 外）

这样可以在不依赖具体库版本的情况下，先把清单检查跑起来。

---

## 相关文档

- 算法/协议清单：`openspec/changes/compatibility-test-guard/jdk-algorithm-protocol-inventory.md`
- 第一层实施计划：`openspec/changes/compatibility-test-guard/layer1-inventory-check.md`
- 架构设计：`openspec/changes/compatibility-test-guard/design.md`
