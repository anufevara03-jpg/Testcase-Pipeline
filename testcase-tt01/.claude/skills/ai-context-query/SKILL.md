---
name: ai-context-query
description: ai-context/ 知识库的只读查询入口。按"确认数据库内容"（表名/字段名/枚举名定位定义并跨文件拼装）和"加载模块测试设计上下文"（为某端某模块组装 5 文件+相关 shared，按 6 可测维度组织）两个场景读取知识库，保留每条溯源标注、聚合 `待补充` 缺口透传给下游，绝不编造。当用户提及"查一下某字段的定义"、"确认数据库表结构"、"为某模块加载测试上下文"、"从知识库取模块信息"或下游 testcase-create/biz-test-analysis 需要 ai-context 上下文时使用。
---

# ai-context-query (知识库只读查询)

## 概述

`ai-context/` 的**规范读取入口**：把"端 × 模块 × 关注点 + shared 公共层"的双维度矩阵知识，按下游可用方式读取、跨文件拼装、按维度组织后输出。

**核心边界：本 skill 只读、不写。** `ai-context/` 的唯一写入方是 `ai-context-maintainer`；本 skill 不落盘任何中间产物，全部结果在对话中呈现，交由调用方（`biz-test-analysis` / `testcase-create` 或用户）消费。

**只聚焦两个场景**（不做通用全文检索）：

- **场景 A · 确认数据库内容**：输入表名/字段名/枚举名 → 定位并拼装完整定义。
- **场景 B · 加载模块测试设计上下文**：输入端+模块 → 组装该模块测试设计所需的全部相关知识，按可测维度组织。

**三 skill 分工：**

| 职责 | ai-context-ingest | ai-context-maintainer | ai-context-query（本 skill） |
|---|---|---|---|
| 读 ai-context/ 查询 | ❌（仅比对冲突） | ❌（仅定位写入位置） | ✅ |
| 跨文件拼装 / 按维度组织 | ❌ | ❌ | ✅ |
| 保留/透传溯源与 `待补充` | ✅（提取时） | ✅（写入时） | ✅（读取时） |
| 写 ai-context/ | ❌ | ✅ 唯一 | ❌ |
| 冲突裁决 | ❌ 只列 | 处理 | ❌ 只透传标 `（未证实）` |

## Instructions

### 1. 输入与触发

- **场景 A 入参**：表名 / 字段名 / 枚举名（必填其一）+ 可选端/模块提示。
- **场景 B 入参**：端名 + 模块名（均必填）。
- 触发场景：用户要"确认/查/核对"某表/字段/枚举；或下游 `testcase-create`/`biz-test-analysis` 需要从知识库取某模块上下文。
- 入参不全（如场景 B 缺端名或模块名）→ 先问清，**不擅自假定端/模块**。可用 maintainer §3.3 的目录扫描列出候选端/模块供用户选。

### 2. KB 地图（要找什么 → 去哪读）

完整速查表见 `references/kb-map.md`，要点：

| 要找 | 主源 | 补源 |
|---|---|---|
| 表名/列类型/约束/DDL | `shared/common-data-model.md`（§二表清单定位→§三/四/五对应小节；公共字段在 §一.3） | 模块 `data-model.md`；原文档 `TemporaryFile/SCRM迭代001_数据库设计文档.md` |
| 字段业务语义/枚举值 | 模块 `data-model.md` | 模块 `requirement.md`；shared |
| 业务规则编号/适用端 | `shared/common-business-rules.md`（规则 #1~#17） | 模块 `requirement.md` 引用 |
| 接口路径/参数/错误码 | 模块 `api.md` | `shared/common-api.md` |
| 前端校验/跳转/异常文案 | 模块 `ui-spec.md` | —— |
| 主流程/验收/用户故事 | 模块 `requirement.md` | `ui-spec.md` 页面跳转 |
| 已知缺陷/回归重点/测试数据 | 模块 `test-notes.md` | —— |
| 端关系/数据流 | `system-overview.md` | 对应端 `platform-overview.md` |

> **公共字段在模块文件常省略**（`merchant_id`/`user_id`、`create_time`/`update_time`、`create_user_id`/`update_user_id`、`remark`、`deleted_flag`、`status`）——确认字段完整定义时，**必须回 `shared/common-data-model.md §一.3` 补全**，否则会漏。

### 3. 场景 A · 确认数据库内容

1. **定位模块**：
   - 表名 → 查 `shared/common-data-model.md §二` 模块与表清单总览（表名→域/模块/维护端）。
   - 字段名/枚举名 → grep `platforms/*/modules/*/data-model.md` 找命中模块。
2. **读模块 `data-model.md`**：取该表的业务字段、语义、约束、状态流转、来源标注。
3. **读 `shared/common-data-model.md`**：
   - §一.3 公共字段（模块文件常省略，必补）。
   - §一 设计约定（金额 `DECIMAL(12,4)`、时间戳 `BIGINT`、枚举 `TINYINT`、命名 `snake_case`、租户隔离等）。
   - 该表在 shared 里的权威列类型小节（如激活码表在 §四 B1）；**若该小节仅概述无字段表**（如 §四 B4 客户管理、B5/B7/B8 多为概述），**则列类型同样标 `待补充`，不反复回查、不编造**。
4. **必要时回原文档**：`TemporaryFile/SCRM迭代001_数据库设计文档.md` 取原始 DDL/注释（仅当 shared 与模块都不足时）。
5. **拼装输出**：完整字段表 = 公共字段(shared §一.3) ∪ 业务字段(模块)；**每行保留 `（源：…）`**；缺项标 `待补充` + 注明需提供什么。

> **UI 输入类型 ≠ DB 列类型**：部分模块 `data-model.md`（如 `customer-profile-management`）顶部带 ⚠️ 说明"类型"指 UI 输入类型（下拉单选/文本/日期），DB 列类型标 `待补充`。输出时**必须显式区分**，不可把 UI 类型当列类型上报。

**示例**（查"客户画像自定义字段名最长多少字"）：
- 模块 `customer-profile-management/data-model.md` → 字段配置约束：自定义字段名最长 **10** 字、分组名最长 **30** 字（源：客户画像管理模块.md §四§1 / §三§1）。
- 该模块 DB 列类型/长度/索引/外键全部 `待补充`（源文档未提供 DDL）——输出时单列为缺口，**不臆造**列类型。

### 4. 场景 B · 加载模块测试设计上下文

1. **完整度检查**：复用 maintainer §3.3 逻辑判该模块 5 文件齐全度；缺失文件记入输出"缺口"区。
2. **读模块 5 文件**（requirement/api/data-model/ui-spec/test-notes）。
3. **读相关 shared**：
   - `common-business-rules.md` 中"适用端"含本端的规则，**按相关性筛**（与全局文件一致，不全量塞）：与本模块直接相关的规则（如画像模块→#1 多租户隔离、#10 逻辑删除）展开正文；弱相关规则（如画像模块→#3/#4/#5 端口、#7 翻译计费）**仅编号枚举**，避免稀释下游信号。`#13~#17` 标 `待确认` 的原样透传。
   - `common-data-model.md` §二 定位本模块表 → 读对应小节。
4. **读相关全局**（按与模块相关性筛，**不全量塞**）：`system-overview.md`（端关系/数据流）、`roles-permissions.md`（角色）、`glossary.md`（术语）。
5. **扫 `cross-platform/`**：找涉及本模块的跨端流程；若该目录仅 `README.md` 无具体流程文件（当前现状），记"跨端端到端流程 `待补充`"入缺口，不视为漏读。
6. **按 6 可测维度重组**（核心增值，维度↔KB来源映射见 `references/dimension-playbook.md`）：

| 维度 | KB 来源 |
|---|---|
| 主流程 | requirement 用户故事/验收标准 + ui-spec 页面跳转/联动 + common-business-rules 适用规则 |
| 边界 | data-model 字段约束数值（长度/上限）+ ui-spec 校验规则 |
| 枚举 | data-model 状态枚举 + api 参数枚举值 + common-data-model §一.6 枚举约定 |
| 校验 | ui-spec 前端校验规则 + api 错误码 + 业务规则（重复/唯一/必填） |
| 异常 | test-notes 已知缺陷 + api 错误码 + 状态流转非法跳转 + 业务规则失效模式 |
| 回归 | test-notes 回归重点 + common-business-rules 跨端规则 |

7. **待补充聚合**：把散落各文件的 `待补充`（尤其高危：错误码/列类型/测试数据）聚成一份清单，标注"需提供什么"——这是下游用例生成的**缺口待办源**。
8. **输出 KB 上下文包**（按 `references/output-schema.md`），全程保留来源标注与 `（未证实）`/`待确认` 标记。

**示例**（场景 B 加载 `web-merchant/customer-profile-management`）：
- 主流程：4 条验收标准（新增/编辑/删除隐藏/分组排序字段）（源：requirement.md）。
- 边界：字段名 10 字、分组名 30 字（源：data-model.md §四§1/§三§1）。
- 校验：分组名称重复→"分组名称已存在"、字段名称重复→"字段名称已存在"、删除含字段分组→拦截（源：ui-spec.md 校验规则表）。
- 异常：删除分组/字段二次确认文案（源：ui-spec.md）。
- 回归：字段管理/分组管理/系统默认保护/预览/边界值/异常 6 组回归重点（源：test-notes.md）。
- **缺口聚合**：api.md 全部接口详情 `待补充`、data-model 列类型/索引/外键 `待补充`、测试数据 `待补充`——下游用例 API 层预期值须占位"待产品确认"，不得编造。

### 5. 输出格式

- **溯源标注 100% 透传**：原样保留 `（源：{文件名} §{章节}）`，无裸事实。
- **`待补充` 聚合成独立清单**：每项注明"需提供什么"，不混入事实正文，不自行补全。
- **`（未证实）`/`待确认` 透传不裁决**：如 common-business-rules 的 #13~#17（标 `待确认`）、模块内标 `（未证实）` 的项，原样带出。
- **场景 B 按 6 维度组织**；**场景 A 输出完整字段表**（公共字段 ∪ 业务字段，标注每行来源）。
- 完整格式契约见 `references/output-schema.md`。

### 6. 与下游接口

- 本 skill 产出的 **KB 上下文包**是 `biz-test-analysis`（首选输入）与 `testcase-create`（经 biz-test-analysis）的上下文来源。下游应经本 skill 取上下文，**不直接读 `ai-context/`**，避免漏拼装、漏溯源。
- **`kb_anchor` 回溯锚点规范**：场景 B 输出时为每条可测事实生成锚点，供下游 `coverage.json` 回溯 KB（下游据此报"哪些 KB 锚点未覆盖"，而非裸 FP 列表）：

| 锚点形态 | 含义 | 示例 |
|---|---|---|
| `req-AC-{n}` | requirement.md 验收标准第 n 条 | `req-AC-2`（编辑字段后两端同步） |
| `rule:#{n}` | common-business-rules 规则编号 | `rule:#3`（端口按会话计） |
| `dim:{维度}:{标识}` | 可测维度项 | `dim:boundary:fieldNameLength`（10字） |
| `field:{字段名}` | 具体字段 | `field:username`（激活码） |
| `enum:{枚举名}` | 状态/参数枚举 | `enum:range_type`（去重范围 1-6） |

> **中文标识锚点**：KB 字段/枚举名多为中文（如"客户价值"、"活跃度"），anchor 标识直接用中文（UTF-8，如 `field:客户价值`、`enum:活跃度`），同一项目内保持一致即可。
>
> `待补充` 的项**无锚点**，进"缺口桶"，不计入下游覆盖率分母。

### 7. 读后闸门（行为约束）

镜像 maintainer §5.1 防幻觉铁律，但落地为读取侧的"读后闸门"——输出前逐条自检，**任一不通过则该项不报、转为缺口**：

1. **溯源是否齐全**：每条事实是否都带 `（源：…）`？无裸事实。
2. **有无编造**：高危项（字段类型/枚举/数值/错误码/测试数据）要么有源、要么标 `待补充`，绝不以推断填充。
3. **推测是否隔离**：推断/未证实内容是否标 `（未证实）` 或置于"待确认"区，未混入事实正文？
4. **公共字段是否补全**：确认字段定义时是否回 shared §一.3 补了公共字段？
5. **UI 类型 vs 列类型**：是否显式区分，未把 UI 输入类型当 DB 列类型？

### 8. 依赖

- 依赖 `ai-context-maintainer` 已初始化的 `ai-context/` 结构与已填充内容。
- 继承项目根 `CLAUDE.md` 的防幻觉铁律（铁律 A/B）。
- 完整速查表：`references/kb-map.md`；维度组织法：`references/dimension-playbook.md`；输出契约：`references/output-schema.md`。

## 注意事项

- 所有路径相对于项目根目录。
- 本 skill **只读**：不修改 `ai-context/`、不写业务代码、不落盘中间产物。
- 不裁决冲突：只透传标 `（未证实）`/`待确认`，等用户或 maintainer 处理。
- 输出是 query→biz-test-analysis 的契约，格式须严格（见 `references/output-schema.md`），便于下游直接消费。
