# 系统总览

## 项目

**NexSCRM** —— 面向多账号、多三方聊天平台（WhatsApp / Telegram / Line / Instagram / Messenger / Facebook / Twitter / Zalo / TikTok / Teams / Snapchat 等 19 个平台）的社交化客户关系管理（SCRM）平台，并集成消息翻译能力。参照成品 ChatKnow。

核心授权链路：**商家购买套餐 → 套餐含「激活码额度 + 端口数」→ 商家生成激活码（含各类聊天/去重/翻译配置）→ 客户端用激活码登录 → 在激活码下登录多个三方平台账号（占用端口）→ 与客户聊天产生线索 → 消息可走翻译线路翻译并计费。**

## 三端架构

| 端 | 目录 | 使用者 | 职责 |
| -- | ---- | ------ | ---- |
| 管理后台 web-admin | `platforms/web-admin/` | 平台运营 / 管理员 | 三方平台管理、套餐管理、翻译线路与计费配置、商家管理、商家套餐分配、banner / 联系我们 / 意见反馈、数据指标查看 |
| 商家后台 web-merchant | `platforms/web-merchant/` | 商家（租户）及其员工 | 激活码管理、账号 / 设备管理、客户 / 线索管理、社群管理、运营物料、代理 IP、组织与权限 |
| PC 客户端 pc-client | `platforms/pc-client/` | 商家分发的激活码登录端（Electron 桌面应用 v1.0.7） | 激活码登录、设备信息同步、三方平台信息查询、账号登录授权与端口限制、客户 / 消息同步、客户画像、聚合翻译 |

```
                    ┌───────────────────────┐
                    │  管理后台 web-admin   │  平台运营：商家/套餐/翻译线路/平台接入/UI/数据
                    └───────────┬───────────┘
                                │ 维护 A 域字典与配置
                                ▼
   ┌──────────────────────┐   统一数据库 (MySQL 8.0)   ┌──────────────────────┐
   │  商家后台 web-merchant│ ◀─────────────────────────▶ │   PC 客户端 pc-client │
   │  商家租户：激活码/客户/ │      B 域 + C 域读写         │  Electron：会话/翻译/  │
   │  账号/运营/组织权限     │                             │  端口占用/消息同步     │
   └──────────────────────┘                             └──────────────────────┘
```

## 共享与跨端

- 各端共用规则见 `shared/common-business-rules.md`；统一数据模型见 `shared/common-data-model.md`。
- 三端共用同一数据库：web-admin 维护 A 域（平台级字典/配置）、web-merchant 读写 B 域（商家域业务）、pc-client 经 C 域接口支撑表与 B 域表交互。
- 跨端联动流程见 `cross-platform/`（待补充：如套餐购买→激活码生成→客户端登录的端到端流程）。

## 数据来源

- `TemporaryFile/SCRM迭代001_数据库设计文档.md`（V1.0.0）—— 数据库设计权威来源。
- `TemporaryFile/NexSCRM项目描述（桌面应用）.md` —— pc-client 端描述。
- `TemporaryFile/NexSCRM项目描述(web商家后台).md` —— web-merchant 端历史/补充描述（部分与迭代001 设计存在差异，见规则 #13~#17 待确认）。
- web-admin 端暂仅提供需求描述（见 `platforms/web-admin/` 各模块 requirement.md）。
