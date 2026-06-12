# 步骤 5 — 三阶段递进深度复检

**命令**: `/icode deepcheck`
**产出**: `{ICODE_OUT_DIR}/05_deepcheck.jsonl`
**会话**: 主会话

## 前置校验

检查 `{ICODE_OUT_DIR}/03_plan_final.md` 和步骤4创建的代码文件是否存在，缺失则报错并提示先执行 `/icode code`。

## 三阶段说明

| 顺序 | 阶段 | 输入 | 目标 |
|------|------|------|------|
| 1 | **Reverse**（逆推） | 只给代码 | 从代码逆推需求规格，对比计划找差异 |
| 2 | **Fixed**（固定维度） | 计划 + 最新代码 | 7 个固定维度，逐项覆盖 |
| 3 | **Free**（自由探索） | 计划 + 最新代码 | 一次性完整覆盖15个角度 |

**阶段切换规则**：
- Reverse：单次执行，完成后进入 Fixed
- Fixed → Free：Fixed 首次全 clean 后切换
- Free：单次完整执行后终止

## 关键：代码新鲜度

**每轮开始前必须重新读取所有代码文件**。上一轮发现的问题已修复，必须基于最新代码分析。

## Free 阶段角度管理（15 个）

| # | 角度 | # | 角度 |
|---|------|---|------|
| A1 | 计划实施一致性 | A9 | 性能热点 |
| A2 | 逻辑闭环 | A10 | 可测试性 |
| A3 | 异常处理完备性 | A11 | 可维护性 |
| A4 | 边界与极端值 | A12 | 编译器/构建兼容 |
| A5 | 代码规范与风格 | A13 | 跨平台与移植 |
| A6 | 安全漏洞 | A14 | API/ABI 兼容 |
| A7 | 并发与重入安全 | A15 | 文档与注释一致性 |
| A8 | 资源与内存管理 | | |

Free 阶段一次性完整覆盖全部 15 个角度。

### 反偷懒机制

**必须先建立计划-代码追溯矩阵**（逐条列出计划功能点/接口/约束，标记代码对应位置和完成状态），再逐维度评估。禁止跳过追溯直接给"全部通过"。

## 执行步骤

1. 检测最新目录，确定 `ICODE_OUT_DIR`
2. 读取 `03_plan_final.md` 和 `.ico_metadata.json`
  - 若 `.ico_metadata.json.status == "code_compile_failed"`，输出 `⚠️ 步骤4编译失败，仍继续复检` 警告
3. **深度思考**（必须先执行）：梳理代码清单 → 回顾计划要点 → 制定逆推/Fixed/Free 检查策略
4. **分步续跑**：若 `status == "deepcheck_in_progress"`，从 metadata 恢复 `deepcheck_total_rounds` / `deepcheck_clean_rounds` / `deepcheck_phase`，同时读取已存在的 `05_deepcheck.jsonl`；若其中已存在 `phase = "reverse"` 的记录，则跳过 Reverse
5. 否则初始化 `deepcheck_clean_rounds = 0`, `deepcheck_total_rounds = 1`, `deepcheck_phase = "reverse"`, `status = deepcheck_in_progress`
6. 输出：`▶ 步骤5 复检开始`

### 阶段 1 — Reverse（逆推）

**重新读取所有代码文件**。基于代码**逆推**需求规格——不允许参考计划或需求文档，只从代码推断。

**反偷懒机制**：必须先逐文件列出所有导出接口/数据结构/函数签名（含文件+行号），再基于证据推断行为。禁止跳过逐文件列举直接写"代码实现了全部功能"等笼统结论。

**逆推内容**：
- 列出所有导出函数/接口（签名、参数、返回值）
- 列出所有数据结构/类型/枚举
- 描述每个模块/函数的实际行为（含分支、错误处理、边界）
- 描述跨文件调用关系和数据流
- 验证代码注释与实际执行路径是否一致
- 列出**从代码无法确定**的需求（标注 "unclear"）

向 `{ICODE_OUT_DIR}/05_deepcheck.jsonl` 追加一行 `phase = "reverse"` 的 JSON 记录。**必须严格遵守下方固定 schema**，禁止自行新增、删除、改名顶层字段；若某阶段字段本轮不适用，必须保留字段并填空数组、空字符串或 `false`，不得省略。

**对比**：读取 `03_plan_final.md`，与逆推规格做机械 diff：
- **欠实现**：计划有，逆推没有
- **偏离/冗余**：逆推有，计划没提

发现问题则用 Edit 修复代码。更新计数器，并在 `05_deepcheck.jsonl` 中继续追加当前轮结果。`deepcheck_phase` 切换为 `"fixed"`。

### 阶段 2 — Fixed（固定维度）

**重新读取所有代码文件**（含 Reverse 修复后的最新版）。

**反偷懒机制**：必须先建立计划-代码追溯矩阵（逐条列出计划功能点/接口/约束，标记代码对应位置和完成状态），再逐维度检查。禁止跳过追溯直接写"维度X: 通过"。

7 维度逐项检查：
1. 计划实施一致性 — 逐条对照每个功能点/接口/约束
2. 逻辑闭环 — 数据流、控制流、跨文件调用链
3. 异常处理 — 错误码、异常场景、边界条件
4. 边界场景 — 空值、越界、超时、并发
5. 规范写法 — 项目代码风格
6. 潜在隐患 — 内存泄漏、死锁、资源竞争、安全漏洞
7. 跨文件一致性 — 接口变更全链路同步

向 `05_deepcheck.jsonl` 追加一行当前轮 JSON，仍必须满足完整固定 schema。

### 阶段 3 — Free（自由探索）

**重新读取所有代码文件**。一次性完整覆盖全部 15 个角度（A1-A15），每个角度须给出 3+ 具体检查点（文件名+行号）。严禁"整体通过"等偷懒措辞。

### 固定 JSONL Schema

`05_deepcheck.jsonl` 的每一行都必须是**完全相同的顶层结构**，字段顺序固定如下：

```json
{
  "schema_version": "icode.deepcheck.v1",
  "step": "deepcheck",
  "phase": "fixed",
  "round": 2,
  "has_issues": false,
  "summary": "Fixed 阶段首轮 clean，进入 Free",
  "traceability_matrix": [
    {
      "plan_item": "实现 rain_sensor_read 接口",
      "code_refs": ["src/rain_sensor.c#L88", "include/rain_sensor.h#L21"],
      "status": "implemented",
      "notes": "参数校验与错误码返回已落地"
    }
  ],
  "files_snapshot": [
    {
      "path": "src/rain_sensor.c",
      "exports": ["int rain_sensor_read(rain_sensor_t *ctx, int *mm)"],
      "key_types": ["struct rain_sensor_t"],
      "key_behaviors": ["读取前检查 ctx/mm 非空", "失败时返回 -EINVAL"]
    }
  ],
  "reverse_analysis": {
    "inferred_spec": ["驱动提供初始化和读数接口"],
    "unclear_requirements": ["是否需要支持异步读取"],
    "plan_gaps": ["计划未说明重试次数上限"],
    "code_extras": ["代码新增调试日志开关"]
  },
  "fixed_results": [
    {
      "dimension": "plan_alignment",
      "status": "clean",
      "evidence": ["03_plan_final.md#L52 对应 src/rain_sensor.c#L88"],
      "issue_ids": []
    },
    {
      "dimension": "logic_closure",
      "status": "clean",
      "evidence": ["初始化到读取的数据流已闭环"],
      "issue_ids": []
    },
    {
      "dimension": "error_handling",
      "status": "issue_found",
      "evidence": ["超时重试失败后缺少日志分支"],
      "issue_ids": ["D2-1"]
    },
    {
      "dimension": "boundary_cases",
      "status": "clean",
      "evidence": ["已覆盖空指针与 nack 场景"],
      "issue_ids": []
    },
    {
      "dimension": "style_consistency",
      "status": "clean",
      "evidence": ["命名与错误码风格一致"],
      "issue_ids": []
    },
    {
      "dimension": "latent_risks",
      "status": "clean",
      "evidence": ["未见资源泄漏或竞态"],
      "issue_ids": []
    },
    {
      "dimension": "cross_file_consistency",
      "status": "clean",
      "evidence": ["头文件声明与实现同步"],
      "issue_ids": []
    }
  ],
  "free_results": [
    {
      "angle": "A1",
      "status": "clean",
      "evidence": ["src/rain_sensor.c#L88", "include/rain_sensor.h#L21", "tests/rain_sensor_test.c#L14"],
      "issue_ids": []
    }
  ],
  "issues": [
    {
      "id": "D2-1",
      "severity": "major",
      "title": "超时失败后缺少日志",
      "code_refs": ["src/rain_sensor.c#L114"],
      "suggestion": "补充超时失败日志并保持错误码透传"
    }
  ],
  "fixes_applied": [
    {
      "issue_id": "D2-1",
      "files": ["src/rain_sensor.c"],
      "summary": "补充超时失败日志分支"
    }
  ],
  "next_phase": "free"
}
```

固定约束：

- `schema_version` 固定为 `icode.deepcheck.v1`
- `step` 固定为 `deepcheck`
- `phase` 只能是 `reverse`、`fixed`、`free`
- `traceability_matrix[*].status` 只能是 `implemented`、`partial`、`missing`、`unclear`
- `fixed_results` 必须严格按以下顺序输出 7 项：`plan_alignment`、`logic_closure`、`error_handling`、`boundary_cases`、`style_consistency`、`latent_risks`、`cross_file_consistency`
- `fixed_results[*].status` 只能是 `clean` 或 `issue_found`
- `free_results` 仅允许使用角度编号 `A1` 到 `A15`；Free 阶段必须覆盖全部 15 项且顺序固定，Reverse/Fixed 阶段则必须输出空数组
- `issues[*].severity` 只能是 `critical`、`major`、`minor`
- `next_phase` 只能是 `fixed`、`free`、`done`
- Reverse 阶段：`fixed_results`、`free_results` 必须为空数组，`reverse_analysis` 必须填写
- Fixed 阶段：`reverse_analysis` 可保留为空对象内容，`fixed_results` 必须 7 项齐全，`free_results` 必须为空数组
- Free 阶段：`fixed_results` 必须为空数组，`free_results` 必须 15 项齐全
- `has_issues = false` 时 `issues` 必须为空数组；`has_issues = true` 时 `issues` 不得为空

### Issue 固定结构

```json
{
  "id": "D2-1",
  "severity": "major",
  "title": "超时失败后缺少日志",
  "code_refs": ["src/rain_sensor.c#L114"],
  "suggestion": "补充超时失败日志并保持错误码透传"
}
```

### 循环控制

- `deepcheck_total_rounds += 1`
- **实时落盘**：`status = deepcheck_in_progress`，写入当前 `deepcheck_total_rounds`/`deepcheck_clean_rounds`/`deepcheck_phase` 到 metadata
- **阶段切换时重置 `deepcheck_clean_rounds = 0`**（每个阶段独立计数）
- Reverse 阶段：单次执行后始终进入 Fixed，重置 `deepcheck_clean_rounds = 0`，不参与循环
- has_issues → 修复 → `deepcheck_clean_rounds = 0` → 回到**当前阶段**重新执行（重新读代码）。**Free 阶段修复后不重跑**
- 无 issues → `deepcheck_clean_rounds += 1`
  - Fixed 首次全 clean → 切换 `deepcheck_phase = "free"`，`deepcheck_clean_rounds = 0`
  - Free 完成后 → 终止
- 终止后更新 `.ico_metadata.json`：`status = deepcheck_done`，`completed_steps` 追加 `"5"`
- 全流程模式：**立即继续执行步骤6**

### 最小合法示例

```json
{
  "schema_version": "icode.deepcheck.v1",
  "step": "deepcheck",
  "phase": "fixed",
  "round": 2,
  "has_issues": false,
  "summary": "Fixed 阶段首轮 clean，进入 Free",
  "traceability_matrix": [],
  "files_snapshot": [],
  "reverse_analysis": {
    "inferred_spec": [],
    "unclear_requirements": [],
    "plan_gaps": [],
    "code_extras": []
  },
  "fixed_results": [
    {"dimension": "plan_alignment", "status": "clean", "evidence": [], "issue_ids": []},
    {"dimension": "logic_closure", "status": "clean", "evidence": [], "issue_ids": []},
    {"dimension": "error_handling", "status": "clean", "evidence": [], "issue_ids": []},
    {"dimension": "boundary_cases", "status": "clean", "evidence": [], "issue_ids": []},
    {"dimension": "style_consistency", "status": "clean", "evidence": [], "issue_ids": []},
    {"dimension": "latent_risks", "status": "clean", "evidence": [], "issue_ids": []},
    {"dimension": "cross_file_consistency", "status": "clean", "evidence": [], "issue_ids": []}
  ],
  "free_results": [],
  "issues": [],
  "fixes_applied": [],
  "next_phase": "free"
}
```
