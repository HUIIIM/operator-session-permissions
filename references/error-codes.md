# Error Code Registry (extract)

Extracted verbatim from section 6.3 of the canonical design (`references/design-v0.2.md`). Semantics, HTTP statuses, occurrence points, and write guarantees are unchanged. Every rejection in the workflow (SKILL.md S1-S5) must cite a code from this registry; codes must never be invented, renamed, or reused with a different meaning.

| HTTP | 错误码 | 出现点 | 写入保证 |
|---|---|---|---|
| 400 | `session_config_unknown_key` | 建会话/更新含未知键（**含任意嵌套层级**） | 零配置写入（仅安全拒绝日志） |
| 400 | `policy_archived` | 引用已归档/未生效（pending_review）策略建会话 | 零写入 |
| 400 | `policy_empty_scope` | 激活零授予策略 | 策略状态不变 |
| 400 | `policy_rule_unresolved` | 激活时 disable 引用不可解析 | 策略状态不变；错误体列出不可解析引用 |
| 400 | `policy_version_not_pinnable` | 钉非当前 active 版本，或钉带 `security_contracted` 标记的版本 | 零写入 |
| 400 | `preset_limit_exceeded` | 预载工具数超上限 | 零写入 |
| 400 | `session_update_mode_forbidden` | 对 `rebuild_on_expiry` 会话发起版本化更新（§3.4） | 零写入 |
| 403 | `policy_forbidden` | 跨作用域引用策略 | 零写入，不泄露策略内容 |
| 403 | `admin_role_required` | 管理面端点调用方角色不足（§6.2 矩阵） | 零写入，写审计 |
| 403 | `session_not_owner` | 非所有者（且非管理员）PATCH/关闭他人会话 | 零写入，写审计 |
| 403 | `tool_not_in_snapshot` | 取 schema/执行快照外工具 | 零副作用，写审计 |
| 403 | `connection_not_in_snapshot` | 执行指向未绑定/他人名下/已失效连接 | 零副作用，写审计 |
| 403 | `connection_forbidden` | 建会话声明的连接 owner 与会话 owner 不符 | 零写入 |
| 404 | `policy_not_found` | 引用不存在的策略 | 零写入 |
| 404 | `connection_not_found` | 建会话声明不存在的连接 | 零写入 |
| 409 | `session_config_version_conflict` | 更新版本不符 | **零字段写入**，须重读后显式重试 |
| 410 | `redemption_consumed` / `redemption_expired` | 赎回令牌重放/过期；PENDING 态二次回调亦回 410 语义 | 状态不变，写审计 |
| 422 | `tool_args_schema_invalid` | 执行入参未过 schema 校验 | 零副作用，写审计 |
| 503 | `session_config_unavailable` | 运行期配置加载失败；会话被紧急 blocked（§3.6） | 会话置 blocked，不改既有配置/快照 |
