# 兼容性看护测试套件

## 背景

服务存在大量对接第三方 Kafka、SFTP 的场景。在 JDK 升级、Kafka Client 升级、Apache SSHD 升级过程中，经常因算法、协议、证书、SASL、Kerberos 等变更导致与第三方服务不兼容，且问题多在上线后被动发现。

## 目标

将兼容性验证从上线后发现左移到日常开发阶段，通过可运行、可维护的测试套件提前暴露风险。

## 范围

### 包含

- 第三方 Kafka 传输层兼容性（TLS、SASL、证书）
- 第三方 Kafka Kerberos 认证兼容性（高优先级专题）
- 第三方 SFTP/SSH 兼容性（KEX、cipher、MAC、host key、认证）
- JDK 算法支持性基线验证

### 不包含

- PR 合并门禁（不拦截合并流程）
- 独立看护平台或 Diff 引擎系统
- 生产环境监控或告警系统

## 方案概述

以工程化测试套件形式落地：

- 提供按专题组织的测试用例
- 提供可复用的探测辅助工具
- 使用基线文件保存各三方期望协商结果
- 默认使用代表服务运行，扩展支持三方真实环境
- 在本地、CI、nightly 均可运行
- 测试失败仅暴露问题，不阻塞流程

## 预期收益

| 场景 | 预期效果 |
|------|---------|
| JDK 17 升级 | 跑测试即可发现 cipher / Kerberos etype 不兼容 |
| Kafka Client 升级 | 测试显示 SASL / hostname verification 行为变化 |
| Apache SSHD 升级 | 测试显示 KEX / cipher 协商变化 |
| 证书轮换 | 测试显示 fingerprint / 有效期变化 |
| 日常配置变更 | 跑测试验证兼容性 |

## 关键设计决策

1. **不做 PR 门禁**：测试失败不拦截合并，由开发者决策处理。
2. **测试套件形态**：不建独立平台，以工程化测试落地。
3. **Kerberos 独立专题**：因依赖 KDC 和 JDK 强封装，问题模式独特。
4. **代表服务优先**：日常测试用容器 / 本地服务，稳定可控。
5. **基线随代码管理**：基线数据作为测试资源，和代码一起版本化。
6. **三层测试**：单元测试 + 集成测试 + E2E 测试。

## 相关文档

- 采集规范：`openspec/specs/compatibility-guard/spec.md`
- 架构设计：`openspec/changes/compatibility-test-guard/design.md`
- 执行任务：`openspec/changes/compatibility-test-guard/tasks.md`
