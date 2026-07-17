# 社交平台接入 需求说明

> 端：web-admin ｜ 模块：social-platform-integration ｜ 对应表：`zs_scrm_social_platform`

## 用户故事

- 作为平台运营，我希望维护三方聊天平台字典（名称/编码/图标/链接/排序/状态），以便客户端拉取并展示可接入的社交平台。
- 作为平台运营，我希望新增或下架平台，以便动态扩展或收缩支持的平台范围。

## 验收标准

- [ ] 场景：新增平台，预期：填写名称、编码（唯一，如 whatsapp）、图标 URL、链接、排序、状态，落库 `zs_scrm_social_platform`，`uk_code` 保证编码唯一。
- [ ] 场景：编码重复，预期：校验失败并提示，不落库。
- [ ] 场景：下架平台（status=0），预期：客户端不再展示该平台，已绑定该平台的激活码/账号按业务策略处理。
- [ ] 场景：客户端拉取平台列表，预期：返回启用中的平台图标/名称/编码，按 sort 排序（参照成品 `/user/UserInfoChannel/list` 返回 19 个平台）。

## 业务规则

1. 平台字典为全平台公共字典，由本端维护，web-merchant / pc-client 只读消费。
2. 编码 `code` 唯一，客户端据此识别平台；激活码-平台绑定（`zs_scrm_user_info_child_channel`）与平台账号（`zs_scrm_channel_account`）通过 `channel_id` 关联。
3. 支持平台范围：WhatsApp / Telegram / Line / Instagram / Messenger / Facebook / Twitter / Zalo / TikTok / Teams / Snapchat 等 19 个。
4. 平台下架不影响历史数据（逻辑保留），仅控制新接入可见性。

## 约束与依赖

- 下游消费方：web-merchant 激活码绑定平台、pc-client 平台适配器。
- 接口/数据模型/UI/测试笔记：暂未提供，待补充（数据模型见 `shared/common-data-model.md` A1）。
