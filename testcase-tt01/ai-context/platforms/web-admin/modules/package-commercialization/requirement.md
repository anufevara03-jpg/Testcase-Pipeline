# 商业化套餐管理 需求说明

> 端：web-admin ｜ 模块：package-commercialization ｜ 对应表：`zs_scrm_setmeal_info`、`zs_scrm_setmeal_specs`、`zs_scrm_setmeal_order`、`zs_scrm_translate_billing_rule`

## 用户故事

- 作为平台运营，我希望定义套餐档位（基础/专业/旗舰等）及其权益，以便商家按需选购。
- 作为平台运营，我希望为每档套餐配置多档「时长+价格」售卖规格（SKU），以便支持月/年/字符版等多种计费。
- 作为平台运营，我希望配置资源消耗触发规则（翻译字符/端口/消息的计费方式、单价、免费额度、触发条件），以便自动结算商家消耗。
- 作为平台运营，我希望查看付款记录（充值套餐 / 购买账号），以便对账与发货管理。

## 验收标准

- [ ] 场景：创建套餐主体，预期：填写名称、档位 type（控制菜单可见性 type0~type4）、计费模式（1按时长 2按字符）、激活码额度、端口数、是否不限端口/免费/含翻译，落库 `zs_scrm_setmeal_info`。
- [ ] 场景：为套餐添加售卖规格，预期：配置时长(duration/duration_unit)、折算秒数(total_seconds)、翻译字符额度、赠送字符、激活码数、售价/原价，落库 `zs_scrm_setmeal_specs`；字符版时长相关字段留空/为 0。
- [ ] 场景：套餐上架/下架，预期：`status` 切换 0/1，下架套餐不可被新购。
- [ ] 场景：配置翻译计费规则，预期：选择适用线路分组/具体线路/套餐，设置计费方式（1字符 2条数 3Token 4包月）、单价(DECIMAL(12,4))、最低消费、免费额度、币种、生效时间，落库 `zs_scrm_translate_billing_rule`。
- [ ] 场景：查看付款记录，预期：按 `order_category` 分两个 Tab（1充值套餐 / 2购买账号），充值套餐展示订单号/套餐/金额/支付状态/支付时间/渠道，购买账号额外展示发货状态。
- [ ] 场景：购买账号发货，预期：维护 `deliver_state`（0待发货 1已发货 2部分 3失败 4已退款）与发货内容（加密存储）。

## 业务规则

1. 套餐档位 `type` 决定菜单可见性（规则 #9）：`company_menu.type0~type4` × 商家订阅 `setmeal_type`。
2. 计费模式分按时长（标准/专业版，total_seconds 秒级）与按字符（字符版，translate_char_total）。
3. 资源消耗触发规则由 `zs_scrm_translate_billing_rule` 定义，实际扣减在 pc-client 翻译消费时产生 `translate_record` 并结算到余额或套餐额度（规则 #7）。
4. 付款记录统一表用 `order_category` 区分充值套餐 / 购买账号，专用字段按类别填充。
5. 套餐时长以秒（BIGINT）计，便于暂停续期（pause_state/pause_start_time）精确扣减。
6. 金额：售价 `DECIMAL(12,2)`，计费单价 `DECIMAL(12,4)`。

## 约束与依赖

- 依赖接口接入模块（api-integration）提供翻译线路，计费规则按线路/分组生效。
- 套餐分配结果供商家管理模块（merchant-management）写入 `zs_scrm_merchant_setmeal`。
- 付款记录的购买账号发货依赖 `account_product` 商品定义。
- 接口/数据模型/UI/测试笔记：暂未提供，待补充（数据模型见 `shared/common-data-model.md` A2/A3.2）。
