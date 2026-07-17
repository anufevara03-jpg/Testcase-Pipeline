# welcome-message-management 数据模型

## 分组表 (welcome_group)

| 字段 | 类型 | 是否必填 | 说明 | 约束 |
| :--- | :--- | :--- | :--- | :--- |
| **id** | bigint | 是 | 分组ID | 主键，自增 |
| **merchant_id** | bigint | 是 | 商户ID | 外键，关联商户表 |
| **group_name** | varchar(20) | 是 | 分组名称 | 最长20字符 |
| **is_default** | tinyint | 是 | 是否默认分组 | 0-否，1-是 |
| **created_at** | datetime | 是 | 创建时间 | |
| **updated_at** | datetime | 是 | 更新时间 | |

**约束说明：**

- 同一商户下分组名称唯一（group_name + merchant_id）
- 新增商户时自动生成默认分组，is_default=1
- 默认分组不允许编辑、删除

## 欢迎语表 (welcome_message)

| 字段 | 类型 | 是否必填 | 说明 | 约束 |
| :--- | :--- | :--- | :--- | :--- |
| **id** | bigint | 是 | 欢迎语ID | 主键，自增 |
| **merchant_id** | bigint | 是 | 商户ID | 外键，关联商户表 |
| **title** | varchar(20) | 是 | 欢迎语标题 | 最长20字符 |
| **group_id** | bigint | 是 | 分组ID | 外键，关联分组表 |
| **send_setting** | varchar(20) | 是 | 发送设置 | 枚举值：original-原文发送, translated-翻译后发送 |
| **delay_min** | int | 是 | 最小延时间隔（秒） | 范围：2-5 |
| **delay_max** | int | 是 | 最大延时间隔（秒） | 范围：2-5 |
| **scope_type** | varchar(20) | 是 | 适用范围类型 | 枚举值：all-全部激活码,部分-部分激活码 |
| **activation_codes** | json | 否 | 适用激活码列表 | 当scope_type=部分时必填 |
| **status** | varchar(20) | 是 | 状态 | 枚举值：enabled-启用, disabled-禁用 |
| **created_by** | bigint | 是 | 创建人 | 外键，关联员工表 |
| **created_at** | datetime | 是 | 创建时间 | |
| **updated_at** | datetime | 是 | 更新时间 | |

**约束说明：**

- delay_min ≤ delay_max，且都在2-5秒范围内
- activation_codes为JSON数组，存储激活码ID列表

## 欢迎语内容表 (welcome_message_content)

| 字段 | 类型 | 是否必填 | 说明 | 约束 |
| :--- | :--- | :--- | :--- | :--- |
| **id** | bigint | 是 | 内容ID | 主键，自增 |
| **message_id** | bigint | 是 | 欢迎语ID | 外键，关联欢迎语表 |
| **material_id** | bigint | 是 | 素材ID | 外键，关联话术素材库 |
| **material_type** | varchar(20) | 是 | 素材类型 | text-文本, image-图片 |
| **sort_order** | int | 是 | 排序序号 | 从小到大排序 |
| **created_at** | datetime | 是 | 创建时间 | |

**约束说明：**

- 一个欢迎语最多配置2条内容
- sort_order从1开始，按从小到大排序

## 状态枚举值

| 字段 | 枚举值 | 说明 |
| :--- | :--- | :--- |
| **send_setting** | original | 原文发送 |
| | translated | 翻译后发送 |
| **scope_type** | all | 全部激活码 |
| | 部分 | 部分激活码 |
| **status** | enabled | 启用 |
| | disabled | 禁用 |
| **material_type** | text | 文本 |
| | image | 图片 |

（源：临时文件/商户端模块说明/欢迎语管理模块.md §四、五）