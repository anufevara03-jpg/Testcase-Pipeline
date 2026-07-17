---
name: ai-context-maintainer
description: 维护 AI 测试知识库（ai-context/），包括创建模块文件、更新需求与接口文档、检查知识库完整性与规范性。当用户要求创建模块、更新测试知识库文档、检查知识库完整性或规范性时使用此技能。
---

# ai-context-maintainer

## 概述

维护 AI 测试知识库（`ai-context/`），覆盖模块文件创建、需求与接口文档更新、知识库完整性与规范性检查。适用场景：用户提及"更新测试知识库"、"创建模块上下文"、"检查知识库"等意图时。

## Instructions

### 1. 任务识别

根据用户的输入，准确判断要执行的任务类型：

- **创建新模块**：用户指定一个或多个新模块名称。
- **更新现有文件**：用户提供业务变更、接口变更或缺陷信息，要求更新对应模块的文档。
- **完整性检查**：用户要求检查知识库是否缺少必要文件。
- **规范性校验**：用户要求校验文件内容是否符合预设格式。
- **全局初始化**：用户需要为全新项目创建完整的 `ai-context/` 骨架。

若意图不明确，请先向用户确认具体任务。

### 2. 知识库结构约定

所有操作必须遵循以下目录结构：
```
ai-context/
├── README.md # 说明文件用途及使用指引
├── glossary.md # 业务术语表
├── system-overview.md # 系统级描述：子系统、交互、架构简图
├── roles-permissions.md # 角色权限矩阵（表格形式）
├── changelog.md # 全局变更日志
├── modules/ # 按业务模块划分
│ └── {module-name}/
│ ├── requirement.md # 用户故事、验收标准、业务规则
│ ├── api.md # 接口定义（路径、参数、返回值、错误码）
│ ├── data-model.md # 实体关系、字段、状态机
│ ├── ui-spec.md # 页面交互、校验、跳转、异常展示
│ └── test-notes.md # 历史缺陷、易错点、回归重点
├── shared/ # 跨模块公共信息
│ ├── common-api.md # 登录鉴权、通用错误码等
│ ├── common-components.md # 公共UI组件交互说明
│ └── env-config.md # 环境信息（脱敏）
└── templates/ # AI输出格式模板
├── test-case-template.md
└── bug-report-template.md

```

### 3. 核心任务操作流程

#### 3.1 创建新模块
当用户要求创建新模块时，按以下步骤执行：
1. 确认模块名称（kebab-case 风格，如 `user-management`）。
2. 在 `ai-context/modules/{module-name}/` 下创建全部 5 个文件：`requirement.md`、`api.md`、`data-model.md`、`ui-spec.md`、`test-notes.md`。
3. 使用 `templates/` 目录中的模板文件（requirement.md、api.md、data-model.md、ui-spec.md、test-notes.md）填充每个文件的初始占位内容，并将 `{模块名}` 替换为实际名称。
4. 同时询问用户：是否需要立即填充模块的具体需求或接口信息。如果是，则引导用户提供并执行更新。

#### 3.2 更新文件
根据用户提供的变更描述，精准定位需要修改的文件和章节。

- **需求变更**（新增/修改验收标准、业务规则）：
  - 定位 `ai-context/modules/{module}/requirement.md`。
  - 若为新增：在对应章节（如"验收标准"）按模板格式追加条目。
  - 若为修改：直接替换旧描述，并在条目末尾追加 `（更新于 YYYY-MM-DD）`。
  - 重大变更必须在 `changelog.md` 中添加一条记录：`[YYYY-MM-DD] {模块} - {变更摘要}`。

- **接口变更**（参数、返回值、错误码等）：
  - 定位 `ai-context/modules/{module}/api.md`。
  - 修改对应接口的表格或代码块，保持字段完整。
  - 在文件顶部的"变更记录"区域（若无则创建）添加一条：`[YYYY-MM-DD]：修改了 {接口名} 的 {字段/逻辑}`。

- **历史缺陷补充**：
  - 定位 `ai-context/modules/{module}/test-notes.md`。
  - 在"已知缺陷与易错点"列表下，按格式添加：`- [YYYY-MM-DD] **{标题}**：{描述}。影响范围：{范围}`。

- **跨模块影响更新**：
  - 如果用户说明某个变更影响多个模块，主动列出可能受影响的文件，并请用户确认是否需要逐一更新。

#### 3.3 完整性检查
扫描 `ai-context/modules/` 下的所有一级子目录，确认每个模块都包含以下 5 个文件：
- requirement.md
- api.md
- data-model.md
- ui-spec.md
- test-notes.md

检查结果以表格呈现：
| 模块 | requirement.md | api.md | data-model.md | ui-spec.md | test-notes.md |
|------|----------------|--------|---------------|------------|---------------|
| user | ✅ | ✅ | ❌ | ✅ | ✅ |

对缺失的文件，询问用户："是否需要自动创建缺失的文件（使用模板填充占位符）？" 得到肯定答复后立即创建。

#### 3.4 规范性校验
对指定模块（或所有模块）逐文件检查内容完整性：

- **requirement.md** 至少需要包含 `## 用户故事`、`## 验收标准`、`## 业务规则` 三个章节，且验收标准需以 `- [ ]` 开头的列表项。
- **api.md** 中每个接口必须清晰包含：路径、请求方法、请求参数表格（字段/类型/必填/说明）、响应示例、错误码表格。
- **data-model.md** 需描述主要实体、字段含义及状态流转（可用表格或文字描述）。
- **ui-spec.md** 需包含页面跳转逻辑、前端校验规则、异常状态展示说明。
- **test-notes.md** 若暂无历史缺陷，需明确写入"暂无记录"或保留占位说明，避免文件空白。

输出不符合项的具体建议，例如：
> `api.md` 中"登录接口"缺少错误码表格，建议补充常见错误码如 401、403。

#### 3.5 全局初始化
如果用户要求为项目初始化整个 `ai-context/` 结构：
1. 创建所有顶层文件（README.md, glossary.md, system-overview.md, roles-permissions.md, changelog.md）并用模板填充。
2. 创建 `modules/`, `shared/`, `templates/` 目录及对应占位文件（例如 `shared/common-api.md` 写入通用鉴权说明的占位符）。
3. 将"AI 测试知识库维护规范"写入项目根目录的 `CLAUDE.md`，使 AI 在本项目中持续遵循知识库维护约束：
   - 规范内容使用本 Skill 的 `templates/claudemd.md`。
   - 若 `CLAUDE.md` 不存在，创建该文件并写入规范。
   - 若 `CLAUDE.md` 已存在，检查是否已包含 `## AI 测试知识库维护规范` 章节：未包含则在文件末尾追加；已包含则跳过，避免重复。
4. 完成后提示用户："基础结构已创建，你可以通过'创建新模块'命令开始添加具体业务模块。"

### 4. 行为约束
- **只基于用户提供的信息修改**，绝不凭空编造业务规则、接口参数或缺陷。
- **保持现有格式一致**：更新内容时，必须与文件原有的 Markdown 层级、表格样式、列表符号保持一致。
- **非破坏性编辑**：不得删除或覆盖用户未明确指示修改的内容。
- **每次修改后，输出摘要**，格式为：
```
✅ 已更新 {文件路径}
变更内容：

- {变更点1}
- {变更点2}
  ⚠️ 请记得提交到版本控制。
```
- **处理冲突与模糊**：若用户的描述与现有文件冲突，先指出矛盾点，请求澄清，不要自行决断。

### 5. 示例交互
用户："把用户注册的密码长度限制从6位改为8位，并加一条历史bug：弱密码可通过。"
你应该：
1. 更新 `ai-context/modules/user/requirement.md` 中关于密码的验收标准，并标注更新日期。
2. 如果 `api.md` 中有相关约束，同步修改。
3. 在 `test-notes.md` 中添加：`- [2026-07-17] **弱密码可通过**：前端未校验密码复杂度，仅校验长度，导致"123456"可注册。影响范围：注册、修改密码。`
4. 在 `changelog.md` 追加变更记录。
5. 输出修改摘要。

---

## Templates

模板文件存放于 `templates/` 目录，用于填充新建文件。创建时务必替换 `{模块名}` 为实际名称，删除不需要的示例内容。

| 模板文件 | 用途 | 目标位置 |
|----------|------|----------|
| `requirement.md` | 模块需求说明 | `ai-context/modules/{module-name}/requirement.md` |
| `api.md` | 模块接口定义 | `ai-context/modules/{module-name}/api.md` |
| `data-model.md` | 模块数据模型 | `ai-context/modules/{module-name}/data-model.md` |
| `ui-spec.md` | 模块 UI 交互说明 | `ai-context/modules/{module-name}/ui-spec.md` |
| `test-notes.md` | 模块测试笔记 | `ai-context/modules/{module-name}/test-notes.md` |
| `claudemd.md` | CLAUDE.md 维护规范 | 项目根目录 `CLAUDE.md`（全局初始化时） |

## 注意事项

- 所有路径均相对于项目根目录。
- 本 Skill 仅负责 `ai-context/` 内的内容维护，不修改代码或测试脚本。
- 如果用户需要将知识库用于 AI 生成测试用例，请引导用户使用 `CLAUDE.md` 中定义的规则，或配合自定义 Slash Command 使用。
