# 管理（manage）表设计方案：日志与关联

## Summary
- 在现有项目表设计规范（统一审计字段 + 逻辑删除 + 多租户）基础上，新增 4 张以 `manage_` 为前缀的表：
  - 对话日志表（AI 对话消息级别）
  - 接口调用日志表（对外/对内调用日志，支持关联到对话消息）
  - 活动类型与通用图片关联表（活动类型多态 + 通用图片 file_id）
  - 用户-活动-规则关联表（多态关联：`*_type + *_id`）

## Current State Analysis
- 项目已有统一基础字段规范（审计/逻辑删除/租户）：
  - [BaseDO.java](file:///workspace/yudao-framework/yudao-spring-boot-starter-mybatis/src/main/java/cn/iocoder/yudao/framework/mybatis/core/dataobject/BaseDO.java#L24-L66)：`creator/createTime/updater/updateTime/deleted`
  - [TenantBaseDO.java](file:///workspace/yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/db/TenantBaseDO.java#L12-L21)：`tenantId`
- 项目已有日志表设计样例，可复用字段与索引习惯：
  - API 访问日志：`infra_api_access_log` [ruoyi-vue-pro.sql](file:///workspace/sql/mysql/ruoyi-vue-pro.sql#L20-L53)
  - 异常日志：`infra_api_error_log` [ruoyi-vue-pro.sql](file:///workspace/sql/mysql/ruoyi-vue-pro.sql#L60-L96)
  - 操作日志：`system_operate_log` [ruoyi-vue-pro.sql](file:///workspace/sql/mysql/ruoyi-vue-pro.sql#L3338-L3366)
- 关联表样例（同样包含审计/租户/逻辑删除字段）：`system_role_menu` [ruoyi-vue-pro.sql](file:///workspace/sql/mysql/ruoyi-vue-pro.sql#L3438-L3454)

## Proposed Changes
### 1) 新增表：manage_chat_log（对话日志表，AI 消息级别）
**目的**
- 记录用户与 AI 助手的每一条消息（可用 `conversation_id` 串起同一次会话），并记录模型、token、耗时与错误信息，便于审计与问题定位。

**字段设计（MySQL）**
- 业务主键
  - `id` bigint auto_increment：日志主键
- 会话/链路
  - `conversation_id` varchar(64)：会话编号（同一次对话的多条消息共用）
  - `trace_id` varchar(64)：链路追踪编号（对齐项目日志习惯）
- 用户
  - `user_id` bigint：用户编号
  - `user_type` tinyint：用户类型（对齐现有日志表的 `user_type` 语义）
- 消息主体
  - `role` tinyint：消息角色（1 user / 2 assistant / 3 system）
  - `content` longtext：消息内容（建议保留原始文本）
  - `content_type` tinyint：内容类型（1 text / 2 markdown / 3 json 等）
- 模型与消耗（AI 场景）
  - `model` varchar(64)：模型标识（如 gpt-4o-mini / deepseek-chat）
  - `prompt_tokens` int、`completion_tokens` int、`total_tokens` int：token 统计
- 处理结果
  - `status` tinyint：状态（0 成功 / 1 失败）
  - `error_code` int：错误码
  - `error_msg` varchar(512)：错误信息摘要
- 请求环境（可选但建议）
  - `user_ip` varchar(50)
  - `user_agent` varchar(512)
- 计时
  - `begin_time` datetime：开始时间
  - `end_time` datetime：结束时间
  - `duration` int：耗时（ms）
- 通用字段（按项目规则）
  - `creator` varchar(64)、`create_time` datetime
  - `updater` varchar(64)、`update_time` datetime
  - `deleted` bit(1)
  - `tenant_id` bigint

**索引建议**
- `idx_create_time(create_time)`
- `idx_conversation_id(conversation_id)`
- `idx_trace_id(trace_id)`
- `idx_user_id(user_id)`

**DDL 落点**
- 将表结构追加到 [ruoyi-vue-pro.sql](file:///workspace/sql/mysql/ruoyi-vue-pro.sql) 中，按现有风格增加 `-- Table structure for ...` 区块。

### 2) 新增表：manage_api_invoke_log（接口调用日志表）
**目的**
- 记录系统在对话处理过程中发生的“对外/对内接口调用”（包含模型调用、第三方 HTTP、内部 RPC 等），并可关联到 `manage_chat_log` 某条消息，便于排查耗时与失败原因。

**字段设计（MySQL）**
- 主键
  - `id` bigint auto_increment
- 关联
  - `chat_log_id` bigint：关联的对话日志 id（0 表示无关联）
  - `conversation_id` varchar(64)
  - `trace_id` varchar(64)
- 用户
  - `user_id` bigint
  - `user_type` tinyint
- 调用对象
  - `invoke_type` tinyint：调用类型（1 LLM / 2 HTTP / 3 内部服务 / 4 其他）
  - `application_name` varchar(50)：应用名（对齐 `infra_api_access_log`）
  - `service_name` varchar(100)：服务/提供方名称（如 openai / qwen / internal-user-service）
- 请求信息
  - `request_method` varchar(16)
  - `request_url` varchar(255)
  - `request_headers` text
  - `request_body` longtext
- 响应信息
  - `response_status` int
  - `response_body` longtext
- 结果
  - `success` bit(1)
  - `result_code` int
  - `result_msg` varchar(512)
- 计时
  - `begin_time` datetime
  - `end_time` datetime
  - `duration` int
- 通用字段（按项目规则）
  - `creator/create_time/updater/update_time/deleted/tenant_id`

**索引建议**
- `idx_create_time(create_time)`
- `idx_conversation_id(conversation_id)`
- `idx_chat_log_id(chat_log_id)`
- `idx_trace_id(trace_id)`

### 3) 新增表：manage_activity_type_image_rel（活动类型与通用图片的关联表）
**目的**
- 维护“活动类型 → 通用图片”的可配置映射，用于前端展示或活动配置默认图。

**关联策略（多类型混用）**
- 使用 `activity_type` 描述活动类型，不直接依赖某一张活动主表。
- 图片使用项目通用文件表 `infra_file` 的 `id` 作为 `image_file_id`（仅保存 id，不强制 FK）。

**字段设计（MySQL）**
- `id` bigint auto_increment
- `activity_type` varchar(50)：活动类型（建议与字典/枚举保持一致）
- `image_file_id` bigint：通用图片文件 id（对齐 `infra_file.id`）
- `sort` int：排序
- `remark` varchar(500)
- 通用字段（按项目规则）：`creator/create_time/updater/update_time/deleted/tenant_id`

**索引建议**
- `idx_activity_type(activity_type)`
- `idx_image_file_id(image_file_id)`
- `idx_create_time(create_time)`

### 4) 新增表：manage_user_activity_rule_rel（用户id、活动id、活动规则关联表）
**目的**
- 在“多活动类型、多规则类型”的前提下，为用户保存关联关系（例如：用户参与了某活动，并匹配到某条规则）。

**关联策略（多类型混用）**
- 活动：`activity_type + activity_id`
- 规则：`rule_type + rule_id`
- 不做外键，仅建索引，避免跨模块/跨表强耦合。

**字段设计（MySQL）**
- `id` bigint auto_increment
- 用户
  - `user_id` bigint
  - `user_type` tinyint
- 活动
  - `activity_type` varchar(50)
  - `activity_id` bigint
- 规则
  - `rule_type` varchar(50)
  - `rule_id` bigint
- 状态/扩展
  - `status` tinyint：状态（0 启用 / 1 停用）
  - `remark` varchar(500)
- 通用字段（按项目规则）：`creator/create_time/updater/update_time/deleted/tenant_id`

**索引建议**
- `idx_user_id(user_id)`
- `idx_activity(activity_type, activity_id)`
- `idx_rule(rule_type, rule_id)`
- `idx_create_time(create_time)`

## Assumptions & Decisions
- 表名前缀统一为 `manage_`（用户要求）。
- 对话日志为“用户-AI助手”场景（用户选择）。
- 活动/规则采用多态关联（`*_type + *_id`），不做外键（用户选择）。
- DDL 仅落 MySQL（用户选择），并复用项目现有 DDL 风格与字段命名（snake_case + 审计字段 + `deleted` bit(1) + `tenant_id`）。

## Verification
- 静态核对 DDL 风格：字段命名/默认值/字符集与现有表一致（参考 `infra_api_access_log`）。
- 在本地 MySQL 8 执行新增 DDL，确认无语法错误并可正常创建索引。
- （可选）编写简单插入/查询用例，验证常用索引字段（`conversation_id/user_id/create_time`）查询命中索引。

