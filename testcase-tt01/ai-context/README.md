# AI 测试知识库

本目录是多端 AI 测试知识库的根目录，用于沉淀各端（PC 客户端、Web 管理端、Web 商户端等）的业务知识，供 AI 在生成测试用例、缺陷报告及回归分析时加载上下文。

## 目录结构

```
ai-context/
├── README.md                 # 本文件
├── glossary.md               # 全局术语表
├── system-overview.md        # 各端关系、架构简图
├── roles-permissions.md      # 全局角色-权限矩阵（标注各端适用范围）
├── changelog.md              # 全局变更日志
├── shared/                   # 所有端共用的铁律
│   ├── common-business-rules.md
│   ├── common-data-model.md
│   ├── common-api.md
│   ├── common-components.md
│   └── env-config.md
├── platforms/                # 按"端"划分
│   └── {platform-name}/
│       ├── platform-overview.md
│       └── modules/{module-name}/
│           ├── requirement.md
│           ├── api.md
│           ├── data-model.md
│           ├── ui-spec.md
│           └── test-notes.md
├── cross-platform/           # 跨端联动流程
│   └── README.md
└── templates/                # AI 输出格式模板
    ├── test-case-template.md
    └── bug-report-template.md
```

## 命名约定

- **端名**统一使用 kebab-case（如 `pc-client`、`web-admin`、`web-merchant`）。
- **模块名**同样使用 kebab-case（如 `user-management`、`order-detail`）。

## 使用方式

- 维护操作通过 `ai-context-maintainer` Skill 完成。
- 新建模块文件必须读取 `.claude/skills/ai-context-maintainer/templates/` 中的对应模板。
- 任何功能变更必须同步更新对应端模块的需求和接口文件。
- 完成更新后应执行完整性检查。
- 读取/查询上下文（确认数据库内容、为模块加载测试设计上下文）通过 `ai-context-query` Skill 完成；下游用例生成（`testcase-create` / `biz-test-analysis`）经它取上下文，不直接读 `TemporaryFile/` 原始文档。
