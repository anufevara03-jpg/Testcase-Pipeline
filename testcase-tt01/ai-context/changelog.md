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
