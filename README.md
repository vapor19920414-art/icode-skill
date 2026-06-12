# ICode — 六步全流程编码工作流

ICode 是一个 Claude Code 技能（Skill），将需求到交付拆解为 6 个严格步骤，每步可单独调用，手动切换模型时操作更灵活。

## 特性

- **计划 → 审查 → 定稿 → 编码 → 复检 → 终审**，六步闭环
- 每步可独立调用，使用当前会话模型
- 全流程模式（`/icode new`）自动串联 6 步
- 所有步骤在主会话中执行，无子 Agent 隔离问题
- 产物保存在 `.icode_output_N/` 目录，支持跨会话恢复
- 元信息管理（`.ico_metadata.json`），以 `status` 作为单一流程状态真值
- 三阶段递进复检（Reverse → Fixed → Free），逆推视角防偷懒复读
- Plan 步骤强制断言验证（Read/Grep 实证，`[已验证]`/`[未验证]` 标记）
- 架构决策记录（ADR）章节，集中管理方案选型与取舍
- Review 支持指定轮数（`/icode review [N]`）和增量审查模式
- Review 意见结构化输出（影响章节/建议/否决风险）
- 步骤2产物压缩为单一 `02_review.jsonl`，每轮一行 JSON，且固定 schema，减少自由发挥
- 步骤5产物压缩为单一 `05_deepcheck.jsonl`，按阶段/轮次顺序追加 JSONL 记录，且固定 schema
- `audit` 只出报告，不在终审阶段自动改码

## 安装

将本仓库克隆到 Claude Code skills 目录：

```bash
git clone <repo-url> ~/.claude/skills/icode
```

## 快速开始

```bash
# 一步走得全流程
/icode new 实现MCU雨量传感器I2C驱动

# 推荐：需求文件模式（类似 add file to chat）
/icode plan @.icode_reqs/bluetooth_open.md

# 或者分步执行
/icode plan 实现MCU雨量传感器I2C驱动   # 步骤1：拟定计划
/icode review                          # 步骤2：专项审查（默认3轮）
/icode review 5                        # 步骤2：指定5轮审查
/icode merge                           # 步骤3：合并定稿
/icode code                            # 步骤4：编码实施
/icode deepcheck                       # 步骤5：循环复检
/icode audit                           # 步骤6：终极终审
```

## 命令一览

| 命令 | 功能 | 创建目录 |
|------|------|----------|
| `/icode help` | 帮助：输出使用流程示例 | 否 |
| `/icode new <需求\|@需求文件>` | 全流程：创建目录 → 步骤1→6 | ✅ |
| `/icode plan <需求\|@需求文件>` | 仅步骤1：拟定项目计划 | ✅ |

## 需求文件模式（推荐）

如果需求信息较长，或你希望带上关联文件、验收标准等上下文，建议在项目根目录维护 `.icode_reqs/`：

```bash
mkdir -p .icode_reqs
cp requirements/REQ_TEMPLATE.md .icode_reqs/bluetooth_open.md

# 编辑需求文件后执行
/icode plan @.icode_reqs/bluetooth_open.md
```

探索范围优先级：
- 显式 `--focus` 最高优先级
- 未传 `--focus` 且需求文件含“关联文件/Focus Paths”时，按需求文件路径做精准探索
- 两者都没有时，才回退关键词驱动搜索（仍禁止全项目扫描）
| `/icode review [N]` | 仅步骤2：专项审查计划（N=轮数，默认3） | 否 |
| `/icode merge` | 仅步骤3：合并审查意见定稿 | 否 |
| `/icode code` | 仅步骤4：落地编码实施 | 否 |
| `/icode deepcheck` | 仅步骤5：三阶段递进复检 | 否 |
| `/icode audit` | 仅步骤6：终极终审并出具报告 | 否 |

## 操作约定

- `new` 和 `plan` 会创建新的 `.icode_output_N/` 目录；其他命令默认使用最新目录。
- 步骤4编译失败时，只写 `status = code_compile_failed`，不再额外维护单独布尔字段。
- 步骤2只写 `02_review.jsonl`；每轮一行 JSON，且必须遵守步骤2文档定义的固定 schema，对步骤3来说它就是唯一审查真值源。
- 步骤5只写 `05_deepcheck.jsonl`；每轮一行 JSON，且必须遵守步骤5文档定义的固定 schema。
- 步骤6只负责终审和出报告；如果报告仍有问题，回到 `/icode code` 或 `/icode deepcheck` 后再重新审计。
- 步骤2/5 支持中断续跑；恢复时直接重跑同一命令，按 `.ico_metadata.json` 中的 `status`、轮次、phase 接着执行。

## 执行方式

所有 6 个步骤在主会话中执行，使用当前会话模型。不主动切换，用户可自行 `/model`。

## 目录结构

```
.icode_output_N/
├── .ico_metadata.json      # 元信息（状态、代码文件列表）
├── 01_plan.md              # 步骤1：项目计划
├── 02_review.jsonl         # 步骤2：审查流水（JSONL，每轮一行）
├── 03_plan_final.md        # 步骤3：定稿计划
├── 05_deepcheck.jsonl      # 步骤5：逆推/Fixed/Free 全部复检流水（JSONL）
└── 06_audit.md             # 步骤6：终审报告
```

## 中断与续跑

可以。ICode 的步骤 2、5 和全流程串联都支持中断后继续，恢复依据是 `.icode_output_N/.ico_metadata.json`。

常见恢复方式：

```bash
# 1) 先确认最近一次产出目录
ls -d .icode_output_*

# 2) 查看当前状态
cat .icode_output_N/.ico_metadata.json

# 3) 按状态恢复
/icode review      # status=review_in_progress 时，继续步骤2
/icode deepcheck   # status=deepcheck_in_progress 时，继续步骤5
/icode merge       # status=review_done 时，进入步骤3
/icode code        # status=plan_finalized 或 code_compile_failed 时，进入/重做步骤4
/icode audit       # status=deepcheck_done 时，进入步骤6
```

恢复规则：

- `/icode new` 串联执行时如果会话中断，重新进入后按 metadata 的 `completed_steps` 或 `status` 继续下一步即可，不需要新建目录。
- `/icode review [N]` 会恢复 `total_rounds`、`clean_rounds`、`max_rounds`，并继续向 `02_review.jsonl` 追加。
- `/icode deepcheck` 会恢复 `deepcheck_total_rounds`、`deepcheck_clean_rounds`、`deepcheck_phase`，并继续向 `05_deepcheck.jsonl` 追加。
- 如果你是有意重跑某一步，直接再次执行该命令；该步骤会覆盖或续写自己的产物，不影响其他步骤文件。

## 工作流程

```
[步骤1] 拟定计划 → [步骤2] 专项审查 → [步骤3] 合并定稿
                                              ↓
[步骤6] 终审报告 ← [步骤5] 循环复检 ← [步骤4] 编码实施
```

## 版本

当前版本：v1.6.0

详细说明请见 [SKILL.md](SKILL.md)。

## 许可证

MIT
