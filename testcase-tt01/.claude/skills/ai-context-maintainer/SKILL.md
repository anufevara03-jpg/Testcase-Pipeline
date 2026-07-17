---
name: ai-context-maintainer
description: 维护多端 AI 测试知识库（ai-context/），支持 PC客户端、Web管理端、Web商户端等多端项目。负责创建/更新各端模块文件、管理跨端共享逻辑、检查知识库完整性与规范性。当用户提及"更新测试知识库"、"创建模块上下文"、"检查知识库"或类似意图时使用。
---

# ai-context-maintainer (Multi-Platform Edition)

## 概述

维护多端 AI 测试知识库（`ai-context/`），支持**多端项目**（如 PC客户端、Web管理端、Web商户端）。负责创建/更新各端模块文件、管理跨端共享逻辑、检查知识库完整性与规范性。适用场景：用户提及"更新测试知识库"、"创建模块上下文"、"检查知识库"等意图时。

## Instructions

### 1. 知识库多端目录结构

所有操作必须遵循以下约定：
```
ai-context/
├── README.md
├── glossary.md # 全局术语表
├── system-overview.md # 各端关系、架构简图
├── roles-permissions.md # 全局角色-权限矩阵（标注各端适用范围）
├── changelog.md # 全局变更日志
│
├── shared/ # 所有端共用的铁律
│   ├── common-business-rules.md # 核心业务规则（如一手机号全网唯一）
│   ├── common-data-model.md # 共用实体（用户、订单主表等）
│   ├── common-api.md # 多端调用的同一套接口
│   ├── common-components.md # 共用UI组件交互（可选）
│   └── env-config.md # 环境信息（脱敏）
│
├── platforms/ # 按"端"划分
│   └── {platform-name}/ # 如 pc-client, web-admin, web-merchant
│       ├── platform-overview.md # 该端特有说明、技术栈、交互范式
│       └── modules/
│           └── {module-name}/
│               ├── requirement.md
│               ├── api.md
│               ├── data-model.md
│               ├── ui-spec.md
│               └── test-notes.md
│
├── cross-platform/ # 跨端联动流程
│   ├── refund-flow.md # 如退款：商户端→管理端→客户端
│   └── notification-rules.md
│
└── templates/ # AI 输出格式模板（非 Skill 模板）
    ├── test-case-template.md
    └── bug-report-template.md
```

- **端名**统一使用 kebab-case（如 `pc-client`, `web-admin`, `web-merchant`）。
- **模块名**同样使用 kebab-case（如 `user-management`, `order-detail`）。

### 2. 任务识别

根据用户输入判断任务类型：

- **创建新模块**：需明确"哪个端"+"什么模块"。
- **更新文件**：可能针对具体端的模块、共享层（`shared/`）或跨端流程（`cross-platform/`）。
- **完整性检查**：检查所有端及其模块是否缺文件。
- **规范性校验**：校验指定文件/模块内容是否符合规范。
- **全局初始化**：为新项目创建完整的多端骨架。
- **提炼共享内容**：将某端文档中重复出现的规则提升至 `shared/` 或 `cross-platform/`。
- **按提取报告写入**：接收 `ai-context-ingest` 产出的提取报告，按报告内容写入对应端/模块文件（见 §3.7）。

若意图或必要参数（如端名）缺失，请先向用户确认。

### 3. 核心任务操作流程

#### 3.1 创建新模块（多端）

1. 确认**端名**和**模块名**。
2. 检查 `platforms/{端名}` 是否存在，若不存在则先创建该端目录及 `platform-overview.md`（使用 `.claude/skills/ai-context-maintainer/templates/platform-overview.md` 模板填充占位）。
3. 在 `platforms/{端名}/modules/{模块名}/` 下创建 5 个文件：`requirement.md`、`api.md`、`data-model.md`、`ui-spec.md`、`test-notes.md`。
4. **所有文件必须使用 `.claude/skills/ai-context-maintainer/templates/` 中的对应模板**：
   - 读取 `.claude/skills/ai-context-maintainer/templates/requirement-template.md` 填充至 `requirement.md`
   - 读取 `.claude/skills/ai-context-maintainer/templates/api-template.md` 填充至 `api.md`
   - 读取 `.claude/skills/ai-context-maintainer/templates/data-model-template.md` 填充至 `data-model.md`
   - 读取 `.claude/skills/ai-context-maintainer/templates/ui-spec-template.md` 填充至 `ui-spec.md`
   - 读取 `.claude/skills/ai-context-maintainer/templates/test-notes-template.md` 填充至 `test-notes.md`
5. 填充时将模板中的 `{平台名}`、`{模块名}` 替换为实际值，并移除不相关的示例内容。
6. 创建完成后询问用户是否立即填充具体内容。

#### 3.2 更新文件（多端感知）

根据变更描述，定位受影响的文件。必须考虑**多端联动**：

- **需求/接口变更**（仅影响一端）：
  - 直接更新 `platforms/{端名}/modules/{模块}/` 下的对应文件。
  - 新增条目按模板格式追加；修改条目在末尾追加 `（更新于 YYYY-MM-DD）`。

- **共享规则变更**（影响多端）：
  - 若修改 `shared/` 或 `cross-platform/` 中的内容，应主动提醒用户："此变更将影响所有引用该规则的端，建议同步检查以下模块：..."
  - 列出可能受影响的端和模块（可根据关键字搜索 `platforms/*/modules/` 下的文件确定）。

- **新增跨端流程**：
  - 在 `cross-platform/` 下新建或更新文件，并注明涉及哪些端的哪些模块。

- **历史缺陷补充**：
  - 明确缺陷发生在哪个端。若非通用缺陷，只记录在对应端的 `test-notes.md` 中。
  - 若为公共组件/接口缺陷，应在受影响的各端 `test-notes.md` 中都添加，或记录在 `shared/common-defects.md`（需创建）中。

- **所有更新同步**：
  - 重大变更必须追加到根目录 `changelog.md`：`[YYYY-MM-DD] [{端名或 shared}] {模块} - {变更摘要}`

#### 3.3 完整性检查（多端）

扫描 `platforms/` 下所有一级子目录（即所有端），并检查每个端 `modules/` 下的所有模块是否包含必要文件。

输出格式按端分组：
```
### pc-client

| 模块 | requirement.md | api.md | data-model.md | ui-spec.md | test-notes.md |
| ---- | -------------- | ------ | ------------- | ---------- | ------------- |
| user | ✅             | ✅     | ❌            | ✅         | ✅            |

### web-admin

...
```

若某端本身缺少 `platform-overview.md`，也应在报告中指出。
对缺失的文件，询问是否自动创建（使用模板填充）。

#### 3.4 规范性校验（多端扩展）

除校验各模块文件的章节完整度外（各文件要求见下），增加多端专项检查：

- **requirement.md** 至少需包含 `## 用户故事`、`## 验收标准`、`## 业务规则` 三个章节，且验收标准需以 `- [ ]` 开头的列表项。
- **api.md** 中每个接口必须清晰包含：路径、请求方法、请求参数表格（字段/类型/必填/说明）、响应示例、错误码表格。
- **data-model.md** 需描述主要实体、字段含义及状态流转。
- **ui-spec.md** 需包含页面跳转逻辑、前端校验规则、异常状态展示说明。
- **test-notes.md** 若暂无历史缺陷，需明确写入"暂无记录"或保留占位说明，避免空白。
- **平台级文件检查**：各端 `platform-overview.md` 不应为空白。
- **共享文件检查**：`shared/` 中的关键文件（如 `common-business-rules.md`）不应只有占位符。
- **跨平台引用检查**：若某模块的需求明确提及"同 pc-client 逻辑"但未给出具体内容，应提示补全或确认引用是否有效。

输出不符合项的具体建议，例如：
> `web-admin/refund/api.md` 中"退款接口"缺少错误码表格，建议补充常见错误码如 400、403。

#### 3.5 全局初始化（多端骨架）

1. 创建 `ai-context/` 及其所有顶级文件（README.md, glossary.md, system-overview.md, roles-permissions.md, changelog.md）并填充基础占位。
2. 创建 `shared/` 下所有文件（common-business-rules.md、common-data-model.md、common-api.md、common-components.md、env-config.md），使用初始占位。
3. 创建 `cross-platform/` 目录，并放入 `README.md` 说明用途。
4. 创建 `platforms/pc-client/` 默认端及其 `platform-overview.md`（使用 `.claude/skills/ai-context-maintainer/templates/platform-overview.md` 模板填充占位）。
5. 创建 `templates/` 目录（存放用例模板 test-case-template.md、bug-report-template.md，非 Skill 模板）。
6. 将"AI 测试知识库维护规范"写入项目根目录的 `CLAUDE.md`，使 AI 在本项目中持续遵循知识库维护约束：
   - 规范内容：
     ```
     ## AI 测试知识库维护规范

     - 知识库存放于 `ai-context/`，结构遵循 `ai-context/README.md` 的描述。
     - 任何功能变更必须同步更新对应端模块的需求和接口文件。
     - 文件格式必须符合 `.claude/skills/ai-context-maintainer/templates/` 中定义的模板。
     - 在完成更新后，应执行完整性检查。
     ```
   - 若 `CLAUDE.md` 不存在，创建该文件并写入规范。
   - 若 `CLAUDE.md` 已存在，检查是否已包含 `## AI 测试知识库维护规范` 章节：未包含则在文件末尾追加；已包含则跳过，避免重复。
7. 完成后提示："多端知识库骨架已就绪（已预创建 pc-client 端）。请使用'创建新模块'命令为具体端添加业务模块。"

#### 3.6 提炼共享内容（辅助功能）

当用户说"把这个规则提升为共享规则"时：

1. 确认规则内容及适用的端。
2. 将规则写入 `shared/` 下最相关的文件（如 `common-business-rules.md`）。
3. 原端文件中的该规则替换为引用："（见 `shared/common-business-rules.md` 规则 #X）"。

#### 3.7 按提取报告写入（与 ai-context-ingest 协作）

接收 `ai-context-ingest` 产出的提取报告（格式见 `.claude/skills/ai-context-ingest/references/report-schema.md`），据此写入知识库：

1. **核对报告**：确认目标端/模块、写入计划；端/模块不存在时先按 §3.1 创建（含 `platform-overview.md`）。
2. **写前闸门（铁律 B）**：复核报告中高危内容是否逐项溯源、有无编造填充、推测是否隔离；不通过项退回 ingest 或标 `待补充`。
3. **按目标文件分组写入**：依据报告"一、提取内容"的 requirement/api/data-model/ui-spec/test-notes 分组，使用 `.claude/skills/ai-context-maintainer/templates/` 对应模板填充；保留报告中的来源标注 `（源：...）`。
4. **冲突处理**：报告"三、冲突/待确认清单"中的项按 `（未证实）` 写入或置于"待确认"区，必要时先向用户确认，不裁决。
5. **完整性检查**：写入后按 §3.3 检查该端/模块文件齐全度。
6. **更新 changelog**：追加 `[YYYY-MM-DD] [{端名}] {模块} - 按 {源文件名} 提取报告写入`。

> 本 skill 是 `ai-context/` 的唯一写入方；ingest 不写文件。两 skill 协作链路：用户提供文档 → `/ai-context-ingest` 解析产报告 → `/ai-context-maintainer` 按报告写入。

### 4. 模板使用规范

- 本 Skill 依赖的**文件模板**存放于 `.claude/skills/ai-context-maintainer/templates/`，共 6 个：
  - `requirement-template.md`
  - `api-template.md`
  - `data-model-template.md`
  - `ui-spec-template.md`
  - `test-notes-template.md`
  - `platform-overview.md`
- 所有新建文件必须严格读取对应模板，保持格式统一。
- 若用户需要自定义模板，请引导其直接修改上述文件。

### 5. 行为约束

- **端隔离原则**：默认情况下，对一个端的修改不应自动修改其他端，除非明确指示或共享层变更。
- **共享层谨慎修改**：修改 `shared/` 或 `cross-platform/` 前，必须提示影响范围。
- **只基于用户提供的信息修改**，绝不编造业务规则或接口细节。
- **保持格式一致**：与现有 Markdown 风格、表格结构完全对齐。
- **非破坏性编辑**：不得删除或覆盖用户未明确指示修改的内容。
- **每次修改后输出摘要**，格式为：
  ```
  ✅ 已更新 {文件路径}
  变更：

  - {变更点1}
    ⚠️ 影响范围提醒：{如有}
    📌 请提交到版本控制。
  ```
- **冲突处理**：若用户指令与现有内容矛盾，列出矛盾点并请求澄清，不要自行决断。

#### 5.1 防幻觉铁律（高危内容溯源 + 写前闸门）

为避免编造信息污染知识库，执行任何写入前须遵守以下两条铁律：

**铁律 A · 高危内容逐项溯源**

下列"高危内容"在写入时**必须逐项标注来源**，不得笼统标注、不得缺省、不得编造：

- API 错误码表（每个错误码）
- 字段类型与约束（每个字段）
- 状态枚举值（每个枚举项）
- 具体数值（百分比、超时、额度上限、手续费、阈值等）
- 测试数据与测试账号

有效来源仅限三类，标注格式：
- 文档：`（源：{文档名} §{章节}）`
- 代码：`（源：代码 {文件:行}）`
- 用户：`（源：用户提供 {YYYY-MM-DD}）`

某高危项缺来源时：**写 `待补充` 并注明"需提供什么"**，绝不以推断内容填充。

**铁律 B · 写前闸门**

任何知识库写入前，逐条自检，**全部通过方可写入**；任一不通过则该部分不写、转为待办输出给用户：

1. 本批涉及的高危内容（铁律 A 范围）是否每一项都挂了有效来源？
2. 有无高危项因缺来源而被编造填充？（应为 `待补充`）
3. 推测性内容是否已显式标注 `（未证实）` 或置于"待确认"区，未混入事实正文？

非高危内容维持现有行为约束（基于用户提供信息、不编造），不强制逐项溯源。

### 6. 示例交互

**用户**："在 web-admin 端创建一个 refund 模块。"
**执行**：
1. 检查 `platforms/web-admin/` 是否存在，否→创建并使用 `platform-overview.md` 模板填充占位。
2. 创建 `platforms/web-admin/modules/refund/` 下 5 个文件，使用对应模板。
3. 输出摘要及下一步建议。

**用户**："pc-client 和 web-merchant 的退款接口都增加了 `reason` 字段，更新一下。"
**执行**：
1. 更新 `platforms/pc-client/modules/refund/api.md` 和 `platforms/web-merchant/modules/refund/api.md`。
2. 若接口定义完全相同，提示："该接口可能在两个端完全一致，是否需要将其移至 `shared/common-api.md` 作为共享接口？"
3. 更新完成后输出摘要及影响范围。

**用户**："检查整个知识库是否完整。"
**执行**：
1. 扫描 `platforms/` 下所有端及模块，按端分组输出检查表格。
2. 同时检查 `shared/`、`cross-platform/` 下文件是否空占位。
3. 给出缺失清单。

---

## 注意事项

- 所有路径均相对于项目根目录。
- 本 Skill 仅维护 `ai-context/`，不直接修改业务代码。
- 若需使用 AI 生成测试用例，请配合 `CLAUDE.md` 或自定义命令，使用 `@platforms/{端}/modules/{模块}` 加载上下文。
