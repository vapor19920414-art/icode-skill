# 步骤 3 — 吸纳评审意见、合并优化定稿

**命令**: `/icode merge`
**产出**: `{ICODE_OUT_DIR}/03_plan_final.md`
**模型**: 全流程模式 `opus`；分步模式使用当前会话模型

## 前置校验

检查 `{ICODE_OUT_DIR}/01_plan.md` 和 `{ICODE_OUT_DIR}/02_review.md` 是否存在，缺失则报错并提示先执行对应步骤。

## 执行步骤

### ⚠️ 强制规则：禁止主 Agent 直接合并定稿

定稿合并**必须由子 Agent 独立完成**。主 Agent 只负责确定目录、读取文件、启动子 Agent、写文件。**主 Agent 不得自行撰写定稿计划。**

1. 执行目录管理中的「检测最新目录」逻辑，确定 `ICODE_OUT_DIR`
2. 读取 `{ICODE_OUT_DIR}/01_plan.md`（原始计划）和 `{ICODE_OUT_DIR}/02_review.md`（审查意见）
3. 按「通用规则」确定当前模型
4. 输出模型确认：`▶ 步骤3 使用模型：{当前模型名称}`
5. **启动子 Agent** 执行合并定稿，prompt 为：

```
当前使用模型：{当前模型名称}。

请将以下审查意见合并优化进原始计划。

原始计划：
{读取 {ICODE_OUT_DIR}/01_plan.md 的全部内容}

审查意见（markdown 格式，每轮一个区块）：
{读取 {ICODE_OUT_DIR}/02_review.md 的全部内容}

**解析方法**：从每个 ```json 代码块提取 JSON，重点关注：
- 首轮的 `file_review.key_findings`（通读实际代码发现的问题）
- 所有轮的 `new_issues`（维度审查发现的问题）

要求：
1. 逐条甄别审查意见（含 `new_issues` 和 `file_review.key_findings`），两者同等重要
2. `file_review.key_findings` 中提到的接口约束、命名模式、隐式依赖等，若计划未覆盖，必须补充到详细设计或风险评估中
3. 每处修改标注 [审查采纳 #编号] 或 [通读发现] 标记
4. 保持整体架构不变
5. 输出前必须自检：章节完整、编号连续、校验项 checkbox 格式正确

**优先使用 prompt 中已提供的信息进行分析，必要时可执行工具调用**。

直接输出完整定稿计划，以 ===FINAL PLAN START=== 开头、===FINAL PLAN END=== 结尾。
```

6. **提取定稿内容**（从 ===FINAL PLAN START=== 到 ===FINAL PLAN END===），写入 `{ICODE_OUT_DIR}/03_plan_final.md`
7. **定稿计划自检**：读取刚写入的 `{ICODE_OUT_DIR}/03_plan_final.md`，逐项检查：
   - 8个章节是否完整（概述、功能需求、架构设计、详细设计、异常处理、实现步骤、校验项、风险评估）
   - 校验项 checkbox 格式是否正确（`- [ ]` 或 `- [x]`）
   - 章节编号是否连续、无重复
   - 所有 [审查采纳] 标记是否与审查意见对应
   - 任何缺失、断裂、矛盾处必须修复后再继续
   - **无需添加新功能或重构，仅修复格式和结构问题**
8. **更新 `.ico_metadata.json`**：`status = plan_finalized`，`completed_steps` 追加 `"3"`
9. 全流程模式：**立即继续执行步骤4**
