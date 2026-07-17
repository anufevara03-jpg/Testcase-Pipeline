# 变更日志

> 记录知识库的重大变更。格式：`[YYYY-MM-DD] [{端名或 shared}] {模块} - {变更摘要}`

- [2026-07-17] [shared] 全局 - 初始化多端知识库骨架（预创建 pc-client 端）。
- [2026-07-17] [shared] 全局 - 基于《SCRM迭代001_数据库设计文档》更新共享数据模型与业务规则（规则 #1~#17）。
- [2026-07-17] [shared] 全局 - 更新系统总览、术语表、角色权限矩阵为三端架构（web-admin / web-merchant / pc-client）。
- [2026-07-17] [pc-client] 平台概览 - 基于《NexSCRM项目描述（桌面应用）》填充。
- [2026-07-17] [web-merchant] 平台概览 - 基于 DB B 域创建（含历史描述差异说明）。
- [2026-07-17] [web-admin] 平台概览 + 模块 - 新建管理端及 6 个需求模块（商家管理、商业化套餐、社交平台接入、接口接入、商户端UI管理、数据指标）。
- [2026-07-17] [shared] 全局 - 落地防幻觉铁律（铁律 A 高危内容逐项溯源 + 铁律 B 写前闸门），写入 `CLAUDE.md` 与 `SKILL.md §5.1`。
- [2026-07-17] [web-merchant] activation-code-management - 基于《激活码管理模块.md》新建模块 5 文件；高危内容逐项溯源，接口参数/错误码/测试数据待补充。
- [2026-07-17] [shared] 全局 - 新建 `ai-context-ingest` skill（文档解析入库，只提取+移交），扩展 `ai-context-maintainer` 增加"按提取报告写入"任务类型（§3.7），登记两 skill 协作关系到 `CLAUDE.md`。
- [2026-07-17] [web-merchant] material-library - 按《话术素材库模块.md》提取报告写入（ingest→maintainer 端到端测试）；C1 类型枚举按模块文档三类为准，C2/C3 标未证实。
- [2026-07-17] [web-merchant] customer-profile-management - 按《客户画像管理模块.md》提取报告写入；字段定义/枚举值/数值约束（分组名30字、字段名10字）逐项溯源；接口详情、数据库列类型、国家枚举列表标待补充；销售信息分组命名（§二销售信息 vs §一§3销售意向）标未证实待确认。
- [2026-07-17] [web-merchant] quick-reply-management - 按《快捷回复管理模块.md》提取报告写入（新建模块 5 文件）；data-model 用真实表 zs_scrm_reply_quick(_group)、列类型溯源 DB §B7.3/B7.4；启用/禁用、≤5 素材、适用激活码、分组维度等与 DB 结构差异 C-Q1~Q7 标未证实待确认；接口参数/错误码/测试数据待补充。
- [2026-07-17] [web-merchant] material-library - 按《话术素材库模块.md》补 §五.4 移动分组/批量移动分组（目标分组=当前素材类型下已有分组）至 requirement.md / ui-spec.md。
- [2026-07-17] [web-merchant] keyword-reply-management - 按《关键词回复管理.md》提取报告写入（新建模块 5 文件）；data-model 用真实表 zs_scrm_reply_keyword(_group)、列类型溯源 DB §B7.5/B7.6；C-K1~K11 标未证实待确认；接口参数/错误码/测试数据待补充。
- [2026-07-17] [web-merchant] keyword-reply-management - 用户确认 C-K2「翻译后发送」机制（请求三方翻译接口将原文翻译成译文再发送，源：用户提供），回填至 requirement/data-model/test-notes；C-K3/C-K4/C-K5 维持待确认。
