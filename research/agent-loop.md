# 调研记录：Agent 主循环与编排（docs/agent-loop.md 的事实依据）

事实基线：上游 main `8d30c4e`（2026-09-29）。本文档是写作依据，不进正文。
审核时拿正文断言对照本记录的「源码依据」逐条核对。

## 一、官方 claim 清单与验证结果

| # | 官方说法 | 来源 | 验证结果 |
|---|---------|------|---------|
| 1 | 主循环完全同步，while-loop，带中断检查、预算、one-turn grace call | agent/AGENTS.md | ✅ conversation_loop.py:1636 循环条件一致 |
| 2 | `run_agent.py` 是门面，AIAgent 由 mixins 组装，构造走 init_agent | agent/AGENTS.md | ✅ run_agent.py:241 AIAgent(14 个 Mixin) |
| 3 | 每个 turn 阶段一个 sibling 文件，共 31 个 turn_*.py | agent/AGENTS.md | ✅ `ls` 确认 31 个 |
| 4 | max_iterations 默认 500（agent.max_turns 配置） | developer-guide/agent-loop.md | ⚠️ **分层默认**：AIAgent 构造默认 `sys.maxsize`（无限，run_agent.py:269），500 是 CLI 应用层默认（hermes_cli/cli_config_load.py:201 `"max_turns": 500`） |
| 5 | subagent 预算独立，delegation.max_iterations 默认 50 | developer-guide + iteration_budget.py docstring | ⚠️ **过时**：tools/delegate_tool.py:71 `DEFAULT_MAX_ITERATIONS = 250`（config 可覆盖；caller 传值被忽略，config 权威）。docstring 的 50 未更新 |
| 6 | 子代理独立预算，父子总量可超父上限 | iteration_budget.py | ✅ 每个子代理独立 IterationBudget |
| 7 | 中断时 API 线程被放弃、响应丢弃、无部分响应进历史 | developer-guide | ⚠️ **不完全**：中断时若已有流出文本，部分文本**会**作为 assistant 行持久化（turn_api_call.py:194-208，供下轮参考）；丢弃的是未完成的 HTTP 响应本身。redirect 交叉时不丢 redirect |
| 8 | 单 tool call 主线程执行；多个并发 ThreadPool，interactive 强制顺序，结果按原序回填 | developer-guide | ⚠️ **已演进**：run_agent.py:1326 `_execute_tool_calls`——≤1 个顺序执行；多个走**分段规划**（tool_dispatch_helpers 的 segment planner：并行安全段 + 顺序屏障），全并行安全才并发 |
| 9 | 三种 API mode（chat_completions/codex_responses/anthropic_messages），统一收敛 OpenAI 内部格式 | developer-guide | ✅（本文只作背景） |
| 10 | Agent 级工具由 tool_executor 拦截，在 handle_function_call 之前 | agent/AGENTS.md | ✅ inline_tool_executors.py:237 表共 15 个 |
| 11 | 严格角色交替，唯一例外 /steer（tool result 后独立 user row） | agent/AGENTS.md | ✅ turn_iteration_prep.py:265 |
| 12 | 系统 prompt 会话内字节稳定，唯一 mutation 是压缩 | agent/AGENTS.md | ✅ _restore_or_build_system_prompt 从 session row 逐字节恢复（conversation_loop.py:799-801） |
| 13 | 100% 预算时停下返回工作摘要 | developer-guide | ⚠️ 近似：预算耗尽 break 后 finalize_turn 有预算回退解析与 abnormal-exit 解释；"摘要"实为多种合成文案之一 |
| 14 | 压缩生成新 session lineage ID（"child" session） | developer-guide | 与 agent/AGENTS.md "in-place compaction keeps a single stable session id" 冲突 → 属压缩篇（第 3 篇）议题，本文不展开，只提"轮换会话恢复"存在于 turn 前置 |

## 二、机制事实清单（正文断言依据）

### 入口与准入
- 两层入口：`chat()` 取 final_response 的薄包装（turn_facade.py:218）；`run_conversation()` 返回完整 dict。门面 run_conversation 先取消后台 review、拿 **durable 跨进程会话轮次租约 (turn lease)** + 续约/看门狗定时器（turn_facade.py:82 `admit_durable_turn_lease`）。
- **租约等待语义（第一轮审核核实）**：竞争租约不是立即失败而是阻塞等待——TTL 300s（LEASE_TTL_SECONDS）、等待上限 1800s（LEASE_WAIT_SECONDS），期间向外发状态提示（turn_facade_lease.py:23-24、:282-288）。等待被用户软中断 → 未消费的用户消息由 carry_unadmitted_user_message 带入 early_result 供下次投递（:341-352，仅 interrupted 且非硬停止时）；等待超时 → early_result 只带调用方传入的历史、提示用户重发（:375-421）。等待实现在 hermes_state_compression.py:588-648。
- 续约与活性看门狗跑在共享周期调度器上，非每轮线程（turn_facade_lease.py:1-8）。
- 门面还设置 relay/观测/accounting 上下文（Portal 标签、token 记账），再转发真正主循环（turn_facade.py:143-156）。
- AIAgent = 14 个 Mixin 组装（run_agent.py:241-244）；run_agent.py 是门面文件。

### turn 前置（build_turn_context，turn_context.py:989）
- 顺序敏感：DB session row 在系统 prompt 恢复/构建**之后**才建（否则存 NULL prompt 导致下轮 cache miss，注释 #45499）；在 preflight 压缩之前。
- 恢复轮换过的压缩会话（recover_rotated_compression_session）。
- 系统 prompt：会话库有存储且 runtime 身份（model/provider/cwd/session id）匹配 → **逐字节复用**；失配才重建并记录 cache break。工具列表 pin 到会话已发顺序（tools freeze，mcp_tool_agent）。
- 用户消息 append；图像 token 单价绑定（按该模型真实 usage 学到的成本）。
- preflight 压缩 token 估算优先级：provider usage anchor > native Responses 剪枝估算 > 粗估；粗估被 last_real_prompt_tokens 下限兜底（非 ASCII 低估 ~2x，防静默截断死亡螺旋，conversation_loop.py:396-423）。

### 主循环（conversation_loop.py:1636-1692）
- 条件：`(api_call_count < max_iterations and iteration_budget.remaining > 0) or _budget_grace_call`。
- 阶段顺序：begin_iteration → prepare_iteration → assemble_api_request → run_preflight_gate → announce_api_call → 内层重试循环 → apply_retry_restarts → normalize_model_response →（run_tool_round | finish_text_response）→ 循环；退出后 finalize_turn。
- `_LoopState` dataclass 打包全部循环局部变量；`_run_phase` 按 helper 函数签名**按名反射传递/回收**字段（conversation_loop.py:1384-1498）——31 个阶段文件是无状态 helper，verdict（action: fallthrough/continue/break/return）驱动控制流。
- MoA（mixture-of-agents）与 codex_app_server 是循环旁路：后者把整个 turn 交给子进程 runtime，失败才回落通用循环（conversation_loop.py:1622-1634）。

### 迭代开始（begin_iteration，turn_iteration_prep.py:333）
- 顺序：drain pending redirect（应用为改偏）→ checkpoint dedup 重置 → 中断检查（break）→ review 输入预算检查（break）→ **api_call_count += 1 + touch activity →（先计数）** grace call 消费标志（不消费预算）else 预算 consume（失败 break）。计数在消费预算之前。
- prepare_iteration：step 回调（gateway agent:step 事件）→ /steer 排水为独立 user row 插在最新 tool result 后（无 tool result 则 requeue 等下个时机）→ run-budget 80% wrap-up notice → 迭代预算警告（opt-in ratio，注入最新 tool result 尾部）→ tool 参数消毒（增量 cursor）→ 隐藏 ghost 行过滤 → 角色交替修复（修复后**重锚定**当前 turn user 行索引）。

### steer / redirect / stop 三分（第一轮审核核实，interrupt_control.py:250-303）
- `steer()`：排队不打断；当前工具批次结束后作为独立 user row 落在最后一条工具结果后；多次调用换行拼接。工具执行期间的 redirect 也退化为 steer（绝不杀在跑的工具，还会请求工具线程让路 yield）。
- `redirect()`：仅模型请求期间生效——取消该次在途请求（已完成消息保留、partial reasoning 成为 assistant 上下文、修正作为真实 user 行追加、循环重建重试）；响应已完成则排队新 turn。
- redirect 重建时追加「助手占位行 + 带脚手架的 user 行」（脚手架在 api_content，conversation_loop.py:320-357）。
- 「唯一合成 user 消息」说法不成立：空响应提示、截断续写、Codex 只思考提示、预算总结请求都是合成 user 行；steer/redirect 是仅有的**用户发起**合成行。

### 请求组装与 preflight 闸门
- 历史 → API 副本做**结构克隆**（写路径重写永不触及持久 transcript，#80498）；tool-call 参数 JSON 规范化（canonicalize + memo 缓存，malformed 走修复）；Anthropic cache 计划打标（strip + rebuild per provider）。
- preflight gate（turn_preflight_gate.py）：Ollama runtime context 下限 → provider 溢出重查 arming → insufficient-progress blocker（比较完整组装的请求而非原始消息）→ 压缩调用。压缩是唯一被允许的上下文 mutation。

### API 调用（turn_api_call.py:68）
- 流式决策：**默认偏好流式**（即使无消费者，为了 stale-stream/read-timeout 健康检查）；ACP/外部进程 provider、无显示消费者的 MoA、测试 Mock 关闭。
- 可中断：HTTP 在 worker 线程；主线程等 response-ready / interrupt / timeout 三事件；**每请求独立 client**（中断只关那一个）；stale-call 检测器杀连接并抛错进重试循环（chat_completion_helpers.py:1293）。
- LLM 执行中间件包装；call_role 元数据 primary/fallback/delegated。
- **redirect 交叉检查**：响应与用户纠偏竞态 → 丢弃 stale 响应、armed restart、从纠偏重建（turn_api_call.py:148-156）。

### 响应分支（turn_response_intake / turn_final_response）
- 文本 → finish_text_response；tool_calls → run_tool_round。
- 病态响应的合成 nudge（re-prompt）：截断续写（boost 输出上限 ladder + ephemeral reasoning-off 一次调用防思考吃掉输出）、空响应 after tools、Codex 仅 reasoning / ack-only、碎片化收尾。nudge 走 user 侧合成提示，不是改系统 prompt。
- InterruptedError：redirect 引起 → armed restart；真中断 → **已流出文本持久化为 assistant 行**（供下轮参考），runaway repetition 字节被替换为隐藏占位（防循环自播）。

### 工具轮（turn_tool_round.py:46）
- 顺序：validate（无效名字 → error tool result；混合批次只执行合法的，但 assistant 行保留全部发射）→ 去重 + delegate 数量上限裁剪 → **persist-before-execute**（增量持久化 tool-call 轮；失败 → turn fail，绝不从内存态执行工具）→ interim commentary（必须在 DB append 之后才可见）→ 执行 → 工具结果持久化失败检查 → guardrail halt（identical call streak / batch-cycle）→ post-tool 压缩 → touch activity（防 gateway 不活动超时误杀，#69559）。
- 参数修复语义（第一轮审核核实）：请求前消毒器把修不了的参数就地改写成空对象 + 补带标记的工具结果（agent_runtime_helpers.py:232-305）；校验环节对非法 JSON 先重试整次调用最多 3 次，仍不行注入错误工具结果让模型自纠（助手消息保留，turn_tool_validation.py:169-224）——不是"丢弃"。
- 执行编排（run_agent.py:1326, tool_dispatch_helpers.py:198-244, tool_executor.py:1895-1919）：≤1 顺序；多调用由 segment planner 切成有序的 (parallel|sequential, calls) 段——能并发的聚成 parallel 段（<2 个降级 sequential，相邻 sequential 合并），段按顺序执行，每段跑完（concurrent executor join 全部 future）才进下一段＝屏障。planner docstring 原话：实际执行的调用顺序与副作用边界和全串行执行完全一致（"a later call never crosses an earlier barrier"），并行只压缩段内时间。结果按 tool_call 原序回填。
- **并行安全判定（tool_dispatch_helpers.py:28-53, 168-195 核实修正）**：保守白名单制。`_NEVER_PARALLEL_TOOLS`（clarify、manage_connections、manage_catalog）永不并发；`_PARALLEL_SAFE_TOOLS` 只读白名单（read_file、search_files、web_search、web_extract、session_search、skill_view、skills_list、vision_analyze、image_generate、connectors__execute、ha_*）；文件类路径准入（_PATH_SCOPED_READERS={read_file,search_files}、WRITERS={write_file,patch}）：reader↔reader 同子树可同段，writer 与任何重叠关段开新段；MCP 工具按 server 声明 opt-in；参数 JSON 解析失败/非 dict → 串行屏障；其余（terminal、execute_code、todo 等）默认串行。段形成（_plan_tool_batch_segments:198-244）：并发段 <2 个调用降级串行、相邻串行段合并。~~破坏性终端命令模式识别~~（更正：`_is_destructive_command` 是写前文件备份 checkpoint 用的，tool_executor.py:1027，与并行判定无关）。
- agent 级内联工具 15 个（inline_tool_executors.py:237）：todo_list、message_agent、session_search、memory、clarify、read_terminal、desktop_preview、drive_preview、annotate_preview、read_window_below、gui_tour、manage_connections、manage_catalog、setup_mcp、delegate_task——需要 live AIAgent 状态，绕过 registry。
- 裁剪细节（run_agent.py:1201-1237）：**去重**——同一批内 (tool_name, canonicalized_args) 相同的调用只留第一份；参数先规范化（合法 JSON 重排序键、压缩空白）再比较，键序/空白差异绕不过去重。**delegate 上限**——一批内 delegate_task 数量超过 max_concurrent_children 的直接丢弃（其余非 delegate 调用全保留）。裁剪发生在 stage/persist 之前，被丢的调用不进落库的助手行，所以不产生「有调用无结果」的协议问题。
- persist-before-execute 的准确语义（turn_tool_round.py:53-56, 118-141）：先把含这批 tool call 的助手行增量写进会话库，写成功才开始执行；执行中崩溃/被破坏性工具重启，resume 能看到「这批调用已发过」（最多缺结果行）。落库失败 → 整轮失败（exit_reason=session_persistence_failed），绝不从纯内存状态执行——保证永不出现「库里没记录、实际执行了」。
- 崩溃后悬空调用（有调用、无结果）的恢复处理：**不重执行**（transcript 无重放语义）。清理在下一轮请求前，两层：
  1. `repair_message_sequence`（turn_iteration_prep.py:203-204 每圈请求准备时跑，docstring 明说覆盖 resumed histories）的 `_prune_unanswered_tool_calls` pass（agent_runtime_helpers.py:552-588）：immediately-following tool run 没应答的调用从助手行剪掉；剪完无 payload 的助手行整条删（空助手行 provider 400）；原地改写后重新落库。
  2. `_sanitize_api_messages`（turn_request_assembly.py:148，每次调用前跑请求副本）的 `_pair_tool_calls_positionally`（agent_runtime_helpers.py:2858-2929）：positional pairing——请求副本里漏网的 unanswered call 补 stub 结果 "[Result unavailable — see context summary above]"；不紧跟其调用的结果丢弃（positional orphans）；`_dedupe_tool_call_ids` 注释明说 crash/resume glitches 是重复 id 来源之一，此层兜底。
  另：压缩侧 `_sanitize_tool_pairs`（context_compressor.py:4482-4520）对尾部 in-flight 助手行的未应答调用保留原文（执行器还没来得及贴结果），也靠 pre-API stub 兜底。
- 心跳的准确机制（turn_tool_round.py:211-220 注释 + activity_tracking.py:41-89）：gateway 有不活跃看门狗 HERMES_AGENT_TIMEOUT（默认 1800s），agent 的活动时间戳超过该窗口没刷新就杀会话（#69559、#69131）。时间戳不是进程活着就自动刷，只在事件发生时由 _touch_activity 打点（发消息、收响应、贴结果等）。危险窗口：工具结果贴回 ~0s → 随后的压缩评估/落库/下一圈慢模型调用期间无任何事件打点 → 看门狗误杀健康会话。修法：贴回结果后立刻 _touch_activity 一次再进下一圈。_touch_activity 同时刷内存时间戳（看门狗读）+ 限速 60s 的 SessionDB 持久活动行。
- 工具异常归一化：异常/超时/取消都归一为托管结果类型（_ManagedToolResult/_ToolTimeoutResult/_ToolCancelledResult）回填模型，错误文本有长度上限；只有落库失败或护栏终止工具轮。
- delegate_task：顶层委托后台执行（句柄返回、结果以消息回流）；嵌套 orchestrator 同步。
- housekeeping 工具轮（memory/todo_list/skill_manage/session_search）静音 tool progress。
- execute_code-only 轮退预算（只退预算计数器，不退 api_call_count）。

### 错误恢复（turn_api_error.py:52）
- 内层重试循环：nous 限流 guard → build → call → check；异常 → handle_api_error。
- **跨会话限流记忆是 Nous Portal 限定**（第一轮审核核实）：共享文件机制只在 provider == "nous" 时检查（turn_api_call.py:241-290；nous_rate_guard.py 模块头自述）；其他 provider 走凭证池/降级链。
- 错误分类器输出：reason / retryable / should_compress / should_rotate_credential / should_fallback。换 key 机制（credential_pool.py「持久化的同服务商多凭证池」）：同一 provider 配多把 API key，当前 key 出问题（429 限流、402 欠费、401/403 鉴权、模型没权限）就换池里下一把，坏 key 标记冷却/耗尽。判定表（error_classifier.py:455-468）：billing=不重试+换key+换服务商；rate_limit=退避重试+换key+换服务商；auth=不重试+换key+换服务商；content_policy=不重试+可换服务商（换 key 没用，请求本身被拦）。「不重试 ≠ 直接结束」：换 key/换 provider 走完才算无路可走。
- 两层数字的准确出处（核实修正）：**内层重试上限默认 3**（agent_init.py:1450 `api_max_retries` 可配），`_run_api_retry_loop` while retry_count<max_retries（conversation_loop.py:1506）；换备用 provider 成功激活、传输层一次性自愈、Copilot 凭证自愈都会把 retry_count 清零（turn_api_error.py:327/363/377/388）。**外层 8**（conversation_loop.py:235 `_MAX_OUTER_LOOP_ERRORS = 8`）是逃逸异常兜底：只接分类器没接住的异常（多半是本地处理 bug），缝补（ unanswered tool call 补错误结果）后让循环重来；外层错误数 ≥ min(8, max_iterations) → repeated_outer_errors 终止轮次；本地确定性错误（local_processing_error）立即终止不烧预算（turn_loop_errors.py:149-170）。**自动恢复梯子**（turn_recovery_autorecover.py）：重试耗尽 + fallback 链空 + 临时性错误（overloaded/server_error/timeout）+ 用户还没收到答案 → 挂起轮次倒数等待，默认 5 轮（agent_init.py:1457 `auto_recovery_cycles`），schedule 15/30/60/60/60s 带抖动，Retry-After 优先（≤120s），每轮回来 retry_count=0 重新进重试循环；等待可见可中断。
- 恢复管线：分类前恢复 → 分类后恢复（凭证轮换/池）→ 路由（fallback 激活）→ 溢出恢复（压缩）→ 终局 settle。
- **每次 fallback 激活必须以 restart_with_rebuilt_messages 离开重试循环**（让 preflight 对 fallback 的上下文窗口重跑，#84733）；401/403 先试凭证刷新再 failover。
- restart 退款有上限：restart_count（per-turn 累计）≤ max_retries，防 runaway redirect/rebuild 无限退款持有租约（turn_iteration_prep.py:448-527）。
- 外层异常上限 8 个/turn（#92450）；billing / content-policy / interpreter-shutdown 不可重试直接终局。

### 预算（iteration_budget.py）
- 线程安全 consume/refund 计数器；每个 agent（父/子）独立。
- 默认值分层：AIAgent 构造 sys.maxsize（无限）→ CLI 层 500（agent.max_turns 配置）→ 子代理 delegation.max_iterations 默认 **250**。
- grace call：**置位条件很窄**（第一轮审核核实，turn_truncation.py:621-624 唯一置位点）——Codex Responses 路由 reasoning-only streak≥3 触发 fallback 激活、且此时 api_call_count≥max_iterations 或预算耗尽，给 fallback 一次额外迭代（否则循环在 fallback 得到机会前退出）；消费后是**普通迭代**（工具照常执行，turn_iteration_prep.py:387-398）。
- 真正「预算耗尽让模型说完」的通用机制在收尾：`_handle_max_iterations` 补发一次总结请求（turn_finalizer.py:149-169）；chat 主路由总结**带 tools**（保持缓存前缀一致，SGLang 在 tools=None 下 KV 前缀发散，chat_completion_helpers.py:2339-2347），但工具调用被丢弃不执行（:2298-2308）；Codex 摘要弹掉 tools（:2311-2316）、Anthropic 摘要 tools=None（:2326-2331）。
- 退款点（第一轮审核补充）：不止四类——起飞前压缩（turn_preflight.py:111/154/179、turn_context_compaction.py:102-113）、空响应后降级（turn_empty_response.py:287）、execute_code-only 轮（只退预算不退计数，turn_tool_round.py:188-191）、redirect/rebuilt 重启（计数+预算一起退）。原则：没到 provider 或不算真实模型轮次的都退。
- 重启上限归属：`restart_count`（per-turn 累计）只覆盖 redirect 重启与 rebuilt/fallback 重启；压缩重启只 `retry_count += 1` 无独立重启上限（turn_iteration_prep.py:448-527）。

### 验证闸门（第一轮审核新核实，turn_stop_gates.py / verification_stop.py）
- 模型纯文本停止时三类闸门可拦截：verify-on-stop（#65919）、pre_verify 插件钩子、kanban worker 终端工具守卫。
- 拦截动作：答案降级为 interim 行 + 合成 user nudge 继续循环；候选答案保留为 `pending_verification_response`（预算耗尽时的兜底 final）。
- verification_stop 是 policy-only：编辑代码后无新鲜验证证据就收尾时，把被动验证台账转为有界后续追问；纯文本类文件（.md 等）豁免。

### 重试退避（第一轮审核新核实，retry_utils.py）
- jittered backoff：min(base*2^(attempt-1), max) + uniform jitter（base 5s、cap 120s、jitter_ratio 0.5），防止多会话同时撞限流的重试风暴（decorrelated retries）。
- 解析 Retry-After（数值/HTTP 日期，双大小写头）优先遵从；特定 provider（Z.AI 429/1305）3 次后进入加宽阶梯 (30/60/90/120s)。等待可中断。

### 收尾（turn_finalizer.py:509）
- 预算回退解析（verification gate 挂起的答案作为 fallback final）。
- **silent stop 修复**：非中断轮掉出循环、尾部是 tool result 无 assistant 文本 → failed + 合成可见收尾（防 UI 空转与 tool→user 违例，#55316）。
- completed 判定：final_response 非 None + 不 failed + 不 interrupted +（未超迭代 或 以文本响应退出）；**在清理与持久化之前算出**（轨迹保存要用，turn_finalizer.py:566-577 vs :584-621）。
- 收尾清理全部 fail-open（延迟标题升级、轨迹、任务资源）；persist step 顺序固定：drop scaffolding → **流文本恢复**（用户已看到的不丢，#95514）→ 输出变换 → transcript 尾部关闭 → micro-compaction → 持久化。
- **记忆与钩子**（第一轮审核补核）：`_sync_external_memory_for_turn` 同步完成轮次给外部记忆 provider + 排队下次预取（:746-747）；后台记忆/技能复盘在回答交付后异步跑（`_should_review_memory`/`_should_review_skills`，:752-764，文件头自述 memory/skill review）；on_turn_complete 通知 context engine（fail-open）；surrogate 清理 chokepoint。
- 失败轮 durable 尾为 user 时补 assistant 边界行（幂等，_close_durable_failed_turn，conversation_loop.py:1745；**上下文压力类除外**——compression_exhausted/compression_deferred/context_overflow 不追加，防增长循环）。

### 消息不变量（贯穿）
- 内部统一 OpenAI 风格消息格式；reasoning 存于 assistant 消息附带字段。
- 严格角色交替：不允许连续同角色（tool 并行结果除外）；/steer 是唯一「合成 user 行」例外。
- 系统 prompt 字节稳定；中途注入只走 user 消息或**当前** tool result 尾部追加（旧行已缓存不可动）。
