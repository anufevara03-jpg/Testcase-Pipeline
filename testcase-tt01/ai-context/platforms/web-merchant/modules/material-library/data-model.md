# 话术素材库 数据模型

> 主表 `zs_scrm_material`（素材），分类表 `zs_scrm_material_type`，分组表 `zs_scrm_biz_group`(type=6)。
> **列类型来源**：SCRM迭代001_数据库设计文档.md §B7 / §B0；**字段语义来源**：话术素材库模块.md。

## 实体关系

- `zs_scrm_biz_group`(type=6 素材) 1---N `zs_scrm_material`：分组-素材（源：DB 文档 §B0 / 模块文档 §三）
- `zs_scrm_material_type` 1---N `zs_scrm_material`：分类-素材（源：DB 文档 §B7）

## 核心实体字段

### zs_scrm_material (素材表)

| 字段 | 类型 | 约束 | 说明 | 来源 |
| ---- | ---- | ---- | ---- | ---- |
| id | BIGINT UNSIGNED | PK | 素材 ID（源：DB 文档 §B7） |
| merchant_id | BIGINT UNSIGNED | NOT NULL | 商家 ID（源：DB 文档 §B7） |
| type_id | BIGINT UNSIGNED | | 素材分类 ID（源：DB 文档 §B7） |
| name | VARCHAR(128) | | 素材名称；图片默认取文件名，≤200 字符（源：DB 文档 §B7 / 模块文档 §五.2） |
| material_type | TINYINT | | 素材类型：文字/图片/名片（源：模块文档 §一）；数字编码待补充 |
| content | TEXT | | 文本内容，≤2000 字符（源：DB 文档 §B7 / 模块文档 §五.1） |
| file_url | VARCHAR(255) | | 媒体 URL（源：DB 文档 §B7） |
| file_size | BIGINT | | 文件大小；图片单张≤10MB（源：DB 文档 §B7 / 模块文档 §五.2） |

### zs_scrm_material_type (素材分类表)

| 字段 | 类型 | 约束 | 说明 | 来源 |
| ---- | ---- | ---- | ---- | ---- |
| id | BIGINT UNSIGNED | PK | （源：DB 文档 §B7） |
| merchant_id | BIGINT UNSIGNED | NOT NULL | 商家 ID（源：DB 文档 §B7） |
| name | VARCHAR(64) | | 分类名（源：DB 文档 §B7） |
| type | TINYINT | | 素材大类：0全部/未分组 1图片 2视频 3文件 4文本（源：DB 文档 §B7） |

### zs_scrm_biz_group (分组，type=6)

- 字段：id / merchant_id / name / type / sort；分组名同类型（文本/图片）不可重复，最长 20 字符（源：DB 文档 §B0 / 模块文档 §三.2）

## 状态流转

- 本模块无复杂状态流转；素材为新增/编辑/删除生命周期。

## 冲突备注（未证实）

- **C1（已确认）**：素材类型枚举——模块文档为「文字/图片/名片」（源：模块文档 §一），DB 文档 `material_type` 为「1文本 2图片 3语音 4视频 5文件」（源：DB 文档 §B7）。**经用户确认按模块文档三类为准**；DB 五类暂不采用，数字编码待补充。
- **C3（未证实）**：分组模型——模块文档分组按素材类型(文本/图片)分别管理（源：模块文档 §三.2）vs DB 文档 `biz_group`(type=6) 统一素材分组（源：DB 文档 §B0）。
