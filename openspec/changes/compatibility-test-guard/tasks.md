# 兼容性看护测试套件 - 执行任务

## 第一阶段：基础设施 + 清单检查 + Kafka/HTTPS Transport（2 周）

**目标**：搭建测试框架，实现清单扫描器，跑通 Kafka Transport 和 HTTPS 入站/出站专题。

| 任务 | 描述 | 验收标准 | 负责人 | 状态 |
|------|------|---------|--------|------|
| 1.1 建立测试模块结构 | 创建 `src/test/java/.../compat/` 目录，按 inventory / kafka / kerberos / sftp / https / jdk 组织 | 目录结构清晰 | 待定 | pending |
| 1.2 设计清单文件格式 | 定义 `compatibility-inventory.yaml` 结构，覆盖 JDK / Kafka / SFTP / HTTPS | 格式通过评审 | 待定 | pending |
| 1.3 实现清单扫描器 | 扫描 JDK TLS / cipher / provider / Kerberos etype 支持性 | 能正确输出扫描结果 | 待定 | pending |
| 1.4 实现清单比对断言 | 将扫描结果与 `compatibility-inventory.yaml` 比对 | PASS/FAIL/WARN 分类正确 | 待定 | pending |
| 1.5 实现基线加载器 | 读取 `src/test/resources/baseline/*.json`，解析为领域对象 | 能正确加载示例基线 | 待定 | pending |
| 1.6 实现 Kafka Transport Probe Helper | 封装 Kafka 握手探测，输出 TLS / SASL / cipher / 证书信息 | 输出字段覆盖 spec 中 Kafka Transport P0 项 | 待定 | pending |
| 1.7 编写首个 Kafka Transport 测试用例 | 针对一个三方伙伴验证 TLS + SASL 协商结果 | 测试可通过 | 待定 | pending |
| 1.8 建立首个三方 Kafka 基线文件 | 创建 `baseline/partner-a-kafka.json` | 字段完整，可被加载器读取 | 待定 | pending |
| 1.9 实现 HTTPS Inbound Probe Helper | 启动本地 HTTPS 服务端，用代表客户端连接，读取服务端协商结果 | 输出字段覆盖 spec 中 HTTPS 入站 P0 项 | 待定 | pending |
| 1.10 实现 HTTPS Outbound Probe Helper | 使用 HTTP Client 连接代表服务端，读取客户端侧协商结果 | 输出字段覆盖 spec 中 HTTPS 出站 P0 项 | 待定 | pending |
| 1.11 编写首个 HTTPS 入站测试用例 | 验证我方 HTTPS 服务端与代表客户端的 TLS 协商结果 | 测试可通过 | 待定 | pending |
| 1.12 编写首个 HTTPS 出站测试用例 | 验证我方 HTTPS 客户端与代表服务端的 TLS 协商结果 | 测试可通过 | 待定 | pending |
| 1.13 建立首个三方 HTTPS 基线文件 | 创建 `baseline/partner-a-https.json` | 字段完整，可被加载器读取 | 待定 | pending |
| 1.14 建立首个三方算法清单 | 创建 `inventory/partner-a-inventory.yaml` | 字段完整，可被扫描器读取 | 待定 | pending |
| 1.15 集成 Kafka 代表服务 | 使用 Testcontainers 启动算法配置可定制的 Kafka broker | CI 和本地均可启动 | 待定 | pending |
| 1.16 集成 HTTPS 代表服务 | 使用 Testcontainers 启动算法配置可定制的 HTTPS 服务端 | CI 和本地均可启动 | 待定 | pending |
| 1.17 接入 CI 作为普通 test job | 在现有 CI 中增加兼容性测试 job，配置 allow_failure | CI 能运行并生成报告 | 待定 | pending |

**第一阶段产出**：
- 可运行的清单检查
- 可运行的 Kafka Transport 兼容性测试
- 可运行的 HTTPS 入站/出站兼容性测试
- 基线加载工具
- CI 集成配置

---

## 第二阶段：Kerberos 专题（2 周）

**目标**：完成 Kafka Kerberos 独立专题，包含清单检查和握手探测。

| 任务 | 描述 | 验收标准 | 负责人 | 状态 |
|------|------|---------|--------|------|
| 2.1 设计 Kerberos 清单项 | 在 `compatibility-inventory.yaml` 中定义 etype / JVM exports 需求 | 字段完整 | 待定 | pending |
| 2.2 扩展清单扫描器支持 Kerberos | 扫描 permitted enctypes、检测是否需要 add-exports | 输出字段覆盖 spec 中 Kerberos 清单项 | 待定 | pending |
| 2.3 实现 Kerberos Probe Helper | 封装 Kerberos 登录、etype 采集、ticket 生命周期采集 | 输出字段覆盖 spec 中 Kerberos P0 项 | 待定 | pending |
| 2.4 搭建 Kerberos 测试 KDC | 使用 Testcontainers 或现有测试 KDC | 测试可连接并获取 TGT / Service Ticket | 待定 | pending |
| 2.5 编写 Kerberos 认证测试 | 验证使用 keytab 登录成功 | 测试可通过 | 待定 | pending |
| 2.6 编写 etype 协商测试 | 验证协商出的 etype 符合基线 | 测试可通过 | 待定 | pending |
| 2.7 编写 ticket 续期测试 | 验证 ticket renewable 能力 | 测试可通过 | 待定 | pending |
| 2.8 编写 JDK 17+ 强封装测试 | 验证当前代码不需要额外 JVM exports | 在 JDK 17+ 上测试通过 | 待定 | pending |
| 2.9 建立 Kerberos 基线文件 | 创建 `baseline/partner-a-kerberos.json` | 字段完整 | 待定 | pending |
| 2.10 更新适配建议库 | 整理 Kerberos 常见失败模式与修复方法 | 文档化 | 待定 | pending |

**第二阶段产出**：
- Kerberos 专题清单检查
- Kerberos 专题测试套件
- Kerberos 基线数据
- JVM exports 检测能力

---

## 第三阶段：SFTP/SSH 专题（1 周）

**目标**：完成 SFTP/SSH 兼容性测试，包含清单检查和握手探测。

| 任务 | 描述 | 验收标准 | 负责人 | 状态 |
|------|------|---------|--------|------|
| 3.1 设计 SFTP 清单项 | 在 `compatibility-inventory.yaml` 中定义 KEX / cipher / MAC / host key 需求 | 字段完整 | 待定 | pending |
| 3.2 扩展清单扫描器支持 SFTP | 扫描 SSHD Client 支持的 KEX / cipher / MAC / host key | 输出字段覆盖 spec 中 SFTP 清单项 | 待定 | pending |
| 3.3 实现 SFTP Probe Helper | 封装 SSH handshake 探测，输出 KEX / cipher / MAC / host key / 认证方式 | 输出字段覆盖 spec 中 SFTP P0 项 | 待定 | pending |
| 3.4 集成 SFTP 代表服务 | 使用 Testcontainers 启动算法配置可定制的 SFTP 服务 | CI 和本地均可启动 | 待定 | pending |
| 3.5 编写 KEX 协商测试 | 验证密钥交换算法符合基线 | 测试可通过 | 待定 | pending |
| 3.6 编写 cipher / MAC 协商测试 | 验证加密和完整性算法符合基线 | 测试可通过 | 待定 | pending |
| 3.7 编写 host key 测试 | 验证 host key 算法与 fingerprint | 测试可通过 | 待定 | pending |
| 3.8 编写认证方式测试 | 验证 publickey / password 等方式可用 | 测试可通过 | 待定 | pending |
| 3.9 建立 SFTP 基线文件 | 创建 `baseline/partner-a-sftp.json` 等 | 字段完整 | 待定 | pending |

**第三阶段产出**：
- SFTP/SSH 专题清单检查
- SFTP/SSH 专题测试套件
- SFTP 基线数据

---

## 第四阶段：JDK 算法基线（1 周）

**目标**：补齐 JDK 层面的单元测试和清单项。

| 任务 | 描述 | 验收标准 | 负责人 | 状态 |
|------|------|---------|--------|------|
| 4.1 实现 JDK Security Probe Helper | 读取 JDK 版本、disabledAlgorithms、providers、Kerberos etypes | 输出字段覆盖 spec 中 JDK P0/P1 项 | 待定 | pending |
| 4.2 完善 JDK 清单项 | 在 `compatibility-inventory.yaml` 中定义 provider / disabled algorithms 需求 | 字段完整 | 待定 | pending |
| 4.3 编写 TLS 算法支持性测试 | 验证业务需要的 TLS 协议和 cipher 未被禁用 | 测试可通过 | 待定 | pending |
| 4.4 编写 Kerberos etype 支持性测试 | 验证业务需要的 etype 在 JDK 中可用 | 测试可通过 | 待定 | pending |
| 4.5 编写 provider 检测测试 | 验证必要的 provider（如 BouncyCastle）已加载 | 测试可通过 | 待定 | pending |
| 4.6 编写 java.security 配置检测测试 | 验证关键配置项符合预期 | 测试可通过 | 待定 | pending |

**第四阶段产出**：
- JDK 算法基线单元测试
- JDK 专题清单检查
- JDK 安全配置检测能力

---

## 第五阶段：完善与扩展（1-2 周）

**目标**：提升覆盖面和易用性。

| 任务 | 描述 | 验收标准 | 负责人 | 状态 |
|------|------|---------|--------|------|
| 5.1 支持三方真实环境标签化 | 使用 JUnit Tag 区分清单检查、代表服务测试和三方环境测试 | 可通过参数单独触发 | 待定 | pending |
| 5.2 建立 nightly 全量矩阵回归 | 配置 CI 定时任务，跑多 JDK / 多组件版本矩阵 | nightly 任务稳定运行 | 待定 | pending |
| 5.3 完善适配建议库 | 整理所有专题常见失败模式与修复方法 | 文档化并关联到测试 | 待定 | pending |
| 5.4 生成兼容性测试报告 | 汇总清单结果、测试结果、diff、适配建议 | 报告可读 | 待定 | pending |
| 5.5 编写使用文档 | 说明如何运行测试、如何更新清单/基线、如何排查失败 | 文档完整 | 待定 | pending |
| 5.6 关键三方全覆盖 | 为所有重要三方伙伴建立清单和基线并编写测试（含 HTTPS 入站/出站） | 覆盖率达到预期 | 待定 | pending |

**第五阶段产出**：
- nightly 回归能力
- 完整兼容性测试报告
- 使用文档

---

## 时间线

```
第 1-2 周      第 3-4 周        第 5 周        第 6 周        第 7-8 周
   │              │              │             │              │
   ▼              ▼              ▼             ▼              ▼
┌──────┐      ┌──────┐      ┌──────┐     ┌──────┐      ┌──────┐
│基础+ │      │Kerberos│     │ SFTP │     │ JDK  │      │完善  │
│Kafka │      │专题   │      │专题  │     │基线  │      │扩展  │
│HTTPS │      └──────┘      └──────┘     └──────┘      └──────┘
└──────┘
```

---

## 依赖与风险

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| Kerberos KDC 环境难以搭建 | 第二阶段延期 | 优先调研 Testcontainers MIT KDC 或申请测试 KDC |
| 三方测试环境不稳定 | E2E 测试不可靠 | 以代表服务测试和清单检查为主，E2E 仅 nightly 运行 |
| HTTPS 代表服务端证书管理复杂 | 测试维护成本高 | 使用自签名 CA，测试代码自动生成/加载 |
| keytab 等敏感凭证管理复杂 | 安全风险 | 使用 CI secret manager，测试代码不硬编码 |
| 基线/清单维护成本高 | 数据过时 | 明确更新责任人，定期刷新 |
| 测试运行速度慢 | 开发者不愿本地跑 | 保证清单检查和单元测试足够快 |

---

## 相关文档

- 采集规范：`openspec/specs/compatibility-guard/spec.md`
- 方案提案：`openspec/changes/compatibility-test-guard/proposal.md`
- 架构设计：`openspec/changes/compatibility-test-guard/design.md`
