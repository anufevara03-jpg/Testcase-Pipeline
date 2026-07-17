# 共享数据模型

> 全平台统一数据库设计。来源：`TemporaryFile/SCRM迭代001_数据库设计文档.md`（V1.0.0，对应飞书《SCRM迭代001》V1.0.2，参照成品 ChatKnow）。
> 数据库：MySQL 8.0（InnoDB / utf8mb4 / utf8mb4_general_ci）。
> 三端共用同一库：管理后台（web-admin）维护 A 域、商家后台（web-merchant）读写 B 域、PC 客户端（pc-client）经接口支撑表 C 域。

## 一、设计约定

1. **存储引擎 / 字符集**：统一 `InnoDB` + `utf8mb4` + `utf8mb4_general_ci`，支持 emoji 与多语言。
2. **主键**：统一 `id BIGINT UNSIGNED AUTO_INCREMENT`；分布式场景可替换为雪花 ID（保持 BIGINT）。
3. **公共字段**（绝大多数业务表都包含）：
   - `merchant_id` / `user_id BIGINT UNSIGNED` —— 所属商家 ID（**租户隔离字段**，几乎所有商家域数据都带，且建索引）。注：DB 文档中部分表写作 `user_id`，语义同 `merchant_id`。
   - `create_time datetime DEFAULT CURRENT_TIMESTAMP` / `update_time datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`。
   - `create_user_id bigint` / `update_user_id bigint` —— 创建人 / 更新人（商家员工 ID）。
   - `remark VARCHAR(255)` —— 备注。
   - `deleted_flag tinyint UNSIGNED NOT NULL DEFAULT 0` —— 逻辑删除（0 正常 / 1 删除）。
   - `status int DEFAULT NULL` —— 通用状态位（基础语义 0-关闭 / 1-开启，具体语义见各表注释）。
4. **金额**：统一 `DECIMAL(12,4)`（计费单价精度高，4 位小数）；余额 `DECIMAL(12,2)` / `DECIMAL(18,2)`。
5. **时间戳**：业务需"秒级时间戳"（如套餐到期）用 `BIGINT`；展示时间用 `DATETIME`。
6. **枚举**：用 `TINYINT` + 注释；多选/可变集合用 `JSON` 或关联表。
7. **命名**：表名、字段名一律 `snake_case`（如 `user_info`、`user_info_child`、`user_info_channel`）。表前缀 `zs_scrm_`。
8. **索引命名**：主键 `PRIMARY`；唯一索引 `uk_字段`；普通索引 `idx_字段`。
9. **租户隔离**：商家域所有查询强制带 `user_id`/`merchant_id`，相关索引均以之为最左前缀。

## 二、模块与表清单总览

| 序号 | 域 | 模块 | 主要数据表 | 维护端 |
| --- | --- | --- | --- | --- |
| A1 | A 平台管理 | 三方聊天平台管理 | `zs_scrm_social_platform` | web-admin |
| A2 | A | 套餐管理 / 付款记录 | `zs_scrm_setmeal_info`、`zs_scrm_setmeal_specs`、`zs_scrm_setmeal_order`、`zs_scrm_merchant_setmeal`、`account_product` | web-admin |
| A3 | A | 翻译线路(=接口服务商)/计费/语种 | `zs_scrm_translate_line`、`zs_scrm_translate_billing_rule`、`zs_scrm_translate_record`、`zs_scrm_translate_language` | web-admin |
| A4 | A | 商家管理 | `zs_scrm_merchant_info`、`user_agent_info` | web-admin |
| A5 | A | CMS：banner / 联系我们 / 意见反馈 / 公告 | `zs_scrm_banner`、`zs_scrm_contact_us`、`zs_scrm_feedback`、`sys_notice` | web-admin |
| B0 | B 商家后台 | 通用分组 | `zs_scrm_biz_group`（type 区分：1标签 3激活码 5ws 6素材） | web-merchant |
| B1 | B | 激活码（分组/列表） | `zs_scrm_user_info_child`、`zs_scrm_user_info_child_channel`、`zs_scrm_user_info_child_dedup` | web-merchant |
| B2 | B | 账号 / 设备管理 | `zs_scrm_device_user_rel`(设备)、`zs_scrm_channel_account`(三方账号) | web-merchant |
| B3 | B | 代理设置 | `zs_scrm_proxy_ip`、`ws_account` | web-merchant |
| B4 | B | 客户管理 | `zs_scrm_customer`、`zs_scrm_customer_profile_config`、`zs_scrm_customer_customer_label`、`zs_scrm_customer_label_rel`、`customer_inherit` | web-merchant |
| B5 | B | 线索 / 消息 | `base_contact`、`sync_fans`、`zs_scrm_customer_message` | web-merchant |
| B6 | B | 社群管理 | `community_group`、`group_work_order`、`group_label` | web-merchant |
| B7 | B | 运营管理 | `zs_scrm_material(_type)`、`zs_scrm_reply_quick(_group)`、`zs_scrm_reply_keyword(_group)`、`zs_scrm_reply_welcome(_group)` | web-merchant |
| B8 | B | 组织与权限 | `company_dept`、`company_role`、`company_staff`、`company_menu`、`company_role_menu`、`company_staff_role` | web-merchant |
| C | C 客户端接口 | 客户端接口支撑 | `client_login_log`、`zs_scrm_device_port`、`api_key`、`open_api_log` | pc-client |

> 备注：`base_contact`/`sync_fans`/`community_group` 等为支撑模块运转的业务表；**分流链接（`shunt_link`）、平台工单（`work_order`）本期暂不实现，已从设计中移除。**

## 三、A 域核心实体（管理后台维护）

### A1. 三方聊天平台 `zs_scrm_social_platform`
全平台公共字典，由管理后台维护，客户端通过接口拉取「图标、名称、编码」。

| 字段 | 类型 | 约束 | 说明 |
| ---- | ---- | ---- | ---- |
| id | BIGINT UNSIGNED | PK | 平台 ID |
| name | VARCHAR(64) | NOT NULL | 平台名称，如 WhatsApp / Telegram |
| code | VARCHAR(64) | NOT NULL, UK | 平台编码（客户端识别用，如 whatsapp） |
| icon | VARCHAR(255) | | 平台图标 URL |
| link | VARCHAR(255) | | 平台链接 URL |
| sort | INT | | 排序，越小越靠前 |
| status | int | | 0禁用 1启用 |

> 成品接口 `/user/UserInfoChannel/list` 返回 19 个平台（WhatsApp / Telegram / Line / Instagram / Messenger / Facebook / Twitter / Zalo / TikTok / Teams / Snapchat 等）。

### A2. 套餐管理

#### A2.1 套餐主体 `zs_scrm_setmeal_info`

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| id | BIGINT UNSIGNED PK | |
| name | VARCHAR(64) | 套餐名称（如 专业版） |
| type | VARCHAR(64) | 套餐档位：1基础 2专业 3旗舰…（对应菜单可见性 type0~type4） |
| bill_model | TINYINT | 计费模式：1按时长 2按字符 |
| child_total | INT | 可生成激活码数量额度 |
| port_total | INT | 端口数（同时在线账号数） |
| infinite_port | TINYINT | 是否不限端口：0否 1是 |
| is_free | TINYINT | 是否免费套餐 |
| enable_translate | TINYINT | 是否含翻译功能 |
| setmeal_desc | text | 套餐权益 |
| status | int | 0下架 1上架 |

#### A2.2 套餐售卖规格 `zs_scrm_setmeal_specs`（一套餐多档「时长+价格」SKU）

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| id | BIGINT UNSIGNED PK | |
| setmeal_id | BIGINT UNSIGNED | 所属套餐主体 ID |
| duration | INT | 时长数量（配合 duration_unit；字符版留空） |
| duration_unit | TINYINT | 1时 2天 3月 4年 |
| total_seconds | BIGINT | 折算总秒数（字符版为 0） |
| translate_char_total | BIGINT | 含翻译字符额度（字符版必填） |
| gift_char_total | BIGINT | 本规格固定赠送字符数 |
| child_total | INT | 本规格可创建激活码数 |
| price | DECIMAL(12,2) | 售价 |
| original_price | DECIMAL(12,2) | 原价/划线价 |
| status | int | 0下架 1上架 |

#### A2.3 付款记录统一表 `zs_scrm_setmeal_order`
一张表存所有付款记录，用 `order_category` 区分两个 Tab：**1充值套餐** / **2购买账号**。

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| id / order_no | BIGINT / VARCHAR(64) UK | 订单号 |
| merchant_id | BIGINT UNSIGNED | 下单商家 ID |
| order_category | TINYINT | 1充值套餐 2购买账号 |
| product_name | VARCHAR(128) | 展示名 |
| amount / pay_amount | DECIMAL(12,2) | 订单金额 / 实付 |
| currency | VARCHAR(8) | 默认 USD |
| pay_channel | VARCHAR(32) | Alipay/WechatPay/Plisio/Balance/Corporate |
| pay_type | TINYINT | 1支付宝 2微信 3余额 4对公 5加密货币 |
| pay_state | TINYINT | 0待支付 1已支付 2已取消 3已退款 |
| pay_time | DATETIME | 支付时间 |
| setmeal_id / specs_id | BIGINT | [充值套餐] 套餐/规格 ID |
| child_total / port_total / total_seconds | | [充值套餐] 额度快照 |
| order_type | TINYINT | [充值套餐] 1新购 2续费 3增加端口 4升级 |
| product_id / channel_id / quantity | | [购买账号] 商品/平台/数量 |
| deliver_state | TINYINT | [购买账号] 0待发货 1已发货 2部分 3失败 4已退款 |
| deliver_time / deliver_content | | [购买账号] 发货时间/内容（加密） |

#### A2.4 商家在用套餐 `zs_scrm_merchant_setmeal`（= 成品 use-setmeal）
记录商家当前生效套餐及余量；支持暂停 / 到期。

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| id | BIGINT UNSIGNED PK | |
| merchant_id | BIGINT UNSIGNED | 商家 ID |
| order_id | BIGINT | 来源付款记录 ID |
| setmeal_id / specs_id | BIGINT | 套餐主体/规格 ID |
| setmeal_name / setmeal_type | | 套餐快照（type 控制菜单可见性） |
| child_total / child_count | INT | 激活码额度总量 / 已生成数 |
| port_total / infinite_port | INT | 端口总数 / 不限端口标记 |
| total_seconds / surplus_seconds | BIGINT | 总时长 / 剩余时长（秒） |
| translate_char_total / translate_char_surplus | BIGINT | 翻译字符额度总量 / 剩余 |
| start_time / end_time | DATETIME | 生效 / 到期 |
| is_free | TINYINT | 是否免费套餐 |
| pause_state | TINYINT | 0正常 1已暂停 |
| pause_start_time | DATETIME | 暂停开始时间 |
| status | int | 0失效 1生效 |

### A3. 翻译线路 / 计费 / 语种（本迭代新增）

#### A3.1 翻译线路 `zs_scrm_translate_line`（= 接口服务商，三合一）
**「翻译线路」与「翻译接口/服务商」是同一概念**。线路分组（标准线路 / 专享线路）走**字典表**，本表只存 `group_code`。

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| id | BIGINT UNSIGNED PK | 线路 ID |
| name | VARCHAR(64) | 线路/服务商名称（百度/有道/Google/DeepL） |
| code | VARCHAR(64) UK | 线路编码（baidu/youdao，语种与翻译记录据此关联） |
| group_code | VARCHAR(32) | 线路分组字典 code（标准/专享） |
| api_url | VARCHAR(255) | 接口地址 |
| app_id / app_secret | VARCHAR | 接入 AppID/账号 / 密钥（加密存储） |
| status | int | 0停用 1启用 |

#### A3.2 翻译计费规则 `zs_scrm_translate_billing_rule`

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| id | BIGINT UNSIGNED PK | |
| group_code | VARCHAR(32) | 适用线路分组字典 code（null=全局） |
| line_code | VARCHAR(64) | 适用具体线路 code（null=按分组生效） |
| setmeal_id | BIGINT | 适用套餐（null=不限） |
| bill_mode | TINYINT | 1按字符 2按条数 3按Token 4包月 |
| unit_price | DECIMAL(12,4) | 单价 |
| min_charge | DECIMAL(12,4) | 单次最低消费 |
| free_quota | BIGINT | 赠送免费额度 |
| currency | VARCHAR(8) | 默认 CNY |
| status / effective_time | | 0停用 1启用 / 生效时间 |

#### A3.3 翻译记录 / 流水 `zs_scrm_translate_record`
客户端消息翻译落库 + 计费依据 + 对账明细。数据量大，建议按月分表/归档。

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| id | BIGINT UNSIGNED PK | |
| merchant_id / child_id | BIGINT | 商家 ID / 激活码 ID |
| message_id | BIGINT | 关联消息 ID |
| line_id | BIGINT | 使用的翻译线路 ID |
| source_lang / target_lang | VARCHAR(16) | 源/目标语种 |
| source_text / target_text | TEXT | 原文 / 译文 |
| char_count | INT | 字符数 |
| bill_mode / unit_price | | 计费方式/单价（快照） |
| cost | DECIMAL(12,4) | 本次费用 |
| direction | TINYINT | 1收到翻译 2发送翻译 |
| status | int | 0失败 1成功 |

#### A3.4 翻译语种配置 `zs_scrm_translate_language`
一条线路对应一套语种配置（line 1:N language）。`code` 直接存该线路服务商自己的语种代码（百度 `jp` / 有道 `ja`）。新增线路只需以新 `line_id` 灌一套语种，**无需改表结构**。

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| id | BIGINT UNSIGNED PK | |
| line_id | BIGINT UNSIGNED | 所属翻译线路 ID |
| code | VARCHAR(32) | 该线路服务商语种代码（auto=自动检测） |
| name | VARCHAR(64) | 语种中文名（如 日语） |
| is_auto | TINYINT | 是否"自动检测"伪语种 |
| support_source / support_target | TINYINT | 是否支持作为源/目标语言 |
| sort / status | | 排序 / 0关闭 1开启 |

> 唯一约束 `uk_line_code(line_id, code)` 保证同一线路下语种代码不重复。语种列表以百度、有道官方文档为准导入。

### A4. 商家（租户）`zs_scrm_merchant_info`

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| merchant_id | bigint PK | 主键 |
| setmeal_id | BIGINT UNSIGNED | 所属套餐主体 ID |
| setmeal_name | VARCHAR(20) | 套餐名称 |
| expiration_time | datetime | 到期时间 |
| balance | DECIMAL(18,2) | 余额 |
| status | int | 状态 |
| signature | VARCHAR(255) | 签名 |
| optimistic | int | 版本号（乐观锁） |

> 成品 `userinfo` 字段参考：`email、nickname、username、avatar、payState、state、type、balance、is_master、timezone、userAgentInfoId、remarks、createTime`。代理商 `user_agent_info` 与商家 1:N。

### A5. CMS：banner / 联系我们 / 意见反馈

- `zs_scrm_banner`：position（1商家后台首页 2登录页 3客户端）、sort、status、start_time/end_time。
- `zs_scrm_contact_us`：type（1客服 2电话 3邮箱 4Telegram 5WhatsApp 6二维码）、content、qr_code。
- `zs_scrm_feedback`：问题类型（1文字翻译 2图片翻译 3其他）、问题描述(限200字)、支持平台 channel_id、联系方式、images(JSON，最多4张≤2M)、handle_state（0待处理 1处理中 2已处理 3已关闭）、reply。

## 四、B 域核心实体（商家后台）

### B0. 通用分组 `zs_scrm_biz_group`
统一分组表 + `type` 区分（实测：type=3 激活码、1 标签、5 ws、6 素材）。字段：`id, merchant_id, name, type, sort`。各回复/关键词分组因带 `channel_id` 单独建表。

### B1. 激活码 `zs_scrm_user_info_child`
**激活码是授权与配置中心**，一条记录承载约 20 项业务开关。关键字段：

| 字段 | 说明 |
| ---- | ---- |
| id / merchant_id | 激活码 ID / 商家 ID |
| username | 激活码（12 位，如 5lVFY6B1UfUw，UK） |
| remarks | 备注（限 50 字） |
| group_id | 所属分组 ID -> biz_group(type=3) |
| port_alloc_type | 端口分配：1动态 2固定 |
| fixed_port_num | 固定分配端口数 |
| share_type | 分享页面权限 |
| range_type | 线索去重范围：1当前会话 2含底库 3账号下全部 4关联账号 5分组内 6指定激活码 |
| range_childs | range_type=6 时指定去重激活码 ID 列表(JSON) |
| work_time_type / work_time_start / work_time_end | 工作时间 |
| enable_keyword_reply / enable_welcome_reply | 自动回复开关 |
| multi_device_login / chat_backup / profile_share / data_masking | 多设备登录/聊天备份/画像共享/数据脱敏 |
| reset_time | 置零时间（每日清零时刻） |
| dept_id | 数据权限-归属部门 ID |
| status | 0停用 1正常 |

#### B1.2 激活码-社交平台绑定 `zs_scrm_user_info_child_channel`（多对多，UK(child_id, channel_id)）

### B2. 账号 / 设备管理

- `zs_scrm_device_user_rel`（设备）：`device_id, device_type, user_id, first_login, last_login, login_count`。UK(device_id, user_id)。
- `zs_scrm_channel_account`（三方账号）：挂在会话 `device_port` 上，**不单独占端口**。字段含 `child_id, client_user_id, device_port_id, channel_id, account_no, nickname, avatar, port_no, proxy_id, ws_account_id, online_state, auth_state(0未授权1已授权2已失效)`。

### B3. 代理设置
`zs_scrm_proxy_ip`：`protocol(1HTTP/2HTTPS/3SOCKS5), host, port, account, password(加密), country, check_state(0未检1可用2不可用)`。`ws_account`（ws 协议号）。

### B4. 客户管理
- `zs_scrm_customer`：含基础信息 + 销售信息（customer_level/sale_phase/source）+ 公司信息 + `custom_fields(JSON)`。
- `zs_scrm_customer_profile_config`：每商家一份，定义字段分类/顺序/隐藏/自定义（contactInfo/salesInfo/companyInfo）。
- `zs_scrm_customer_customer_label` + `zs_scrm_customer_label_rel`：标签 + 客户-标签关联。
- `customer_inherit`：客户继承。

### B5. 线索 / 消息
- `base_contact`：去重底库，按 `(merchant_id, channel_id, platform_uid)` 唯一。
- `sync_fans`：同步粉丝。
- `zs_scrm_customer_message`：聊天消息表。字段含 `direction(1接收2发送), msg_type(1文本2图片3语音4视频5文件6位置7名片), content, translate_content, translate_state, send_state, msg_time`。UK(channel_account_id, platform_msg_id) 幂等去重。**聊天备份(激活码 chat_backup)关闭时不落正文**。

### B7. 运营管理
素材（`zs_scrm_material_type` / `zs_scrm_material`）、快捷回复（`zs_scrm_reply_quick(_group)`）、关键词回复（`zs_scrm_reply_keyword(_group)`）、欢迎语（`zs_scrm_reply_welcome(_group)`）。**回复类均带 `channel_id`，即按平台维度配置**。

### B8. 组织与权限
`company_dept / company_role / company_staff / company_menu / company_role_menu / company_staff_role`。菜单可见性由 `company_menu.type0~type4` × `zs_scrm_merchant_setmeal.setmeal_type` 控制。

## 五、C 域：客户端接口支撑

### `zs_scrm_device_port`（会话/挂机位 = 端口占用单位）

| 字段 | 说明 |
| ---- | ---- |
| id / merchant_id / child_id | 会话 ID / 商家 / 激活码 |
| client_user_id | 所属设备 ID（一设备可开多会话） |
| port_no | 端口序号 |
| channel_id | 计划/已登录平台 ID（开会话时可空） |
| channel_account_id | 已登录三方账号 ID（NULL=空会话） |
| occupy_state | 1占用(会话开启) 0已释放(会话关闭) |
| account_state | 0未登录(空会话) 1已登录 2登出/掉线 |
| open_time / close_time | 会话开启/关闭时间 |

> 另有 `client_login_log`、`api_key`、`open_api_log`。

## 六、核心实体关系（ER 概览）

```
user_agent_info (代理商)
      │1:N
zs_scrm_merchant_info (商家) ─1:N─ zs_scrm_setmeal_order (付款记录:充值套餐+购买账号) ─N:1─ zs_scrm_setmeal_info ─1:N─ zs_scrm_setmeal_specs
   │  │                                  └─N:1─ account_product (购买账号商品)
   │  └──1:1── zs_scrm_merchant_setmeal (在用套餐)        type0~4 控制菜单可见
   │                                              company_menu (菜单)
   ├──1:N── company_dept / company_role / company_staff ──N:M── company_menu
   │
   ├──1:N── zs_scrm_user_info_child (激活码) ──N:M── zs_scrm_social_platform (三方平台)
   │             │                              （经 user_info_child_channel）
   │             ├──1:N── device_user_rel (设备) ──1:N── device_port (会话/挂机位=占端口) ──0..1── channel_account (三方账号)
   │             │                                        │   ├──N:1── proxy_ip
   │             │                                        │   └──N:1── ws_account
   │             ├──1:N── customer_message (消息) ──1:1── translate_record
   │             ├──N:1── customer (客户) ──N:M── customer_label
   │             └──1:N── community_group (社群) ──1:N── group_work_order
   │
   ├──1:N── base_contact / sync_fans / customer_inherit
   └──1:N── material / reply_quick / reply_keyword / reply_welcome (均可带 channel_id)

zs_scrm_translate_line (翻译线路=接口服务商, 分组走字典 group_code:标准/专享) ──1:N── translate_language
zs_scrm_translate_billing_rule ──(计费)── translate_record
```

## 七、关键设计说明

1. **多租户隔离**：所有商家域表均含 `user_id`/`merchant_id`，索引以之为最左前缀；查询层强制注入，杜绝越权。
2. **激活码是授权与配置中心**：`user_info_child` 承载约 20 项业务开关，是客户端登录后所有行为的"配置下发源"；建议这些开关同时缓存 Redis，登录时随 token 下发。
3. **端口限制（按会话计，非按账号）**：详见 [[common-business-rules]] 规则 #3/#4。
4. **去重底库**：`base_contact` 按 `(user_id, channel_id, platform_uid)` 唯一；新进线索按激活码 `range_type` 决定比对范围。
5. **统一分组 `biz_group` + type**：源于成品「各类 group-list 同构」；回复类分组因绑定 `channel_id` 单独建表。
6. **消息与翻译大表**：`message`、`translate_record`、`sys_oper_log` 高写入，建议按 `user_id` 或月份分表/分区，冷数据归档。
7. **翻译计费闭环**：`translate_line` → 产生 `translate_record`(含 cost) → 按 `translate_billing_rule` 结算到 `merchant_info.balance` 或套餐额度。
8. **菜单套餐可见性**：`company_menu.type0~type4` × `zs_scrm_merchant_setmeal.setmeal_type`，登录时计算可见菜单。
9. **金额/时长**：套餐时长用秒（BIGINT）便于精确扣减与暂停续期；计费金额 `DECIMAL(12,4)`。
10. **逻辑删除与审计**：统一 `deleted_flag`，关键表保留 `create_user_id/update_user_id`，配合 `sys_oper_log` 满足审计与内控。
