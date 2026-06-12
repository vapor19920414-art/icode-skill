# 步骤 3 — 吸纳评审意见、合并优化定稿

**命令**: `/icode merge`
**产出**: `{ICODE_OUT_DIR}/03_plan_final.md`
**会话**: 主会话

## 前置校验

检查 `{ICODE_OUT_DIR}/01_plan.md` 和 `{ICODE_OUT_DIR}/02_review.jsonl` 是否存在，缺失则报错并提示先执行对应步骤。

## 执行步骤

1. 执行目录管理中的「检测最新目录」逻辑，确定 `ICODE_OUT_DIR`
2. 读取 `{ICODE_OUT_DIR}/01_plan.md`（原始计划）和 `{ICODE_OUT_DIR}/02_review.jsonl`（审查详情）
3. **深度思考**（必须先执行）：逐条甄别审查意见 → 判断采纳/驳回 → 规划修改策略
4. 输出步骤确认：`▶ 步骤3 定稿开始`

### 合并定稿

解析 `02_review.jsonl` 中各轮审查意见前，先逐行做 schema 校验：
- 每行必须是合法 JSON，且 `schema_version == "icode.review.v1"`、`step == "review"`
- 顶层字段必须齐全，不得缺少固定 schema 中的任何字段
- `round` 必须按追加顺序严格递增；`mode` 必须满足首轮 `full`、后续轮次 `incremental`
- `dimension_results` 必须严格为 6 项且顺序固定；其中 `issue_ids` 只能引用本行 `new_issues[*].id`
- `has_new_issues` 与 `new_issues` 必须一致：`true` 时不得为空，`false` 时必须为空数组
- 任一行缺字段、错字段、枚举值非法、ID 对不上、JSON 损坏，都必须立即报错并停止定稿，禁止猜测补全或跳过坏行

通过校验后，按 `round` 升序消费全部记录，重点关注：
- 首轮的 `file_review.key_findings`（通读实际代码发现的问题）
- 所有轮的 `new_issues`（维度审查发现的问题，含 `affected_sections`/`suggestion`/`rejection_risk` 结构化字段）

**固定 schema 消费细则**：
1. `summary` 仅用于快速浏览，不得替代结构化字段；最终判断必须逐条读取 `file_review.key_findings`、`new_issues`、`comparison_analysis`、`incremental_scope`、`dimension_results`
2. 首轮 `file_review.key_findings` 是步骤3吸收“通读代码约束”的唯一真值源；即使 `file_review.summary` 写得更详细，也不得只凭 summary 采纳或否决
3. `new_issues` 是步骤3吸收“结构化待处理问题”的唯一 issue 清单；不得从 `summary`、`comparison_analysis`、`dimension_results.evidence` 中自行再发明新的 issue 编号
4. 对每条 `new_issues[*]`，必须把 `affected_sections`、`suggestion`、`rejection_risk` 三个字段成组读取后再裁决，禁止只看其中一项就采纳/否决
5. `comparison_analysis` 与 `incremental_scope` 仅用于理解该轮审查的上下文背景，不参与步骤 3 的裁决决策；如果其中内容未落到 `new_issues` 或首轮 `file_review.key_findings`，不得直接当成独立待处理项
6. 当 `has_new_issues = false` 时，本轮不得从其他字段反推“隐含 issue”；只能把该轮视为无新增问题的背景信息

**要求**：
1. 逐条甄别审查意见（含 `new_issues` 和 `file_review.key_findings`），两者同等重要
2. 利用 issue 的 `affected_sections` 字段定位计划中需修改的章节，利用 `suggestion` 字段理解建议修改，利用 `rejection_risk` 评估否决后果
3. 对每条 issue 做出判断：采纳 / 部分采纳 / 否决。否决必须写明理由（rejection_reason）
4. `file_review.key_findings` 中的接口约束、命名模式、隐式依赖等，若计划未覆盖，必须补充
5. 每处修改标注 `[审查采纳 #编号]` 或 `[通读发现]` 或 `[审查否决 #编号: 理由]` 标记
6. 保持整体架构不变
7. 输出前必须自检：章节完整、编号连续、校验项 checkbox 格式正确

**写入定稿**：使用当前会话可用的文件写入工具写入 `{ICODE_OUT_DIR}/03_plan_final.md`。

### 定稿自检

读取刚写入的 `{ICODE_OUT_DIR}/03_plan_final.md`，逐项检查：

- 9 章节完整（概述、功能需求、架构设计、架构决策记录ADR、详细设计、异常处理、实现步骤、校验项、风险评估）
- 校验项 checkbox 格式正确（`- [ ]` 或 `- [x]`）
- 章节编号连续无重复
- 所有 [审查采纳] 标记与审查意见对应
- 任何缺失、断裂、矛盾处必须修复后再继续
- **无需添加新功能或重构，仅修复格式和结构问题**

### 强制操作

- **更新 `.ico_metadata.json`**：`status = plan_finalized`，`completed_steps` 追加 `"3"`
- 全流程模式：**立即继续执行步骤4**
