# 快捷回复管理 数据模型

> 主表 `zs_scrm_reply_quick`（快捷回复），分组表 `zs_scrm_reply_quick_group`。
> **列类型来源**：SCRM迭代001_数据库设计文档.md §B7.3 / §B7.4；**字段语义来源**：快捷回复管理模块.md。
> 注：回复类（快捷/关键词/欢迎语）均带 `channel_id`，按社交平台维度配置；分组因带 channel_id 单独建表（源：DB 文档 §B7、shared/common-data-model.md §七.5）。

## 实体关系

- `zs_scrm_reply_quick_group` 1---N `zs_scrm_reply_quick`：分组-快捷回复（源：DB 文档 §B7.3 / §B7.4）
- 快捷回复引用话术素材库 `zs_scrm_material`（回复内容来源于素材，源：模块文档 §一.2、§五.1）

## 核心实体字段

### zs_scrm_reply_quick_group (快捷回复分组表)

| 字段 | 类型 | 约束 | 说明 | 来源 |
| ---- | ---- | ---- | ---- | ---- |
| id | BIGINT UNSIGNED | PK AUTO_INCREMENT | 分组 ID | DB §B7.3 |
| merchant_id | BIGINT UNSIGNED | NOT NULL | 商家 ID | DB §B7.3 |
| channel_id | BIGINT UNSIGNED | DEFAULT NULL | 适用平台 ID（成品 reply 分组带 channel_id） | DB §B7.3 |
| name | VARCHAR(64) | NOT NULL | 分组名；模块文档限制≤20 字符 | DB §B7.3 / 模块文档 §三.1 |
| sort | INT | DEFAULT 0 | 排序 | DB §B7.3 |
| create_time | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 | DB §B7.3 |
| deleted_flag | tinyint UNSIGNED | NOT NULL DEFAULT 0 | 删除状态 | DB §B7.3 |

### zs_scrm_reply_quick (快捷回复表)

| 字段 | 类型 | 约束 | 说明 | 来源 |
| ---- | ---- | ---- | ---- | ---- |
| id | BIGINT UNSIGNED | PK AUTO_INCREMENT | 快捷回复 ID | DB §B7.4 |
| merchant_id | BIGINT UNSIGNED | NOT NULL | 商家 ID | DB §B7.4 |
| group_id | BIGINT UNSIGNED | DEFAULT 0 | 分组 ID | DB §B7.4 |
| channel_id | BIGINT UNSIGNED | DEFAULT NULL | 适用平台 | DB §B7.4 |
| title | VARCHAR(128) | DEFAULT NULL | 标题/快捷指令；模块文档限制≤20 字符 | DB §B7.4 / 模块文档 §五.1 |
| content_type | TINYINT | DEFAULT 1 | 内容类型：1文本 2图片 3视频 4文件 | DB §B7.4 |
| content | TEXT | | 回复内容 | DB §B7.4 |
| file_url | VARCHAR(255) | DEFAULT NULL | 媒体 URL | DB §B7.4 |
| sort | INT | DEFAULT 0 | 排序 | DB §B7.4 |
| create_time | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 | DB §B7.4 |
| update_time | datetime | ON UPDATE CURRENT_TIMESTAMP | 更新时间 | DB §B7.4 |
| deleted_flag | tinyint UNSIGNED | NOT NULL DEFAULT 0 | 删除状态 | DB §B7.4 |

## 状态流转

- 模块文档定义 启用↔禁用（源：模块文档 §四.2）。
- **注意（未证实，C-Q1）**：DB `zs_scrm_reply_quick`（§B7.4）**无 status 列**（reply_keyword/reply_welcome 有 status，reply_quick 无）。启用/禁用状态的存储方式待确认。

## 冲突备注（未证实）

- **C-Q1（未证实，关键）**：启用/禁用状态——模块文档围绕启用/禁用构建（顶部指标"启用回复/禁用回复"、列表"状态"列、§四.2），但 DB `zs_scrm_reply_quick`（§B7.4）无 status 列。待确认状态如何存储 / 是否需新增字段。
- **C-Q2（未证实）**：多素材——模块文档"回复内容"支持选最多 5 项素材（§五.1），DB 为单 `content`+`file_url`+`content_type`（单素材，§B7.4）。待确认是否拆子表（参照 welcome 多内容设计）。
- **C-Q3（未证实）**：适用激活码——模块文档有"适用激活码（全部/部分）"（§五.1），DB `reply_quick` 无 activation_code 字段（仅 channel_id 平台，§B7.4）。待确认激活码范围如何存储。
- **C-Q4（未证实）**：内容类型枚举——模块文档"文本/图片"（源素材库 文字/图片/名片，§五.1）vs DB content_type"1文本 2图片 3视频 4文件"（§B7.4）。枚举不一致，待确认（延续 material-library C1 思路）。
- **C-Q5（未证实）**：分组唯一性——模块文档"当前商户下唯一"（§三.1）vs DB `reply_quick_group` 带 channel_id + KEY(user_id,channel_id)（§B7.3）暗示按平台维度。待确认唯一性是商户级还是平台级。
- **C-Q6（未证实，轻微）**：字符上限——模块文档标题/分组名≤20 字符（§五.1、§三.1）vs DB title VARCHAR(128) / name VARCHAR(64)（§B7.4/§B7.3）。DB 更宽松，UI 限制更紧（备注，非硬冲突）。
- **C-Q7（未证实，沿用）**：项目名 SCEM（模块文档标题）vs NexSCRM（知识库）。与 material-library C2、welcome-message 同类。
