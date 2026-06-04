# 步骤 3 — 吸纳评审意见、合并优化定稿

**命令**: `/icode merge`
**产出**: `{ICODE_OUT_DIR}/03_plan_final.md`
**会话**: 主会话

## 前置校验

检查 `{ICODE_OUT_DIR}/01_plan.md` 和 `{ICODE_OUT_DIR}/02_review.md` 是否存在，缺失则报错并提示先执行对应步骤。

## 执行步骤

1. 执行目录管理中的「检测最新目录」逻辑，确定 `ICODE_OUT_DIR`
2. 读取 `{ICODE_OUT_DIR}/01_plan.md`（原始计划）和 `{ICODE_OUT_DIR}/02_review.md`（审查意见）
3. **深度思考**（必须先执行）：逐条甄别审查意见 → 判断采纳/驳回 → 规划修改策略
4. 输出步骤确认：`▶ 步骤3 定稿开始`

### 合并定稿

解析 02_review.md 中的审查意见——从每个 json 代码块提取 JSON，重点关注：
- 首轮的 `file_review.key_findings`（通读实际代码发现的问题）
- 所有轮的 `new_issues`（维度审查发现的问题）

**要求**：
1. 逐条甄别审查意见（含 `new_issues` 和 `file_review.key_findings`），两者同等重要
2. `file_review.key_findings` 中的接口约束、命名模式、隐式依赖等，若计划未覆盖，必须补充
3. 每处修改标注 `[审查采纳 #编号]` 或 `[通读发现]` 标记
4. 保持整体架构不变
5. 输出前必须自检：章节完整、编号连续、校验项 checkbox 格式正确

**写入定稿**：使用 Write 工具写入 `{ICODE_OUT_DIR}/03_plan_final.md`。

### 定稿自检

读取刚写入的 `{ICODE_OUT_DIR}/03_plan_final.md`，逐项检查：
- 8 章节完整（概述、功能需求、架构设计、详细设计、异常处理、实现步骤、校验项、风险评估）
- 校验项 checkbox 格式正确（`- [ ]` 或 `- [x]`）
- 章节编号连续无重复
- 所有 [审查采纳] 标记与审查意见对应
- 任何缺失、断裂、矛盾处必须修复后再继续
- **无需添加新功能或重构，仅修复格式和结构问题**

### 强制操作

- **更新 `.ico_metadata.json`**：`status = plan_finalized`，`completed_steps` 追加 `"3"`
- 全流程模式：**立即继续执行步骤4**
