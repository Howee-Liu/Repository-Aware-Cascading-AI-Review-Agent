---
name: review-class
description: 类级代码审查。审查指定类的职责合理性、依赖健康度、设计模式、架构违规。需要 CodeGraph MCP 索引。触发词：review-class、类审查、审查 PaymentService。
version: 2.3.0
skill_level: class
parent_skill:
  - review-repository
next_skill:
  - review-method
---

# Class Review — 类级代码审查

基于 Repository Context Engine（CodeGraph MCP）的类级审查。采用"多角度独立发现 + 三态验证"架构，对指定类执行职责、依赖、设计、架构多维度分析，输出结构化审查报告。

## 推荐审查工作流（三级级联）

本 Skill 是三级审查工作流的 **第二级（类级深入）**：

```
review-repository（整库扫描）
    ↓ 报告中 [Major/Critical] 涉及的类
▶ review-class（逐类深入）  ← 当前
    ↓ 报告中 [Major/Critical] 涉及的方法
review-method（逐方法深入）
```

**上游衔接：** 如果已经执行过 review-repository，可将其报告中标记为 Major/Critical 的类（如 God Class、模块边界违规类）作为本 Skill 的审查目标，无需重新发现。

**下游引导：** 本 Skill 报告末尾的「优先修复建议」中，每条涉及具体方法的建议后附加提示：`→ 建议执行 review-method <ClassName>.<methodName>() 深入分析`

## 前置条件

项目已通过 `codegraph init -i` 建立索引。

## 核心原则（反幻觉约束）

- 每个发现必须引用具体代码行
- 对架构违规和安全问题：深入分析，不跳过窄触发场景
- 对设计建议：确认真实存在再标记，用具体场景解释 WHY
- 不标记推测性问题，除非能指出具体的受影响代码路径
- 不标记风格偏好或有意的设计选择
- 当置信度有限但潜在影响高时，报告并标注不确定性

## 审查流程

### Phase 0: 上下文收集（确定性步骤）

1. **定位目标类**: `codegraph_explore(query="<ClassName>", projectPath)`
2. **依赖关系**: `codegraph_explore(query="<ClassName> dependencies imports implements extends", projectPath, maxFiles=15)`
3. **职责边界**: `codegraph_explore(query="<依赖类名> <接口名>", projectPath, maxFiles=15)`

确认收集到：类完整定义、字段列表、方法列表、继承关系、接口实现、注入依赖、框架注解。

### Phase 0 质量门控

收集完成后检查，任一不满足则暂停并提示用户：

| 检查项 | 要求 | 不满足时 |
|--------|------|---------|
| 目标代码可见性 | 能看到完整类定义 | 提示确认 projectPath 和索引状态 |
| 返回文件数 | ≥ 1 个相关文件 | 换关键词重试一次，仍空则停止 |
| 核心依赖覆盖率 | 直接依赖 ≥ 60% 可见 | 继续执行，报告中整体标注 ⚠️ 上下文覆盖不足，发现可信度受限 |

### Phase 1: 多角度独立发现（6 个 Review Angle）

**角度之间互不压制**——同一行/同一个类的不同维度问题都要保留。

**Angle A — 职责合理性 (SRP / Single Responsibility)**
- 类的职责是否单一？统计方法数量和方法行数
- 是否存在 "God Class" 反模式（>500 行、>15 个方法、>5 个注入依赖）？
- 方法之间是否存在职责分散（一个方法做太多不相关的事）？
- 是否存在本应属于其他类的逻辑？

**Angle B — 依赖健康度 (Dependency Health)**
- 依赖数量是否过多？（>7 个注入依赖需警示）
- 是否存在循环依赖？
- 是否依赖了实现而非接口？（违反 DIP）
- 依赖注入方式是否合理？（字段注入 vs 构造器注入）
- 是否存在未使用的依赖？

**Angle C — 设计模式与抽象 (Design & Abstraction)**
- 是否使用了合适的设计模式？是否存在过度设计或模式误用？
- 接口设计是否符合接口隔离原则（ISP）？
- 抽象层次是否一致？（混合了高层抽象和低层细节）
- 是否容易扩展而不修改现有代码？（OCP）
- 子类化是否安全？（LSP）

**Angle D — 架构合规 (Architecture Compliance)**
- 是否存在层级违规？（Controller 直接调用 DAO/Repository）
- 是否存在模块边界违规？（跨模块直接依赖实现类）
- 事务边界是否合理？（@Transactional 位置、传播行为）
- 是否存在层间数据模型泄漏？（Entity 暴露到 Controller 层）
- 是否存在贫血模型或充血模型失衡？

**Angle E — 安全与健壮性 (Security & Robustness)**
- 注入依赖是否有适当的校验和防护？
- 公开方法是否有输入校验？
- 异常处理是否完整？是否有吞异常/空 catch？
- 是否存在敏感数据暴露风险？
- 并发访问是否安全？（共享可变状态）

**Angle F — 类级代码质量 (Class-Level Code Quality)**

> 范围约束：只审查类层面可直接观察的信号，方法体内部复杂度属于 review-method 范畴。

- **可见性边界**：是否有应为 private/protected 的方法被设为 public？public 方法是否暴露了内部状态？
- **方法命名一致性**：同类中方法命名风格是否统一？（如混用 get/fetch/load/query）
- **方法数量热点**：方法数是否超过 20 个？是否存在方法簇暗示类应该拆分？
- **常量管理**：类级别是否存在硬编码常量应提取为命名常量或配置项？
- **类内部组织**：字段、构造器、public 方法、private 方法排列是否有规律？是否影响可读性？

每个 Angle 最多产出 **6 个候选发现**，每个含：
- `file`, `line`, `angle`, `category`, `summary`, `failure_scenario`

**failure_scenario 写法规范：**
格式：`[触发条件] → [执行路径（含方法名）] → [可观测后果（异常类型/错误响应/数据状态）]`

合格示例：
> 当 userId 为 null → UserService.getUser() 未做 null 检查直接调用 user.getId() → NullPointerException，请求 500

不合格示例（拒绝）：
> 可能出现空指针

### Phase 2: 三态验证

对 Phase 1 的每个候选发现，逐一验证并标记状态：

- **CONFIRMED** — 能指出触发问题的具体代码和场景。引用代码行。
- **PLAUSIBLE** — 问题机制真实，触发条件不确定。说明确认方法。
- **REFUTED** — 事实性错误或其他地方已有防护。引用证明。

**保留策略：**

| 状态 | 是否进入报告 | 报告中的处理 |
|------|------------|------------|
| CONFIRMED | ✅ 是 | 正常展示 |
| PLAUSIBLE | ✅ 是 | 标注 ⚠️ Needs Verification，注明确认方法 |
| REFUTED | ❌ 否 | 不进报告 |

排序规则：同严重度内，CONFIRMED 排在 PLAUSIBLE 前面。

### Phase 2.5: Pattern Expansion（模式扩展搜索）

对 Phase 2 产出的每个 **CONFIRMED Critical/Major** 发现，执行模式扩展搜索。

**Step 1 — 提取模式 + 分类**

从发现中抽象出 code pattern 并生成 `pattern_signature`，格式为 `前缀_具体模式`。**安全类发现必须使用标准前缀**（见 review-orchestrator-lite 的 Security Pattern Signature Convention）：`sqli_`、`xxe_`、`auth_bypass_`、`rce_`、`cors_`、`deser_`、`path_traversal_`。非安全类自由命名。

同时为每个 pattern 分配 `pattern_type`（内部路由决策，不进入 review_targets）：

| pattern_type | 定义 | 示例 | 检索策略 |
|-------------|------|------|---------|
| **symbolic** | 可映射到具体 API/类/方法调用的模式 | `DocumentBuilderFactory.newInstance()`、字段注入 `@Autowired` | search + callers + impact |
| **structural** | 基于代码语法结构的模式 | `catch(Exception e){}`、`if-else` 嵌套过深 | grep |
| **convention** | 基于框架约定/字符串插值的模式 | `${dto.whereSql}`、`@Value("${...}")` | grep + search |

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
  total_hits: 45
  sampled_hits: 5            # ≤ max_grep_samples（class 级默认 5）
  validated_hits: 4
  true_positive_rate: 0.80
```

**Step 4 — 更新发现**

```yaml
expansion_metrics:
  pattern_signature: xxe_unsafe_document_builder
  pattern_type: symbolic
  affected_locations: [...]
  affected_files: 3
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
- 模式过于特殊，不太可能在其他地方出现（如特定类的特定设计模式选择）

### Phase 3: 生成审查报告

```
# Class Review Report: <ClassName>

## 基本信息
| 项目 | 值 |
|------|-----|
| 类 | `<完整类名>` |
| 文件 | `<文件路径>` |
| 方法数 | <N> |
| 依赖数 | <N> 个注入依赖 |
| 继承 | <父类 / 接口列表> |
| 审查时间 | <日期> |

## 类结构概览
（字段、核心方法、辅助方法、内部类的结构描述）

## 依赖关系图
（文字描述所有注入依赖及其用途）

## 发现问题

### [Critical/Major/Minor/Info] — <issue_header>
- **位置**: `file:line`
- **角度**: <A/B/C/D/E/F> — <category>
- **验证状态**: CONFIRMED / ⚠️ PLAUSIBLE (Needs Verification: <确认方法>)
- **描述**: <问题是什么，违反了什么原则>
- **失败场景**: <[触发条件] → [执行路径] → [可观测后果]>
- **证据**: <具体代码>
- **建议**: <重构方案>

## 设计评分
| 维度 | 评分 (1-5) | 依据 |
|------|-----------|------|
| 职责单一性 | | <基于 Angle A 发现> |
| 依赖合理性 | | <基于 Angle B 发现> |
| 设计质量 | | <基于 Angle C 发现> |
| 架构合规性 | | <基于 Angle D 发现> |
| 安全健壮性 | | <基于 Angle E 发现> |
| 类级代码质量 | | <基于 Angle F 发现> |

## 审查统计
- 候选发现: <N> | CONFIRMED: <N> | PLAUSIBLE: <N> | REFUTED: <N>
- Critical: <N> | Major: <N> | Minor: <N> | Info: <N>

## 上下文覆盖
- 已分析依赖类: <N> | 依赖图深度: <N> 层
- 未覆盖依赖: <列表>
- 覆盖置信度: 高 / 中（⚠️ 依赖覆盖不足，发现可信度受限）

## 综合评价
<一段话总结类设计质量>

## 下一步建议（Next Review Action）

若需要方法级证据，建议继续审查：

| 发现特征 | 建议动作 |
|---------|---------|
| Critical/Major 方法（NPE、注入、逻辑缺陷） | → `review-method <ClassName>.<methodName>()` |
| PLAUSIBLE 且缺少证据的问题 | → `review-method` 获取调用链级确认 |
| 职责划分、依赖设计、架构边界问题 | → `review-repository` 确认是否为系统性模式 |

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

- CodeGraph 调用 ≤ 3 次
- 最终发现 ≤ 10 个
- 设计评分必须有明确依据
- REFUTED 的发现不进入报告
- graph_depth_limit: 4（调用链追踪不超过 4 跳；超出节点标记 `truncated_beyond_review_scope` 并停止展开）
- Pattern Expansion 预算参数：
  - max_same_pattern_search ≤ 5 次（CodeGraph 工具调用上限）
  - max_grep_hits: 50（grep 统计上限，超出只记录 total_hits 不展开）
  - max_grep_samples: 5（grep 抽样验证数量，class 级默认）
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

review-class, review class, 类审查, 审查类, analyze class, class review, 类级审查, 职责分析, 依赖分析

## LLM 使用建议

本 Skill 的审查分析由执行它的 Agent 模型直接完成，无需额外配置模型 API Key。

**模型选择建议：**
- 推荐：GPT-4o、Claude Sonnet/Opus、Qwen-Max 等具备强代码推理能力的模型
- 类级审查对模型的架构理解能力要求较高，弱模型在 SOLID 原则和设计模式评估上可能不准确
- 参考数据：BitsAI-CR 实践中，未经工程流水线优化的 LLM 基线精确率约 10-35%，经过流水线优化后达 75%

**无需额外配置：**
- CodeGraph MCP 负责结构化上下文收集，不涉及 LLM 调用
- 审查分析、三态验证、报告生成均由 Agent 模型完成
- 不依赖 OpenCodeReview 或其他外部 Review Engine