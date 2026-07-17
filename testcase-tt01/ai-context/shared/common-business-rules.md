# 共享业务规则

> 所有端共同遵守的核心业务规则。新增规则按 `规则 #N` 编号，各端模块引用时写 `（见 shared/common-business-rules.md 规则 #N）`。
> 来源：`TemporaryFile/SCRM迭代001_数据库设计文档.md`、`NexSCRM项目描述（桌面应用）.md`、`NexSCRM项目描述(web商家后台).md`。

## 规则列表

- **规则 #1 多租户隔离**：所有商家域表均含 `user_id`/`merchant_id`，索引以之为最左前缀；查询层强制注入该字段，杜绝跨租户越权。
  - 适用端：web-admin / web-merchant / pc-client（全部）

- **规则 #2 激活码是授权与配置中心**：`zs_scrm_user_info_child` 一条记录承载约 20 项业务开关（端口分配、去重范围、自动回复、聊天备份、画像共享、数据脱敏、工作时间等），是客户端登录后所有行为的"配置下发源"。建议这些开关缓存 Redis，登录时随 token 下发。
  - 适用端：web-merchant（配置）/ pc-client（消费）

- **规则 #3 端口按会话计，非按账号**：PC 端「开启一个会话」那一步即占用 1 个端口（此时可还没登录任何社交账号，**空会话也占端口**）；账号登录是会话内的二级动作，**不额外占端口**；**关闭会话**才释放端口。占用单位是 `zs_scrm_device_port`（会话/挂机位）；`zs_scrm_channel_account` 挂在会话上、不单独占端口。
  - 适用端：pc-client / web-merchant

- **规则 #4 端口校验时机 = 开会话时**：`当前 occupy_state=1 的会话数 < zs_scrm_merchant_setmeal.port_total + infinite_port` 才允许开会话；满了直接提示「端口已满」（与成品一致：端口满时点开会话即被拦，无需先登账号）。动态分配激活码走商家总端口池；固定分配激活码（`port_alloc_type=2`）按其 `fixed_port_num` 单独校验。
  - 适用端：pc-client / web-merchant

- **规则 #5 端口实时计数与回收**：`online_count` = 当前开启的会话数（= 占用端口数），实时计数走 Redis（开会话 +1 / 关会话 −1）+ 定时回写 `device_port` 对账；心跳超时回收幽灵会话。套餐到期（`end_time`）/ 暂停（`pause_state=1`）→ 释放全部会话端口、禁止新开会话。
  - 适用端：pc-client / web-merchant

- **规则 #6 去重底库**：`base_contact` 按 `(user_id, channel_id, platform_uid)` 唯一；新进线索按激活码 `range_type` 决定去重比对范围（1当前会话 / 2含底库 / 3账号下全部 / 4关联账号 / 5分组内 / 6指定激活码）。
  - 适用端：pc-client / web-merchant

- **规则 #7 翻译计费闭环**：`zs_scrm_translate_line`（翻译线路=接口服务商，分组走字典 `group_code`：标准/专享）→ 产生 `zs_scrm_translate_record`(含 cost) → 按 `zs_scrm_translate_billing_rule` 结算到 `zs_scrm_merchant_info.balance` 或套餐翻译字符额度。
  - 适用端：web-admin（配置线路/计费）/ pc-client（产生翻译）/ web-merchant（查看消耗）

- **规则 #8 翻译语种按线路配置**：`zs_scrm_translate_language` 一张表按 `line_id` 区分，每条线路维护自己一套语种列表（`code` 直接存该服务商代码，百度 `jp` / 有道 `ja`）。翻译时按「当前线路」取其语种行；新增线路只需以新 `line_id` 灌一套语种数据，表结构不变。语种列表以百度、有道官方文档为准。
  - 适用端：web-admin（维护）/ pc-client（使用）

- **规则 #9 菜单套餐可见性**：`company_menu.type0~type4` 对应不同套餐档位的功能可见性，结合 `zs_scrm_merchant_setmeal.setmeal_type` 在登录时计算可见菜单。
  - 适用端：web-merchant / pc-client

- **规则 #10 逻辑删除与审计**：业务表统一 `deleted_flag`（0正常/1删除），关键表保留 `create_user_id/update_user_id`，配合 `sys_oper_log` 满足审计与内控。
  - 适用端：全部

- **规则 #11 消息落库受聊天备份开关控制**：`zs_scrm_user_info_child.chat_backup` 关闭的激活码不落 `zs_scrm_customer_message` 正文，仅留会话摘要。
  - 适用端：pc-client / web-merchant

- **规则 #12 回复/关键词按平台维度配置**：快捷回复、关键词回复、欢迎语均带 `channel_id`，即按社交平台维度独立配置。
  - 适用端：web-merchant

> 以下规则来源于《NexSCRM项目描述(web商家后台).md》的历史/补充描述，涉及字符额度分配、推广分销、提现结算等。该描述与当前迭代001 数据库设计（激活码/客户/运营）存在差异，**落地前需与产品确认是否仍适用**。

- **规则 #13 字符分配强事务**（待确认）：主账号剩余字符数更新与子账号增加字符数必须在同一数据库事务中完成，避免资源凭空消失或超额分配。
  - 适用端：web-merchant（待确认）

- **规则 #14 消息扣减字符最终一致性**（待确认）：每次在会话中发送一条消息，系统须同步扣减主账号/子账号的字符额度；保证最终一致性（消息发送成功后异步扣除，或预扣）。
  - 适用端：pc-client / web-merchant（待确认）

- **规则 #15 推广分销佣金**（待确认）：首次交易奖励 15%，后续交易奖励 10%；推广业绩在推广报表展示。
  - 适用端：web-merchant（待确认）

- **规则 #16 提现强事务与补偿**（待确认）：提现申请与余额扣除为强事务操作，须调用第三方支付/USDT 链上转账接口，设计「提现中 → 提现成功/提现失败」补偿机制，防止资金丢失；规则为 3 个工作日内到账、手续费 1%。
  - 适用端：web-merchant（待确认）

- **规则 #17 敏感词风控**（待确认）：敏感词模块设置拦截词库，对沟通中违规发送内容进行拦截与记录，保障账号安全。
  - 适用端：pc-client / web-merchant（待确认）
