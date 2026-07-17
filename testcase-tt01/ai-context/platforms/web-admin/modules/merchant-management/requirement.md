# 商家管理 需求说明

> 端：web-admin ｜ 模块：merchant-management ｜ 对应表：`zs_scrm_merchant_info`、`user_agent_info`、`zs_scrm_merchant_setmeal`

## 用户故事

- 作为平台运营，我希望创建和维护商家（租户）账号，以便为商家分配套餐并管理其生命周期。
- 作为平台运营，我希望查看商家余额、到期时间、套餐状态，以便进行运营干预（暂停/续期/调整套餐）。
- 作为平台运营，我希望管理代理商与商家的归属关系，以便按代理体系组织商家。

## 验收标准

- [ ] 场景：新建商家，预期：必填字段（商家名/账号/套餐）校验通过后落库 `zs_scrm_merchant_info`，初始余额、状态、乐观锁版本号正确。
- [ ] 场景：为商家分配套餐，预期：生成 `zs_scrm_merchant_setmeal` 订阅记录，激活码额度（child_total）、端口数（port_total）、时长（total_seconds）、翻译字符额度正确写入，生效/到期时间正确。
- [ ] 场景：商家套餐到期或暂停（pause_state=1），预期：联动释放该商家全部会话端口、禁止新开会话（见 [[common-business-rules]] 规则 #5）。
- [ ] 场景：并发更新商家余额，预期：乐观锁 `optimistic` 字段生效，并发冲突时更新失败并提示。
- [ ] 场景：查看商家列表，预期：支持按代理商、套餐、状态、到期时间筛选与搜索。
- [ ] 场景：逻辑删除商家，预期：`deleted_flag=1`，不物理删除，关联订阅/激活码按业务策略处理。

## 业务规则

1. 商家为租户隔离单位，`merchant_id` 全局唯一，所有商家域数据以之隔离（规则 #1）。
2. 商家套餐分配产生 `zs_scrm_merchant_setmeal` 订阅记录，余量字段（surplus_seconds、translate_char_surplus、child_count）随消费扣减。
3. 套餐到期/暂停触发端口释放与新会话禁止（规则 #5）。
4. 余额变更须通过乐观锁控制并发（`optimistic` 版本号）。
5. 代理商（user_agent_info）与商家 1:N，代理商仅可见其下级商家。

## 约束与依赖

- 依赖套餐管理模块（package-commercialization）提供套餐主体定义。
- 依赖付款记录（`zs_scrm_setmeal_order`）作为订阅来源。
- 商家状态变更影响 web-merchant / pc-client 的可用性（跨端联动）。
- 接口/数据模型/UI/测试笔记：暂未提供，待补充（数据模型见 `shared/common-data-model.md` A4）。
