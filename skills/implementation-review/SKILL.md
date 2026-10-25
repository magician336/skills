---
name: implementation-review
description: |
  Review code that an agent (or a human) just wrote, judging delivery completeness and evidence
  validity rather than stylistic defects: every promised acceptance row traced to a real
  implementation seam and current-head evidence, no unrun check reported as passed, no test that
  shadows the production code it claims to cover. Use this skill when the user says
  "评审一下刚写的代码" / "复审" / "代码写完了帮我看看" / "能不能合了" / "验收一下" / "review this change",
  or asks whether an agent's implementation is ready to merge or hand to a human reviewer.
  Do NOT use for pure defect-hunting on a diff (use code-review), for reviewing a plan before
  implementation (use plan-review), or for module-structure assessment (use architecture-health-review).
---

# 实现评审（代码已写完）

判断「这次实现是否真的完成了它承诺的事，以及证据是否可信」。评审对象是**交付完整性与证据有效性**，不是代码风格、命名偏好或重构建议。

如果用户只想要「这段 diff 有什么 bug」，用 `code-review`；本 skill 处理的是「这个 agent 说它做完了，是真的吗」。

## Inputs to collect

- 当前 head（`git rev-parse HEAD`）与比较基（base / PR 目标分支）——证据必须绑定 head，不接受「上次跑过是绿的」；
- 被承诺的验收矩阵 / 计划 / spec（没有就从 issue 读，或向用户要）；
- 变更面：改了哪些文件，触及哪些持久化 / 权限 / 并发 / 审计路径。

先读仓库指令文件（AGENTS.md、CONTEXT.md、docs/adr/ 一类）确认本项目的验收口径，不要用通用习惯替代项目约定。

## Procedure

### 0. 先确认改动存在，再谈质量

先确认交付说明里的改动在当前 head 上**真的存在**：文件在不在、路由注册了没有、提交/PR/SHA 对不对得上。

这一步排在所有质量讨论之前——如果改动不存在，读每一行代码都是空谈，评审结论应直接是"退回"，而不是"风格建议"。交付说明里的每个断言都是**待证命题**，直到被仓库事实证实。

### 1. 重建「承诺了什么」

把验收矩阵逐行抄出来（每行：用户入口 / 领域不变量 / 成功 / 失败-权限-恢复 / 持久化与并发 / 证据）。后面每条发现都挂到某一行上，否则它只是意见。

### 2. 逐行找实现接缝与证据

对矩阵每一行：

1. 找到实现接缝（文件 + 函数/路由 + 关键行）；
2. 找到证据（测试名、确切命令、CI 链接、运行报告）；
3. 判定状态：`PASS` / `FAIL` / `NOT RUN` / `BLOCKED` / `DEFERRED_MANUAL` / `N/A` / `WARNING`。

状态口径见 `references/evidence-rules.md`。**没跑的不能写成跑了**：跳过等于 `NOT RUN`，不等于通过。

### 3. 验证据是否真的钉住了生产代码

这是评审 agent 产出时最容易出问题的一环，逐条查：

- **替身遮蔽生产代码**：测试替身（fake / stub / mock）是不是自己实现了被测行为？断言读的是替身返回的对象，还是生产链路的输出？替身自己置了标记、自己算结果时，断言就钉不住生产代码——把生产侧那行赋值删掉，测试可能照样绿。
- **变异的反证**：挑 1–2 个关键断言做反向验证（临时改坏生产代码，确认测试转红），改完立刻还原。全部转红说明测试确实钉住了。
  **只读评审里做不了变异时**（评审任务限定不改文件），把它降级成一条**要求提交方补做**的检查项并写进结论，不要跳过，也不要为了验证去改生产代码。同理，"测试是否真落库"这类只能靠跑测试确认的点，读代码判断不了就标「待提交方用反向变异证明」，不要猜。
- **断言方向**：断言读的是替身仓库的内部 map，还是重新读了一次库？替身的 setter 是真落库还是空实现？空实现 + 服务层就地改内存，等于断言结构上不可能失败。
- **接缝真实性**：跨模块、持久化、权限、并发、审计的改动必须证明真实的 application / HTTP seam 与事务行为。直接 store 测试、内存替身、或「CI 绿」都不能替代生产路径证据。
- **术语一致**：代码、迁移、接口契约、测试名、PR 描述用同一套术语。术语冲突说明契约没定，不是文档问题。

### 4. 核对范围

对照计划的范围内 / 范围外：

- 少做的：矩阵行找不到实现接缝 → `Blocker`；
- 多做且未声明的：超出范围外但没有对应的矩阵行或决策记录 → `Blocker`（它没被验收过）。

### 5. 判严重级别并给结论

- `Blocker`：违反规格、事务 / 权限 / 数据一致性、缺当前 head 证据，或证据无效（钉不住生产代码）；
- `Required`：本轮应补齐的契约、测试或文档同步；
- `Suggestion`：记录为后续事项，不阻塞。

同一类问题连续两轮修完又冒新 `Blocker` → 停止继续打补丁，回到 `plan-review` 重新切分目的地或拆成更小的片。

## Output contract

1. 当前 head 与比较基；
2. 验收矩阵逐行状态表（行 / 状态 / 实现接缝 / 证据 / 缺什么）；
3. 发现清单，按 `Blocker` → `Required` → `Suggestion` 排序，每条挂到矩阵行或具体文件行号；
4. 结论：**可交人审 / 不可交人审**，以及还差哪几条。

常规测试、日志整理、证据汇总这类 agent 自己能完成的事，不要塞进交给用户的清单——评审的产物是**可执行的判断**，不是待办转交。

## Failure handling

- 找不到计划 / 矩阵 → 明说「无法判断交付完整性」，改为对照用户陈述的意图评审，并把这一限制写进结论。
- 证据指向另一个 commit → 标 `NOT RUN`（当前 head 上未验证），不要采信旧证据。
- 环境跑不起来 → 标 `NOT RUN` / `BLOCKED` 并写明确切命令与失败原因，不要改写成通过。
- 变更过大、难以逐行对照 → 先按模块分块审，明确声明哪几块只做了抽样；不要用「整体看过」掩盖未覆盖的部分。

## Examples

输入：「agent 说这活干完了，PR 在 #123，帮我看看能不能合。」

输出：head `abc1234`；矩阵 5 行中 3 行 `PASS`、1 行 `NOT RUN`（并发场景，当前 head 未跑）、1 行 `DEFERRED_MANUAL`（真实环境人工验收，已挂 #130）；1 条 `Blocker`：权限矩阵行的断言读的是替身仓库内部 map，且替身 setter 是空实现——删掉服务层赋值测试仍绿，该断言无效；结论：不可交人审。
