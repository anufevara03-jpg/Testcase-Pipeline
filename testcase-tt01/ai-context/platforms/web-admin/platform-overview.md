# web-admin 端概览

## 端简介

- **端名称**：web-admin
- **端类型**：Web 管理后台（平台运营端）
- **目标用户**：平台运营 / 管理员（平台方人员）

> 平台运营端，负责全局配置与商家管理。对应数据库 A 域。该端暂仅提供需求文档（见 `modules/` 各模块 requirement.md），接口/数据模型/UI 细节待补充。

## 核心职责

1. **商家管理**：商家（租户）的创建、维护、套餐分配、状态与余额管理。
2. **商业化管理**：套餐规格与定价（套餐主体 + 售卖规格 SKU）、资源消耗触发规则管理（翻译字符/端口等资源的计费与触发规则）。
3. **各社交平台的接入**：三方聊天平台字典维护（19 个平台，图标/名称/编码/状态）。
4. **各接口的接入**：翻译线路（=接口服务商）接入、计费规则、语种配置；其他外部接口对接。
5. **商户端界面 UI 管理**：banner、联系我们、意见反馈、公告等商户端展示内容管理。
6. **各项数据指标的查看**：商家/套餐/翻译消耗/付款等多维数据指标查看。

## 技术栈

- **前端框架**：{待补充}
- **UI 组件库**：{待补充}
- **状态管理**：{待补充}

## 交互范式

- Web 表格密集型管理后台，平台运营全局视角。
- 不受商家租户隔离约束（跨租户管理），但自身需平台管理员鉴权。
- 维护 A 域字典与配置表，供 web-merchant / pc-client 读取消费。

## 模块清单

| 模块 | 主要表 | 需求文档 |
| ---- | ------ | -------- |
| 商家管理 merchant-management | `zs_scrm_merchant_info`、`user_agent_info`、`zs_scrm_merchant_setmeal` | ✅ requirement.md |
| 商业化套餐管理 package-commercialization | `zs_scrm_setmeal_info`、`zs_scrm_setmeal_specs`、`zs_scrm_setmeal_order`、`zs_scrm_translate_billing_rule` | ✅ requirement.md |
| 社交平台接入 social-platform-integration | `zs_scrm_social_platform` | ✅ requirement.md |
| 接口接入 api-integration | `zs_scrm_translate_line`、`zs_scrm_translate_language`、`api_key`、`open_api_log` | ✅ requirement.md |
| 商户端 UI 管理 merchant-ui-management | `zs_scrm_banner`、`zs_scrm_contact_us`、`zs_scrm_feedback`、`sys_notice` | ✅ requirement.md |
| 数据指标查看 data-metrics | （聚合 A/B 域表，只读统计） | ✅ requirement.md |

> 各模块的 api.md / data-model.md / ui-spec.md / test-notes.md 暂未提供，待补充。数据模型可参考 `shared/common-data-model.md` 中对应 A 域表定义。

## 与其他端的关系

- 维护 A 域字典与配置，供 web-merchant / pc-client 消费：
  - 三方平台字典（`zs_scrm_social_platform`）→ 客户端拉取。
  - 套餐/规格 → 商家订阅（`zs_scrm_merchant_setmeal`）→ 决定激活码额度与端口数。
  - 翻译线路/计费/语种 → 客户端翻译消费并产生 `translate_record` 计费（[[common-business-rules]] 规则 #7/#8）。
- 商家管理：为商家分配套餐，商家在 web-merchant 生成激活码供 pc-client 使用。
- 跨端流程：套餐定价（web-admin）→ 商家购买（付款记录）→ 商家生成激活码（web-merchant）→ 客户端登录（pc-client）→ 翻译计费回流（translate_record）。

## 约定与备注

- 资源消耗触发规则（如翻译字符计费、端口超限）的配置入口在本端，实际扣减/拦截在 pc-client / web-merchant 执行（见规则 #4/#7）。
- 金额精度：计费单价 `DECIMAL(12,4)`，余额 `DECIMAL(18,2)`；套餐时长以秒（BIGINT）计。
- 付款记录统一表 `zs_scrm_setmeal_order` 用 `order_category` 区分充值套餐 / 购买账号两个 Tab。
