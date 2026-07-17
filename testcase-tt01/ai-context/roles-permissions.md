# 全局角色-权限矩阵

> 三端使用者与权限范围。✅ 可用 / ❌ 不可用 / ⚠️ 受条件限制。

## 角色清单

| 角色 | 说明 | 适用端 |
| ---- | ---- | ------ |
| 平台运营 / 管理员 | 平台方人员，维护全局配置、商家、套餐、翻译线路、CMS、数据 | web-admin |
| 代理商（user_agent_info） | 商家上级代理，关联多个商家 | web-admin |
| 商家主账号（is_master） | 租户所有人，管理本租户全部资源与组织权限 | web-merchant |
| 商家员工 / 子账号（company_staff） | 主账号创建的下属，按 `company_role` 分权 | web-merchant / pc-client |
| 激活码登录端 | 商家分发的激活码在 PC 客户端登录使用 | pc-client |

## 权限矩阵（核心操作）

| 操作 / 资源 | 平台运营 | 代理商 | 商家主账号 | 商家员工 | 激活码端 | 适用端 |
| ----------- | ------- | ------- | ---------- | -------- | -------- | ------ |
| 三方平台字典维护 | ✅ | ❌ | ❌ | ❌ | ❌ | web-admin |
| 套餐主体/规格维护 | ✅ | ❌ | ❌ | ❌ | ❌ | web-admin |
| 翻译线路/计费配置 | ✅ | ❌ | ❌ | ❌ | ❌ | web-admin |
| 商家管理（增删/套餐分配） | ✅ | ⚠️仅下级 | ❌ | ❌ | ❌ | web-admin |
| banner/联系我们/反馈处理 | ✅ | ❌ | ❌ | ❌ | ❌ | web-admin |
| 数据指标查看 | ✅ | ⚠️下级 | ⚠️本租户 | ❌ | ❌ | web-admin / web-merchant |
| 激活码生成/配置 | ❌ | ❌ | ✅ | ⚠️按角色 | ❌ | web-merchant |
| 账号/设备/代理管理 | ❌ | ❌ | ✅ | ⚠️按角色 | ❌ | web-merchant |
| 客户/线索/运营管理 | ❌ | ❌ | ✅ | ⚠️按角色 | ❌ | web-merchant |
| 组织/角色/菜单权限 | ❌ | ❌ | ✅ | ❌ | ❌ | web-merchant |
| 开会话/占端口 | ❌ | ❌ | ❌ | ❌ | ⚠️受套餐端口数限制 | pc-client |
| 三方账号登录授权 | ❌ | ❌ | ❌ | ❌ | ⚠️受激活码配置限制 | pc-client |
| 消息收发/翻译 | ❌ | ❌ | ❌ | ❌ | ✅ | pc-client |

> 商家域内部权限由 `company_dept / company_role / company_staff / company_menu / company_role_menu / company_staff_role` 实现 RBAC；菜单可见性受套餐档位 `type0~type4` 约束（见 [[common-business-rules]] 规则 #9）。
