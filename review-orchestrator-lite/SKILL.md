---
name: review-orchestrator-lite
description: 审查编排器。自动串联 review-repository → review-class → review-method，基于 review_targets 优先级队列调度，合并 Pattern Family 报告。触发词：review-orchestrator、全链路审查、自动审查编排。
version: 1.3.0
skill_level: orchestrator
consumes:
  - review-repository
  - review-class
  - review-method
---

# Review Orchestrator Lite — 全链路审查编排

编排层，不是审查层。不调 CodeGraph、不做三态验证、不评判代码。

只做四件事：**调度（Dispatch）、传递线索（Seed Context）、收集结果（Collect）、合并报告（Merge）。**

## 推荐审查工作流

```
review-orchestrator-lite（编排入口）
  ├─ Phase 0: review-repository（整库扫描）
  ├─ Phase 1: 解析 review_targets → 优先级队列
  ├─ Phase 2: review-class × N（逐类深入，携带 Seed Context）
  ├─ Phase 3: review-method × N（逐方法深入，携带 Seed Context）
  ├─ Phase 4: Pattern Family Aggregation（模式聚类 + 证据链构建）
  └─ Phase 5: Final Report（合并报告）
```

三个 review Skill 可独立执行。Orchestrator 是可选的自动化编排层。

## 前置条件

- review-repository、review-class、review-method 三个 Skill 已安装
- 项目已通过 `codegraph init -i` 建立索引

## 核心原则

### 原则 1: 确定性调度

所有编排决策由确定性规则驱动。Orchestrator 不做 LLM 自由判断。

| 决策点 | 规则 | 反例（禁止） |
|--------|------|-------------|
| 执行顺序 | `priority_score` 降序 | "选择最重要的问题" |
| 是否入队 | `priority_score >= min_priority_score` | "判断这个问题值不值得审" |
| 预算截断 | 取 Top N | "选择最值得深入的 N 个" |
| 目标去重 | 同一 `target` 保留最高 score | "判断哪个更有价值" |

### 原则 2: Seed Context = 线索，不是结论

级联时**只传递调查线索（evidence + target）**，**不传递结论（severity / reason / confidence）**。

原因：如果 Repository 判断"SQL Injection → Critical"并传递给 Class，Class 会被引导去**证明** SQL Injection 存在，而不是**独立验证**。错误会沿级联链放大。

详见下方「Seed Context 协议」章节。

### 原则 3: Skill 独立性

三个 review Skill 无需知道 Orchestrator 的存在。它们单独执行和被 Orchestrator 调用时行为一致。

Orchestrator 通过调用时的 prompt 注入 Seed Context，不修改 Skill 文件，不要求 Skill 增加"接收 seed"的接口。

## Review Budget 配置

每次执行前确定以下预算参数（用户可覆盖，否则使用默认值）：

```yaml
review_budget:
  max_classes: 5           # 最多审查几个类
  max_methods_per_class: 3 # 每个 class review 产出的 method targets 最多取几个
  max_methods_total: 10    # 方法级审查总数上限（含 repository 直接产出的 + class 产出的）
  min_priority_score: 12   # 最低入队分数
  max_seed_findings: 3     # 每个下游 target 最多注入几条 seed context（防止 prompt 爆炸）
```

**min_priority_score 参考值：**

| 组合 | priority_score | 是否入队（默认 12） |
|------|---------------|-------------------|
| Critical CONFIRMED 跨模块 | 4 × 1.0 × 3 = 12 | ✅ 刚好入队 |
| Critical CONFIRMED 单模块多文件 | 4 × 1.0 × 2 = 8 | ❌ 不入队 |
| Critical PLAUSIBLE 跨模块 | 4 × 0.6 × 3 = 7.2 | ❌ 不入队 |
| Major CONFIRMED 跨模块 | 3 × 1.0 × 3 = 9 | ❌ 不入队 |

默认值 12 = 只审查 Critical CONFIRMED 且影响跨模块的问题。用户可降低以扩大覆盖面。

**security_impact_override（安全影响提权）：**

`priority_score` 公式中 `impact_scope` 衡量的是代码影响范围，但某些安全问题的代码位置在单文件、实际安全影响却是系统级的（如认证绕过、全局配置缺陷）。对这类 finding，标准公式会低估其风险。

提权规则（确定性）：

```
IF finding.severity == Critical
   AND finding.pattern_signature matches any prefix in Security Pattern Signature Convention
   AND finding.priority_score < min_priority_score:
     → force_enqueue(finding)   # 强制入队，忽略 min_priority_score
     → mark: security_override = true
```

提权后的 finding 在队列中标记 `security_override: true`，在 Final Report 的 Runtime Metrics 中统计 override 次数。

**提权不修改 priority_score 本身**（遵守 No-Rewrite Rule），仅影响入队判断。

## Security Pattern Signature Convention（安全模式签名约定）

三个 review skill 生成 `pattern_signature` 时，**安全类发现必须使用以下标准前缀**，以确保 Orchestrator 的 `security_impact_override` 能正确识别。

| 漏洞类型 | 标准前缀 | 正确示例 | ❌ 错误示例 |
|---------|---------|---------|-----------|
| SQL 注入 | `sqli_` | `sqli_raw_where`, `sqli_string_concat` | `raw_where_sql` |
| XXE | `xxe_` | `xxe_unsafe_document_builder` | `unsafe_document_builder` |
| 认证绕过 | `auth_bypass_` | `auth_bypass_wildcard_skip`, `auth_bypass_missing_filter` | `skip_urls_wildcard` |
| 授权绕过 | `authz_bypass_` | `authz_bypass_missing_role_check` | — |
| CORS 配置缺陷 | `cors_` | `cors_wildcard_credentials` | ✅ (已正确) |
| 不安全反序列化 | `deser_` | `deser_untrusted_objectinput` | `unsafe_deserialization` |
| 远程代码执行 | `rce_` | `rce_class_forname_newinstance`, `rce_script_eval` | `unsafe_reflection_newinstance` |
| 路径遍历 | `path_traversal_` | `path_traversal_unvalidated_input` | — |

**非安全类 pattern_signature** 不受此约定约束，继续使用自由命名（如 `swallowed_exception`、`field_injection`、`zero_test_coverage`）。

**前缀匹配规则：** Orchestrator 的 override 使用 `pattern_signature.startsWith(prefix)` 进行前缀匹配。例如 `sqli_raw_where` 匹配 `sqli_` 前缀，`xxe_unsafe_document_builder` 匹配 `xxe_` 前缀。

## 审查流程

### Phase 0: Repository Review

**输入：** 用户指定的 `projectPath`

**执行：**
1. 调用 `review-repository` Skill，传入 projectPath
2. 等待 Repository Review 完成，获取完整报告

**输出：** Repository Review Report（含 review_targets YAML 块）

**异常处理：**
- Repository Review 失败 → 终止编排，报告错误
- Repository Review 完成但无 review_targets → 终止编排，报告"未发现需要深入审查的问题"

### Phase 1: Extract & Build Queue

**确定性步骤，无 LLM 判断。**

**Step 1.1 — 提取 review_targets**

从 Repository Report 中提取 `review_targets` YAML 块，解析为结构化列表。

**Step 1.2 — 过滤**

```
queue = [t for t in targets if t.priority_score >= min_priority_score]
```

**Step 1.3 — 目标去重**

同一 `target` 出现多次时（如一个类被多个 finding 指向），只保留 `priority_score` 最高的条目。

```
deduped = {}
for t in sorted(queue, key=priority_score, desc):
    if t.target not in deduped:
        deduped[t.target] = t
queue = list(deduped.values())
```

**Step 1.4 — 分离队列**

```
class_queue  = [t for t in queue if t.entity_type == "class"]
method_queue = [t for t in queue if t.entity_type == "method"]
```

**Step 1.5 — 预算截断**

```
class_queue = class_queue[:max_classes]
method_queue = method_queue[:max_methods_total]  # 后续 Phase 2 会追加
```

**Step 1.6 — 初始化 visited_targets**

Orchestrator 维护一个全局已调度集合，防止循环调度和重复调度：

```
visited_targets = set()

# 初始化时将所有已确定调度的 target 加入
for t in class_queue + method_queue:
    visited_targets.add(f"{t.skill}:{t.target}")
```

**调度前检查（Phase 2 / Phase 3 通用）：**

```
if f"{target.skill}:{target.target}" in visited_targets:
    skip()  # 已调度过，跳过
```

**Forward-Only Cascade 规则：**

调度方向只允许向下，不允许回跳：

```
review-repository → review-class   ✅ 允许
review-repository → review-method  ✅ 允许
review-class      → review-method  ✅ 允许
review-class      → review-class   ❌ 禁止（横向循环）
review-method     → review-class   ❌ 禁止（反向回跳）
review-method     → review-method  ❌ 禁止（横向循环）
```

实现：如果下游 Skill 的 review_targets 中出现 `skill: review-class` 或 `skill: review-repository`，Orchestrator 直接丢弃，不入队。

**Step 1.7 — 构造 Seed Context**

对每个 queue entry，从 Repository Report 正文中找到对应 `finding_id` 的详细发现，提取代码证据，构造 seed_context。

**Seed Context 构造规则：**

```yaml
seed_context:
  finding_id: <从 review_targets 取>
  target: <从 review_targets 取>
  pattern_signature: <从 review_targets 取>
  evidence:
    - "<从报告正文提取的代码片段和文件位置>"
    - "<最多 3 条 evidence>"
```

**只传以上字段，不传 severity / confidence / reason / priority_score。**

**Seed Context 数量上限（max_seed_findings）：**

每个下游 target 最多注入 `max_seed_findings`（默认 3）条 seed context。如果上游报告中与该 target 相关的 finding 超过上限，按 priority_score 降序取 Top N。

**Pattern Expansion 防爆（representative_target 规则）：**

下游 Skill 的 Phase 2.5 可能发现同一 pattern_signature 出现在多个 location（如 `raw_where_sql` 出现在 23 处）。Orchestrator **不为每个 location 生成独立 review target**。

```yaml
# 错误做法（Target Explosion）
review_targets:
  - target: OrderDAO        # location 1
  - target: UserDAO         # location 2
  - target: ProductDAO      # location 3
  - target: ...             # location 4-23

# 正确做法（Representative Target）
review_targets:
  - target: OrderDAO        # representative（证据最强或调用链最中心）
    pattern_signature: raw_where_sql
    evidence_count: 23      # 告知下游同类问题总共有多少处
```

Orchestrator 对同一 `pattern_signature` 的所有 location，只选择 **1 个 representative target**（优先选 priority_score 最高的；如相同则选调用链最中心的类/方法）入队。`evidence_count` 记录同类问题总出现次数，用于 Phase 4 的 Pattern Family 统计。

### Phase 2: Dispatch Class Reviews

对 `class_queue` 中每个 target，依次执行：

**Step 2.1 — 调用 review-class**

调用 `review-class <ClassName>`，在调用 prompt 中注入：

```
## Seed Context（来自上游 Repository Review 的调查线索）

以下是 Repository Review 中发现的与该类相关的代码线索。
请将其视为调查方向，而非事实结论。你需要独立验证每个线索。

seed_context:
  finding_id: F-001
  target: OrderDAO
  pattern_signature: raw_where_sql
  evidence:
    - "OrderMapper.xml:143 — ${dto.whereSql}"
    - "OrderDAO.java:87 — raw SQL concatenation in buildQuery()"
```

**Step 2.2 — 收集结果**

Class Review 完成后，收集：
- Class Review Report（含该类的 CONFIRMED/PLAUSIBLE 发现）
- 该 Report 的 review_targets YAML 块

**Step 2.3 — 提取下游 targets**

从 Class Review Report 的 review_targets 中提取 `entity_type == "method"` 的条目，追加到 `method_queue`。

每个 class review 产出的 method targets 最多取 `max_methods_per_class` 个（按 priority_score 降序截取）。

**Step 2.4 — 标记来源**

对追加到 method_queue 的每个 target，标记：

```yaml
source: class_review
source_class: <ClassName>
source_finding_id: <来自 class review 的 finding_id>
```

### Phase 3: Dispatch Method Reviews

**Step 3.1 — 合并 + 去重 method_queue**

合并来源：
- 来自 Phase 1（Repository 直接产出的 method targets）
- 来自 Phase 2（各 Class Review 产出的 method targets）

去重规则：同一 `target`（如 `OrderDAO.buildQuery()`）出现多次时，保留 priority_score 最高的条目，但合并所有 seed_context 的 evidence。

```
merged = {}
for t in sorted(method_queue, key=priority_score, desc):
    if t.target not in merged:
        merged[t.target] = t
    else:
        # 合并证据（同一 target 可能从不同上游 finding 获得不同 evidence）
        merged[t.target].seed_context.evidence += t.seed_context.evidence
method_queue = list(merged.values())
```

**Step 3.2 — 预算截断**

```
method_queue = method_queue[:max_methods_total]
```

**Step 3.3 — 构造 Seed Context 并调用 review-method**

对每个 target：

1. 从上游报告（Repository 或 Class）中提取对应 finding 的代码证据
2. 构造 seed_context（同 Phase 1 Step 1.6 的规则）
3. 调用 `review-method <ClassName>.<methodName>()`，注入 seed_context

调用 prompt 格式同 Phase 2。

**Step 3.4 — 收集结果**

收集每个 Method Review Report（含 review_targets，但 method review 的 targets 不再级联——这是编排的最后一层）。

### Phase 4: Pattern Family Aggregation

**确定性步骤：按 pattern_signature 聚类，无 LLM 判断。**

**Step 4.1 — 收集全部 findings**

从三层报告中收集所有 CONFIRMED/PLAUSIBLE 发现：

```
all_findings = []
for finding in repository_report.findings:
    finding.level = "Repository"
    all_findings.append(finding)
for class_report in class_reports:
    for finding in class_report.findings:
        finding.level = "Class"
        all_findings.append(finding)
for method_report in method_reports:
    for finding in method_report.findings:
        finding.level = "Method"
        all_findings.append(finding)
```

**Step 4.2 — 按 pattern_signature 聚类**

```
families = {}
for f in all_findings:
    sig = f.pattern_signature
    if sig not in families:
        families[sig] = []
    families[sig].append(f)
```

**Step 4.3 — 标记聚类类型**

对每个 Pattern Family：

| 条件 | 标记 |
|------|------|
| findings 来自 ≥ 2 个不同 level（如 Repository + Method） | **Corroborated**（多层级证据链） |
| findings 来自同一 level 但 ≥ 2 个不同 target | **Related**（同模式，不同位置） |
| 仅 1 个 finding | **Single**（独立发现） |

**Step 4.4 — 排序 Pattern Families**

排序规则（确定性）：
1. Corroborated > Related > Single
2. 同类型内，按 Family 中最高 severity 排序（Critical > Major > Minor > Info）
3. 同 severity 内，按 finding 数量降序

**Step 4.5 — 计算 family_confidence**

对每个 Pattern Family，统计内部 findings 的 confidence 分布：

```
family_confidence:
  evidence_nodes: 3       # Family 内 finding 总数
  confirmed: 1            # CONFIRMED 数量
  plausible: 1            # PLAUSIBLE 数量
  refuted: 1              # REFUTED 数量（跨级联中某层推翻）
```

**family_confidence 聚合规则（确定性）：**

| 条件 | family_confidence 标记 |
|------|----------------------|
| confirmed == evidence_nodes（全部 CONFIRMED） | **HIGH** |
| confirmed >= 1 且 refuted == 0 | **MEDIUM** |
| confirmed >= 1 且 refuted >= 1 | **MIXED**（跨层级结论不一致） |
| confirmed == 0 且 plausible >= 1 | **LOW** |
| refuted == evidence_nodes（全部 REFUTED） | **REFUTED** |

**禁止取最高等级。** 如果一个 Family 中 Repository = CONFIRMED 但 Method = REFUTED，标记为 MIXED 而非 HIGH。这让最终报告的可信度显著提高。

**family_confidence 在 Final Report 中展示：** 每个 Pattern Family 标题行后显示 `family_confidence: MIXED` 及分布明细。

**Step 4.6 — 聚合 family_statistics**

对每个 Pattern Family，从下游 findings 的 `expansion_metrics` 中聚合统计数据：

```yaml
family_statistics:
  total_hits: 145              # Family 内所有 findings 的 total_hits 最大值
  validated_hits: 9            # 对应最高 total_hits 的 finding 的 validated_hits
  true_positive_rate: 0.90     # 对应的 true_positive_rate
  affected_files: 37           # Family 内所有 findings 的 affected_files 并集计数
  affected_modules: 8          # Family 内所有 findings 的 affected_modules 并集计数
```

**聚合规则（确定性）：**
- `total_hits`：取 Family 内 findings 的最大值（同一 pattern 在不同层级的 grep 结果可能有差异，取最大代表最广覆盖）
- `validated_hits` / `true_positive_rate`：取与最大 `total_hits` 对应的值
- `affected_files` / `affected_modules`：取 Family 内所有 findings 的文件/模块并集计数

**family_statistics 在 Final Report 中展示：** 每个 Pattern Family 的 Evidence Chain 后追加统计行：

```
**Expansion Statistics:** total_hits: 145, true_positive_rate: 0.90, affected_files: 37, affected_modules: 8
```

**family_statistics 与 systemic 判断的关系：** Orchestrator 不重新判断 systemic（这是下游 Skill 的职责），但在 Final Report 中交叉验证：如果某个 Family 在 Repository 层标为 systemic 但在 Method 层未标，检查 `family_statistics` 的 `affected_modules` 是否 >= 3 以确认上游判断的合理性。

### Phase 5: Final Report

生成 Full-Chain Review Report。**合并 Repository Report 的基础章节与 Pattern Family 分析为一份完整交付物**，不产出独立中间报告。

```markdown
# Full-Chain Code Review: <项目名称>

> 审查模式：review-orchestrator-lite 全链路编排
> 审查时间：<日期>
> 审查深度：Repository → Class → Method（三层级）

## 项目概况

<从 Repository Report 的项目概况章节直接迁移>

| 项目 | 值 |
|------|-----|
| 项目名 | <名称> |
| 技术栈 | <框架/语言/构建工具> |
| 模块数 | <N> |
| 代码规模 | <估算 LOC> |
| 测试覆盖 | <测试文件数 / 源文件数> |

## 1. 架构评估

<从 Repository Report 的架构评估章节迁移，含架构概览和综合评分表>

| 维度 | 评分 (1-5) | 依据 |
|------|-----------|------|
| 架构合理性 | | |
| 模块化质量 | | |
| 代码质量 | | |
| 安全合规性 | | |
| 依赖健康度 | | |
| 可维护性 | | |

## 2. Pattern Families

审查通过 Repository → Class → Method 三层级下钻，按 pattern_signature 聚类。
跨层级证据链（Corroborated）的发现拥有最高置信度。

### Family: <pattern_signature>
**Classification:** Corroborated / Related / Single
**family_confidence:** HIGH / MEDIUM / MIXED / LOW / REFUTED (confirmed: N, plausible: N, refuted: N)
**Instances:** N | **Evidence Levels:** Repository → Class → Method

| Level | Finding | Target | Severity | Confidence | priority_score |
|-------|---------|--------|----------|------------|----------------|
| Repository | F-001 | OrderDAO | Critical | CONFIRMED | 12 |
| Class | F-014 | OrderDAO | Critical | CONFIRMED | 12 |
| Method | F-038 | OrderDAO.buildQuery() | Critical | CONFIRMED | 12 |

**Evidence Chain:**
- Repository: <该层级的证据摘要>
- Class: <该层级的证据摘要>
- Method: <该层级的证据摘要>

**Recommended Fix:**
<基于最深层级证据给出的修复建议>

---
（更多 Pattern Families...）

## 3. 其他发现（未形成 Pattern Family）

| # | Level | Target | Pattern | Severity | Confidence | Summary |
|---|-------|--------|---------|----------|------------|---------|
| 1 | Class | UserService | field_injection | Major | CONFIRMED | ... |

## 4. 全局防护识别

<从 Repository Report 迁移，列出已识别的全局安全防护机制及其有效性状态>

## 5. 审查执行统计

| Phase | Targets | Completed | Skipped | Notes |
|-------|---------|-----------|---------|-------|
| Repository | 1 | 1 | 0 | |
| Class Reviews | N | N | 0 | |
| Method Reviews | N | N | 0 | |

**Budget Configuration:**

| Parameter | Configured | Actual |
|-----------|-----------|--------|
| max_classes | 5 | |
| max_methods_per_class | 3 | |
| max_methods_total | 10 | |
| min_priority_score | 12 | |
| max_seed_findings | 3 | |

**Pattern Family 统计：**

| Metric | Value |
|--------|-------|
| Total findings (pre-aggregation) | N |
| Pattern Families | N |
| Corroborated (multi-level) | N |
| Single (independent) | N |
| New from downstream reviews | N |

## 6. Cascade Trace

| Seed (Upstream) | → Dispatched Skill | → Downstream Finding | Relationship |
|-----------------|--------------------|----------------------|--------------|
| F-001 (Repository, raw_where_sql) | review-class OrderDAO | F-014 (Class, raw_where_sql) | Corroborated |

## 7. Runtime Metrics（Orchestrator 调优数据）

记录编排执行过程中的关键指标，用于调优 Budget/Queue/Family 聚合：

- **Queue Stability**：入队数、截断触发、队列抖动次数
- **Seed Context**：注入次数、超限次数、Confirmation Bias 疑似案例
- **Target Explosion Prevention**：representative_target 触发次数、被抑制的 targets 数
- **Loop Detection**：visited_targets 检查次数、循环拦截次数
- **Priority Score Distribution**：各分数段 finding 数量分布

## 8. 优先修复建议

<从 Repository Report 迁移，按 priority_score 和实际安全影响排序>

1. [Critical] <问题及建议>
2. [Major] <问题及建议>
...

## 9. 总结

<2-3 段综合评价。重点回答：>
<- 最高风险 Pattern Family 是什么？>
<- 哪些问题有跨层级证据链（Corroborated）？>
<- 哪些问题是独立的（Single），可能优先级更低？>
```

**报告合并规则（确定性）：**

| 章节 | 来源 | 合并方式 |
|------|------|---------|
| 项目概况 | Repository Report | 直接迁移 |
| 架构评估 + 综合评分 | Repository Report | 直接迁移 |
| Pattern Families | Phase 4 聚类结果 | 新生成 |
| 其他发现 | Phase 4 非 Family findings | 新生成 |
| 全局防护识别 | Repository Report | 直接迁移 |
| 审查执行统计 | Orchestrator 执行记录 | 新生成 |
| Cascade Trace | Orchestrator 执行记录 | 新生成 |
| Runtime Metrics | Orchestrator 执行记录 | 新生成 |
| 优先修复建议 | Repository Report + 下游新发现 | 合并排序 |
| 总结 | Orchestrator 综合 | 新生成 |

**Orchestrator 产出的是唯一最终报告，不额外输出独立的 Repository Review Report。**

## Seed Context 协议

### 传入（调查线索）

| 字段 | 来源 | 性质 |
|------|------|------|
| finding_id | review_targets | 跨级追踪标识 |
| target | review_targets | 审查目标名称 |
| pattern_signature | review_targets | 代码模式标识符（结构性信息，非结论） |
| evidence | 报告正文提取 | 代码片段 + 文件位置（事实，非判断） |

### 不传（结论，会导致 Confirmation Bias）

| 字段 | 不传原因 |
|------|---------|
| severity | 防止下游被锚定（"上游说 Critical，我不敢判 Minor"） |
| confidence | 防止下游跳过独立验证（"上游已 CONFIRMED，我不需要再验"） |
| reason | 防止下游被引导去证明上游结论（"上游说 SQL Injection，我去找 SQL Injection"） |
| priority_score | 下游需要基于自己的 severity/confidence/impact_scope 重新计算 |

### Evidence 提取规则

从上游报告正文中，为每个 finding 提取：
1. **代码片段**：包含问题的具体代码（如 `${dto.whereSql}`）
2. **文件位置**：文件名 + 行号（如 `OrderDAO.java:87`）
3. 每个 finding 最多 **3 条** evidence，避免 seed 过大

### Seed Context 注入格式

在调用下游 Skill 时，在 prompt 开头注入：

```
## Seed Context（来自上游审查的调查线索）

以下是上游审查中发现的与该目标相关的代码线索。
请将其视为**调查方向**，而非事实结论。你需要独立验证每个线索：
- 线索可能指向真实问题 → CONFIRMED
- 线索可能是误报 → REFUTED
- 线索可能指向不同问题 → 你的发现可能和线索的 pattern_signature 不同

seed_context:
  finding_id: <id>
  target: <target>
  pattern_signature: <signature>
  evidence:
    - "<代码片段 + 文件位置>"
    - "<代码片段 + 文件位置>"
```

## 级联去重逻辑（Cascade Deduplication）

每次 Class/Method Review 完成后，将其 findings 与上游 findings 比对：

| 条件 | 判定 | 处理 |
|------|------|------|
| 相同 `pattern_signature` + target 属于同一条调用链 | **Corroborated** | 保留两者，Phase 4 归入同一 Family |
| 相同 `pattern_signature` + target 不在同一调用链 | **Related** | 保留两者，Phase 4 归入同一 Family |
| 不同 `pattern_signature` | **New** | 独立 finding |

**Corroborated 不删除、不降级。** 多层级证据是审查质量的体现，在 Final Report 中以 Evidence Chain 形式展示。

## 质量约束

- Orchestrator 自身不调用 CodeGraph MCP（所有代码探索由下游 Skill 完成）
- Orchestrator 不做 severity / confidence 判断（所有判断由下游 Skill 独立完成）
- Seed Context 严格遵守"传线索不传结论"协议
- 所有排序、过滤、去重操作必须是确定性的（可复现）
- Final Report 的 Pattern Family 聚类基于 pattern_signature 精确匹配
- 不修改下游 Skill 文件
- Phase 间数据传递使用结构化格式（YAML / JSON），不使用自然语言描述
- **Priority Score 不可变（No-Rewrite Rule）**：`priority_score` 在 finding 产生时一次性计算，之后不再修改。即使下游 Review 推翻了上游判断（如 Repository 判 CONFIRMED → Class 判 REFUTED），上游的 `priority_score` 不回写。下游 finding 拥有自己独立计算的 `priority_score`。原因：回写会导致队列抖动（已排序的队列因为 score 变化而重新排列），破坏编排确定性。

## 异常处理

| 场景 | 处理 |
|------|------|
| Repository Review 失败 | 终止编排，输出错误信息 |
| Repository Review 无 review_targets | 终止编排，输出"未发现需要深入审查的问题" |
| class_queue 为空 | 跳过 Phase 2，直接进入 Phase 3（仅 method_queue） |
| method_queue 为空 | 跳过 Phase 3，直接进入 Phase 4（仅 Repository + Class findings） |
| 某个 Class Review 失败 | 记录失败，继续下一个 target |
| 某个 Method Review 失败 | 记录失败，继续下一个 target |
| Seed Context 无法提取 evidence | 使用空 evidence（`evidence: []`），在 Seed Context 中标注"上游报告未提供代码细节" |

## 触发词

review-orchestrator, review orchestrator, orchestrator-lite, 全链路审查, 自动审查编排, full-chain review, automated review, 编排审查, 自动编排

## LLM 使用建议

本 Skill 的编排逻辑由执行它的 Agent 模型直接完成。

**模型选择建议：**
- Orchestrator 的核心操作（提取 YAML、排序、过滤、聚类）是确定性的，对模型推理能力要求低于三个 review Skill
- 但 Orchestrator 需要处理大量文本（多个 review report），对模型的长上下文管理能力要求高
- 推荐：与 review Skill 使用相同或更高规格的模型

**Token 预算参考：**
- Repository Report: ~3,000-8,000 tokens
- 每个 Class Review Report: ~2,000-5,000 tokens
- 每个 Method Review Report: ~1,500-3,000 tokens
- Full-Chain Report: ~5,000-15,000 tokens
- 典型执行（1 repo + 5 class + 10 method）总上下文: ~50,000-100,000 tokens

**无需额外配置：**
- 编排操作（解析、排序、聚类）由 Agent 按确定性规则执行
- 审查分析由下游 Skill 完成
- CodeGraph MCP 由下游 Skill 调用
