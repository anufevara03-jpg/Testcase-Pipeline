# 接口接入 需求说明

> 端：web-admin ｜ 模块：api-integration ｜ 对应表：`zs_scrm_translate_line`、`zs_scrm_translate_language`、`api_key`、`open_api_log`

## 用户故事

- 作为平台运营，我希望接入并配置翻译线路（接口服务商，如百度/有道/Google/DeepL），以便客户端调用翻译能力。
- 作为平台运营，我希望为每条线路维护其支持的语种列表，以便翻译时按线路选取正确语种代码。
- 作为平台运营，我希望管理线路分组（标准线路/专享线路），以便按分组差异化计费与路由。
- 作为平台运营，我希望管理对外开放接口的 API Key 与调用日志，以便第三方对接与审计。

## 验收标准

- [ ] 场景：新增翻译线路，预期：填写名称、编码（唯一）、分组字典 code（标准/专享）、接口地址、AppID、密钥（加密存储）、状态，落库 `zs_scrm_translate_line`。
- [ ] 场景：为线路导入语种配置，预期：以 `line_id` 关联批量写入 `zs_scrm_translate_language`，每行存该服务商语种代码（如百度 jp / 有道 ja）、中文名、是否自动检测、是否可作源/目标语言；`uk_line_code(line_id, code)` 保证不重复。
- [ ] 场景：新增线路只需灌一套语种数据，预期：表结构不变，新增 `line_id` 的语种行即可（规则 #8）。
- [ ] 场景：停用线路，预期：`status=0`，客户端不再路由到该线路；已产生的翻译记录保留。
- [ ] 场景：管理 API Key，预期：生成/吊销开放接口密钥，调用记入 `open_api_log`。
- [ ] 场景：语种列表导入，预期：以百度、有道官方文档为准（百度《通用文本翻译-语种列表》、有道《文本翻译-支持语言》）。

## 业务规则

1. **翻译线路 = 翻译接口/服务商**（三合一概念），分组（标准/专享）走字典表，本表只存 `group_code`（规则 #7/#8）。
2. 一条线路对应一套语种配置（line 1:N language），`code` 直接存该线路服务商自己的语种代码；新增线路不改表结构。
3. 密钥类字段（app_secret）加密存储。
4. 线路与计费规则关联：`zs_scrm_translate_billing_rule` 按 `group_code`/`line_code` 生效（见 package-commercialization 模块）。
5. 翻译时客户端按「当前线路」取其语种行；部分线路存在方向限制，用 `support_source/support_target` 控制。

## 约束与依赖

- 上游：依赖字典表（标准/专享线路分组）。
- 下游：pc-client 聚合翻译引擎调用线路产生 `translate_record`；package-commercialization 模块按线路配置计费规则。
- 接口/数据模型/UI/测试笔记：暂未提供，待补充（数据模型见 `shared/common-data-model.md` A3.1/A3.4）。
