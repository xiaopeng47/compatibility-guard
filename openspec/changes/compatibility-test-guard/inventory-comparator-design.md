# 清单比对规则实现分析

基于清单扫描器输出的当前环境支持情况，本节分析如何将 `compatibility-inventory.yaml` 中的依赖项与扫描结果进行比对。

## 核心目标

对每一项依赖，判断：
- 当前 JDK/库是否**支持**它？
- 当前 JDK/库是否**默认启用**它？
- 它是否被 `java.security` 等配置**禁用**？
- 所需的 Provider 是否已**加载**？

输出：`PASS` / `WARN` / `FAIL`。

## 四条核心规则

| 规则 | 含义 | 适用场景 | 失败等级 |
|------|------|---------|---------|
| `REQUIRED_SUPPORTED` | 依赖算法必须在支持列表中 | TLS 协议、cipher、Kerberos etype、SSH 算法 | FAIL |
| `REQUIRED_ENABLED` | 依赖算法最好在默认启用列表中 | TLS 协议、cipher | WARN |
| `NOT_DISABLED` | 依赖算法不能在禁用列表中 | TLS 协议、cipher、证书签名算法 | FAIL |
| `PROVIDER_PRESENT` | 依赖 Provider 必须已加载 | SunJSSE、SunJCE、SunJGSS、BouncyCastle 等 | FAIL |

## 规则与字段的默认映射

```yaml
required:
  jdk:
    tlsProtocols: ["TLSv1.2"]           # REQUIRED_SUPPORTED + REQUIRED_ENABLED
    tlsCipherSuites: ["..."]            # REQUIRED_SUPPORTED + NOT_DISABLED
    kerberosEnctypes: ["..."]           # REQUIRED_SUPPORTED
    providers: ["SunJSSE"]              # PROVIDER_PRESENT
```

| 字段 | 默认规则 | 扫描来源 |
|------|---------|---------|
| `tlsProtocols` | REQUIRED_SUPPORTED, REQUIRED_ENABLED | `supportedTlsProtocols`, `enabledTlsProtocols` |
| `tlsCipherSuites` | REQUIRED_SUPPORTED, NOT_DISABLED | `supportedTlsCipherSuites`, `disabledTlsAlgorithms` |
| `kerberosEnctypes` | REQUIRED_SUPPORTED | `permittedKerberosEnctypes` |
| `providers` | PROVIDER_PRESENT | `securityProviders` |
| Kafka `securityProtocols` | REQUIRED_SUPPORTED | Kafka Client 能力 |
| Kafka `saslMechanisms` | REQUIRED_SUPPORTED | JDK SASL 能力 |
| SSH `kexAlgorithms` | REQUIRED_SUPPORTED | SSH Client 能力 |
| SSH `ciphers` | REQUIRED_SUPPORTED | SSH Client 能力 |
| SSH `macs` | REQUIRED_SUPPORTED | SSH Client 能力 |
| SSH `hostKeyAlgorithms` | REQUIRED_SUPPORTED | SSH Client 能力 |

## 规则详细说明

### REQUIRED_SUPPORTED

**判定逻辑**：`requiredValue ∈ supportedValues`

```java
boolean isSupported = supportedValues.stream()
    .anyMatch(v -> v.equalsIgnoreCase(requiredValue));
```

**示例**：
- 依赖：`TLSv1.2`
- 扫描结果：`supportedTlsProtocols = ["TLSv1.2", "TLSv1.3"]`
- 结果：`PASS`

**失败场景**：
- JDK 升级后彻底移除了某个协议（如 JDK 11+ 默认仍支持 TLSv1.2，但某些定制 JDK 可能移除）
- 依赖了当前 JDK 不支持的 cipher

### REQUIRED_ENABLED

**判定逻辑**：`requiredValue ∈ enabledValues`

```java
boolean isEnabled = enabledValues.stream()
    .anyMatch(v -> v.equalsIgnoreCase(requiredValue));
```

**示例**：
- 依赖：`TLSv1.2`
- 扫描结果：`enabledTlsProtocols = ["TLSv1.3"]`（TLSv1.2 被显式关闭）
- 结果：`WARN`（仍在支持列表，但默认不启用）

**为什么用 WARN 不是 FAIL**：
- 多数算法可以通过显式配置重新启用
- 但如果用户没意识到被关闭，实际握手可能协商出不一样的结果

### NOT_DISABLED

**判定逻辑**：`requiredValue ∉ disabledValues`

这是最复杂的规则，因为 `java.security` 的禁用列表不全是精确匹配。

#### 禁用列表示例

```
SSLv3, RC4, DES, MD5withRSA, DH keySize < 1024, EC keySize < 224, 3DES_EDE_CBC
```

#### 匹配类型

| 类型 | 示例 | 匹配方式 |
|------|------|---------|
| 精确算法名 | `SSLv3`, `RC4` | 精确匹配 |
| 算法组件 | `3DES_EDE_CBC` | cipher suite 名中包含该子串 |
| 约束条件 | `RSA keySize < 1024` | 算法名包含 `RSA` 时可能受影响 |
| include 指令 | `include jdk.disabled.namedCurves` | 需递归解析 |

#### 简化实现（第一版）

```java
private boolean isAlgorithmDisabled(String required, List<String> disabledAlgorithms) {
    String normalizedRequired = required.toUpperCase();
    for (String disabled : disabledAlgorithms) {
        String normalizedDisabled = disabled.toUpperCase().trim();
        
        // 1. 精确匹配
        if (normalizedDisabled.equals(normalizedRequired)) {
            return true;
        }
        
        // 2. 子串匹配（适用于 cipher suite 组件）
        if (normalizedRequired.contains(normalizedDisabled)) {
            return true;
        }
        
        // 3. 处理约束条件，如 "RSA keySize < 1024"
        if (normalizedDisabled.contains(" ")) {
            String algorithmPart = normalizedDisabled.split(" ")[0];
            if (normalizedRequired.contains(algorithmPart)) {
                return true; // 保守判定为禁用
            }
        }
    }
    return false;
}
```

**注意**：这是启发式匹配，可能有误报。后续可以维护更精确的算法组件映射。

### PROVIDER_PRESENT

**判定逻辑**：`requiredProvider ∈ loadedProviders`

```java
boolean isPresent = loadedProviders.stream()
    .anyMatch(p -> p.equalsIgnoreCase(requiredProvider));
```

**示例**：
- 依赖：`BouncyCastle`
- 扫描结果：`securityProviders = ["SUN", "SunJSSE", "BC"]`
- 结果：`FAIL`（名称不匹配，实际 BouncyCastle provider 名为 `BC`）

**注意点**：
- Provider 名称大小写敏感，但实际比较时建议不区分大小写
- BouncyCastle 在不同场景下可能叫 `BC` 或 `BouncyCastle Provider`

## 类设计

```java
public class InventoryComparator {
    private final Inventory inventory;
    private final InventoryScanResult scanResult;
    
    public InventoryComparator(Inventory inventory, InventoryScanResult scanResult) {
        this.inventory = inventory;
        this.scanResult = scanResult;
    }
    
    public List<InventoryCheck> compare() {
        List<InventoryCheck> checks = new ArrayList<>();
        checks.addAll(compareJdk());
        checks.addAll(compareKafka());
        checks.addAll(compareSftp());
        checks.addAll(compareHttps());
        return checks;
    }
    
    private List<InventoryCheck> compareJdk() {
        List<InventoryCheck> checks = new ArrayList<>();
        JdkInventory jdk = inventory.getJdk();
        
        for (String protocol : jdk.getTlsProtocols()) {
            checks.add(checkSupported("jdk.tls.protocol", protocol, scanResult.getSupportedTlsProtocols()));
            checks.add(checkEnabled("jdk.tls.protocol.enabled", protocol, scanResult.getEnabledTlsProtocols()));
        }
        
        for (String cipher : jdk.getTlsCipherSuites()) {
            checks.add(checkSupported("jdk.tls.cipher.supported", cipher, scanResult.getSupportedTlsCipherSuites()));
            checks.add(checkNotDisabled("jdk.tls.cipher", cipher, scanResult.getDisabledTlsAlgorithms()));
        }
        
        for (String provider : jdk.getProviders()) {
            checks.add(checkProviderPresent("jdk.provider", provider, scanResult.getSecurityProviders()));
        }
        
        for (String etype : jdk.getKerberosEnctypes()) {
            checks.add(checkSupported("jdk.kerberos.etype", etype, scanResult.getPermittedKerberosEnctypes()));
        }
        
        return checks;
    }
    
    private InventoryCheck checkSupported(String subject, String required, List<String> actual) {
        boolean found = containsIgnoreCase(actual, required);
        return InventoryCheck.builder()
            .subject(subject)
            .required(required)
            .actual(found ? "supported" : "not supported")
            .status(found ? CheckStatus.PASS : CheckStatus.FAIL)
            .rule("REQUIRED_SUPPORTED")
            .message(found ? required + " is supported" : required + " is NOT supported")
            .build();
    }
    
    private InventoryCheck checkEnabled(String subject, String required, List<String> actual) {
        boolean found = containsIgnoreCase(actual, required);
        return InventoryCheck.builder()
            .subject(subject)
            .required(required)
            .actual(found ? "enabled by default" : "not enabled by default")
            .status(found ? CheckStatus.PASS : CheckStatus.WARN)
            .rule("REQUIRED_ENABLED")
            .message(found ? required + " is enabled by default" : required + " is supported but NOT enabled by default")
            .build();
    }
    
    private InventoryCheck checkNotDisabled(String subject, String required, List<String> disabled) {
        boolean isDisabled = isAlgorithmDisabled(required, disabled);
        return InventoryCheck.builder()
            .subject(subject)
            .required(required)
            .actual(isDisabled ? "disabled" : "not disabled")
            .status(isDisabled ? CheckStatus.FAIL : CheckStatus.PASS)
            .rule("NOT_DISABLED")
            .message(isDisabled ? required + " is disabled by java.security" : required + " is not disabled")
            .build();
    }
    
    private InventoryCheck checkProviderPresent(String subject, String required, List<String> providers) {
        boolean found = containsIgnoreCase(providers, required);
        return InventoryCheck.builder()
            .subject(subject)
            .required(required)
            .actual(found ? "loaded" : "not loaded")
            .status(found ? CheckStatus.PASS : CheckStatus.FAIL)
            .rule("PROVIDER_PRESENT")
            .message(found ? required + " provider is loaded" : required + " provider is NOT loaded")
            .build();
    }
    
    private boolean containsIgnoreCase(List<String> values, String target) {
        return values.stream().anyMatch(v -> v.equalsIgnoreCase(target));
    }
}
```

## 输出结构

```java
public class InventoryCheck {
    private String subject;          // "jdk.tls.protocol"
    private String required;         // "TLSv1.2"
    private String actual;           // "supported" / "not supported" / "disabled"
    private CheckStatus status;      // PASS / WARN / FAIL
    private String rule;             // "REQUIRED_SUPPORTED"
    private String message;          // 人类可读说明
}
```

## 控制台输出示例

```
[PASS] jdk.tls.protocol TLSv1.2 is supported (REQUIRED_SUPPORTED)
[WARN] jdk.tls.protocol TLSv1.2 is supported but NOT enabled by default (REQUIRED_ENABLED)
[PASS] jdk.tls.cipher TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 is supported (REQUIRED_SUPPORTED)
[PASS] jdk.tls.cipher TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 is not disabled (NOT_DISABLED)
[PASS] jdk.provider SunJSSE provider is loaded (PROVIDER_PRESENT)
[FAIL] jdk.kerberos.etype rc4-hmac is NOT supported (REQUIRED_SUPPORTED)
```

## 与 JUnit 集成

```java
@Test
public void inventoryCheckShouldPass() {
    Inventory inventory = InventoryLoader.load("compatibility-inventory.yaml");
    InventoryScanResult scanResult = new JdkInventoryScanner().scan();
    List<InventoryCheck> checks = new InventoryComparator(inventory, scanResult).compare();
    
    List<InventoryCheck> failures = checks.stream()
        .filter(c -> c.getStatus() == CheckStatus.FAIL)
        .toList();
    
    List<InventoryCheck> warnings = checks.stream()
        .filter(c -> c.getStatus() == CheckStatus.WARN)
        .toList();
    
    // WARN 只打印，不失败
    warnings.forEach(w -> System.out.println("[WARN] " + w.getMessage()));
    
    // FAIL 导致测试失败
    assertThat(failures)
        .withFailMessage(() -> formatFailures(failures))
        .isEmpty();
}
```

## 边界情况

| 情况 | 处理 |
|------|------|
| 扫描结果为空（如 Kerberos etype 拿不到） | 标记为 `UNKNOWN`，不 PASS 也不 FAIL，提示信息 |
| 依赖算法有多种命名 | 统一转大写比较，必要时维护别名映射 |
| Provider 名称别名 | BouncyCastle 常见名 `BC`，可在清单中允许写 `BouncyCastle` |
| 禁用列表包含 include 指令 | 递归解析引用的 property |
| 约束条件如 `keySize < 1024` | 第一版保守判定为可能影响，后续完善精确解析 |

## YAML 规则表示方式

### 方案 A：隐式规则（推荐第一版）

字段名决定规则，YAML 简洁：

```yaml
required:
  jdk:
    tlsProtocols: ["TLSv1.2"]
    tlsCipherSuites: ["TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]
    providers: ["SunJSSE"]
```

### 方案 B：显式规则

更灵活，但 YAML 更复杂：

```yaml
required:
  jdk:
    tlsProtocols:
      - value: "TLSv1.2"
        rule: REQUIRED_SUPPORTED
    tlsCipherSuites:
      - value: "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
        rules: [REQUIRED_SUPPORTED, NOT_DISABLED]
```

### 建议

- 第一版用**方案 A**（隐式规则）
- 后续如需要更细粒度控制，再支持方案 B 的显式规则覆盖

## 第一阶段实现建议

实现 `InventoryComparator` 的 JDK 部分：

1. `REQUIRED_SUPPORTED` for `tlsProtocols`
2. `REQUIRED_SUPPORTED` + `REQUIRED_ENABLED` for `tlsProtocols`
3. `REQUIRED_SUPPORTED` + `NOT_DISABLED` for `tlsCipherSuites`
4. `PROVIDER_PRESENT` for `providers`
5. `REQUIRED_SUPPORTED` for `kerberosEnctypes`

暂时不实现：
- Kafka / SFTP / HTTPS 的专用规则（依赖库级扫描器）
- 复杂的禁用算法条件解析（先做子串匹配）

## 相关文档

- 扫描器设计：`openspec/changes/compatibility-test-guard/inventory-scanner-design.md`
- 算法/协议清单：`openspec/changes/compatibility-test-guard/jdk-algorithm-protocol-inventory.md`
- 第一层实施计划：`openspec/changes/compatibility-test-guard/layer1-inventory-check.md`
- 架构设计：`openspec/changes/compatibility-test-guard/design.md`
