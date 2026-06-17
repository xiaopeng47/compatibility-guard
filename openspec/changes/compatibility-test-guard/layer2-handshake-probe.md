# 第二层：动态握手探测 - 整体方案

## 目标

回答一个问题：**升级后，实际和第三方/代表服务协商出来的结果变了吗？**

```
我方代码 + 当前 JDK/Client
        │
        ▼
   连接目标（代表服务 / 真实三方）
        │
        ▼
   完成握手（TLS / SASL / SSH / Kerberos）
        │
        ▼
   读取协商结果
        │
        ▼
   与 baseline/*.json 期望结果 diff
```

## 与第一层的关系

| 问题 | 第一层（清单检查） | 第二层（握手探测） |
|------|------------------|------------------|
| 算法被禁用了吗？ | ✅ | ✅（表现为握手失败或协商结果变化） |
| 算法还在支持列表吗？ | ✅ | ❌ |
| 默认启用偏好变了吗？ | ⚠️ WARN | ✅ |
| 实际能协商成功吗？ | ❌ | ✅ |
| 三方侧配置变了吗？ | ❌ | ✅ |
| 证书/SAN/有效期变了吗？ | ❌ | ✅ |

**运行顺序**：先跑第一层（秒级），再跑第二层（分钟级）。

## 探测目标

| 目标类型 | 优点 | 缺点 | 适用场景 |
|---------|------|------|---------|
| **代表服务（Testcontainers）** | 稳定、快速、可重复、无网络依赖 | 不能反映三方真实变化 | 日常 CI、本地开发 |
| **真实三方测试环境** | 最真实 | 可能不稳定、需网络/凭证 | Nightly、发布前 |
| **生产只读采样** | 最兜底 | 风险高、需授权 | 特殊场景 |

## 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                     握手探测测试套件                         │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │          BaselineLoader（基线加载器）                │   │
│   │   读取 baseline/<partner>-<subject>.json            │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │          ProbeHelper（探测辅助工具）                 │   │
│   │   KafkaTransportProbe / SshProbe / KerberosProbe   │   │
│   │   HttpsInboundProbe / HttpsOutboundProbe            │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │          TargetProvider（目标提供者）                │   │
│   │   Testcontainers / Real Environment / Production   │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │          BaselineComparator（基线比对器）            │   │
│   │   当前结果 vs 基线 → diff                           │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 各专题探测方案

### 1. Kafka Transport 探测

#### 目标

- TLS 版本、Cipher suite
- SASL 机制
- 证书信息（subject、SAN、issuer、有效期、fingerprint、签名算法）
- API 版本
- Hostname verification 行为

#### 实现方式

**方案 A：裸 TLS Socket 探测（推荐用于 TLS 层）**

```java
SSLSocketFactory factory = SSLContext.getDefault().getSocketFactory();
SSLSocket socket = (SSLSocket) factory.createSocket(brokerHost, brokerPort);
socket.startHandshake();
SSLSession session = socket.getSession();

String tlsVersion = session.getProtocol();
String cipherSuite = session.getCipherSuite();
Certificate[] certs = session.getPeerCertificates();
```

优点：直接拿到 TLS 协商结果。
缺点：不做 SASL，只能用于 `SSL` 和 `SASL_SSL` 的 TLS 层。

**方案 B：Kafka AdminClient + 自定义 SSL 工厂（推荐用于 SASL_SSL）**

通过自定义 `SslEngineFactory` 或包装 `SSLEngine` 捕获协商结果。Kafka Client 内部使用 `SSLEngine`，需要包装以记录 handshake 后的 `SSLSession`。

```java
// 伪代码：包装 SSLEngine
SSLEngine wrappedEngine = new SslEngineWrapper(delegate) {
    @Override
    public void beginHandshake() throws SSLException {
        delegate.beginHandshake();
    }
    
    // handshake 完成后读取 session
    public SSLSession getSessionAfterHandshake() {
        return delegate.getSession();
    }
};
```

优点：覆盖 TLS + SASL 完整流程。
缺点：实现复杂，依赖 Kafka Client 内部结构。

**方案 C：AdminClient 连接 + 单独 TLS 探测（推荐第一版）**

- 用 `AdminClient.describeCluster()` 验证 SASL/认证是否成功
- 用裸 TLS Socket 探测 TLS 协商结果
- SASL 机制从配置中读取

**第一版建议**：方案 C。

### 2. Kafka Kerberos 探测

#### 目标

- JAAS 配置解析
- Kerberos etype 协商结果
- Ticket 生命周期
- JDK 17+ 强封装是否需要 add-exports

#### 实现方式

使用 `Krb5LoginModule` 登录并读取 Ticket：

```java
LoginContext lc = new LoginContext("KafkaClient", callbackHandler);
lc.login();
Subject subject = lc.getSubject();

Set<Ticket> tickets = subject.getPrivateCredentials(Ticket.class);
for (Ticket ticket : tickets) {
    EncryptionKey key = ticket.getSessionKey();
    int etype = key.getEType();
    Date startTime = ticket.getStartTime();
    Date endTime = ticket.getEndTime();
    boolean renewable = ticket.isRenewable();
}
```

**JDK 17+ 处理**：
- `Ticket` 类在 `javax.security.auth.kerberos` 包，公开 API
- 但读取某些内部字段可能需要 `sun.security.krb5` 内部类
- 探测时尝试无 add-exports 读取，失败则标记需要 add-exports

### 3. SFTP/SSH 探测

#### 目标

- KEX 算法
- Host key 算法 / fingerprint
- Cipher（C2S / S2C）
- MAC（C2S / S2C）
- 认证方式
- SFTP 协议版本

#### 实现方式（Apache SSHD Client）

```java
SshClient client = SshClient.setUpDefaultClient();
client.start();

ClientSession session = client.connect(username, host, port)
    .verify(timeout)
    .getSession();

session.addPasswordIdentity(password);
session.auth().verify();

// 读取协商结果
String kex = session.getNegotiatedKexAlgorithm();
PublicKey serverKey = session.getServerKey();
String hostKeyAlgo = serverKey.getAlgorithm();
String hostKeyFingerprint = KeyUtils.getFingerPrint(serverKey);
String c2sCipher = session.getNegotiatedCipher(ClientSession.ClientServer.C2S);
String s2cCipher = session.getNegotiatedCipher(ClientSession.ClientServer.S2C);
String c2sMac = session.getNegotiatedMac(ClientSession.ClientServer.C2S);
String s2cMac = session.getNegotiatedMac(ClientSession.ClientServer.S2C);

// SFTP 版本
SftpClient sftp = session.createSftpClient();
int sftpVersion = sftp.getVersion();
```

### 4. HTTPS 入站探测

#### 目标

- 我方 HTTPS 服务端与代表客户端协商出的 TLS 版本、Cipher
- 证书信息
- ALPN / HTTP 协议版本
- mTLS 客户端证书要求

#### 实现方式

```java
// 1. 启动本地 HTTPS 服务端
HttpServer server = HttpsServer.create(new InetSocketAddress(8443), 0);
SSLContext sslContext = createServerSslContext();
server.setHttpsConfigurator(new HttpsConfigurator(sslContext) {
    public void configure(HttpsParameters params) {
        params.setSSLParameters(sslContext.getDefaultSSLParameters());
    }
});

server.createContext("/probe", exchange -> {
    SSLSession session = exchange.getSSLSession();
    // 记录 session.getProtocol(), session.getCipherSuite(), 证书等
    exchange.sendResponseHeaders(200, 0);
    exchange.close();
});
server.start();

// 2. 用代表客户端连接
HttpClient client = HttpClient.newBuilder()
    .sslContext(createClientSslContext())
    .build();
client.send(HttpRequest.newBuilder(URI.create("https://localhost:8443/probe")).build(),
    HttpResponse.BodyHandlers.discarding());

// 3. 服务端读取到的协商结果即为入站探测结果
```

**注意**：
- 代表客户端需要配置与第三方客户端对齐的算法列表
- 如果第三方客户端信息未知，可用一个“现代浏览器/JDK 默认”客户端近似

### 5. HTTPS 出站探测

#### 目标

- 我方 HTTP Client 与第三方服务端协商出的 TLS 版本、Cipher
- 服务端证书信息
- ALPN / HTTP 协议版本
- Hostname verification 行为

#### 实现方式

**方案 A：裸 TLS Socket 探测**

```java
SSLSocketFactory factory = SSLContext.getDefault().getSocketFactory();
SSLSocket socket = (SSLSocket) factory.createSocket(host, port);
socket.startHandshake();
SSLSession session = socket.getSession();
```

优点：简单。
缺点：不做 HTTP/ALPN，看不到 HTTP/2 协商。

**方案 B：JDK 11+ HttpClient + 自定义 SSLContext**

```java
SSLContext context = SSLContext.getDefault();
HttpClient client = HttpClient.newBuilder()
    .sslContext(context)
    .build();

HttpResponse<String> response = client.send(
    HttpRequest.newBuilder(URI.create("https://partner-api.example.com/health")).build(),
    HttpResponse.BodyHandlers.ofString()
);
```

拿到 TLS 协商结果需要包装 `SSLEngine` 或 `SSLContext`，较复杂。

**第一版建议**：方案 A 做 TLS 层探测，ALPN/HTTP 版本通过实际 HttpClient 请求后读取响应头推断（如 `server` 头、HTTP/2  SETTINGS）。

## 代表服务配置

代表服务需要配置成与第三方算法一致。配置来源：

| 来源 | 说明 |
|------|------|
| 基线文件 | 从 baseline 中读取期望算法，反推代表服务配置 |
| Partner Profile | 每个伙伴一份 Docker Compose / 容器启动配置 |
| 手动配置 | 特殊场景人工维护 |

### 示例：Kafka 代表服务

```yaml
# docker-compose.yml
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_LISTENERS: SSL://:9093
      KAFKA_SSL_KEYSTORE_FILENAME: kafka.keystore.jks
      KAFKA_SSL_KEYSTORE_CREDENTIALS: pass
      KAFKA_SSL_KEY_PASSWORD: pass
      KAFKA_SSL_TRUSTSTORE_FILENAME: kafka.truststore.jks
      KAFKA_SSL_TRUSTSTORE_CREDENTIALS: pass
      KAFKA_SSL_ENABLED_PROTOCOLS: TLSv1.2
      KAFKA_SSL_CIPHER_SUITES: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

### 示例：SFTP 代表服务

```dockerfile
# 基于 atmoz/sftp，自定义 sshd_config
RUN echo "KexAlgorithms curve25519-sha256" >> /etc/ssh/sshd_config \
    && echo "Ciphers aes256-ctr" >> /etc/ssh/sshd_config \
    && echo "MACs hmac-sha2-256" >> /etc/ssh/sshd_config
```

## 基线比对

### 比对逻辑

```java
public class BaselineComparator {
    public BaselineDiff compare(HandshakeResult actual, Baseline expected) {
        List<FieldDiff> diffs = new ArrayList<>();
        
        compareField(diffs, "tlsVersion", expected.getTlsVersion(), actual.getTlsVersion());
        compareField(diffs, "cipherSuite", expected.getCipherSuite(), actual.getCipherSuite());
        compareField(diffs, "saslMechanism", expected.getSaslMechanism(), actual.getSaslMechanism());
        compareCertificate(diffs, expected.getServerCertificate(), actual.getServerCertificate());
        
        return new BaselineDiff(diffs);
    }
    
    private void compareField(List<FieldDiff> diffs, String name, Object expected, Object actual) {
        if (!Objects.equals(expected, actual)) {
            diffs.add(new FieldDiff(name, expected, actual));
        }
    }
}
```

### 忽略字段

某些字段每次运行都会变化，不应参与比对：
- `collectedAt`
- `handshakeTime`
- 证书序列号（如果每次代表服务重新生成证书）

可在基线中配置 `ignoredFields`。

## JUnit 集成

```java
@Test
@Tag("integration")
@DisplayName("Partner A Kafka Transport 握手探测")
void partnerAKafkaTransportShouldMatchBaseline() {
    Baseline baseline = BaselineLoader.load("baseline/partner-a-kafka.json");
    
    try (KafkaBrokerContainer kafka = new KafkaBrokerContainer(baseline)) {
        kafka.start();
        
        KafkaTransportResult result = KafkaTransportProbe.builder()
            .bootstrapServers(kafka.getBootstrapServers())
            .securityProtocol("SASL_SSL")
            .saslMechanism("GSSAPI")
            .build()
            .probe();
        
        BaselineDiff diff = new BaselineComparator().compare(result, baseline);
        
        assertThat(diff.getDiffs())
            .withFailMessage(() -> "Baseline mismatch: " + diff)
            .isEmpty();
    }
}
```

### 标签控制

| 标签 | 运行内容 |
|------|---------|
| `inventory` | 第一层清单检查 |
| `integration` | 第二层代表服务握手探测 |
| `e2e` | 第二层真实三方握手探测 |

### CI 策略

| 场景 | 运行内容 | CI 配置 |
|------|---------|---------|
| PR CI | 清单检查 + 代表服务探测 | 代表服务探测 allow_failure |
| Nightly | 清单检查 + 全量代表服务 + 真实三方探测 | 真实三方 allow_failure |
| 本地开发 | 按需运行 | 手动触发 |

## 安全与凭证

| 凭据 | 存储方式 |
|------|---------|
| Kafka keytab | CI secret / 本地安全存储 |
| SFTP 密码/私钥 | CI secret / 本地安全存储 |
| HTTPS 客户端证书 | CI secret / 本地安全存储 |
| 服务端证书私钥 | 测试资源自签名 CA，不提交私钥 |

**基线文件**：不包含任何私钥、密码、keytab 路径等敏感信息。

## 第一阶段实现建议

第二层比第一层复杂，建议分阶段实现：

```
第一阶段：Kafka Transport 代表服务探测
  ├── Testcontainers Kafka（TLS / SASL_SSL）
  ├── KafkaTransportProbe
  └── baseline/partner-a-kafka.json

第二阶段：SFTP/SSH 代表服务探测
  ├── Testcontainers SFTP
  ├── SshProbe
  └── baseline/partner-a-sftp.json

第三阶段：Kerberos 专题
  ├── Testcontainers KDC
  ├── KerberosProbe
  └── baseline/partner-a-kerberos.json

第四阶段：HTTPS 入站/出站
  ├── Testcontainers HTTPS 服务端/客户端
  ├── HttpsProbe
  └── baseline/partner-a-https.json

第五阶段：真实三方 E2E
  └── 接入真实测试环境，定时触发
```

## 与第一层衔接

```
本地开发:
  mvn test -Dtest=CompatibilityInventoryTest
  mvn test -Dtest=KafkaTransportProbeTest -Dtags=integration

PR CI:
  job 1: mvn test                       (清单检查，失败不阻塞)
  job 2: mvn test -Dgroups=integration  (代表服务探测，allow_failure)

Nightly:
  job: mvn test -Dgroups=e2e            (真实三方探测，allow_failure)
```

## 相关文档

- 第一层实施计划：`openspec/changes/compatibility-test-guard/layer1-inventory-check.md`
- 扫描器设计：`openspec/changes/compatibility-test-guard/inventory-scanner-design.md`
- 比对规则设计：`openspec/changes/compatibility-test-guard/inventory-comparator-design.md`
- JUnit 集成设计：`openspec/changes/compatibility-test-guard/inventory-junit-integration.md`
- 架构设计：`openspec/changes/compatibility-test-guard/design.md`
