---
name: plan-review
description: |
  Review an implementation plan, spec, or technical design BEFORE any code is written, against a fixed
  discipline: one destination, explicit in/out-of-scope, a complete acceptance matrix, immutable domain
  contracts, and a clear boundary on what still needs the user. Use this skill when the user says
  "评审计划" / "审一下方案" / "这个设计能不能开工" / "plan review" / "看看这个开发计划有没有漏洞",
  or hands over a plan/spec/design doc and asks whether it is ready to implement.
  Do NOT use it to write the plan itself (use to-spec), to interview the user about requirements
  (use grill-me), to review code that already exists (use implementation-review), or to assess
  overall module structure (use architecture-health-review).
---

# 计划评审

评审一份还没开工的计划，判断它能不能按原样施工——不是把它改写成另一份计划。评审对象是「目的地是否锁定、边界是否明确、验收是否可证」，不是文笔、格式、命名或技术选型偏好。

## Inputs to collect

先读自己能拿到的东西：仓库里的计划/Spec 文档、issue 描述、契约文件、迁移、CI 配置、当前工作区。读完再判断缺什么。

把缺失信息分成四类，只对后两类发问：

- `[DERIVED]`：仓库或 issue 已给出，直接采用；
- `[GENERATED]`：你能自己造的（测试账号、合成数据、临时库、浏览器、失败注入夹具）；
- `[REQUIRED]`：只有用户能给（外部供应商、模型名、endpoint、密钥、SSO 账号、外部 URL）；
- `[APPROVAL]`：涉及费用、生产写入、发布、合并的授权。

`[REQUIRED]`/`[APPROVAL]` 合并成**一次**提问并附推荐默认值，收到答复后继续，不要逐项回问。某个输入缺失时只阻塞依赖它的那几行，其余照常评审。

## Procedure

### 1. 锁定目的地

确认计划只对应一个主要交付和一个规格，并写清：用户可见结果、范围内、范围外、前置依赖、真实消费者（领域服务 / application port / HTTP / 前端 / 导出入口）。

「把整个 Spec 都做了」不是目的地。如果它还依赖尚未决定的业务规则，先要求一条决策记录，而不是允许在代码里猜。

- 范围外没写 → `Blocker`：范围外是评审时无法判断「这是漏做还是不做」的那一半。
- 真实消费者没写 → `Blocker`：没有消费者的改动无法验收。

### 2. 验收矩阵是否成立

每个 User Story、失败场景、恢复场景至少一行，列定义与状态口径见 `references/acceptance-matrix.md`。

矩阵缺行、缺列或状态全是 TODO 时**不进入下一步**——矩阵不是交付物，是判断「做完没有」的唯一依据。指出具体缺哪一行、缺哪个不变量，而不是笼统说「验收不足」。

### 3. 不可变契约是否已写下

写代码前必须已决定（而不是「实现时再说」）：

- 哪个模块拥有身份、版本、有效配置和哈希的真相；
- 哪个事务同时保护读取、校验、写入和审计；
- 哪些 reader/writer port 必需，组合根缺少能力时如何 fail-closed；
- 失败时哪些对象必须保持零写入，重试如何幂等；
- 审计记录如何被生产 seam 读到，而不是只在 store 测试里可见。

这几条含糊 → `Blocker`。它们是「变化会怎么放大」的地基，落地后改动成本最高。

### 4. 并行面与所有权

若计划包含多个可并行工作流，逐个确认：每个面有明确交付物、文件所有权路径集互不重叠、零阻塞理由成立。路径集重叠 → `Blocker`（两个面会互相覆盖）。

依赖边要分档：「行为未稳定」是真阻塞，「验收未满足」只要求契约冻结即可开工。把两种边混为一谈会让可并行的面被无谓串行化。

### 5. 风险与未决决策

未决规则应当是决策记录或 issue，不是实现任务里的「待确认」。计划把未决项写成 TODO 却没有承接载体 → `Blocker`。

### 6. 出结论

每条发现给一个严重级别：

- `Blocker`：违反规格、契约/事务/权限/数据一致性不明确，或缺少可证伪的验收定义——必须改；
- `Required`：本轮应补齐的契约、测试或文档同步；
- `Suggestion`：记录为后续事项，不阻塞开工。

## Output contract

1. 结论：**可以开工 / 改完再开工**（一句话理由）；
2. 发现清单，按 `Blocker` → `Required` → `Suggestion` 排序，每条指明计划里的具体位置；
3. 缺失的验收矩阵行（逐行列出该补什么）；
4. 一次合并提问（仅 `[REQUIRED]`/`[APPROVAL]`），附推荐默认值；
5. 范围外与未决决策一览。

不要把计划的复述当成评审输出。评审的价值是指出「哪里还不可证」。

## Failure handling

- 计划薄到无法评审 → 不要替用户补写计划。指出缺哪几块（目的地/矩阵/契约），给出可开工的最小增量。
- 计划与仓库现状冲突（契约已变、模块已不存在）→ 标 `Blocker`，指出冲突位置。
- 用户不同意某条发现 → 记下裁决和理由，不反复重提；只有当它仍会阻塞开工时才保留。

## Examples

输入：「这个改造计划你看看有没有问题，能直接开工吗？」

输出：结论「改完再开工」；2 条 `Blocker`（验收矩阵缺权限与恢复两行；审计记录没有生产 seam 读取路径）；1 条 `Required`（迁移与重启读回未纳入矩阵）；提问项：无（全部输入可从仓库推导）。
