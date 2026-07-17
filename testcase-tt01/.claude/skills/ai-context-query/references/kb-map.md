# KB 地图（要找什么 → 去哪读）

> ai-context-query 的速查表。知识库是"端 × 模块 × 关注点 + shared 公共层"双维度矩阵，确认一个事实往往要跨 2-3 文件拼装。本表给出每种信息的**主源**与**补源**。

## 路径前缀

- 模块文件：`ai-context/platforms/{端}/modules/{模块}/{文件}.md`
- 公共层：`ai-context/shared/{文件}.md`
- 全局：`ai-context/{文件}.md`
- 原文档：`TemporaryFile/`（仅当 KB 不足时回查）

## 速查表

| 要找 | 主源 | 补源 | 备注 |
|---|---|---|---|
| 表名 → 模块/域/维护端 | `shared/common-data-model.md §二`（模块与表清单总览） | `system-overview.md` 三端架构 | §二给出 序号/域/模块/主要数据表/维护端 |
| 表的列类型/约束/DDL | `shared/common-data-model.md`（§二定位→§三 A域/§四 B域/§五 C域 对应小节） | 模块 `data-model.md`；原文档 `SCRM迭代001_数据库设计文档.md` | 仅 §四 B0/B1/B2/B3 等有字段表；B4/B5/B7/B8 多为概述，列类型靠模块或 `待补充` |
| **公共字段** | `shared/common-data-model.md §一.3` | —— | `merchant_id`/`user_id`、`create_time`/`update_time`、`create_user_id`/`update_user_id`、`remark`、`deleted_flag`、`status`。**模块文件常省略，确认字段必补** |
| 设计约定（金额/时间戳/枚举/命名） | `shared/common-data-model.md §一`（设计约定 1-9） | —— | 金额 `DECIMAL(12,4)`、秒级时间戳 `BIGINT`、枚举 `TINYINT`、命名 `snake_case`、表前缀 `zs_scrm_` |
| 字段业务语义/枚举值 | 模块 `data-model.md` | 模块 `requirement.md`；shared 对应小节 | 注意 UI 输入类型 ≠ DB 列类型（见下） |
| 状态流转 | 模块 `data-model.md`（状态流转节） | —— | ASCII 图，含非法跳转用于异常用例 |
| 业务规则（编号/适用端） | `shared/common-business-rules.md`（规则 #1~#17） | 模块 `requirement.md` 引用 | 每条带"适用端"；#13~#17 标 `待确认` |
| 接口路径/方法/参数/错误码 | 模块 `api.md` | `shared/common-api.md`；原文档成品接口 | 多数模块 api.md 大量 `待补充` |
| 前端校验规则 | 模块 `ui-spec.md`（前端校验规则表） | —— | 含字段/规则/触发时机/错误提示 |
| 页面跳转/联动 | 模块 `ui-spec.md` | `requirement.md` 验收标准 | 联动规则驱动跨端用例 |
| 二次确认/异常文案 | 模块 `ui-spec.md`（关键交互文案/异常状态展示） | —— | 用例预期值的精确文案 |
| 用户故事/验收标准 | 模块 `requirement.md`（验收标准 `- [ ]`） | `ui-spec.md` | 验收标准是覆盖清单分母的主要来源 |
| 已知缺陷/易错点 | 模块 `test-notes.md` | —— | 驱动异常/回归用例 |
| 回归测试重点 | 模块 `test-notes.md`（回归测试重点） | `common-business-rules.md` 跨端规则 | 已按场景分组 |
| 测试数据/测试账号 | 模块 `test-notes.md`（测试数据说明） | —— | 多为 `待补充`（高危） |
| 端定位/技术栈/交互范式 | `platforms/{端}/platform-overview.md` | `system-overview.md` | 端级上下文 |
| 三端关系/数据流 | `system-overview.md`（三端架构 + ASCII 图） | `platform-overview.md` | 共享同一 MySQL 库，A/B/C 域分工 |
| 角色-权限 | `roles-permissions.md` | 各端 platform-overview | 标注各端适用范围 |
| 术语 | `glossary.md` | —— | —— |
| 跨端联动流程 | `cross-platform/*.md` | `system-overview.md` 共享与跨端节 | 多数待补充 |

## 三个重要"别漏"

1. **公共字段别漏**：确认任何业务表字段时，除模块 `data-model.md` 的业务字段外，**必须并上 `shared/common-data-model.md §一.3` 的公共字段**（merchant_id/create_time/update_time/create_user_id/update_user_id/remark/deleted_flag/status），否则字段表不完整。
2. **业务规则别漏**：加载模块上下文时，`common-business-rules.md` 中"适用端"含本端的规则要**全部带上**（grep 规则编号），不只看模块 requirement 里显式引用的。
3. **UI 类型 ≠ 列类型**：`customer-profile-management` 等模块顶部带 ⚠️，其"类型"列是 UI 输入类型（下拉单选/文本/日期），DB 列类型标 `待补充`。**不可混用**。

## 双源字段写法范例

模块 `data-model.md` 的字段常挂双来源（列类型来自 shared、语义来自模块文档），读取时原样保留：

```
| username | varchar(64) | UK | 激活码，12 位随机码（大小写字母+数字）（源：§B1 / §二.2） |
```

- `§B1` → `shared/common-data-model.md` 的列类型来源
- `§二.2` → 模块文档的字段语义来源

（范例来自 `activation-code-management/data-model.md`）
