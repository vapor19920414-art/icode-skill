# 步骤 2 — 多轮专项审查

**命令**: `/icode review [N]`
**产出**: `{ICODE_OUT_DIR}/02_review.jsonl`
**会话**: 主会话

采用**独立计划对比 + 多轮循环审查**模式：
- **首轮**：先基于原始需求独立编制简要计划，再与步骤1计划逐项对比，最后做6维度审查
- **后续轮次**：**增量审查**，只审查上一轮修改的部分 + 跨章节影响分析，**最多 N 轮**（N 由 `/icode review [N]` 指定，默认 3）。连续 2 轮无新问题提前终止

## 前置校验

检查 `{ICODE_OUT_DIR}/01_plan.md` 和 `{ICODE_OUT_DIR}/.ico_metadata.json` 是否存在，缺失则报错。

## 执行流程

1. 执行目录管理中的「检测最新目录」逻辑，确定 `ICODE_OUT_DIR`
2. 读取 `{ICODE_OUT_DIR}/01_plan.md` 和 `.ico_metadata.json` 获取原始需求
3. **深度思考**（必须先执行）：需求分解 → 独立方案构思 → 对比要点预判
4. **分步续跑检测**：
   - 解析命令参数获取 `max_rounds`：若 `/icode review N` 提供了正整数 N，则 `max_rounds = N`；否则 `max_rounds = 3`
  - 若 `.ico_metadata.json.status == "review_in_progress"`，从 metadata 恢复 `total_rounds` / `clean_rounds` 字段
    - 同时读取已存在的 `02_review.jsonl` 汇总历史问题
   - 跳过已完成轮次，从当前 `total_rounds` 继续
   - 续跑时以 metadata 中保存的 `max_rounds` 为准（首次执行时写入 metadata）
   - 输出续跑信息：`▶ 步骤2 续跑，从第{total_rounds}轮开始（已完成{total_rounds-1}轮，最多{max_rounds}轮）`
   - 否则初始化 `clean_rounds = 0`, `total_rounds = 1`，`max_rounds` 由参数决定，并设 `status = review_in_progress`，将 `max_rounds` 写入 metadata
5. 输出步骤确认：`▶ 步骤2 审查开始（最多{max_rounds}轮）`

### 首轮审查（`total_rounds == 1`）

**步骤 2.1 — 独立编制计划**：
基于原始需求独立编制一份简要项目计划（架构思路、功能模块、核心接口、实现步骤），**不要参考步骤1计划**。

**步骤 2.2 — 对比分析**：
读取步骤1计划，与你的独立计划逐项对比：遗漏点、偏差点、多余点，给出裁决。

**步骤 2.3 — 逐文件通读（必须先执行，不可跳过）**：
从步骤1计划中识别所有涉及的文件（新建文件、修改文件、依赖文件），逐一通读。
- 对每个现有源文件，从头到尾阅读：函数/结构体/宏定义签名、调用关系、命名风格、错误处理模式
- 对每个计划新建文件，列出其对外的接口承诺

**反偷懒机制**：必须逐一列出每个文件的路径 + 实际阅读发现的关键约束（函数签名、数据结构、宏定义、命名规范）。禁止用"已通读所有文件，无问题"等笼统措辞跳过。如确实未发现问题，须逐文件给出"已通读，无冲突"记录。

**输出通读记录**：列出读过的文件路径 + 关键发现，特别是计划**未提及但实际代码存在**的约束。

**步骤 2.4 — 断言验证审查**：
重点审查计划中标记为 `[未验证]` 的断言，优先用 Read/Grep 实证验证。验证失败的问题直接记为 issue。

**步骤 2.5 — 逐维审查（6个维度，全部覆盖）**：

**反偷懒机制**：每个维度必须给出至少 1 条具体检查结果（可引用文件/章节/行号）。若某维度确实无问题，须明确写出"经检查 X、Y、Z 等方面，未发现问题"而非仅写"通过"。禁止对所有维度统一输出"全部通过"。

1. 逻辑合理性、2. 流程完整性、3. 场景覆盖度、4. 风险遗漏、5. 落地可行性、6. 现有实现对照

**步骤 2.6 — 写入结果**：
向 `{ICODE_OUT_DIR}/02_review.jsonl` 追加一行 JSON，包含：round、mode（`full`/`incremental`）、independent_plan_summary、file_review（files_read + key_findings）、comparison_analysis、dimension_results、has_new_issues、new_issues（每条 issue 遵循下方 Issue 结构化模板）、summary。

要求：
- 每轮只追加 1 行，不再拆分 `review_round_*.json`
- `summary` 字段必须足够让人直接从 JSONL 快速浏览当前轮结论
- 续跑时读取 `02_review.jsonl` 最后一行即可获取最近一轮结论

### 后续轮次 — 增量审查（`total_rounds > 1`，不再重复通读文件）

**增量审查范围**：

1. **修改区域审查**：只审查上一轮 new_issues 导致计划修改的章节，而非全量重审
2. **跨章节影响分析**：检查修改区域对其他章节的连带影响（如接口变更影响调用方、数据结构变更影响解析逻辑）
3. **断言验证跟进**：审查上一轮 `[未验证]` 断言是否已在计划更新中解决
4. **遗漏深挖**：基于之前轮次的发现继续深入，检查更深层次风险

维度同首轮，但仅针对增量范围。完成后向 `02_review.jsonl` 追加当前轮 JSON。

### Issue 结构化模板

每条 issue 必须包含以下三个字段（首轮和后续轮次统一）：

```json
{
  "id": "R{轮次}-{序号}",
  "affected_sections": ["受影响的计划章节编号/标题"],
  "suggestion": "具体建议修改内容",
  "rejection_risk": "若不采纳可能导致的后果"
}
```

### 循环控制

- `total_rounds += 1`
- **实时落盘**：将 `status` 保持为 `review_in_progress`，写入当前 `total_rounds` 和 `clean_rounds` 到 metadata
- 有 issues：`clean_rounds = 0`，**若 `total_rounds <= max_rounds`** 回到后续轮次继续；否则终止
- 无 issues：`clean_rounds += 1`，若 `clean_rounds < 2` 且 `total_rounds <= max_rounds` 继续后续轮次，否则终止
- 终止后更新 `.ico_metadata.json`：`status = review_done`，`completed_steps` 追加 `"2"`
- 全流程模式：**立即继续执行步骤3**

### JSONL 建议字段结构

```json
{
  "round": 1,
  "mode": "full",
  "has_new_issues": true,
  "summary": "首轮发现 2 个结构性问题，需补充接口约束与异常路径",
  "new_issues": []
}
```
