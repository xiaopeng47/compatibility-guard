# 清单检查 JUnit 集成分析

基于前两步的扫描器和比对规则，本节分析如何将清单检查集成到 JUnit 测试流程中，使其能在本地和 CI 稳定运行。

## 核心目标

1. **作为测试运行**：清单检查是测试套件的一部分，不是独立脚本。
2. **快速反馈**：无外部依赖，秒级运行。
3. **失败可见**：FAIL 导致测试失败，WARN 可见但不失败。
4. **CI 友好**：输出格式便于 CI 日志阅读和报告生成。
5. **不阻塞 PR**：CI 配置 `allow_failure`，但测试本身正常断言失败。

## 测试结构选择

### 方案 A：单一测试方法（推荐第一版）

```java
@DisplayName("兼容性清单检查")
class CompatibilityInventoryTest {

    @Test
    void inventoryShouldPass() {
        Inventory inventory = InventoryLoader.load("compatibility-inventory.yaml");
        InventoryScanResult scanResult = new JdkInventoryScanner().scan();
        List<InventoryCheck> checks = new InventoryComparator(inventory, scanResult).compare();
        
        List<InventoryCheck> failures = checks.stream()
            .filter(c -> c.getStatus() == CheckStatus.FAIL)
            .toList();
        
        List<InventoryCheck> warnings = checks.stream()
            .filter(c -> c.getStatus() == CheckStatus.WARN)
            .toList();
        
        // WARN 打印但不失败
        warnings.forEach(w -> System.out.println("[WARN] " + w));
        
        // FAIL 导致断言失败
        assertThat(failures)
            .withFailMessage(() -> formatFailures(failures))
            .isEmpty();
    }
}
```

**优点**：简单、好维护、一眼看懂。
**缺点**：所有检查在一个测试用例里，粒度粗。

### 方案 B：动态测试（推荐后续优化）

```java
@TestFactory
Stream<DynamicTest> inventoryChecks() {
    Inventory inventory = InventoryLoader.load("compatibility-inventory.yaml");
    InventoryScanResult scanResult = new JdkInventoryScanner().scan();
    List<InventoryCheck> checks = new InventoryComparator(inventory, scanResult).compare();
    
    return checks.stream()
        .map(check -> DynamicTest.dynamicTest(
            check.getSubject() + " - " + check.getRequired(),
            () -> assertCheck(check)
        ));
}

private void assertCheck(InventoryCheck check) {
    switch (check.getStatus()) {
        case PASS -> assertThat(true).isTrue();
        case WARN -> {
            System.out.println("[WARN] " + check);
            assertThat(true).isTrue();
        }
        case FAIL -> fail(check.getMessage());
    }
}
```

**优点**：每个检查一个测试用例，报告清晰。
**缺点**：测试数量随清单增长，大量 PASS 用例噪音。

### 方案 C：只把 FAIL/WARN 生成动态测试

```java
@TestFactory
Stream<DynamicTest> inventoryFailuresAndWarnings() {
    // 只生成 FAIL 和 WARN 的测试
    // FAIL → 断言失败
    // WARN → 断言通过但打印警告
}
```

**优点**：报告聚焦问题。
**缺点**：看不到 PASS 的覆盖情况。

### 第一版建议

用**方案 A**：单一测试方法。实现简单，足够支撑第一阶段。

## JUnit 版本选择

| 版本 | 建议 |
|------|------|
| JUnit 5 (Jupiter) | **推荐**。现代、扩展机制好、`DynamicTest` 支持好 |
| JUnit 4 | 如果项目已经用 JUnit 4，也可以用，但扩展性稍差 |

第一版按 JUnit 5 设计，需要依赖：

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <scope>test</scope>
</dependency>
```

## 清单文件加载

```java
public class InventoryLoader {
    public static Inventory load(String resourcePath) {
        InputStream is = InventoryLoader.class.getClassLoader()
            .getResourceAsStream(resourcePath);
        if (is == null) {
            throw new IllegalStateException("Inventory file not found: " + resourcePath);
        }
        Yaml yaml = new Yaml(new Constructor(Inventory.class));
        return yaml.load(is);
    }
    
    public static Inventory loadFromPath(String path) {
        // 支持从文件系统加载，便于 CI 指定不同环境的清单
    }
}
```

### 路径配置

| 场景 | 路径 |
|------|------|
| 默认 | `src/test/resources/compatibility-inventory.yaml` |
| 系统属性覆盖 | `-Dcompatibility.inventory=path/to/custom-inventory.yaml` |
| CI 多环境 | 不同 job 指定不同清单文件 |

## 断言与错误消息

### FAIL 消息格式

```
Compatibility inventory check failed (2 failures):

[FAIL] jdk.tls.cipher TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 is disabled by java.security
       Rule: NOT_DISABLED
       Suggestion: Check 'jdk.tls.disabledAlgorithms' in java.security

[FAIL] jdk.provider BouncyCastle provider is NOT loaded
       Rule: PROVIDER_PRESENT
       Suggestion: Add BouncyCastle provider or update inventory
```

### 实现

```java
private String formatFailures(List<InventoryCheck> failures) {
    StringBuilder sb = new StringBuilder();
    sb.append("Compatibility inventory check failed (").append(failures.size()).append(" failures):\n\n");
    for (InventoryCheck failure : failures) {
        sb.append("[FAIL] ").append(failure.getMessage()).append("\n");
        sb.append("       Rule: ").append(failure.getRule()).append("\n");
        sb.append("       Suggestion: ").append(getSuggestion(failure)).append("\n\n");
    }
    return sb.toString();
}
```

### WARN 处理

WARN 不应导致测试失败，但要让开发者看见：

```java
if (!warnings.isEmpty()) {
    System.out.println("Compatibility inventory warnings (" + warnings.size() + "):");
    warnings.forEach(w -> System.out.println("[WARN] " + w.getMessage()));
}
```

在 CI 中，这些警告会出现在日志里。

## 与 CI 集成

### 测试本身

清单检查的 JUnit 测试**正常断言失败**。也就是说：
- 如果有 FAIL，测试用例失败
- CI job 退出码非零

### CI 配置 allow_failure

清单检查失败不阻塞 PR，这是 CI 层面的配置：

**GitLab CI：**
```yaml
compatibility-inventory-check:
  stage: test
  script:
    - mvn test -Dtest=CompatibilityInventoryTest
  allow_failure: true
```

**GitHub Actions：**
```yaml
- name: Compatibility Inventory Check
  run: mvn test -Dtest=CompatibilityInventoryTest
  continue-on-error: true
```

### 为什么测试本身要失败？

- 测试框架能生成标准 JUnit XML 报告
- CI 能识别哪些检查失败
- 不隐藏问题

## 运行阶段划分

```
mvn test
├── 单元测试（包含 CompatibilityInventoryTest） ← 清单检查在这里
└── 其他单元测试

mvn integration-test / failsafe
├── Kafka 握手探测
├── SFTP 握手探测
├── HTTPS 握手探测
└── 其他集成测试
```

清单检查没有外部依赖，应该和单元测试一起跑。
握手探测需要 Testcontainers/真实环境，放在集成测试阶段。

## 本地开发体验

### 运行全部清单检查

```bash
mvn test -Dtest=CompatibilityInventoryTest
```

### 只看某类检查

如果后续按专题拆分，可以：

```bash
mvn test -Dtest=CompatibilityInventoryTest#jdkChecks
mvn test -Dtest=CompatibilityInventoryTest#kafkaChecks
```

第一版不需要拆分，一个测试方法就够了。

## 多 JDK 版本矩阵

清单检查特别适合跑多 JDK 矩阵：

```yaml
strategy:
  matrix:
    java-version: [8, 11, 17, 21]
```

每个 JDK 跑同一份清单，结果可能不同：
- JDK 8：某些 cipher 默认可用
- JDK 17：某些旧 cipher 被禁用

这能直观展示升级影响。

## 报告输出

### 控制台输出

```
Compatibility inventory warnings (1):
[WARN] jdk.tls.protocol TLSv1.2 is supported but NOT enabled by default

Compatibility inventory check failed (2 failures):
[FAIL] jdk.tls.cipher TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 is disabled by java.security
[FAIL] jdk.provider BouncyCastle provider is NOT loaded
```

### JUnit XML 报告

标准 `TEST-CompatibilityInventoryTest.xml` 会被生成，CI 可以解析。

### 后续可扩展：JSON 报告

```java
InventoryReport report = InventoryReport.from(checks);
report.writeTo(Path.of("target/compatibility-inventory-report.json"));
```

## 类结构建议

```
src/test/java/.../compat/
├── inventory/
│   ├── CompatibilityInventoryTest.java      # JUnit 入口
│   ├── InventoryLoader.java                  # YAML 加载
│   ├── InventoryScanner.java                 # 扫描器接口
│   ├── JdkInventoryScanner.java              # JDK 扫描器实现
│   ├── InventoryComparator.java              # 比对器
│   ├── InventoryCheck.java                   # 单次检查结果
│   ├── InventoryScanResult.java              # 扫描结果
│   ├── Inventory.java                        # 清单领域对象
│   └── CheckStatus.java                      # PASS/WARN/FAIL
└── ...
```

## 完整测试类示例

```java
package com.example.compat.inventory;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DisplayName("兼容性清单检查")
class CompatibilityInventoryTest {

    @Test
    @DisplayName("所有依赖的算法/协议都应被当前 JDK 支持")
    void inventoryShouldPass() {
        Inventory inventory = InventoryLoader.load("compatibility-inventory.yaml");
        InventoryScanResult scanResult = new JdkInventoryScanner().scan();
        List<InventoryCheck> checks = new InventoryComparator(inventory, scanResult).compare();

        List<InventoryCheck> failures = checks.stream()
            .filter(c -> c.getStatus() == CheckStatus.FAIL)
            .toList();

        List<InventoryCheck> warnings = checks.stream()
            .filter(c -> c.getStatus() == CheckStatus.WARN)
            .toList();

        warnings.forEach(w -> System.out.println("[WARN] " + w.getMessage()));

        assertThat(failures)
            .withFailMessage(() -> formatFailures(failures))
            .isEmpty();
    }

    private String formatFailures(List<InventoryCheck> failures) {
        StringBuilder sb = new StringBuilder();
        sb.append("Compatibility inventory check failed (")
          .append(failures.size())
          .append(" failures):\n\n");
        for (InventoryCheck failure : failures) {
            sb.append("[FAIL] ").append(failure.getMessage()).append("\n");
            sb.append("       Rule: ").append(failure.getRule()).append("\n\n");
        }
        return sb.toString();
    }
}
```

## 边界情况

| 情况 | 处理 |
|------|------|
| 清单文件不存在 | 测试初始化失败，给出明确错误 |
| 扫描器遇到 JDK 17 强封装限制 | 记录 WARN，不影响其他检查 |
| 所有检查都 PASS | 测试通过，无输出或极简输出 |
| 只有 WARN 无 FAIL | 测试通过，日志打印 WARN |
| 有 FAIL 也有 WARN | 测试失败，同时打印 FAIL 和 WARN |

## 第一阶段范围

- 一个 JUnit 5 测试类 `CompatibilityInventoryTest`
- 加载默认清单文件
- 运行 `JdkInventoryScanner` + `InventoryComparator`
- FAIL 断言失败，WARN 打印日志
- 不接入 CI（先本地跑通），但文档说明如何配置 `allow_failure`

## 相关文档

- 比对规则设计：`openspec/changes/compatibility-test-guard/inventory-comparator-design.md`
- 扫描器设计：`openspec/changes/compatibility-test-guard/inventory-scanner-design.md`
- 第一层实施计划：`openspec/changes/compatibility-test-guard/layer1-inventory-check.md`
- 算法/协议清单：`openspec/changes/compatibility-test-guard/jdk-algorithm-protocol-inventory.md`
