# 第一层：静态算法/协议清单检查 - 实施计划

## 目标

回答一个问题：**升级后，我们依赖的算法/协议还在不在、有没有被禁用？**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  依赖清单        │ ──▶ │  清单扫描器      │ ──▶ │  比对结果        │
│  inventory.yaml │     │  扫描当前 JDK    │     │  PASS/FAIL/WARN │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## 第一步：定义清单文件格式

确定 `compatibility-inventory.yaml` 长什么样。

```yaml
inventoryVersion: "2026-06-01"
subject: "compatibility-inventory"

required:
  jdk:
    tlsProtocols: ["TLSv1.2"]
    tlsCipherSuites: ["TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]
    kerberosEnctypes: ["aes256-cts-hmac-sha1-96"]
    providers: ["SunJSSE", "SunJCE", "SunJGSS"]
    
  kafka:
    securityProtocols: ["SASL_SSL", "SSL"]
    saslMechanisms: ["GSSAPI", "SCRAM-SHA-512"]
    
  sftp:
    kexAlgorithms: ["curve25519-sha256"]
    ciphers: ["aes256-ctr"]
    macs: ["hmac-sha2-256"]
    hostKeyAlgorithms: ["rsa-sha2-256", "ssh-ed25519"]
    
  https:
    inbound:
      tlsProtocols: ["TLSv1.2"]
      tlsCipherSuites: ["TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]
    outbound:
      tlsProtocols: ["TLSv1.2"]
      tlsCipherSuites: ["TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]
```

### 需要决策

- 文件放在 `src/test/resources/compatibility-inventory.yaml` 还是 `openspec/...`？
- 每个服务一份，还是整个项目一份？
- 清单版本怎么演进？

## 第二步：实现清单扫描器

扫描器负责读取当前运行时环境，回答“当前支持什么”。

| 扫描项 | 扫描方式 | 说明 |
|--------|---------|------|
| TLS 协议支持性 | `SSLContext.getDefault().getSupportedSSLParameters().getProtocols()` | JDK 支持哪些 TLS 版本 |
| TLS 默认启用协议 | `SSLParameters.getProtocols()` | 默认会协商哪些 |
| TLS 禁用算法 | `Security.getProperty("jdk.tls.disabledAlgorithms")` | 被 `java.security` 禁用的 |
| TLS 支持的 cipher | `SSLParameters.getCipherSuites()` | 客户端/服务端支持列表 |
| Provider 列表 | `Security.getProviders()` | 已加载的安全 provider |
| Kerberos etype | `Security.getProperty("permitted_enctypes")` | 允许的 etype |
| Kerberos 默认 etype | `krb5.conf` / `sun.security.krb5.Config` | 默认偏好 |

### 关键设计

```java
public interface InventoryScanner {
    InventoryScanResult scan();
}

public class JdkInventoryScanner {
    public InventoryScanResult scan() {
        return InventoryScanResult.builder()
            .supportedTlsProtocols(scanSupportedTlsProtocols())
            .enabledTlsProtocols(scanEnabledTlsProtocols())
            .disabledTlsAlgorithms(scanDisabledTlsAlgorithms())
            .supportedTlsCipherSuites(scanSupportedTlsCipherSuites())
            .securityProviders(scanSecurityProviders())
            .permittedKerberosEnctypes(scanPermittedKerberosEnctypes())
            .build();
    }
}
```

### 难点

- SSHD / Kafka Client 的默认支持算法列表，可能没法直接通过标准 API 拿到，需要反射或读取配置。
- “支持”和“默认启用”是两个概念，要分开处理。

## 第三步：实现清单比对规则

定义“依赖算法”和“当前环境”之间的关系。

| 规则 | 含义 | 示例 |
|------|------|------|
| `REQUIRED_SUPPORTED` | 必须在支持列表中 | `TLSv1.2` 在 `supportedTlsProtocols` 里 |
| `REQUIRED_ENABLED` | 最好在默认启用列表中 | `TLSv1.2` 在 `enabledTlsProtocols` 里 |
| `NOT_DISABLED` | 不能在禁用列表中 | cipher 不在 `disabledTlsAlgorithms` 里 |
| `PROVIDER_PRESENT` | provider 必须已加载 | `SunJSSE` 在 provider 列表里 |

### 输出结构

```java
public class InventoryCheck {
    String subject;          // "jdk.tls.protocol"
    String required;         // "TLSv1.2"
    String actual;           // "supported" / "disabled" / "missing"
    CheckStatus status;      // PASS / FAIL / WARN
    String message;          // 人类可读说明
}
```

## 第四步：集成到 JUnit

清单检查最终要作为测试跑起来。

```java
@Test
public void inventoryCheckShouldPass() {
    Inventory inventory = InventoryLoader.load("compatibility-inventory.yaml");
    InventoryScanResult scan = new InventoryScanner().scan();
    List<InventoryCheck> checks = new InventoryComparator(inventory, scan).compare();
    
    List<InventoryCheck> failures = checks.stream()
        .filter(c -> c.getStatus() == CheckStatus.FAIL)
        .toList();
    
    assertThat(failures)
        .withFailMessage(() -> formatFailures(failures))
        .isEmpty();
}
```

### CI 策略

- 作为普通 test job 运行
- `allow_failure`：失败不阻塞 PR
- 输出清单检查报告

## 第五步：生成初始清单

清单不能凭空写，需要从已有信息推导。

| 来源 | 推导方式 |
|------|---------|
| 现有 baseline 文件 | 从 `negotiated` 字段里用的算法反推 |
| 业务配置 | 读取 `application.yml` / `kafka.properties` / `ssl` 配置 |
| 三方对接文档 | 人工整理伙伴要求的算法 |
| 代码扫描 | 搜索 `ssl.enabled.protocols`、`sasl.mechanism` 等关键字 |

### 建议

- 第一版先人工整理最小集合
- 后续逐步从基线和配置中反推补全

## 第六步：输出报告

清单检查要产出可读报告，而不仅仅是 PASS/FAIL。

```
[PASS] JDK TLSv1.2 仍受支持
[PASS] JDK TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 未被禁用
[FAIL] Kerberos etype rc4-hmac 在当前 JDK 默认禁用
[WARN] SSH kex diffie-hellman-group14-sha256 存在但不在默认偏好前列
```

### 报告格式选择

- 控制台文本
- JSON 文件
- JUnit XML（通过测试框架自然生成）
- HTML 报告（后续扩展）

## 第七步：与第二层的关系

清单检查和握手探测的分工：

| 问题 | 清单检查 | 握手探测 |
|------|---------|---------|
| 算法被禁用了吗？ | ✅ | ✅（间接） |
| 算法还在支持列表吗？ | ✅ | ❌ |
| 默认启用偏好变了吗？ | ⚠️ | ✅ |
| 实际能协商成功吗？ | ❌ | ✅ |
| 三方侧配置变了吗？ | ❌ | ✅ |

### 运行顺序

1. 先跑清单检查（秒级，无外部依赖）
2. 清单通过后，再跑握手探测（分钟级，需代表服务/真实环境）

## 实施顺序

```
Step 1: 定义清单 YAML 格式
        │
Step 2: 实现 InventoryLoader（读 YAML）
        │
Step 3: 实现 JDK 清单扫描器（TLS / Provider / Kerberos etype）
        │
Step 4: 实现 InventoryComparator（比对规则）
        │
Step 5: 写一个 JUnit 测试跑起来
        │
Step 6: 接入 CI
        │
Step 7: 补充 Kafka / SFTP / HTTPS 的扫描项
        │
Step 8: 从现有代码/配置生成初始清单
```

## 待确认问题

1. **技术栈**：Maven 还是 Gradle？JDK 版本？（目前项目目录里没有构建文件）
2. **清单文件放哪**：`src/test/resources/compatibility-inventory.yaml` 可以吗？
3. **第一刀切多深**：先做 JDK 层面的 TLS/provider/etype，还是 JDK + Kafka + HTTPS 一起？
4. **扫描 SSHD/Kafka Client 默认算法**：是否需要？这两个通过标准 API 不太好拿。

## 相关文档

- 方案提案：`openspec/changes/compatibility-test-guard/proposal.md`
- 架构设计：`openspec/changes/compatibility-test-guard/design.md`
- 执行任务：`openspec/changes/compatibility-test-guard/tasks.md`
- 能力级规范：`openspec/specs/compatibility-guard/spec.md`
