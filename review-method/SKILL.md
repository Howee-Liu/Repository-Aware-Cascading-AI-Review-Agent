---
name: review-method
description: 方法级代码审查。审查指定方法的调用链、安全风险、异常处理、逻辑缺陷。需要 CodeGraph MCP 索引。触发词：review-method、方法审查、审查 login()。
version: 2.3.0
skill_level: method
parent_skill:
  - review-class
  - review-repository
---

# Method Review — 方法级代码审查

基于 Repository Context Engine（CodeGraph MCP）的方法级审查。采用"多角度独立发现 + 三态验证"架构，对指定方法执行调用链分析、多维度审查，输出结构化审查报告。

## 推荐审查工作流

本 Skill 是三级审查工作流的 **第三级（方法级深入）**：

```
review-repository（整库扫描）
    ↓ 报告中 Major/Critical 涉及的类
review-class（逐类深入）
    ↓ 报告中 Major/Critical 涉及的方法
▶ review-method（逐方法深入）  ← 当前
```

三个 Skill 可独立执行，也可按需级联下钻。

## 前置条件

- **完整模式**：项目已通过 `codegraph init -i` 建立索引（支持调用链追踪）
- **轻量模式**：用户直接粘贴方法代码（跳过 Phase 0 的 CodeGraph 步骤，Angle D 调用链分析标注"受限"）

## 核心原则（反幻觉约束）

- 每个发现必须引用具体代码行，不做臆测
- 对明确的安全/逻辑缺陷：深入分析，不跳过窄触发场景
- 对低严重度问题：确认真实存在再标记，必须用具体场景解释 WHY
- 不标记推测性的潜在问题，除非能指出具体的受影响代码路径
- 不标记风格偏好或有意的设计选择
- 当置信度有限但潜在影响高（数据丢失、安全漏洞）时，报告并标注不确定性；否则宁可不报也不猜

## 审查流程

### Phase 0: 上下文收集（确定性步骤）

使用 CodeGraph 收集结构化上下文，此阶段不涉及 LLM 判断：

1. **定位目标方法**: `codegraph_explore(query="<methodName>", projectPath)`
2. **调用链追踪**: `codegraph_explore(query="<methodName> callers callees call chain", projectPath)`
3. **跨文件依赖**: `codegraph_explore(query="<依赖类名/接口名>", projectPath, maxFiles=15)`

确认收集到：方法签名、方法体源码、上游调用方、下游被调用方、参数类型定义、相关配置。
如返回多个匹配方法，让用户确认目标。

### Phase 0 质量门控

收集完成后检查，任一不满足则暂停并提示用户：

| 检查项 | 要求 | 不满足时 |
|--------|------|---------|
| 目标代码可见性 | 能看到完整方法体定义 | 提示确认 projectPath 和索引状态 |
| 返回文件数 | ≥ 1 个相关文件 | 换关键词重试一次，仍空则停止 |
| 核心依赖覆盖率 | 直接依赖 ≥ 60% 可见 | 继续执行，报告中整体标注 ⚠️ 上下文覆盖不足，发现可信度受限 |

### Phase 1: 多角度独立发现（6 个 Review Angle）

对收集到的上下文，从 6 个独立角度分别扫描。**角度之间互不压制**——如果两个角度对同一行有不同发现，都保留。

**Angle A — 正确性 (Correctness)**
逐行审查方法体。对每一行问：什么输入、状态、时序或平台会使这行出错？
检查：条件反转/错误、off-by-one、null/undefined 解引用、缺失 await/async、falsy-zero 检查、错误变量拷贝粘贴、catch 中吞异常、未转义正则元字符。

**Angle B — 安全风险 (Security)**
- 输入校验：参数是否经过校验和清洗？SQL/XSS/命令注入？
- 认证授权：缺少权限检查？绕过认证？
- 敏感数据：日志/响应/异常中泄露密码、Token、PII？
- 加密安全：密码明文？弱算法？硬编码密钥？

**Angle C — 健壮性 (Robustness)**
- 边界条件：null/空集合/空字符串/超时/并发 是否处理？
- 异常处理：空 catch / 吞异常 / 异常范围过大 / 异常信息丢失？
- 资源管理：连接/流/锁 是否正确释放？try-with-resources？
- 降级策略：外部依赖不可用时是否有降级？

**Angle D — 调用链风险 (Call Chain Risk)**
- 上游：调用方是否正确传递参数？是否存在参数篡改风险？
- 下游：被调用方法是否可靠？异常是否正确传播？
- 链路完整性：调用链中是否存在单点故障？
- 数据流：参数从哪来？返回值到哪去？中间是否有篡改可能？

**Angle E — 性能 (Performance)**
- 是否有 N+1 查询？循环内数据库调用？
- 是否有不必要的对象创建或大数据拷贝？
- 事务范围是否过大？锁粒度是否合理？
- 是否有可用缓存替代重复计算的场景？

**Angle F — 代码质量 (Code Quality)**
- 方法复杂度：圈复杂度是否过高？方法是否过长（>50行）？
- 命名规范：变量/方法命名是否清晰表达意图？
- 重复代码：是否存在可提取的公共逻辑？
- 注释：复杂逻辑是否有注释？注释是否过时？

每个 Angle 最多产出 **6 个候选发现**，每个含：
- `file`: 文件路径
- `line`: 行号
- `angle`: 发现角度（A/B/C/D/E/F）
- `category`: 类别（Correctness/Security/Robustness/CallChainRisk/Performance/CodeQuality）
- `summary`: 一句话描述
- `failure_scenario`: 见下方写法规范

**failure_scenario 写法规范：**
格式：`[触发条件] → [执行路径（含方法名）] → [可观测后果（异常类型/错误响应/数据状态）]`

合格示例：
> 当 username 包含单引号 → UserDao.findByName() 拼接原始 SQL → 返回任意用户数据，认证绕过

不合格示例（拒绝）：
> 当输入不合法时可能导致安全问题

### Phase 2: 三态验证

对 Phase 1 的每个候选发现，逐一验证并标记状态：

- **CONFIRMED** — 能指出触发该问题的具体输入/状态，以及具体的错误输出或崩溃。引用代码行。
- **PLAUSIBLE** — 问题机制是真实的，但触发条件不确定（依赖时序、环境、配置）。说明什么情况下可以确认。
- **REFUTED** — 事实性错误（代码不是那样写的）或在其他地方已有防护。引用证明代码行。

**保留策略：**

| 状态 | 是否进入报告 | 报告中的处理 |
|------|------------|------------|
| CONFIRMED | ✅ 是 | 正常展示 |
| PLAUSIBLE | ✅ 是 | 标注 ⚠️ Needs Verification，注明确认方法 |
| REFUTED | ❌ 否 | 不进报告 |

排序规则：同严重度内，CONFIRMED 排在 PLAUSIBLE 前面。
Deep 模式（用户显式要求）与 Standard 模式保留策略相同，区别仅在于发现上限放宽至 12 个。

### Phase 2.5: Pattern Expansion（模式扩展搜索）

对 Phase 2 产出的每个 **CONFIRMED Critical/Major** 发现，执行模式扩展搜索。

**Step 1 — 提取模式 + 分类**

从发现中抽象出 code pattern 并生成 `pattern_signature`，格式为 `前缀_具体模式`。**安全类发现必须使用标准前缀**（见 review-orchestrator-lite 的 Security Pattern Signature Convention）：`sqli_`、`xxe_`、`auth_bypass_`、`rce_`、`cors_`、`deser_`、`path_traversal_`。非安全类自由命名。

同时为每个 pattern 分配 `pattern_type`（内部路由决策，不进入 review_targets）：

| pattern_type | 定义 | 示例 | 检索策略 |
|-------------|------|------|---------|
| **symbolic** | 可映射到具体 API/类/方法调用的模式 | `SM2Util.decrypt()` 异常未捕获、`Runtime.exec()` | search + callers + impact |
| **structural** | 基于代码语法结构的模式 | `Thread.sleep()` 在循环中、资源未关闭 | grep |
| **convention** | 基于框架约定/字符串插值的模式 | `${dto.whereSql}`、`@Transactional` 缺失 | grep + search |

**Step 2 — 分层检索（Tier 1 → 2 → 3，按 pattern_type 选择路径）**

```
Tier 1 — Symbol Retrieval（图索引驱动）
  codegraph_search("<symbol>")   → 符号位置列表
  codegraph_callers("<symbol>")  → 调用者列表
  适用：symbolic（主力），convention（辅助）

Tier 2 — Graph Expansion（调用链驱动）
  codegraph_callees("<symbol>")  → 被调用者列表
  codegraph_impact("<symbol>")   → 影响传播链
  适用：symbolic，需要理解下游调用链时

Tier 3 — Text Retrieval（文本匹配驱动）
  grep -rn "<pattern>" <project-root> --include="*.<ext>"
  统计 total_hits，抽样 max_grep_samples 个验证
  适用：structural（主力），convention（主力）
```

**pattern_type → 工具路由（确定性）：**

| pattern_type | Tier 1 | Tier 2 | Tier 3 |
|-------------|--------|--------|--------|
| symbolic | ✅ search + callers | ✅ callees/impact（可选） | ❌ |
| structural | ❌ | ❌ | ✅ grep |
| convention | ✅ search（辅助） | ❌ | ✅ grep（主力） |

**Step 3 — 验证（仅 Tier 3 grep 结果需要）**

对 grep 返回的命中，抽样 `max_grep_samples` 个进行 true positive 验证：

```
expansion_metrics:
  total_hits: 20
  sampled_hits: 5            # ≤ max_grep_samples（method 级默认 5）
  validated_hits: 5
  true_positive_rate: 1.0
```

**Step 4 — 更新发现**

```yaml
expansion_metrics:
  pattern_signature: unhandled_decrypt_exception
  pattern_type: symbolic
  affected_locations: [...]
  affected_files: 2
  affected_modules: 1
  impact_scope: 1
  systemic: false
```

**Systemic 阈值（默认值，Orchestrator 可覆盖）：**
```yaml
systemic_threshold:
  affected_modules: 3
  affected_files: 10
  total_hits: 50
  true_positive_rate: 0.8
```
满足 **任一** 条件即标注 Systemic：
- `affected_modules >= 3`
- `affected_files >= 10`
- `total_hits >= 50 AND true_positive_rate >= 0.8`

Pattern Expansion 搜索受质量约束中定义的预算参数限制。搜索完成后，扩展结果合并回原始 finding，不作为独立 finding 输出。

**Pattern Expansion 不执行的情况：**
- PLAUSIBLE 发现（尚未确认，不扩展）
- Minor / Info 发现（投入产出比不合理）
- 模式过于特殊，不太可能在其他地方出现（如特定方法的特定参数边界条件）

### Phase 3: 生成审查报告

```
# Method Review Report: <ClassName>.<methodName>()

## 基本信息
| 项目 | 值 |
|------|-----|
| 方法 | `<完整签名>` |
| 文件 | `<文件路径>` |
| 所属类 | `<类名>` |
| 审查时间 | <日期> |
| 审查模式 | Standard / Deep |

## 调用链概览
上游 → 目标方法 → 下游（文字描述完整调用路径）

## 发现问题

### [Critical/Major/Minor/Info] — <issue_header>
- **位置**: `file:line`
- **角度**: <A/B/C/D/E/F> — <category>
- **验证状态**: CONFIRMED / ⚠️ PLAUSIBLE (Needs Verification: <确认方法>)
- **描述**: <问题是什么，为什么是问题>
- **失败场景**: <[触发条件] → [执行路径] → [可观测后果]>
- **证据**: <引用的具体代码片段>
- **建议**: <修复方案>

（按严重度排序，同严重度按 CONFIRMED 优先）

## 方法风险等级

**🔴 High** / **🟡 Medium** / **🟢 Low** / **⚪ Negligible**

判断规则：
- 任一 Critical CONFIRMED → 🔴 High
- Major CONFIRMED 或 Critical PLAUSIBLE → 🟡 Medium
- 仅 Minor / Info → 🟢 Low
- 无发现 → ⚪ Negligible

## 审查统计
- 候选发现: <N> 个
- CONFIRMED: <N> 个 | PLAUSIBLE: <N> 个 | REFUTED: <N> 个
- Critical: <N> | Major: <N> | Minor: <N> | Info: <N>

## 上下文覆盖
- 已分析跨文件依赖: <N> 个
- 调用链深度: 上游 <N> 层 / 下游 <N> 层
- 未覆盖依赖: <列表>
- 覆盖置信度: 高 / 中（⚠️ 依赖覆盖不足，发现可信度受限）/ 受限（轻量模式，无调用链）

## 综合评价
<一段话总结方法质量，含整体风险判断和优先修复建议>

## 上下文扩展建议（Context Escalation）

若发现问题的根因超出方法级范畴，建议向上追溯：

| 发现特征 | 建议动作 |
|---------|---------|
| 问题源于类的职责划分不当或依赖设计缺陷 | → `review-class <ClassName>` |
| 问题跨多个类/模块，疑似系统性架构问题 | → `review-repository` |
| 同类 NPE/异常模式在多处出现 | → `review-class` 确认是否为类级设计问题 |
```

## Review Targets（机器可消费）

```yaml
review_targets:
  - finding_id: F-001
    entity_type: method
    target: <ClassName.methodName()>
    skill: review-method
    severity: <Critical/Major/Minor/Info>
    confidence: <CONFIRMED/PLAUSIBLE>
    pattern_signature: <category_specificpattern>
    reason: <一句话描述发现>
    priority_score: <N>
  - finding_id: F-002
    entity_type: class
    target: <ClassName>
    skill: review-class
    severity: <Critical/Major/Minor/Info>
    confidence: <CONFIRMED/PLAUSIBLE>
    pattern_signature: <category_specificpattern>
    reason: <一句话描述发现>
    priority_score: <N>
```

按 priority_score 降序排列。每个 CONFIRMED/PLAUSIBLE 发现对应一个条目。finding_id 在级联审查中保持不变，用于跨 Skill 追踪同一发现。

## 严重度判定（Decision Tree）

**Step 1 → Critical（满足任意一项）：**
- 该问题是否出现在无需登录的代码路径上（public endpoint / 无鉴权 Filter）？
- 是否涉及将外部输入直接拼接到 SQL / 命令 / 文件路径？
- 是否在数据写入或删除的主路径上，且无事务保护？

**Step 2 → Major（不满足 Step 1，满足任意一项）：**
- 是否会导致方法在正常业务参数下抛出未捕获异常？
- 是否在循环内或主业务流程中存在显著性能问题（N+1、大对象拷贝）？
- 是否存在明确的架构层级违规（Controller 直调 DAO）？

**Step 3 → Minor：** 不影响功能，但违反一致性或降低可读性。

**Step 4 → Info：** 可选改进，不影响质量。

## 质量约束

- 单次 Method Review 的 CodeGraph 调用 ≤ 3 次
- Standard 模式最终发现 ≤ 8 个，Deep 模式 ≤ 12 个
- 所有发现必须有代码证据，0 个臆测
- REFUTED 的发现不进入最终报告
- graph_depth_limit: 6（调用链追踪不超过 6 跳；超出节点标记 `truncated_beyond_review_scope` 并停止展开）
- Pattern Expansion 预算参数：
  - max_same_pattern_search ≤ 5 次（CodeGraph 工具调用上限）
  - max_grep_hits: 50（grep 统计上限，超出只记录 total_hits 不展开）
  - max_grep_samples: 5（grep 抽样验证数量，method 级默认）
  - max_grep_files: 20（grep 涉及文件上限，超出统计后截断）
- review_targets 必须为每个 CONFIRMED/PLAUSIBLE 发现生成对应条目

**priority_score 计算公式：**
```
priority_score = severity_weight × confidence_weight × impact_scope

severity_weight:  Critical = 4, Major = 3, Minor = 2, Info = 1
confidence_weight: CONFIRMED = 1.0, PLAUSIBLE = 0.6
impact_scope:     跨模块 = 3, 单模块多文件 = 2, 单文件 = 1
```

- priority_score 完全由 Phase 2（三态验证）和 Phase 2.5（Pattern Expansion 提供 impact_scope）的确定性输出计算，不引入额外 LLM 判断
- review_targets 按 priority_score 降序排列
- orchestrator 消费时按 score 取 Top-N 进入审查队列

## 触发词

review-method, review method, 方法审查, 审查方法, analyze method, method review, 方法级审查, 调用链分析

## LLM 使用建议

本 Skill 的审查分析由执行它的 Agent 模型直接完成，无需额外配置模型 API Key。

**模型选择建议：**
- 推荐：GPT-4o、Claude Sonnet/Opus、Qwen-Max 等具备强代码推理能力的模型
- 模型能力直接影响审查质量：弱模型可能产生更多 PLAUSIBLE（需人工确认）的发现，强模型的 CONFIRMED 率更高
- 参考数据：BitsAI-CR 实践中，未经工程流水线优化的 LLM 基线精确率约 10-35%，经过流水线优化后达 75%

**无需额外配置：**
- CodeGraph MCP 负责结构化上下文收集，不涉及 LLM 调用
- 审查分析、三态验证、报告生成均由 Agent 模型完成
- 不依赖 OpenCodeReview 或其他外部 Review Engine