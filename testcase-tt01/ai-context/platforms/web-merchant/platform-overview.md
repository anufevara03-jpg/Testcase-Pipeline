# web-merchant 端概览

## 端简介

- **端名称**：web-merchant
- **端类型**：Web 商家后台（租户端）
- **目标用户**：商家（租户）及其员工（主账号 + 子账号/员工，按 RBAC 分权）

> 商家租户运营管理后台，负责本租户内的激活码、账号/设备、客户/线索、社群、运营物料、代理、组织权限等。对应数据库 B 域。

## 技术栈

- **前端框架**：{待补充}
- **UI 组件库**：{待补充}
- **状态管理**：{待补充}

> 成品参照 ChatKnow。后端走统一 API，数据落 MySQL B 域表。

## 交互范式

- Web 表格密集型管理后台，按菜单模块组织。
- 菜单可见性受套餐档位 `company_menu.type0~type4` × `zs_scrm_merchant_setmeal.setmeal_type` 控制（[[common-business-rules]] 规则 #9）。
- 商家域所有操作强制带 `merchant_id` 租户隔离（规则 #1）。

## 核心功能域（B 域模块）

| 模块 | 主要表 | 职责 |
| ---- | ------ | ---- |
| 激活码管理 | `zs_scrm_user_info_child`、`zs_scrm_user_info_child_channel`、`zs_scrm_biz_group`(type=3) | 激活码分组/列表/编辑（约 20 项配置开关）、激活码-社交平台绑定 |
| 账号 / 设备管理 | `zs_scrm_device_user_rel`、`zs_scrm_channel_account`、`zs_scrm_device_port` | 设备/会话管理、三方平台账号登录授权 |
| 代理设置 | `zs_scrm_proxy_ip`、`ws_account` | 代理 IP（HTTP/HTTPS/SOCKS5）、ws 协议号 |
| 客户管理 | `zs_scrm_customer`、`zs_scrm_customer_profile_config`、标签表、`customer_inherit` | 客户档案、画像配置、标签、客户继承 |
| 线索 / 消息 | `base_contact`、`sync_fans`、`zs_scrm_customer_message` | 去重底库、粉丝同步、聊天消息（受 chat_backup 控制） |
| 社群管理 | `community_group`、`group_work_order`、`group_label` | 社群、工单、标签 |
| 运营管理 | `zs_scrm_material*`、`zs_scrm_reply_quick*`/`keyword*`/`welcome*` | 素材、快捷/关键词/欢迎语回复（均按 channel_id 平台维度） |
| 组织与权限 | `company_dept`/`role`/`staff`/`menu`/`role_menu`/`staff_role` | 部门、角色、员工、菜单 RBAC |

## 与其他端的关系

- 与 web-admin 共享：套餐订阅（`zs_scrm_merchant_setmeal`）、三方平台字典、翻译线路/计费（产生消耗走规则 #7）。
- 与 pc-client 共享：激活码配置下发、账号/设备/会话、客户/消息数据；端口占用联动（规则 #3~#5）。
- 跨端流程：web-admin 分配套餐 → web-merchant 生成激活码 → pc-client 登录使用。

## 历史描述差异说明

`TemporaryFile/NexSCRM项目描述(web商家后台).md` 描述了字符额度分配（子账号管理/字符分配记录）、推广分销（推广中心/推广报表，首单15%/后续10%）、USDT 提现（3 工作日到账、1% 手续费）、敏感词风控等能力。**这些与迭代001 数据库设计（激活码/客户/运营）存在差异，落地前需与产品确认是否仍适用**。相关规则暂记为 [[common-business-rules]] 规则 #13~#17（待确认）。

## 约定与备注

- 回复/关键词/欢迎语均带 `channel_id`，按平台维度配置（规则 #12）。
- 消息落库受激活码 `chat_backup` 开关控制（规则 #11）。
- 通用分组 `zs_scrm_biz_group` + `type` 区分激活码/标签/ws/素材分组。
