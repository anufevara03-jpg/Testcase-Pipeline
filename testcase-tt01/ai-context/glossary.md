# 全局术语表

> 各端通用业务术语，避免不同端对同一概念使用不同命名造成歧义。

| 术语 | 英文/缩写 | 含义 | 出现的端 |
| ---- | --------- | ---- | -------- |
| 商家 / 租户 | merchant / tenant | 平台的客户单位，数据以 `merchant_id`/`user_id` 隔离 | web-admin / web-merchant / pc-client |
| 激活码 | activation code / user_info_child | 商家生成、用于客户端登录的 12 位凭证，承载约 20 项业务开关 | web-merchant / pc-client |
| 子账号 | child account | 历史描述中由主账号创建的下属账号（迭代001 中由"激活码"承接） | web-merchant / pc-client |
| 端口 | port | 同时在线会话数额度，套餐 `port_total` 控制 | web-admin / web-merchant / pc-client |
| 会话 / 挂机位 | session / device_port | PC 端开启的一个会话，占用 1 个端口（空会话也占） | pc-client / web-merchant |
| 三方平台 / 社交平台 | social platform | WhatsApp / Telegram 等 19 个聊天平台，`zs_scrm_social_platform` | web-admin / web-merchant / pc-client |
| 平台账号 | channel_account | 在某激活码、某设备会话里登录的具体三方平台账号 | web-merchant / pc-client |
| 套餐 | setmeal | 商家订阅的服务档位，分主体 `setmeal_info` 与售卖规格 `setmeal_specs` | web-admin / web-merchant |
| 售卖规格 | specs / SKU | 套餐的「时长+价格」档位 | web-admin |
| 翻译线路 | translate_line | 翻译接口服务商（百度/有道/Google/DeepL），线路=服务商 | web-admin / pc-client |
| 线路分组 | group_code | 标准线路 / 专享线路，走字典表 | web-admin |
| 翻译计费规则 | translate_billing_rule | 按字符/条数/Token/包月的单价与免费额度配置 | web-admin |
| 翻译记录 | translate_record | 单次翻译流水，含 cost，作为计费与对账依据 | web-admin / pc-client |
| 去重底库 | base_contact | 按 `(merchant_id, channel_id, platform_uid)` 唯一的线索去重表 | web-merchant / pc-client |
| 去重范围 | range_type | 激活码配置的线索去重比对范围（当前会话/账号/分组/指定激活码等） | web-merchant / pc-client |
| 代理商 | user_agent_info | 商家的上级代理，与商家 1:N | web-admin |
| 菜单可见性 | type0~type4 | `company_menu` 控制不同套餐档位的功能可见性 | web-merchant / pc-client |
| 客户画像 | customer_profile | 客户的联系/销售/公司信息及自定义字段配置 | web-merchant / pc-client |
| 代理 IP | proxy_ip | 防关联的网络出口，HTTP/HTTPS/SOCKS5 | web-merchant / pc-client |
| ws 协议号 | ws_account | WebSocket 协议号资源 | web-merchant / pc-client |
| 字符额度 | char quota | AI 翻译字符数额度，套餐含或单独计费 | web-admin / web-merchant / pc-client |
