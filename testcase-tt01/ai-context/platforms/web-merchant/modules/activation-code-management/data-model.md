# 激活码管理 数据模型

> 主表 `zs_scrm_user_info_child`（激活码），绑定表 `zs_scrm_user_info_child_channel`，分组表 `zs_scrm_biz_group`(type=3)。
> **列类型来源**：SCRM迭代001_数据库设计文档.md §B1；**字段语义来源**：激活码管理模块.md §四。

## 实体关系

- `zs_scrm_biz_group`(type=3) 1---N `zs_scrm_user_info_child`：分组-激活码（源：§B1 / 模块文档 §三）
- `zs_scrm_user_info_child` N---M `zs_scrm_social_platform`：经 `zs_scrm_user_info_child_channel`（源：§B1.2 / 模块文档 §四.4）

## 核心实体字段

### zs_scrm_user_info_child (激活码表)

| 字段 | 类型 | 约束 | 说明 |
| ---- | ---- | ---- | ---- |
| id | BIGINT UNSIGNED | PK | 激活码 ID（源：§B1） |
| merchant_id | BIGINT UNSIGNED | NOT NULL | 所属商家 ID（源：§B1） |
| username | varchar(64) | UK | 激活码，12 位随机码（大小写字母+数字）（源：§B1 / §二.2） |
| remarks | VARCHAR(50) | | 备注，限 50 字（源：§B1 / §四.14） |
| group_id | BIGINT UNSIGNED | | 所属分组 ID -> biz_group(type=3)（源：§B1 / §四.2） |
| port_alloc_type | TINYINT | 默认 1 | 端口分配：1 动态 2 固定（源：§B1 / §四.1） |
| fixed_port_num | INT | | 固定分配端口数（>0）（源：§B1 / §四.1） |
| share_type | VARCHAR(50) | | 分享页面权限（源：§B1 / §四.3） |
| range_type | TINYINT | 默认 1 | 线索去重范围：1当前会话 2含底库 3账号下全部 4关联账号 5分组内 6指定激活码（源：§B1 / §四.5） |
| range_childs | JSON | | range_type=6 时指定去重激活码 ID 列表（源：§B1 / §四.5） |
| work_time_type | TINYINT | 默认 0 | 工作时间：0 暂不设置 1 固定时间（源：§B1 / §四.6） |
| work_time_start / work_time_end | VARCHAR(50) | | 固定工作时间起止（源：§B1 / §四.6） |
| enable_keyword_reply | TINYINT | 默认 0 | 关键词回复开关（源：§B1 / §四.7） |
| enable_welcome_reply | TINYINT | 默认 0 | 欢迎语回复开关（源：§B1 / §四.7） |
| multi_device_login | TINYINT | 默认 1 | 多设备登录：0 关 1 开（源：§B1 / §四.8） |
| chat_backup | TINYINT | 默认 1 | 聊天备份：0 关 1 开（源：§B1 / §四.9） |
| profile_share | TINYINT | 默认 0 | 客户画像共享：0 不共享 1 共享（源：§B1 / §四.10） |
| data_masking | TINYINT | 默认 0 | 数据脱敏：0 关 1 开（源：§B1 / §四.11） |
| reset_time | VARCHAR(16) | | 置零时间，默认 00:00（源：§B1 / §四.12） |
| dept_id | BIGINT UNSIGNED | | 数据权限-归属部门 ID（源：§B1 / §四.13） |
| last_login_time | DATETIME | | 最近登录时间（源：§B1 / §六.2） |
| status | int | | 0 停用 1 正常（源：§B1 / §五.2） |

### zs_scrm_user_info_child_channel (激活码-社交平台绑定)

- UK(child_id, channel_id)（源：§B1.2）

### zs_scrm_biz_group (分组，type=3)

- 字段：id / merchant_id / name / type / sort；分组名同商户不可重复，最长 20 字符（源：§B0 / 模块文档 §三.2）

## 状态流转

```
停用 --(启用)--> 启用
启用 --(停用：PC强制下线 / 回收端口 / 工单置禁用)--> 停用
```
（源：模块文档 §五.2）
