# 关键词回复管理 数据模型

> 主表 `zs_scrm_reply_keyword`（关键词回复），分组表 `zs_scrm_reply_keyword_group`。
> **列类型来源**：SCRM迭代001_数据库设计文档.md §B7.5 / §B7.6；**字段语义来源**：关键词回复管理.md。
> 注：回复类（快捷/关键词/欢迎语）均带 `channel_id`，按社交平台维度配置；分组因带 channel_id 单独建表（源：DB 文档 §B7、shared/common-data-model.md §七.5）。

## 实体关系

- `zs_scrm_reply_keyword_group` 1---N `zs_scrm_reply_keyword`：分组-关键词回复（源：DB 文档 §B7.5 / §B7.6）
- 关键词回复引用话术素材库 `zs_scrm_material`（回复内容来源于素材，源：模块文档 §一.2、§五.1）

## 核心实体字段

### zs_scrm_reply_keyword_group (关键词回复分组表)

| 字段 | 类型 | 约束 | 说明 | 来源 |
| ---- | ---- | ---- | ---- | ---- |
| id | BIGINT UNSIGNED | PK AUTO_INCREMENT | 分组 ID | DB §B7.5 |
| merchant_id | BIGINT UNSIGNED | NOT NULL | 商家 ID | DB §B7.5 |
| channel_id | BIGINT UNSIGNED | DEFAULT NULL | 适用平台 ID（成品 reply 分组带 channel_id） | DB §B7.5 |
| name | VARCHAR(64) | NOT NULL | 分组名；模块文档限制≤20 字符 | DB §B7.5 / 模块文档 §三.1 |
| sort | INT | DEFAULT 0 | 排序 | DB §B7.5 |
| create_time | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 | DB §B7.5 |
| deleted_flag | tinyint UNSIGNED | NOT NULL DEFAULT 0 | 删除状态 | DB §B7.5 |

### zs_scrm_reply_keyword (关键词回复表)

| 字段 | 类型 | 约束 | 说明 | 来源 |
| ---- | ---- | ---- | ---- | ---- |
| id | BIGINT UNSIGNED | PK AUTO_INCREMENT | 关键词回复 ID | DB §B7.6 |
| merchant_id | BIGINT UNSIGNED | NOT NULL | 商家 ID | DB §B7.6 |
| group_id | BIGINT UNSIGNED | DEFAULT 0 | 分组 ID | DB §B7.6 |
| channel_id | BIGINT UNSIGNED | DEFAULT NULL | 适用平台 | DB §B7.6 |
| keyword | VARCHAR(255) | NOT NULL | 关键词（多个用分隔符）；模块文档限制单个≤20 字符 | DB §B7.6 / 模块文档 §五.1 |
| match_type | TINYINT | DEFAULT 1 | 匹配方式：1精确 2模糊/包含 | DB §B7.6 |
| content_type | TINYINT | DEFAULT 1 | 回复内容类型：1文本 2图片 3视频 4文件 | DB §B7.6 |
| content | TEXT | | 回复内容 | DB §B7.6 |
| file_url | VARCHAR(255) | DEFAULT NULL | 媒体 URL | DB §B7.6 |
| status | int | NULL DEFAULT NULL | 状态：0停用 1启用 | DB §B7.6 |
| sort | INT | DEFAULT 0 | 排序 | DB §B7.6 |
| create_time | datetime | DEFAULT CURRENT_TIMESTAMP | 创建时间 | DB §B7.6 |
| update_time | datetime | ON UPDATE CURRENT_TIMESTAMP | 更新时间 | DB §B7.6 |
| deleted_flag | tinyint UNSIGNED | NOT NULL DEFAULT 0 | 删除状态 | DB §B7.6 |

## 状态流转

- 模块文档定义 启用↔禁用（源：模块文档 §四.2）。
- DB `zs_scrm_reply_keyword` 含 `status`（0停用 1启用，源：DB §B7.6），与模块文档启用/禁用语义对应（此点与 quick-reply 的 `reply_quick` 无 status 列不同，关键词回复启用/禁用存储无冲突）。

## 冲突备注（未证实）

- **C-K1（未证实，关键）**：匹配规则枚举命名——模块文档"全匹配/半匹配"（§五.1）vs DB `match_type` "1精确 2模糊/包含"（§B7.6）。语义可能对应（精确=全匹配，模糊=半匹配），待确认对应关系。
- **C-K2（未证实，部分澄清）**：发送设置——模块文档有"发送设置：原文发送/翻译后发送"（§五.1），DB `zs_scrm_reply_keyword`（§B7.6）**无 send_setting 字段**。用户澄清：「翻译后发送」通过请求三方翻译接口将原文翻译成译文再发送（源：用户提供 2026-07-17）；但 send_setting 字段的 DB 存储方案（新增/复用）仍未确认。
- **C-K3（未证实，关键）**：延时间隔——模块文档"范围滑块，默认 2s-5s，自定义 1s-180s"（§五.1），DB（§B7.6）**无 delay 字段**（reply_welcome 有 delay_second，reply_keyword 无）。待确认如何存储。
- **C-K4（未证实，关键）**：回复方式——模块文档"回复全部/随机回复一条"（§五.1），DB（§B7.6）**无 reply_mode 字段**。待确认如何存储。
- **C-K5（未证实，关键）**：适用激活码——模块文档"全部/部分激活码 + 激活码级联"（§五.1），DB `reply_keyword` 无 activation_code 字段（仅 channel_id 平台，§B7.6）。待确认激活码范围如何存储（同 quick-reply-management C-Q3）。
- **C-K6（未证实）**：多素材——模块文档"自动回复内容"支持选最多 5 项素材（§五.1），DB 为单 `content`+`file_url`+`content_type`（单素材，§B7.6）。待确认是否拆子表（参照 welcome 多内容设计）。
- **C-K7（未证实，轻微）**：关键词长度——模块文档关键词≤20 字符（§五.1）vs DB `keyword VARCHAR(255)`（§B7.6）。DB 更宽松，UI 限制更紧（备注，非硬冲突）。
- **C-K8（未证实，轻微）**：分组名长度——模块文档分组名≤20 字符（§三.1）vs DB `name VARCHAR(64)`（§B7.5）。同 C-K7 性质。
- **C-K9（未证实）**：分组唯一性——模块文档"当前商户下唯一"（§三.1）vs DB `reply_keyword_group` 带 channel_id + KEY(user_id,channel_id)（§B7.5）暗示按平台维度。待确认唯一性是商户级还是平台级（同 quick-reply-management C-Q5）。
- **C-K10（未证实，沿用）**：内容类型枚举——模块文档素材来源（文字/图片/名片，来自素材库，§五.1）vs DB content_type"1文本 2图片 3视频 4文件"（§B7.6）。沿用 material-library C1 / quick-reply-management C-Q4 思路。
- **C-K11（未证实，沿用）**：项目名 SCEM（模块文档标题）vs NexSCRM（知识库）。与 material-library C2、quick-reply-management C-Q7 同类。
