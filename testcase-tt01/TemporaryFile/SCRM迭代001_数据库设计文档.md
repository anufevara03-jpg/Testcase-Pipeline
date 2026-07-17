# SCRM 系统数据库设计文档（SCRM 迭代 001）

> 版本：V1.0.0　|　对应需求：飞书《SCRM迭代001》V1.0.2
参照成品：ChatKnow（https://user.chatknow.com）
数据库：MySQL 8.0（InnoDB / utf8mb4）

***

## 一、文档说明

### 1.1 业务背景

本系统是一套面向**多账号、多三方聊天平台（WhatsApp / Telegram / Line / Instagram / Messenger / Facebook / Twitter / Zalo / TikTok / Teams / Snapchat 等 19 个平台）的社交化客户关系管理（SCRM）平台**，并在本迭代中新增**消息翻译**能力。系统分为三端：

| 端 | 使用者 | 职责 |
| --- | --- | --- |
| **管理后台（平台运营端）** | 平台运营 / 管理员 | 三方平台管理、套餐管理、翻译线路与计费配置、商家管理、商家套餐分配、banner / 联系我们 / 意见反馈 |
| **商家后台** | 商家（租户）及其员工 | 激活码管理、账号 / 设备管理、客户 / 线索管理、社群管理、运营物料、代理 IP、组织与权限 |
| **客户端接口（PC 挂机端）** | 商家分发的激活码登录端 | 激活码登录、设备信息同步、三方平台信息查询、账号登录授权与端口限制、客户 / 消息同步、客户画像 |

核心授权链路：**商家购买套餐 → 套餐包含「激活码额度 + 端口数」→ 商家生成激活码（含各类聊天/去重/翻译配置）→ 客户端用激活码登录 → 在激活码下登录多个三方平台账号（占用端口）→ 与客户聊天产生线索 → 消息可走翻译线路翻译并计费。**

### 1.2 设计约定

1. **存储引擎 / 字符集**：统一 `InnoDB` + `utf8mb4` + `utf8mb4_general_ci`，支持 emoji 与多语言。
2. **主键**：统一 `id BIGINT UNSIGNED AUTO_INCREMENT`；分布式场景可替换为雪花 ID（保持 BIGINT）。
3. **公共字段**（绝大多数业务表都包含）：

    - `user_id BIGINT UNSIGNED` —— 所属商家 ID（租户隔离字段，几乎所有商家域数据都带，且建索引）。
    - `create_time datetime NULL DEFAULT CURRENT_TIMESTAMP` / `update_time datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` —— 创建 / 更新时间。
    - `create_user_id bigint` / `update_user_id bigint` —— 创建人 / 更新人（商家员工 ID）。
    - `remark VARCHAR(255)` —— 备注。
    - `deleted_flag tinyint UNSIGNED NOT NULL DEFAULT 0` —— 删除状态（0 正常 / 1 删除，逻辑删除）。
    - `status int NULL DEFAULT NULL` —— 通用状态位（基础语义 0-关闭 / 1-开启，具体语义见各表注释）。

4. **金额**：统一 `DECIMAL(12,4)`（计费单价精度高，故 4 位小数）；余额 `DECIMAL(12,2)`。
5. **时间戳**：业务需要"秒级时间戳"的场景（如套餐到期）用 `BIGINT`（参照成品），展示时间用 `DATETIME`。
6. **枚举**：用 `TINYINT` + 注释；多选/可变集合用 `JSON` 或关联表。
7. **命名**：表名、字段名一律 `snake_case`，与成品保持一致（如 `user_info`、`user_info_child`、`user_info_channel`）。
8. **索引命名**：主键 `PRIMARY`；唯一索引 `uk_字段`；普通索引 `idx_字段`。
9. **租户隔离**：商家域所有查询都强制带 `user_id`，相关索引均以 `user_id` 作为最左前缀。

### 1.3 模块与表清单总览

| 序号 | 模块 | 主要数据表 |
| --- | --- | --- |
| A1 | 三方聊天平台管理 | `user_info_channel` |
| A2 | 套餐管理 / 付款记录 | `zs_order_setmeal_info`、`zs_order_setmeal_specs`、`zs_merchant_payment_record`（付款记录:充值套餐+购买账号）、`zs_merchant_setmeal`（=use-setmeal）、`account_product` |
| A3 | 翻译线路(=接口服务商) / 计费 / 语种 | `translate_line`、`translate_billing_rule`、`translate_record`、`translate_language` |
| A4 | 商家管理 | `user_info`、`user_agent_info` |
| A5 | banner / 联系我们 / 意见反馈 | `sys_banner`、`sys_contact_us`、`sys_feedback`、`sys_notice` |
| B1 | 激活码（分组 / 列表） | `biz_group`、`user_info_child`、`user_info_child_channel`、`user_info_child_dedup` |
| B2 | 账号 / 设备管理 | `client_user`(设备)、`channel_account`(三方账号) |
| B3 | 代理设置 | `proxy_ip`、`ws_account` |
| B4 | 客户管理 | `customer`、`customer_profile_config`、`customer_label`、`customer_label_rel`、`customer_inherit` |
| B5 | 线索 / 消息 | `base_contact`、`sync_fans`、`message` |
| B6 | 社群管理 | `community_group`、`group_work_order`、`group_label` |
| B7 | 运营管理 | `material`、`material_type`、`reply_quick`、`reply_keyword`、`reply_welcome`（含各自分组表） |
| B8 | 组织与权限 | `company_dept`、`company_role`、`company_staff`、`company_menu`、`company_role_menu`、`company_staff_role` |
| C | 客户端接口支撑 | `client_login_log`、`device_port`、`api_key`、`open_api_log` |

***

## 二、A 域：平台管理后台

### A1. 三方聊天平台管理 —— `zs_scrm_social_platform`

> 需求：【管理后台】三方聊天平台管理；【客户端接口】三方聊天平台信息查询（图标、名称、编码）。
成品接口：`/user/UserInfoChannel/list`（返回 19 个平台）。

```sql
CREATE TABLE `zs_scrm_social_platform` (
  `id`             BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键，平台ID',
  `name`           VARCHAR(64)  NOT NULL COMMENT '平台名称，如 WhatsApp / Telegram',
  `code`           VARCHAR(64)  NOT NULL COMMENT '平台编码，客户端用于识别，如 whatsapp',
  `icon`           VARCHAR(255) DEFAULT NULL COMMENT '平台图标URL',
  `link`           VARCHAR(255) DEFAULT NULL COMMENT '平台链接URL',
  `sort`           INT DEFAULT 0 COMMENT '排序，越小越靠前',
  `status`          int NULL DEFAULT NULL COMMENT '状态：0禁用 1启用',
  `create_user_id` bigint NULL DEFAULT NULL COMMENT '创建人',
  `update_user_id` bigint NULL DEFAULT NULL COMMENT '更新人',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='三方聊天平台';
```

> 说明：该表为全平台公共字典，由管理后台维护，客户端通过接口拉取「图标、名称、编码」。

### A2. 套餐管理

> 需求：【管理后台】套餐管理、商家套餐配置。
成品接口：`/userInfoCore/use-setmeal` 揭示了套餐字段（`order_setmeal_info_id/name/type`、`setmeal_total_quantity`、`child_total`、`infinite_port`、`specs_id`、`is_free`、`pause_state` 等）。

#### A2.1 套餐定义 zs_scrm`_setmeal_info`

```sql
CREATE TABLE `zs_scrm_setmeal_info` (
  `id`            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `name`          VARCHAR(64) NOT NULL COMMENT '套餐名称（中文），如 专业版',
  `type`          VARCHAR(64) NOT NULL COMMENT '套餐档位：1基础 2专业 3旗舰 …（对应菜单可见性 type0~type4）',
  `bill_model`    TINYINT NOT NULL DEFAULT 1 COMMENT '计费模式：1按时长(标准/专业版) 2按字符(字符版)',
  `child_total`   INT DEFAULT 0 COMMENT '可生成激活码数量额度',
  `port_total`    INT DEFAULT 0 COMMENT '端口数（同时在线账号数）',
  `infinite_port` TINYINT DEFAULT 0 COMMENT '是否不限端口：0否 1是',
  `is_free`       TINYINT DEFAULT 0 COMMENT '是否免费套餐：0否 1是',
  `enable_translate` TINYINT DEFAULT 0 COMMENT '是否包含翻译功能：0否 1是',
  `setmeal_desc`          text DEFAULT  NULL COMMENT '套餐权益',
  `sort`          INT DEFAULT 0 COMMENT '排序',
  `status`         int NULL DEFAULT NULL COMMENT '状态：0下架 1上架',
  `remark`        VARCHAR(255) DEFAULT NULL COMMENT '套餐说明',
  `create_user_id` bigint NULL DEFAULT NULL COMMENT '创建人',
  `update_user_id` bigint NULL DEFAULT NULL COMMENT '更新人',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='套餐主体表';
```

#### A2.2 套餐规格（时长 / 价格）`zs_scrm_setmeal_specs`

> 一个套餐有多档「时长 + 价格」规格（成品的 `specs_id`、`setmeal_total_quantity` 秒数）。

```sql
CREATE TABLE `zs_scrm_setmeal_specs` (
  `id`             BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `setmeal_id`          BIGINT UNSIGNED NOT NULL COMMENT '所属(套餐主体)ID -> zs_sp.id',
  `duration`       INT DEFAULT NULL COMMENT '时长数量，配合 duration_unit（字符版留空）',
  `duration_unit`  TINYINT DEFAULT 3 COMMENT '时长单位：1时 2天 3月 4年',
  `total_seconds`  BIGINT DEFAULT 0 COMMENT '折算总秒数（=setmeal_total_quantity；字符版为0）',
  `translate_char_total` BIGINT DEFAULT 0 COMMENT '含翻译字符数额度（字符版必填；0=不含翻译字符）',
  `gift_char_total` BIGINT DEFAULT 0 COMMENT '本规格固定赠送字符数（0=无；阶梯满赠另见赠送规则）',
  `child_total`    INT DEFAULT 0 COMMENT '本规格可创建激活码数（字符版：每个字符包送的激活码数；0=回落到SP）',
  `price`          DECIMAL(12,2) NOT NULL COMMENT '售价',
  `original_price` DECIMAL(12,2) DEFAULT NULL COMMENT '原价/划线价',
  `status`          int NULL DEFAULT NULL COMMENT '状态：0下架 1上架',
  `sort`           INT DEFAULT 0,
  `create_user_id` bigint NULL DEFAULT NULL COMMENT '创建人',
  `update_user_id` bigint NULL DEFAULT NULL COMMENT '更新人',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_sp` (`sp_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='套餐售卖规格表（LP/SKU，时长/字符/价格）';
```

#### A2.3 （统一订单）`zs_scrm_setmeal_o`rder

> 对照成品「付款记录」页面：**一张表存所有付款记录**，用 `order_category` 区分两个 Tab——
**充值套餐**（订单号 / 充值套餐 / 订单金额 / 支付状态 / 支付时间 / 支付渠道）与 **购买账号**（多一列 **发货状态**）。前端按 `order_category` 分 Tab 查询；充值套餐专用字段（套餐/额度）与购买账号专用字段（商品/发货）按类别填充。

```sql
CREATE TABLE `zs_scrm_setmeal_order` (
  `id`            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `order_no`      VARCHAR(64) NOT NULL COMMENT '订单号',
  `merchant_id`   BIGINT UNSIGNED NOT NULL COMMENT '下单商家ID -> user_info.id',
  `order_category` TINYINT NOT NULL COMMENT '订单类别：1充值套餐 2购买账号',
  `product_name`  VARCHAR(128) DEFAULT NULL COMMENT '展示名（充值套餐名 / 购买账号商品名，对应列表"充值套餐"或"购买账号"列）',
  `amount`        DECIMAL(12,2) NOT NULL COMMENT '订单金额',
  `pay_amount`    DECIMAL(12,2) DEFAULT 0 COMMENT '实付金额',
  `currency`      VARCHAR(8) DEFAULT 'USD' COMMENT '币种（成品订单以 $ 计价）',
  `pay_channel`   VARCHAR(32) DEFAULT NULL COMMENT '支付渠道（对应"支付渠道"列）：Alipay支付宝 / WechatPay微信 / Plisio加密货币 / Balance余额 / Corporate对公',
  `pay_type`      TINYINT DEFAULT NULL COMMENT '支付方式（归一化）：1支付宝 2微信 3余额 4对公 5加密货币(Plisio) …',
  `pay_state`     TINYINT DEFAULT 0 COMMENT '支付状态：0待支付 1已支付 2已取消 3已退款',
  `pay_time`      DATETIME DEFAULT NULL COMMENT '支付时间',
  `setmeal_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '[充值套餐]套餐ID -> zs_order_setmeal_info.id',
  `specs_id`      BIGINT UNSIGNED DEFAULT NULL COMMENT '[充值套餐]套餐规格ID -> zs_order_setmeal_specs.id',
  `child_total`   INT DEFAULT 0 COMMENT '[充值套餐]本单含激活码额度（快照）',
  `port_total`    INT DEFAULT 0 COMMENT '[充值套餐]本单含端口数（快照）',
  `total_seconds` BIGINT DEFAULT 0 COMMENT '[充值套餐]本单时长（秒，快照）',
  `order_type`    TINYINT DEFAULT NULL COMMENT '[充值套餐]订单类型：1新购 2续费 3增加端口 4升级套餐',
  `product_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '[购买账号]账号商品ID -> account_product.id',
  `channel_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '[购买账号]账号所属平台ID -> user_info_channel.id',
  `quantity`      INT DEFAULT NULL COMMENT '[购买账号]购买数量',
  `deliver_state` TINYINT DEFAULT NULL COMMENT '[购买账号]发货状态：0待发货 1已发货 2部分发货 3发货失败 4已退款',
  `deliver_time`  DATETIME DEFAULT NULL COMMENT '[购买账号]发货时间',
  `deliver_content` TEXT DEFAULT NULL COMMENT '[购买账号]发货内容（交付的账号/凭证信息，加密存储）',
 `create_user_id` bigint NULL DEFAULT NULL COMMENT '创建人',
  `update_user_id` bigint NULL DEFAULT NULL COMMENT '更新人',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),
  KEY `idx_merchant` (`merchant_id`),
  KEY `idx_category` (`order_category`),
  KEY `idx_pay_state` (`pay_state`),
  KEY `idx_deliver_state` (`deliver_state`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='付款记录统一表（充值套餐+购买账号，order_category区分）';
```

#### A2.4 商家在用套餐 `zs_merchant_setmeal`（= 成品 use-setmeal）

> 记录商家当前生效的套餐及其余量；支持暂停（`pause_state`）、到期（`end_time`）。

```sql
CREATE TABLE `zs_scrm_merchant_setmeal` (
  `id`               BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
  `merchant_id`      BIGINT UNSIGNED NOT NULL COMMENT '商家ID（=user_info_id）',
  `order_id`         BIGINT UNSIGNED DEFAULT NULL COMMENT '来源付款记录ID -> zs_merchant_payment_record.id',
  `setmeal_id`            BIGINT UNSIGNED NOT NULL COMMENT '(套餐主体)ID -> zs_sp.id',
  `specs_id`            BIGINT UNSIGNED DEFAULT NULL COMMENT '(售卖规格)ID -> zs_lp.id',
  `setmeal_name`          VARCHAR(64) COMMENT '套餐名称快照',
  `setmeal_type`          TINYINT COMMENT '套餐档位快照（用于菜单可见性 type0~type4）',
  `child_total`      INT DEFAULT 0 COMMENT '激活码额度总量',
  `child_count`      INT DEFAULT 0 COMMENT '已生成激活码数',
  `port_total`       INT DEFAULT 0 COMMENT '端口总数',
  `infinite_port`    INT DEFAULT 0 COMMENT '不限端口标记/额外端口数',
  `total_seconds`    BIGINT DEFAULT 0 COMMENT '套餐总时长（秒）',
  `translate_char_total`   BIGINT DEFAULT 0 COMMENT '翻译字符数额度总量（基础+赠送合计，多笔充值累加）',
  `translate_char_surplus` BIGINT DEFAULT 0 COMMENT '剩余翻译字符数（按字符计费时扣减）',
  `surplus_seconds`  BIGINT DEFAULT 0 COMMENT '剩余时长（秒，=setmeal_surplus_quantity）',
  `start_time`       DATETIME DEFAULT NULL COMMENT '生效时间',
  `end_time`         DATETIME DEFAULT NULL COMMENT '到期时间',
  `is_free`          TINYINT DEFAULT 0 COMMENT '是否免费套餐',
  `pause_state`      TINYINT DEFAULT 0 COMMENT '暂停状态：0正常 1已暂停',
  `pause_start_time` DATETIME DEFAULT NULL COMMENT '暂停开始时间',
  `status`            int NULL DEFAULT NULL COMMENT '状态：0失效 1生效',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_merchant_id` (`merchant_id`),
  KEY `idx_end_time` (`end_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商家在用套餐（订阅）表';
```

### A3. 翻译线路 / 分组 / 计费 / 接口对接（本迭代新增）

> 需求：【管理后台】翻译线路配置，线路分组（标准线路 / 专享线路），翻译计费设置；翻译接口对接。
该能力为成品 ChatKnow 未覆盖的新增模块，按需求文本设计。

#### A3.1 翻译线路 `translate_line`（= 接口服务商，三合一）

> **「翻译线路」与「翻译接口/服务商」是同一个概念**，原 `translate_provider` / `translate_line_group` / `translate_line` 三张表合并为这一张：一行 = 一条可调用的翻译线路（即一个接口服务商）。
线路分组（标准线路 / 专享线路）后续统一用**字典表**维护，本表只存分组的字典编码 `group_code`，不再单独建分组表。

```sql
CREATE TABLE `zs_scrm_translate_line` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '线路ID',
  `name`        VARCHAR(64) NOT NULL COMMENT '线路/服务商名称，如 百度 / 有道 / Google / DeepL',
  `code`        varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NULL DEFAULT NULL COMMENT '线路/服务商编码，如 baidu / youdao（唯一，语种与翻译记录据此关联）',
  `group_code`  VARCHAR(32) DEFAULT NULL COMMENT '线路分组字典code（标准线路/专享线路，走字典表，本表只存code）',
  `api_url`     VARCHAR(255) DEFAULT NULL COMMENT '接口地址',
  `app_id`      VARCHAR(128) DEFAULT NULL COMMENT '接入AppID/账号',
  `app_secret`  VARCHAR(255) DEFAULT NULL COMMENT '密钥（加密存储）',
  `status`      int NULL DEFAULT NULL COMMENT '状态：0停用 1启用',
  `remark`      VARCHAR(255) DEFAULT NULL COMMENT '备注',
  `create_user_id` bigint NULL DEFAULT NULL COMMENT '创建人',
  `update_user_id` bigint NULL DEFAULT NULL COMMENT '更新人',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_code` (`code`),
  KEY `idx_group_code` (`group_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='翻译线路/接口服务商表（线路=服务商，分组走字典）';
```

#### A3.2 翻译计费规则 `translate_billing_rule`

```sql
CREATE TABLE `zs_scrm_translate_billing_rule` (
  `id`            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '计费规则ID',
  `group_code`    VARCHAR(32) DEFAULT NULL COMMENT '适用线路分组字典code（null=全局；标准/专享走字典）',
  `line_code`       varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin COMMENT '适用具体线路code（null=按分组生效）',
  `setmeal_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '适用套餐（null=不限）',
  `bill_mode`     TINYINT NOT NULL DEFAULT 1 COMMENT '计费方式：1按字符 2按条数 3按Token 4包月',
  `unit_price`    DECIMAL(12,4) NOT NULL COMMENT '单价（每字符/每条/每Token）',
  `min_charge`    DECIMAL(12,4) DEFAULT 0 COMMENT '单次最低消费',
  `free_quota`    BIGINT DEFAULT 0 COMMENT '赠送免费额度（字符/条）',
  `currency`      VARCHAR(8) DEFAULT 'CNY' COMMENT '币种',
  `status`         int NULL DEFAULT NULL COMMENT '状态：0停用 1启用',
  `effective_time` DATETIME DEFAULT NULL COMMENT '生效时间',
  `remark`        VARCHAR(255) DEFAULT NULL,
  `create_user_id` bigint NULL DEFAULT NULL COMMENT '创建人',
  `update_user_id` bigint NULL DEFAULT NULL COMMENT '更新人',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_group_code` (`group_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='翻译计费规则表';
```

#### A3.3 翻译记录 / 流水 `translate_record`

> 既用于客户端消息翻译落库，也作为计费依据与对账明细。数据量大，建议按月分表或归档。

```sql
CREATE TABLE `zs_scrm_translate_record` (
  `id`            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '翻译记录ID',
  `merchant_id`   BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `child_id`      BIGINT UNSIGNED DEFAULT NULL COMMENT '激活码ID',
  `message_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '关联消息ID',
  `line_id`       BIGINT UNSIGNED DEFAULT NULL COMMENT '使用的翻译线路ID（=接口服务商）-> translate_line.id',
  `source_lang`   VARCHAR(16) DEFAULT NULL COMMENT '源语种',
  `target_lang`   VARCHAR(16) DEFAULT NULL COMMENT '目标语种',
  `source_text`   TEXT COMMENT '原文',
  `target_text`   TEXT COMMENT '译文',
  `char_count`    INT DEFAULT 0 COMMENT '字符数',
  `bill_mode`     TINYINT DEFAULT 1 COMMENT '计费方式（快照）',
  `unit_price`    DECIMAL(12,4) DEFAULT 0 COMMENT '单价（快照）',
  `cost`          DECIMAL(12,4) DEFAULT 0 COMMENT '本次费用',
  `direction`     TINYINT DEFAULT 1 COMMENT '方向：1收到翻译 2发送翻译',
  `status`         int NULL DEFAULT NULL COMMENT '状态：0失败 1成功',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  KEY `idx_user_time` (`user_id`,`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='翻译记录/计费流水表';
```

#### A3.4 翻译语种配置 `translate_language`

> 需求补充：把各翻译线路（接口服务商）的「支持语种列表」单独建表配置。**一条翻译线路对应一套语种配置（line 1 : N language），用一张表 + `line_id` 区分。**
参照：百度翻译《通用文本翻译-语种列表》https://fanyi-api.baidu.com/doc/21 （文档入口 [https://fanyi-api.baidu.com/doc/23）、有道智云《文本翻译-支持语言》http://ai.youdao.com/DOCSIRMA/html/trans/api/wbfy/index.html](https://fanyi-api.baidu.com/doc/23%EF%BC%89%E3%80%81%E6%9C%89%E9%81%93%E6%99%BA%E4%BA%91%E3%80%8A%E6%96%87%E6%9C%AC%E7%BF%BB%E8%AF%91-%E6%94%AF%E6%8C%81%E8%AF%AD%E8%A8%80%E3%80%8Bhttp://ai.youdao.com/DOCSIRMA/html/trans/api/wbfy/index.html) 。
**设计要点**：同一种语言在不同线路（服务商）的「语种代码」并不一致（如 **日语**：百度 `jp` / 有道 `ja`；**简体中文**：百度 `zh` / 有道 `zh-CHS`；**韩语**：百度 `kor` / 有道 `ko`）。本表每行 = 「某线路 + 该线路的一种语言」，`code` 直接存该线路服务商自己的语种代码；新增线路只需以新的 `line_id` 再灌入一套语种数据，**无需改表结构**。

```sql
CREATE TABLE `zs_scrm_translate_language` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '语种配置ID',
  `line_id`     BIGINT UNSIGNED NOT NULL COMMENT '所属翻译线路ID（=接口服务商）-> translate_line.id',
  `code`        VARCHAR(32) NOT NULL COMMENT '该线路服务商的语种代码，如 百度jp / 有道ja（自动检测=auto）',
  `name`             VARCHAR(64) NOT NULL COMMENT '语种中文名，如 日语',
  `is_auto`     TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '是否“自动检测”伪语种：0否 1是',
  `support_source` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '是否支持作为源语言：0否 1是',
  `support_target` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '是否支持作为目标语言：0否 1是',
  `sort`        INT DEFAULT 0 COMMENT '排序',
  `status`      int NULL DEFAULT NULL COMMENT '状态：0关闭 1开启',
  `create_user_id` bigint NULL DEFAULT NULL COMMENT '创建人',
  `update_user_id` bigint NULL DEFAULT NULL COMMENT '更新人',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_line_code` (`line_id`,`code`),
  KEY `idx_line` (`line_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='翻译语种配置表（每条线路一套，line_id区分）';
```

> 说明：①`UNIQUE(line_id, code)` 保证同一线路下语种代码不重复；②业务侧（线路 `translate_line.support_langs`、翻译记录 `translate_record.source_lang/target_lang`）按「当前线路」选取本表对应行；③部分线路存在方向限制（如有道领域翻译仅中英双向），用 `support_source/support_target` 控制即可。

### A4. 商家管理

> 需求：【管理后台】商家管理。成品 `userinfo` 字段：`email、nickname、username、avatar、payState、state、type、balance、is_master、timezone、userAgentInfoId、remarks、createTime`。

#### A4.1 商家（租户）`zs_merchant_info`

```sql
CREATE TABLE `zs_merchant_info`  (
  `merchant_id` bigint NOT NULL COMMENT '主键',
  `setmeal_id`          BIGINT UNSIGNED NOT NULL COMMENT '所属套餐主体ID',
  `setmeal_name` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '套餐名称',
  `expiration_time` datetime NULL DEFAULT COMMENT '到期时间',
  `balance` decimal(18, 2) NOT NULL COMMENT '余额',
  `status` int NOT NULL COMMENT '状态',
  `signature` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '签名',
  `optimistic` int NOT NULL DEFAULT 0 COMMENT '版本号',
  `create_user_id` bigint NULL DEFAULT NULL COMMENT '创建人',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_user_id` bigint NULL DEFAULT NULL COMMENT '更新人',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`merchant_id`) USING BTREE,
  UNIQUE INDEX `idx_merchant_name`(`merchant_name` ASC) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = '商户管理' ROW_FORMAT = DYNAMIC;

SET FOREIGN_KEY_CHECKS = 1;
```

### A5. CMS：banner / 联系我们 / 意见反馈 / 公告

> 需求：【管理后台】banner、联系我们、意见反馈。成品首页有系统通知（`/user/home/notice`）。

```sql
-- A5.1 Banner
CREATE TABLE `zs_scrm_banner` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `title`       VARCHAR(128) DEFAULT NULL COMMENT '标题',
  `image_url`   VARCHAR(255) NOT NULL COMMENT '图片URL',
  `link_url`    VARCHAR(255) DEFAULT NULL COMMENT '跳转链接',
  `position`    TINYINT DEFAULT 1 COMMENT '展示位置：1商家后台首页 2登录页 3客户端',
  `sort`        INT DEFAULT 0 COMMENT '排序',
  `status`       int NULL DEFAULT NULL COMMENT '状态：0下线 1上线',
  `start_time`  DATETIME DEFAULT NULL COMMENT '展示开始时间',
  `end_time`    DATETIME DEFAULT NULL COMMENT '展示结束时间',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Banner轮播图表';

-- A5.2 联系我们
CREATE TABLE `zs_scrm_contact_us` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `type`        TINYINT DEFAULT 1 COMMENT '渠道类型：1客服 2电话 3邮箱 4Telegram 5WhatsApp 6二维码',
  `title`       VARCHAR(64) DEFAULT NULL COMMENT '名称',
  `content`     VARCHAR(255) DEFAULT NULL COMMENT '内容（号码/账号/链接）',
  `qr_code`     VARCHAR(255) DEFAULT NULL COMMENT '二维码图片URL',
  `sort`        INT DEFAULT 0,
  `status`       int NULL DEFAULT NULL COMMENT '状态：0隐藏 1显示',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='联系我们配置表';

-- A5.3 意见反馈（对照"提交反馈"表单：问题类型/问题描述/支持平台/联系方式/上传图片）
CREATE TABLE `zs_scrm_feedback` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_id`     BIGINT UNSIGNED NOT NULL COMMENT '反馈商家ID',
  `staff_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '反馈员工ID',
  `type`        TINYINT NOT NULL COMMENT '问题类型(必填)：1文字翻译 2图片翻译 3其他问题',
  `content`     VARCHAR(255) NOT NULL COMMENT '问题描述(必填，限200字)',
  `channel_id`  BIGINT UNSIGNED DEFAULT NULL COMMENT '支持平台(选填) -> user_info_channel.id，如 Telegram',
  `contact_type` TINYINT NOT NULL DEFAULT 1 COMMENT '联系方式类型(必填)：1 E-mail 2手机 3Telegram 4WhatsApp 5微信',
  `contact`     VARCHAR(128) NOT NULL COMMENT '联系方式值(必填)',
  `images`      JSON DEFAULT NULL COMMENT '上传图片URL数组（最多4张，单张≤2M，jpg/png/jpeg）',
  `handle_state` TINYINT DEFAULT 0 COMMENT '处理状态：0待处理 1处理中 2已处理 3已关闭',
  `reply`       TEXT DEFAULT NULL COMMENT '平台回复',
  `handle_by`   BIGINT UNSIGNED DEFAULT NULL COMMENT '处理人（平台）',
  `handle_time` DATETIME DEFAULT NULL,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user` (`user_id`),
  KEY `idx_handle_state` (`handle_state`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='意见反馈表';

```

***

## 三、B 域：商家后台

### B0. 通用分组表 `biz_group`

> **重要设计**：成品对「激活码分组、标签分组、ws 分组、素材分组、回复分组」等统一返回 `{id,name,user_id,type`

`,use_count}`，且通过 `type` 区分（实测：type=3 激活码、type=1 标签、type=5 ws）。因此采用**统一分组表 + type 区分**，减少重复表。各回复/关键词分组因带 `channel_id` 单独建表。

```sql
CREATE TABLE `zs_scrm_biz_group` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '分组ID',
  `merchant_id`     BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `name`        VARCHAR(64) NOT NULL COMMENT '分组名称',
  `type`        TINYINT NOT NULL COMMENT '分组类型：1标签 3激活码 5ws协议号 6素材 …',
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_type` (`user_id`,`type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='业务通用分组表（按type区分激活码/标签/ws等）';
```

### B1. 激活码管理

> 需求：【商家后台】激活码分组、激活码列表；【客户端接口】激活码登录。
成品接口：`/user/UserInfoChild/list`、`/base-list`、「编辑激活码」抽屉揭示全部配置项。

#### B1.1 激活码 `user_info_child`

> 「激活码编辑」表单完整字段：端口分配（动态/固定）、分组、分享页面权限（移除账号/线索分析/线索清零）、社交平台多选、线索去重范围、工作时间、自动回复、多设备登录、聊天备份、客户画像共享、数据脱敏、置零时间、数据权限（部门）、备注。

```sql
CREATE TABLE `zs_scrm_user_info_child` (
  `id`             BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '激活码ID',
  `merchant_id`        BIGINT UNSIGNED NOT NULL COMMENT '所属商家ID',
  `username`       varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NULL DEFAULT NULL COMMENT '激活码（如 5lVFY6B1UfUw，12位）',
  `remarks`        VARCHAR(50) DEFAULT NULL COMMENT '备注（限50字）',
  `group_id`       BIGINT UNSIGNED DEFAULT 0 COMMENT '所属分组ID -> biz_group(type=3)',
  `port_alloc_type` TINYINT DEFAULT 1 COMMENT '端口分配：1动态分配 2固定分配',
  `fixed_port_num` INT DEFAULT 0 COMMENT '固定分配端口数（port_alloc_type=2时）',
  -- 分享页面权限
  `share_type` VARCHAR(50) DEFAULT NULL COMMENT '分享页面权限',
  -- 线索去重
  `range_type`     TINYINT DEFAULT 1 COMMENT '线索去重范围：1当前会话 2当前会话(含底库) 3账号下全部会话(含底库) 4关联账号下全部(含底库) 5激活码分组内 6指定激活码',
  `range_childs`   JSON DEFAULT NULL COMMENT 'range_type=6时，指定去重的激活码ID列表',
  -- 工作时间
  `work_time_type` TINYINT DEFAULT 0 COMMENT '工作时间：0暂不设置 1固定时间',
  `work_time_start` VARCHAR(50)DEFAULT NULL COMMENT '固定工作时间配置（开始）',
  `work_time_end` VARCHAR(50)DEFAULT NULL COMMENT '固定工作时间配置（结束）',
  -- 自动回复开关
  `enable_keyword_reply` TINYINT DEFAULT 0 COMMENT '是否启用关键词回复',
  `enable_welcome_reply` TINYINT DEFAULT 0 COMMENT '是否启用欢迎语回复',
  -- 其他开关
  `multi_device_login` TINYINT DEFAULT 1 COMMENT '多设备登录：0关闭 1开启',
  `chat_backup`        TINYINT DEFAULT 1 COMMENT '聊天备份：0关闭 1开启',
  `profile_share`      TINYINT DEFAULT 0 COMMENT '客户画像共享：0不共享 1共享',
  `data_masking`       TINYINT DEFAULT 0 COMMENT '数据脱敏：0关闭 1开启',
  `reset_time`         VARCHAR(16) DEFAULT NULL COMMENT '置零时间（每日清零时刻，如 00:00）',
  `dept_id`            BIGINT UNSIGNED DEFAULT NULL COMMENT '数据权限-归属部门ID',
  `last_login_time`    DATETIME DEFAULT NULL COMMENT '最近登录时间',
  `status`              int NULL DEFAULT NULL COMMENT '状态：0停用 1正常',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`),
  KEY `idx_user_group` (`user_id`,`group_id`),
  KEY `idx_state` (`state`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='激活码表';
```

#### B1.2 激活码-社交平台绑定 `user_info_child_channel`

> 一个激活码可勾选多个社交平台（多对多）。成品接口 `/user/UserInfoChildChannel/share-list`。

```sql
CREATE TABLE `zs_scrm_user_info_child_channel` (
  `id`         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`    BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `child_id`   BIGINT UNSIGNED NOT NULL COMMENT '激活码ID',
  `channel_id` BIGINT UNSIGNED NOT NULL COMMENT '社交平台ID -> user_info_channel.id',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_child_channel` (`child_id`,`channel_id`),
  KEY `idx_user` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='激活码-社交平台绑定表';
```

### B2. 账号 / 设备管理

> 需求：【商家后台】账号管理、设备管理；【客户端接口】设备信息同步、账号登录授权、端口限制。
成品接口：`/client-user/page`（设备/会话）、账号列表 `/multi-account`。

#### B2.1 设备 `client_user`

> 客户端（PC 挂机端）一次登录即一个设备/会话，受套餐端口数限制。

```sql
CREATE TABLE `zs_device_user_rel`  (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `merchant_id`    BIGINT UNSIGNED DEFAULT  NULL COMMENT '商家ID',
  `device_id` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '设备ID',
  `device_type`      VARCHAR(32) NOTNULL COMMENT '设备类型',
  `user_id` bigint NOT NULL COMMENT '账号ID',
  `first_login` datetime NOT NULL COMMENT '该设备首次登录该账号的时间',
  `last_login` datetime NOT NULL COMMENT '最近一次登录时间',
  `login_count` int NOT NULL DEFAULT 1 COMMENT '累计登录次数',
  `remark`          VARCHAR(255) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `uk_dev_user`(`device_id` ASC, `user_id` ASC) USING BTREE,
  INDEX `idx_user`(`user_id` ASC) USING BTREE,
  INDEX `idx_device`(`device_id` ASC) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 607 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci COMMENT = '设备账号关系' ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
```

#### B2.2 三方平台账号 `channel_account`

> 在某激活码、某设备的**会话**里登录的具体三方平台账号。**账号本身不直接占端口——端口由它所在的会话 `device_port` 占用**；账号是"登录到一个已占端口的会话上"。这是"账号列表"的核心。

```sql
CREATE TABLE `zs_scrm_channel_account` (
  `id`            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '账号ID',
  `merchant_id`       BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `child_id`      BIGINT UNSIGNED NOT NULL COMMENT '激活码ID',
  `client_user_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '登录设备ID',
  `device_port_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '所在会话/挂机位ID -> device_port.id（端口由该会话占用）',
  `channel_id`    BIGINT UNSIGNED NOT NULL COMMENT '社交平台ID',
  `account_no`    VARCHAR(128) DEFAULT NULL COMMENT '平台账号（手机号/ID）',
  `user_name`    VARCHAR(128) DEFAULT NULL COMMENT '平台账号用户名',
  `nickname`      VARCHAR(128) DEFAULT NULL COMMENT '账号昵称',
  `avatar`        VARCHAR(255) DEFAULT NULL COMMENT '账号头像',
  `port_no`       INT DEFAULT NULL COMMENT '所在端口号（来自所在会话 device_port.port_no）',
  `proxy_id`      BIGINT UNSIGNED DEFAULT NULL COMMENT '使用的代理IP -> proxy_ip.id',
  `ws_account_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '使用的ws协议号 -> ws_account.id',
  `online_state`  TINYINT DEFAULT 0 COMMENT '在线状态：0离线 1在线',
  `auth_state`    TINYINT DEFAULT 0 COMMENT '登录授权状态：0未授权 1已授权 2已失效',
  `first_login_time` DATETIME DEFAULT NULL COMMENT '首次登录时间',
  `last_active_time` DATETIME DEFAULT NULL COMMENT '最后活跃时间',
  `lat_login_time` DATETIME DEFAULT NULL COMMENT '本次登录时间',
  `client_version`        VARCHAR(255) DEFAULT NULL COMMENT '客户端版本号',
  `status`         int NULL DEFAULT NULL COMMENT '状态：0禁用 1正常',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_merchant_id` (`merchant_id`),
  KEY `idx_child` (`child_id`),
  KEY `idx_channel` (`channel_id`),
  KEY `idx_online` (`online_state`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='三方平台登录账号表';
```

### B3. 代理设置

> 需求：【商家后台】代理设置。成品：`/proxy/list`（代理IP管理）、`/ws/pageAccount`（ws协议号管理）。

```sql
-- B3.1 代理IP
CREATE TABLE `zs_scrm_proxy_ip` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '代理ID',
  `merchant_id`     BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `name`        VARCHAR(64) DEFAULT NULL COMMENT '名称/备注',
  `protocol`    TINYINT DEFAULT 1 COMMENT '协议：1HTTP 2HTTPS 3SOCKS5',
  `host`        VARCHAR(128) NOT NULL COMMENT '代理主机/IP',
  `port`        INT NOT NULL COMMENT '端口',
  `account`     VARCHAR(128) DEFAULT NULL COMMENT '认证账号',
  `password`    VARCHAR(128) DEFAULT NULL COMMENT '认证密码（加密）',
  `country`     VARCHAR(64) DEFAULT NULL COMMENT '国家/地区',
  `check_state` TINYINT DEFAULT 0 COMMENT '检测状态：0未检测 1可用 2不可用',
  `last_check_time` DATETIME DEFAULT NULL,
  `status`       int NULL DEFAULT NULL COMMENT '状态：0停用 1启用',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='代理IP表';
```

### B4. 客户管理

> 需求：【商家后台】客户管理；【客户端接口】客户信息同步、客户画像。
成品：`/customer-list`、`/profile/config`（画像配置，含 contactInfo/salesInfo/companyInfo + 自定义字段）、`/userLabel`（标签）、`/inherit`（客户继承）。

#### B4.1 客户 `customer`

```sql
CREATE TABLE `zs_scrm_customer` (
  `id`            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '客户ID',
  `merchant_id`       BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `channel_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '来源平台ID',
  `channel_account_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '接待账号ID',
  `child_id`      BIGINT UNSIGNED DEFAULT NULL COMMENT '归属激活码ID',
  `platform_uid`  VARCHAR(128) DEFAULT NULL COMMENT '客户在三方平台的唯一ID',
  `platform_name`  VARCHAR(128) DEFAULT NULL COMMENT '客户在三方平台的名称',
  `nickname`      VARCHAR(128) DEFAULT NULL COMMENT '平台昵称',
  `avatar`        VARCHAR(255) DEFAULT NULL COMMENT '头像',
  `remark`        VARCHAR(128) DEFAULT NULL COMMENT '客户备注名',
  `phone`         VARCHAR(32)  DEFAULT NULL COMMENT '电话',
  `area_code`     VARCHAR(16)  DEFAULT NULL COMMENT '国家区号',
  `address`       VARCHAR(255) DEFAULT NULL COMMENT '地址',
  `birthday`      DATE DEFAULT NULL COMMENT '生日',
  `sex`           TINYINT DEFAULT 0 COMMENT '性别：0未知 1男 2女',
  `note`          VARCHAR(500) DEFAULT NULL COMMENT '客户备注/便签',
  `active`        TINYINT DEFAULT 1 COMMENT '是否活跃',
  -- 销售信息
  `customer_level` TINYINT DEFAULT NULL COMMENT '客户等级',
  `sale_phase`     TINYINT DEFAULT NULL COMMENT '销售阶段',
  `source`         VARCHAR(64) DEFAULT NULL COMMENT '客户来源',
  -- 公司信息
  `company_name`    VARCHAR(128) DEFAULT NULL COMMENT '公司名称',
  `company_title`   VARCHAR(64)  DEFAULT NULL COMMENT '职位',
  `company_website` VARCHAR(128) DEFAULT NULL COMMENT '公司网站',
  `company_scale`   VARCHAR(64)  DEFAULT NULL COMMENT '公司规模',
  `company_phone`   VARCHAR(32)  DEFAULT NULL COMMENT '公司电话',
  `company_email`   VARCHAR(128) DEFAULT NULL COMMENT '公司邮箱',
  `company_address` VARCHAR(255) DEFAULT NULL COMMENT '公司地址',
  `custom_fields`   JSON DEFAULT NULL COMMENT '自定义字段值（对应画像配置customCategories）',
  `status`         int NULL DEFAULT NULL COMMENT '状态：0无效 1正常',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user` (`user_id`),
  KEY `idx_platform_uid` (`user_id`,`platform_uid`),
  KEY `idx_channel_account` (`channel_account_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='客户表';
```

#### B4.2 客户画像配置 `customer_profile_config`

> 成品 `/profile/config`：每个商家一份，定义字段分类、顺序、隐藏、自定义字段。

```sql
CREATE TABLE `zs_scrm_customer_profile_config` (
  `id`               BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`          BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `contact_info`     JSON DEFAULT NULL COMMENT '联系信息配置{fieldOrder,hiddenFields,customFields}',
  `sales_info`       JSON DEFAULT NULL COMMENT '销售信息配置',
  `company_info`     JSON DEFAULT NULL COMMENT '公司信息配置',
  `category_order`   JSON DEFAULT NULL COMMENT '分类展示顺序',
  `custom_categories` JSON DEFAULT NULL COMMENT '自定义分类及字段',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='客户画像字段配置表';
```

#### B4.3 标签 `customer_label` 与 关联 `customer_label_rel`

> 成品 `/userLabel/group/group-list`（标签分组 type=1）+ `/userLabel/group/list`（标签）。

```sql
CREATE TABLE `zs_scrm_customer_customer_label` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '标签ID',
  `merchant_id`     BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `group_id`    BIGINT UNSIGNED DEFAULT 0 COMMENT '标签分组 -> biz_group(type=1)',
  `name`        VARCHAR(64) NOT NULL COMMENT '标签名',
  `color`       VARCHAR(16) DEFAULT NULL COMMENT '标签颜色',
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_group` (`user_id`,`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='客户标签表';

CREATE TABLE `zs_scrm_customer_label_rel` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `customer_id` BIGINT UNSIGNED NOT NULL COMMENT '客户ID',
  `label_id`    BIGINT UNSIGNED NOT NULL COMMENT '标签ID',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_customer_label` (`customer_id`,`label_id`),
  KEY `idx_label` (`label_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='客户-标签关联表';
```

#### B5.3 消息 `message`

> 需求：【客户端接口】消息同步。聊天备份（激活码 chat_backup 开关）控制是否落库。数据量极大，建议按 `user_id` 或时间分库分表。

```sql
CREATE TABLE `zs_scrm_customer_message` (
  `id`            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '消息ID',
  `merchant_id`       BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `child_id`      BIGINT UNSIGNED DEFAULT NULL COMMENT '激活码ID',
  `channel_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '平台ID',
  `channel_account_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '账号ID',
  `customer_id`   BIGINT UNSIGNED DEFAULT NULL COMMENT '客户ID',
  `platform_msg_id` VARCHAR(128) DEFAULT NULL COMMENT '平台原始消息ID（幂等去重）',
  `direction`     TINYINT NOT NULL COMMENT '方向：1接收 2发送',
  `msg_type`      TINYINT DEFAULT 1 COMMENT '类型：1文本 2图片 3语音 4视频 5文件 6位置 7名片',
  `content`       TEXT COMMENT '消息内容（文本/媒体URL）',
  `translate_content` TEXT DEFAULT NULL COMMENT '翻译后内容',
  `translate_state` TINYINT DEFAULT 0 COMMENT '翻译状态：0未翻译 1已翻译',
  `send_state`    TINYINT DEFAULT 1 COMMENT '发送状态：0失败 1成功 2发送中',
  `msg_time`      DATETIME DEFAULT NULL COMMENT '消息发生时间',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_platform_msg` (`channel_account_id`,`platform_msg_id`),
  KEY `idx_customer_time` (`customer_id`,`msg_time`),
  KEY `idx_user_time` (`user_id`,`msg_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='聊天消息表';
```

### B7. 运营管理

> 需求：【商家后台】运营管理（素材、快捷回复、关键词回复、欢迎语回复）。
成品：`/material/*`、`/reply/*`（注意回复类均带 `channel_id`，即按平台维度配置）。

```sql
-- B7.1 素材分类
CREATE TABLE `zs_scrm_material_type` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `name`        VARCHAR(64) NOT NULL COMMENT '分类名',
  `type`        TINYINT DEFAULT 0 COMMENT '素材大类：0全部/未分组 1图片 2视频 3文件 4文本',
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='素材分类表';

-- B7.2 素材
CREATE TABLE `zs_scrm_material` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `type_id`     BIGINT UNSIGNED DEFAULT 0 COMMENT '素材分类ID',
  `name`        VARCHAR(128) DEFAULT NULL COMMENT '素材名称',
  `material_type` TINYINT DEFAULT 1 COMMENT '类型：1文本 2图片 3语音 4视频 5文件',
  `content`     TEXT COMMENT '文本内容',
  `file_url`    VARCHAR(255) DEFAULT NULL COMMENT '媒体URL',
  `file_size`   BIGINT DEFAULT 0 COMMENT '文件大小',
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_type` (`user_id`,`type_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='素材表';

-- B7.3 快捷回复分组
CREATE TABLE `zs_scrm_reply_quick_group` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `channel_id`  BIGINT UNSIGNED DEFAULT NULL COMMENT '适用平台ID（成品reply分组带channel_id）',
  `name`        VARCHAR(64) NOT NULL COMMENT '分组名',
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_channel` (`user_id`,`channel_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='快捷回复分组表';

-- B7.4 快捷回复
CREATE TABLE `zs_scrm_reply_quick` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `group_id`    BIGINT UNSIGNED DEFAULT 0 COMMENT '分组ID',
  `channel_id`  BIGINT UNSIGNED DEFAULT NULL COMMENT '适用平台',
  `title`       VARCHAR(128) DEFAULT NULL COMMENT '标题/快捷指令',
  `content_type` TINYINT DEFAULT 1 COMMENT '内容类型：1文本 2图片 3视频 4文件',
  `content`     TEXT COMMENT '回复内容',
  `file_url`    VARCHAR(255) DEFAULT NULL,
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_group` (`user_id`,`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='快捷回复表';

-- B7.5 关键词回复分组
CREATE TABLE `zs_scrm_reply_keyword_group` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `channel_id`  BIGINT UNSIGNED DEFAULT NULL COMMENT '适用平台',
  `name`        VARCHAR(64) NOT NULL,
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_channel` (`user_id`,`channel_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='关键词回复分组表';

-- B7.6 关键词回复
CREATE TABLE `zs_scrm_reply_keyword` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `group_id`    BIGINT UNSIGNED DEFAULT 0,
  `channel_id`  BIGINT UNSIGNED DEFAULT NULL,
  `keyword`     VARCHAR(255) NOT NULL COMMENT '关键词（多个用分隔符）',
  `match_type`  TINYINT DEFAULT 1 COMMENT '匹配方式：1精确 2模糊/包含',
  `content_type` TINYINT DEFAULT 1 COMMENT '回复内容类型：1文本 2图片 3视频 4文件',
  `content`     TEXT COMMENT '回复内容',
  `file_url`    VARCHAR(255) DEFAULT NULL,
  `status`       int NULL DEFAULT NULL COMMENT '状态：0停用 1启用',
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_group` (`user_id`,`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='关键词回复表';

-- B7.7 欢迎语回复分组
CREATE TABLE `zs_scrm_reply_welcome_group` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `channel_id`  BIGINT UNSIGNED DEFAULT NULL COMMENT '适用平台',
  `name`        VARCHAR(64) NOT NULL,
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_channel` (`user_id`,`channel_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='欢迎语回复分组表';

-- B7.8 欢迎语回复
CREATE TABLE `zs_scrm_reply_welcome` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `merchant_id`     BIGINT UNSIGNED NOT NULL,
  `group_id`    BIGINT UNSIGNED DEFAULT 0,
  `channel_id`  BIGINT UNSIGNED DEFAULT NULL,
  `trigger_type` TINYINT DEFAULT 1 COMMENT '触发场景：1新好友 2进群 3首次会话',
  `content_type` TINYINT DEFAULT 1 COMMENT '内容类型：1文本 2图片 3视频 4文件',
  `content`     TEXT COMMENT '欢迎语内容',
  `file_url`    VARCHAR(255) DEFAULT NULL,
  `delay_second` INT DEFAULT 0 COMMENT '延迟发送秒数',
  `status`       int NULL DEFAULT NULL COMMENT '状态：0停用 1启用',
  `sort`        INT DEFAULT 0,
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_flag` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '删除状态',
  PRIMARY KEY (`id`),
  KEY `idx_user_group` (`user_id`,`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='欢迎语回复表'
```

> 四、C 域：客户端接口支撑表
需求：【客户端接口】激活码登录、设备信息同步、账号登录授权、端口限制、客户/消息同步。
多数读写复用上面的 `user_info_child`、`client_user`、`channel_account`、`customer`、`message`、`user_info_channel`。下面补充接口专用表。

```sql
-- C2 会话 / 挂机位（端口占用的真正单位）
-- 关键：PC端「开启一个会话」就占用 1 个端口（此时可以还没登录任何社交账号，空会话也占端口）；
--       账号登录是会话内的二级动作、不额外占端口；关闭会话才释放端口。
CREATE TABLE `zs_scrm_device_port` (
  `id`            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '会话(挂机位)ID',
  `merchant_id`       BIGINT UNSIGNED NOT NULL COMMENT '商家ID',
  `child_id`      BIGINT UNSIGNED NOT NULL COMMENT '激活码ID',
  `client_user_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '所属设备ID（一个设备可开多个会话）',
  `port_no`       INT DEFAULT NULL COMMENT '端口序号',
  `channel_id`    BIGINT UNSIGNED DEFAULT NULL COMMENT '计划/已登录平台ID（开会话时可先选平台，可空）',
  `channel_account_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '已登录的三方账号ID（未登录=NULL，即空会话）',
  `occupy_state`  TINYINT DEFAULT 1 COMMENT '占用状态：1占用(会话开启) 0已释放(会话关闭)',
  `account_state` TINYINT DEFAULT 0 COMMENT '账号状态：0未登录(空会话) 1已登录 2已登出/掉线',
  `open_time`     DATETIME DEFAULT NULL COMMENT '会话开启时间（= 占用端口时间）',
  `close_time`    DATETIME DEFAULT NULL COMMENT '会话关闭时间（= 释放端口时间）',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_user_state` (`user_id`,`occupy_state`),
  KEY `idx_child` (`child_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='会话/挂机位表（一行=一个会话=占1个端口；开会话即占用，空会话也占，关闭才释放）';
```

***

## 五、核心实体关系（ER 概览）

```
user_agent_info (代理商)
      │1
      │N
user_info (商家) ─1:N─ zs_merchant_payment_record (付款记录:充值套餐+购买账号) ─N:1─ zs_order_setmeal_info ─1:N─ zs_order_setmeal_specs
   │  │  │                                  └─N:1─ account_product (购买账号商品) │
   │  │  └──1:1── zs_merchant_setmeal (在用套餐)                       │type0~4 控制可见
   │  │                                              company_menu (菜单)
   │  ├──1:N── company_dept / company_role / company_staff ──N:M── company_menu
   │  │
   │  ├──1:N── user_info_child (激活码) ──N:M── user_info_channel (三方平台)
   │  │             │                              （经 user_info_child_channel）
   │  │             ├──1:N── client_user (设备) ──1:N── device_port (会话/挂机位=占端口) ──0..1── channel_account (三方账号)
   │  │             │                                        │   ├──N:1── proxy_ip
   │  │             │                                        │   └──N:1── ws_account
   │  │             ├──1:N── message (消息) ──1:1── translate_record
   │  │             ├──N:1── customer (客户) ──N:M── customer_label
   │  │             └──1:N── community_group (社群) ──1:N── group_work_order
   │  │
   │  ├──1:N── base_contact / sync_fans / customer_inherit
   │  └──1:N── material / reply_quick / reply_keyword / reply_welcome (均可带 channel_id)
   │
translate_line (翻译线路=接口服务商, 分组走字典 group_code:标准/专享) ──1:N── translate_language
translate_billing_rule ──(计费)── translate_record
```

***

## 六、关键设计说明与注意事项

1. **多租户隔离**：所有商家域表均含 `user_id`，所有索引以 `user_id` 为最左前缀；查询层强制注入 `user_id`，杜绝越权。
2. **激活码是授权与配置中心**：`user_info_child` 一条记录承载约 20 项业务开关（端口分配、去重范围、自动回复、聊天备份、画像共享、数据脱敏、工作时间等），是客户端登录后所有行为的"配置下发源"。建议这些开关同时缓存到 Redis，登录时随 token 下发。
3. **端口限制（按会话计，不是按账号）**：端口在 PC 端\*\*「开启一个会话」那一步就占用\*\*——此时可以还没登录任何社交账号（空会话也占端口）；账号登录是会话内的二级动作、不额外占端口；**关闭会话**才释放端口。

    - **校验时机 = 开会话时**：`当前 occupy_state=1 的会话数 < zs_merchant_setmeal.port_total + infinite_port` 才允许开；满了直接提示「端口已满」（与成品一致：端口满时点开会话即被拦，无需先登账号）。
    - 动态分配激活码走商家总端口池；固定分配激活码（`user_info_child.port_alloc_type=2`）按其 `fixed_port_num` 单独校验。
    - 占用单位是 `device_port`（会话/挂机位）；`channel_account`（登录的账号）挂在会话上、不单独占端口。
    - `online_count` = 当前开启的会话数（= 占用端口数），实时计数走 Redis（开会话 +1 / 关会话 −1）+ 定时回写 `device_port` 对账；心跳超时回收幽灵会话。
    - 套餐到期（`end_time`）/暂停（`pause_state=1`）→ 释放全部会话端口、禁止新开会话。

4. **去重底库**：`base_contact` 按 `(user_id, channel_id, platform_uid)` 唯一；新进线索按激活码的 `range_type` 决定去重比对范围（当前会话 / 账号 / 关联账号 / 分组 / 指定激活码）。
5. **统一分组 `biz_group` + type**：源于成品「各类 group-list 同构（id/name/user_id/type/use_count）」的事实；回复类分组因绑定 `channel_id` 单独建表。
6. **消息与翻译大表**：`message`、`translate_record`、`sys_oper_log` 为高写入大表，建议：① 按 `user_id` 或月份分表/分区；② 冷数据归档；③ `聊天备份` 关闭的激活码不落 `message` 正文，仅留会话摘要。
7. **翻译计费闭环**：`translate_line`（翻译线路=接口服务商，分组走字典 `group_code`：标准/专享）→ 产生 `translate_record`(含 cost) → 按 `translate_billing_rule` 结算到 `user_info.balance` 或套餐额度。

    - **语种配置**：`translate_language` 一张表按 `line_id` 区分，每条线路维护自己的一套语种列表（`code` 直接存该线路服务商代码，百度 `jp` / 有道 `ja`）。翻译时按「当前线路」取其语种行；新增线路只需以新 `line_id` 灌一套语种数据，表结构不变。语种列表以百度、有道官方文档为准导入维护。

8. **菜单套餐可见性**：`company_menu.type0~type4` 对应不同套餐档位的功能可见性（成品实测字段），结合 `zs_merchant_setmeal.setmeal_type` 在登录时计算可见菜单。
9. **金额/时长**：套餐时长用秒（`BIGINT`）便于精确扣减与暂停续期（`pause_state`/`pause_start_time`）；计费金额 `DECIMAL(12,4)`。
10. **逻辑删除与审计**：业务表统一 `deleted_flag`，关键表保留 `create_user_id/update_user_id`，配合 `sys_oper_log` 满足审计与内控（成品「内控管理」）需求。

***

## 七、与需求模块对照检查表

| 需求模块 | 对应表 | 覆盖 |
| --- | --- | --- |
| 【管理后台】三方聊天平台管理 | `user_info_channel` | ✅ |
| 【管理后台】套餐管理 | `zs_order_setmeal_info`/`zs_order_setmeal_specs`/`zs_merchant_payment_record`/`zs_merchant_setmeal` | ✅ |
| 付款记录（充值套餐 / 购买账号） | `zs_merchant_payment_record`（一张表，`order_category` 区分，含发货状态）+ `account_product` | ✅ |
| 【管理后台】翻译线路(=接口服务商)/分组/计费/对接 | `translate_line`(线路=服务商，分组走字典)/`translate_billing_rule`/`translate_record` | ✅ |
| 【管理后台】翻译语种配置（每个服务商一套） | `translate_language` | ✅ |
| 【管理后台】banner/联系我们/意见反馈 | `sys_banner`/`sys_contact_us`/`sys_feedback` | ✅ |
| 【管理后台】商家管理/套餐配置/员工/角色 | `user_info`/`zs_merchant_setmeal`/`company_staff`/`company_role` | ✅ |
| 【商家后台】激活码分组/列表 | `biz_group`/`user_info_child`/`user_info_child_channel` | ✅ |
| 【商家后台】运营管理（素材/快捷/关键词/欢迎语） | `material*`/`reply_quick*`/`reply_keyword*`/`reply_welcome*` | ✅ |
| 【商家后台】客户管理 | `customer`/`customer_profile_config`/`customer_label*`/`customer_inherit` | ✅ |
| 【商家后台】账号管理/设备管理 | `channel_account`/`client_user`/`device_port` | ✅ |
| 【商家后台】代理设置 | `proxy_ip`/`ws_account` | ✅ |
| 【客户端接口】激活码登录/设备信息同步 | `user_info_child`/`client_user`/`client_login_log` | ✅ |
| 【客户端接口】三方平台信息查询 | `user_info_channel` | ✅ |
| 【客户端接口】账号登录授权/端口限制 | `channel_account`/`device_port`/`zs_merchant_setmeal` | ✅ |
| 【客户端接口】客户/消息同步、客户画像 | `customer`/`message`/`customer_profile_config` | ✅ |

> 备注：`base_contact`/`sync_fans`/`community_group` 等为支撑上述模块运转的业务表，需求虽未单列但为成品既有能力，一并纳入设计；**分流链接（`shunt_link`）、平台工单（`work_order`）本期暂不实现，已从设计中移除。**
