# Acceptance Tests (extract)

Extracted verbatim from section 7 of the canonical design (`references/design-v0.2.md`). Test 2 is a release gate: if it fails, the implementation must not ship. Tests are numbered 1-22 and are referenced by number throughout this package.

1. 建会话载荷多塞一个未知键 → 400 且数据库无会话行、无快照行（以行数与哈希双验）；未知键塞在 `rules`/预载清单等**嵌套层级** → 同样 400 并列出路径。
2. **[放行前置——本条不过，整个实现不得放行]** `enable: []` 建会话 → 成功但该会话任何业务工具执行均被拒；把同一载荷的 `enable` 删掉重建 → 行为与空数组**完全一致**（absent=empty 归一化回归）；两次回执均回显空有效工具集与快照哈希（§6.1）。
3. 引用不存在/已归档/他人作用域的策略建会话 → 分别 404/400/403 且零会话行。
4. 人为损坏一条已存会话的快照（改一字节）→ 运行期加载校验失败 → 会话 blocked、零工具可执行、告警发出；任何重试都不产生无限制执行。
5. 两并发更新同基线版本 → 恰一成功；失败者 409，且会话配置的字节级内容与胜者写入前相比除胜者变更外零变化，历史无失败者条目。
6. 回滚：以 v3 内容生成 v7 → 历史链 1..7 完整，v7 快照与 v3 一致（哈希同）。
7. OAuth 固化演练（operator 侧）：U 发起、V 登录赎回 → 连接 FAILED、封存凭据销毁、安全事件含双方 id；同一 session_uri 再赎回 → 410。
8. OAuth 正常路：U 发起、U 赎回（含 §4.2 步骤 6 的明示确认）→ ACTIVE；provider 中途 5xx 一次 → 状态不判死、TTL 内可重试并最终成功（分层错误不误杀）。
9. 搜段隔离：在只授予 A 工具包的会话里搜 B 工具包工具名 → 结果不出现 B 的任何条目；直接 fetch B 工具 schema → 403。
10. 拒绝即停止演练：连接被吊销后执行 → 拒绝上报链完整，执行面无任何替代工具调用记录、无自动改绑记录。
11. 策略 `disable` 引用不存在的工具名 → 激活被拒 `policy_rule_unresolved`、状态不变；`enable` 侧悬空 → 激活通过但告警审计事件＋回执明示齐全。
12. 策略收缩出 v4（v3 已标 `security_contracted`）后：普通调用方新建会话显式钉 v3 → 400 `policy_version_not_pinnable`；审计角色钉 v3 → 同样被拒；审计角色钉未标记的普通历史版本 → 成功且会话带 `audit_repro` 标记、审计事件齐全。
13. 非所有者 PATCH 他人会话配置 → 403 `session_not_owner` 且零写入（配置哈希不变、无历史条目）；无 policy_admin 角色调用 activate → 403 `admin_role_required`。
14. 会话执行时指向未绑定连接 → 403 `connection_not_in_snapshot`＋审计；指向他人名下连接 → 同上；绑定内连接被吊销后执行 → 拒绝且不自动改绑，快照绑定清单不变。
15. A 发起、B 完成同意、A 赎回：(a) 无预声明时，赎回页明示 B 的账号标识、A 显式确认后才 ACTIVE，确认留痕；(b) 有预声明且标识不一致 → FAILED＋凭据销毁＋安全事件；(c) B 的账号命中另一已知我方用户 → 步骤 4 即 FAILED、安全事件先行、无可用凭据封存。
16. PENDING_REDEMPTION 态二次回调（不同 code＋同一 state）→ 封存凭据哈希不变、状态不变、审计有二次回调记录、回调方得 410 语义。
17. execute 入参含 schema 未声明字段 → 422 零副作用。
18. 会话中途工具 schema 变更 → execute 要求重取或按钉版执行（按实现定版断言），不静默漂移。
19. 非幂等工具执行「已执行、响应丢失」后以同幂等键重试 → 只执行一次、结果可重查。
20. 按 policy_id 批量 blocked → 该策略全部未终结会话 503、零工具可执行、告警与审计齐全、快照内容逐字节不变。
21. refresh token 复用（旧 token 重放）→ 检测告警＋连接置 revoked/待复核；revoke 后以连接 id 读凭据 → 不存在（销毁可验证）。
22. verifier 页与全链路日志检查：session_uri 明文不出现在访问日志、代理日志与任何第三方请求中；verifier 页无第三方子资源且带 `Referrer-Policy: no-referrer`。
