---
name: review-repository
description: 仓库级代码审查。审查整个代码仓库的架构设计、模块依赖、技术债、安全风险。需要 CodeGraph MCP 索引。触发词：review-repository、仓库审查、整库审计。
version: 2.3.0
skill_level: repository
next_skill:
  - review-class
  - review-method
---

# Repository Review — 仓库级代码审查

基于 Repository Context Engine（CodeGraph MCP）的仓库级审查。采用"模块化切片 + 多角度发现 + 三态验证"架构，对代码仓库执行架构、依赖、技术债、安全多维度分析，输出结构化审查报告。

## 推荐审查工作流

本 Skill 是三级审查工作流的 **第一级（整库入口）**：

```
review-repository（整库扫描）
    ↓ 报告中 Major/Critical 涉及的类
review-class（逐类深入）
    ↓ 报告中 Major/Critical 涉及的方法
review-method（逐方法深入）
```

三个 Skill 可独立执行，也可按需级联下钻。

## 前置条件

项目已通过 `codegraph init -i` 建立索引。

## 核心原则（反幻觉约束）

- 每个发现必须有代码证据
- 模块化切片分析，避免上下文稀释效应
- 不标记推测性问题
- 对安全/架构问题深入分析，对风格问题确认真实再标记
- 置信度有限但影响高时标注不确定性

## 执行策略：批次化审查

Repository Review 按模块分批执行，每批独立输出发现，用户确认后继续下一批。

| 项目规模 | 核心模块数 | 单批模块数 | 单批 CodeGraph 上限 |
|---------|-----------|-----------|-------------------|
| 小型 | ≤ 3 | 全部 | ≤ 15 次 |
| 中型 | 4-8 | 3 个/批 | ≤ 15 次 |
| 大型 | > 8 | 2 个/批 | ≤ 15 次 |

**批次优先级：** 安全入口模块（Auth/Gateway/Filter）> 核心业务模块（支付/交易/权限）> 其他业务模块 > 工具类/配置类。

每批完成后输出该批发现，提示用户："本批审查完成，是否继续下一批？"

## 审查流程（模块化切片 + Guided Investigation）

### Phase 0: 全景扫描（确定性步骤）

```bash
find <project-root> -maxdepth 3 -type d | head -80
```

```
codegraph_explore(query="project structure main packages modules configuration", projectPath, maxFiles=12)
```

提取：技术栈、模块划分、入口点、配置文件。形成模块清单，作为后续切片和批次划分依据。

### Phase 0 质量门控

收集完成后检查，任一不满足则暂停并提示用户：

| 检查项 | 要求 | 不满足时 |
|--------|------|---------|
| 目标代码可见性 | 能识别出 ≥ 1 个核心模块 | 提示确认 projectPath 和索引状态 |
| 返回文件数 | ≥ 3 个相关文件 | 换关键词重试一次，仍空则停止 |
| 模块覆盖率 | 核心模块 ≥ 60% 可见 | 继续执行，报告中标注 ⚠️ 上下文覆盖不足，发现可信度受限 |

### Phase 1: 模块级切片分析

对每批模块，从 5 个角度独立扫描。**角度之间互不压制。**

**Angle A — 架构合规 (Architecture)**
```
codegraph_explore(query="Controller Service Repository DAO layer", projectPath, maxFiles=15)
```
- 分层是否清晰？跨层调用？反向依赖？层间数据模型泄漏？

**Angle B — 模块依赖 (Module Dependency)**
```
codegraph_explore(query="<模块A> <模块B> dependency", projectPath, maxFiles=15)
```
- 循环依赖？依赖方向不合理？过度耦合？

**Angle C — 横切关注点 (Cross-Cutting Concerns)**
```
codegraph_explore(query="security authentication authorization interceptor filter aspect AOP exception handler", projectPath, maxFiles=12)
```
- 安全机制统一？异常处理统一？日志一致？事务管理规范？

**Angle D — 技术债 (Technical Debt)**
```
codegraph_explore(query="largest classes complex methods utility helpers common", projectPath, maxFiles=15)
```
- 超大类（>500行）、超长方法（>50行）、工具类膨胀、重复代码
- 硬编码 URL/IP/密码、配置未外部化、敏感信息暴露

```bash
grep -r "version" <project-root>/pom.xml 2>/dev/null | head -40
```
- 已知漏洞版本、废弃 API、过旧依赖

**Angle E — 安全风险 (Security)**
```
codegraph_explore(query="SQL query execute input parameter validation sanitize filter security vulnerability", projectPath, maxFiles=15)
```
- SQL 注入、XSS、敏感数据暴露、不安全反序列化、文件操作安全

**Angle E 补充 — 全局防护识别（三态验证前执行）**

```
codegraph_explore(query="security filter interceptor global authentication validator", projectPath, maxFiles=8)
```

识别以下全局防护并记录，后续三态验证时作为 REFUTED 依据：
- SQL 防护：ORM 参数化 / 全局 PreparedStatement 封装
- XSS 防护：全局 HTML 转义 Filter
- 认证防护：Spring Security / Shiro 全局配置
- 输入校验：全局 @Valid 处理器

已识别的全局防护，在三态验证时优先 REFUTED 对应的安全候选发现。

**Discovery Budget：**

| Angle | 单 Angle 最多候选发现数 |
|-------|----------------------|
| A 架构 | ≤ 6 |
| B 依赖 | ≤ 5 |
| C 横切 | ≤ 4 |
| D 技术债 | ≤ 6 |
| E 安全 | ≤ 6 |

**安全特殊规则：** 若安全 Angle CONFIRMED 发现为 0，报告中必须显式注明"安全扫描完成，未发现 CONFIRMED 问题"，不得省略安全章节。

每个 Angle 产出的候选发现，每个含：
- `module`, `file`, `line`, `angle`, `category`, `summary`, `failure_scenario`

**failure_scenario 写法规范：**
格式：`[触发条件] → [执行路径（含方法名）] → [可观测后果（异常类型/错误响应/数据状态）]`

合格示例：
> 当攻击者传入 id=1 OR 1=1 → OrderController.list() 拼接原始 SQL → 返回全部订单数据，越权访问

不合格示例（拒绝）：
> 存在 SQL 注入风险

### Phase 2: 三态验证

- **CONFIRMED** — 能指出具体代码和触发场景。引用代码行。
- **PLAUSIBLE** — 机制真实，触发条件不确定。说明确认方法。
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

从发现中抽象出 code pattern 并生成 `pattern_signature`，格式为 `前缀_具体模式`。**安全类发现必须使用 Orchestrator 定义的标准前缀**（见 review-orchestrator-lite 的 Security Pattern Signature Convention）：`sqli_`、`xxe_`、`auth_bypass_`、`rce_`、`cors_`、`deser_`、`path_traversal_`。非安全类自由命名。

同时为每个 pattern 分配 `pattern_type`（内部路由决策，不进入 review_targets）：

| pattern_type | 定义 | 示例 | 检索策略 |
|-------------|------|------|---------|
| **symbolic** | 可映射到具体 API/类/方法调用的模式 | `DocumentBuilderFactory.newInstance()`、`Runtime.exec()`、`ObjectInputStream.readObject()` | search + callers + impact |
| **structural** | 基于代码语法结构的模式 | `catch(Exception e){}`、`Thread.sleep()`、`System.out.println` | grep |
| **convention** | 基于框架约定/字符串插值的模式，图索引可能不覆盖 | `${dto.whereSql}`、`${ew.customSqlSegment}`、`@Value("${...}")` | grep + search |

**Step 2 — 分层检索（Tier 1 → 2 → 3，按 pattern_type 选择路径）**

```
Tier 1 — Symbol Retrieval（图索引驱动）
  codegraph_search("<symbol>", kind="<method|class>")  → 符号位置列表
  codegraph_callers("<symbol>")                        → 调用者列表
  codegraph_impact("<symbol>", depth=2)                → 影响传播链
  适用：symbolic pattern（主力），convention pattern（辅助）

Tier 2 — Graph Expansion（调用链驱动）
  codegraph_callees("<symbol>")                        → 被调用者列表
  适用：symbolic pattern，需要理解下游调用链时

Tier 3 — Text Retrieval（文本匹配驱动）
  grep -rn "<pattern>" <project-root> --include="*.<ext>"
  统计 total_hits，抽样 max_grep_samples 个验证
  适用：structural pattern（主力），convention pattern（主力）
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
  total_hits: 145            # grep 统计总数（不展开全部）
  sampled_hits: 10           # 抽样验证数量（≤ max_grep_samples）
  validated_hits: 9          # 抽样中真实同模式数量
  true_positive_rate: 0.90   # validated_hits / sampled_hits
```

验证标准：命中必须与原始 pattern 的代码语义一致，而非仅字符串匹配。例如 `whereSql` 的 145 个 grep 命中中，`private String whereSql` 和 `setWhereSql()` 不是漏洞点，只有 `${dto.whereSql}` 才是 true positive。

**Step 4 — 更新发现**

```yaml
expansion_metrics:
  pattern_signature: sqli_raw_where
  pattern_type: convention          # 内部路由，不进入 review_targets
  affected_locations: [...]         # 经验证的同模式位置
  affected_files: 37                # 涉及文件数
  affected_modules: 8               # 涉及模块数
  total_hits: 145                   # grep 统计总数（仅 convention/structural）
  true_positive_rate: 0.90          # 抽样验证率（仅 convention/structural）
  impact_scope: 3                   # 跨模块 = 3 / 单模块多文件 = 2 / 单文件 = 1
  systemic: true
```

**Systemic 阈值（默认值，Orchestrator 可覆盖）：**
```yaml
systemic_threshold:
  affected_modules: 3       # 跨模块数
  affected_files: 10        # 涉及文件数
  total_hits: 50            # grep 总命中数（仅 convention/structural）
  true_positive_rate: 0.8   # 抽样验证最低真实率
```
满足 **任一** 条件即标注 Systemic：
- `affected_modules >= 3`
- `affected_files >= 10`
- `total_hits >= 50 AND true_positive_rate >= 0.8`

Pattern Expansion 搜索受质量约束中定义的预算参数限制。搜索完成后，扩展结果合并回原始 finding，不作为独立 finding 输出。

**Pattern Expansion 不执行的情况：**
- PLAUSIBLE 发现（尚未确认，不扩展）
- Minor / Info 发现（投入产出比不合理）
- 模式过于特殊，不太可能在其他地方出现（如硬编码特定 IP 地址）

### Phase 3: 根因聚类 + 跨模块汇总与优先级排序

**Phase 3.1: 根因聚类（在排序前执行）**

满足以下**全部**条件的发现合并为一条：
1. 相同 `category`（如 ExceptionHandling）
2. 相同 `code pattern`（如 `catch(Exception e){}`）
3. 相同 `layer`（同属 Controller / Service / DAO 层）

合并后输出：
- severity 保持原级别不变
- `affected_locations` 列出所有位置
- 标注："同一根因，建议统一修复策略"

不满足条件 3（跨层）时：分开报告，在各条末尾注明"同类问题另见 [位置]"。

**Phase 3.2: 跨模块汇总排序**

1. Critical CONFIRMED > Critical PLAUSIBLE > Major CONFIRMED > ...
2. 同级别内按影响范围排序（影响多模块 > 影响单模块）
3. 根因聚类已处理去重，此步骤只做排序

### Phase 4: 生成审查报告

```
# Repository Review Report: <项目名称>

## 项目概况
| 项目 | 值 |
|------|-----|
| 项目名 | `<名称>` |
| 技术栈 | <框架/语言/构建工具> |
| 模块数 | <N> |
| 代码规模 | <估算 LOC> |
| 审查批次 | 第 <N> 批 / 共 <N> 批 |
| 审查时间 | <日期> |

## 1. 架构评估
### 1.1 架构概览
（整体架构风格：分层/MVC/微服务/模块化单体）
### 1.2 架构发现
（Angle A 发现，每个含严重度和建议）

## 2. 模块依赖分析
### 2.1 模块关系
### 2.2 依赖问题
（Angle B 发现）

## 3. 横切关注点
（Angle C 发现）

## 4. 技术债评估
### 4.1 代码质量热点
| 文件 | 问题 | 严重度 | 验证状态 |
|------|------|--------|---------|
### 4.2 依赖健康度
### 4.3 配置问题
（Angle D 发现）

## 5. 安全风险
### 5.1 全局防护识别
（已识别的全局防护机制列表）
### 5.2 安全发现
（Angle E 发现，或"安全扫描完成，未发现 CONFIRMED 问题"）
### 5.3 安全建议

## 6. 综合评分
| 维度 | 评分 (1-5) | 依据 |
|------|-----------|------|
| 架构合理性 | | |
| 模块化质量（内聚/耦合综合） | | |
| 代码质量 | | |
| 安全合规性 | | |
| 依赖健康度 | | |
| 可维护性 | | |

## 7. 优先修复建议
1. [Critical] <问题及建议>
2. [Major] <问题及建议>
...

## 审查统计
- 候选发现: <N> | CONFIRMED: <N> | PLAUSIBLE: <N> | REFUTED: <N>
- 聚类合并: <N> 组 → <N> 条（去重节省 <N> 条）
- Critical: <N> | Major: <N> | Minor: <N> | Info: <N>

## 上下文覆盖与置信度
| 区域 | 覆盖程度 | 对发现的影响 |
|------|---------|------------|
| 核心业务层 | 高 / 中 / 低 | <说明> |
| 外部集成层 | 高 / 中 / 低 | <说明> |
| 配置与部署层 | 高 / 中 / 低 | <说明> |

- 已分析模块: <列表>
- 未覆盖区域: <列表>

## 总结
<2-3 段综合评价>

## 下一步建议（Next Review Action）

根据本报告发现类型，建议继续审查：

| 发现特征 | 建议动作 |
|---------|---------|
| Critical/Major 集中在某个类（God Class、职责混乱） | → `review-class <ClassName>` |
| Critical/Major 集中在某个方法（SQL注入、NPE、逻辑缺陷） | → `review-method <ClassName>.<methodName>()` |
| Critical 安全问题需要方法级证据链 | → 优先 `review-method` 获取调用链级证据 |
| 多个模块出现同类模式问题 | → 对最具代表性的类执行 `review-class`，验证是否为系统性设计缺陷 |

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

- 单批 CodeGraph 调用 ≤ 15 次
- 最终发现 ≤ 20 个（聚类去重后）
- 评分必须有明确依据
- REFUTED 的发现不进入报告
- graph_depth_limit: 2（调用链追踪不超过 2 跳；超出节点标记 `truncated_beyond_review_scope` 并停止展开）
- Pattern Expansion 预算参数：
  - max_same_pattern_search ≤ 10 次（CodeGraph 工具调用上限）
  - max_grep_hits: 50（grep 统计上限，超出只记录 total_hits 不展开）
  - max_grep_samples: 10（grep 抽样验证数量）
  - max_grep_files: 20（grep 涉及文件上限，超出统计后截断）
- review_targets 必须为每个 CONFIRMED/PLAUSIBLE 发现生成对应条目

**priority_score 计算公式：**

```
priority_score = severity_weight × confidence_weight × impact_scope

severity_weight:  Critical = 4, Major = 3, Minor = 2, Info = 1
confidence_weight: CONFIRMED = 1.0, PLAUSIBLE = 0.6
impact_scope:     跨模块 = 3, 单模块多文件 = 2, 单文件 = 1
```

- priority_score 完全由 Phase 2（三态验证）、Phase 2.5（Pattern Expansion 提供 impact_scope）和 Phase 3（聚类）的确定性输出计算，不引入额外 LLM 判断
- review_targets 按 priority_score 降序排列
- orchestrator 消费时按 score 取 Top-N 进入审查队列

## 触发词

review-repository, review repository, 仓库级审查, 仓库审计, inspect repository, audit codebase, 代码审计, 架构审查, architecture review, 全库扫, 技术债分析, security audit

## LLM 使用建议

本 Skill 的审查分析由执行它的 Agent 模型直接完成，无需额外配置模型 API Key。

**模型选择建议：**
- 推荐：GPT-4o、Claude Sonnet/Opus、Qwen-Max 等具备强代码推理能力的模型
- 整库审查涉及大量上下文切换和跨模块推理，对模型的长上下文理解能力要求最高
- 参考数据：BitsAI-CR 实践中，未经工程流水线优化的 LLM 基线精确率约 10-35%，经过流水线优化后达 75%

**无需额外配置：**
- CodeGraph MCP 负责结构化上下文收集，不涉及 LLM 调用
- 审查分析、三态验证、报告生成均由 Agent 模型完成
- 不依赖 OpenCodeReview 或其他外部 Review Engine