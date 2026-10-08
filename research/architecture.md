# 调研记录：总体架构与设计思想（docs/architecture.md 的事实依据）

事实基线：上游 main `8d30c4e`（2026-09-29）。本文档是写作依据，不进正文。
审核时拿正文断言对照本记录的「源码依据」逐条核对。行号均为该 commit 下实测。

## 一、官方 claim 清单与验证结果

| # | 官方说法 | 来源 | 验证结果 |
|---|---------|------|---------|
| 1 | 同一个 agent 核心跑在 CLI、~20 平台 gateway、TUI、桌面上 | 根 AGENTS.md | ✅ AIAgent 在 gateway/run_turn.py:1249、cron/scheduler.py:2498、tui_gateway/server.py:2608、acp_adapter/session.py:502、gateway/platforms/api_server.py:2378 均为进程内 import + 构造；batch_runner.py docstring：multiprocessing 池并行跑 JSONL |
| 2 | 入口：CLI / Gateway / ACP / Batch / API server / Python 库 | developer-guide/architecture.md | ✅ 同上；api_server 是 gateway/platforms/ 下的内置适配器（HTTP API 即"又一个平台"） |
| 3 | Gateway 是长驻进程，25+ 适配器（内置 + 插件） | architecture.md「Major Subsystems」 | ✅ 内置 8 个（signal、bluebubbles、weixin、whatsapp_cloud、yuanbao、webhook、api_server、msgraph_webhook）+ plugins/platforms/ 22 个包（telegram、discord、slack、matrix、email、feishu、wecom、dingtalk、line、teams、irc、sms、a2a、homeassistant…）≈ 30 个 |
| 4 | 三条数据流：CLI 会话 / 平台消息 / cron 任务 | architecture.md「Data Flow」 | ✅ 平台消息：适配器 → MessageEvent → 鉴权 → 会话键 → 建/取缓存 agent → 进程内跑轮次 → 经适配器送回（gateway/AGENTS.md Shape + run_turn.py）；cron：tick → jobs.json 到期任务 → 新建 agent（无历史）→ 产出投递到目标平台（cron/AGENTS.md；bot_chat_delivery.py、scheduler_prompt.py） |
| 5 | Provider 解析被 CLI / gateway / cron / ACP / 辅助调用共享，(provider,model)→(api_mode,key,base_url) | architecture.md | ✅ hermes_cli/runtime_provider.py:982；cron/scheduler.py、gateway/run.py、tui_gateway、acp_adapter、api_server 均 import |
| 6 | 70+ 工具、28 工具集、终端 7 后端 | architecture.md | ⚠️ 数字过时（文件系统为准）：TOOLSETS 41 项（toolsets.py 实测）；tools/*.py 内注册调用 64 处（59 registry.register + 5 @register）+ 15 个 agent 级内联工具（inline_tool_executors.py:237，见 agent-loop 调研）+ MCP 动态工具；终端后端 7 种 ✅（tools/environments/：local、docker、ssh、modal、daytona、singularity、vercel_sandbox） |
| 7 | 会话存储 SQLite + FTS5、会话谱系、按平台隔离、原子写 | architecture.md | ✅ 每 home 一个 state.db（hermes_state_dbfile.py）；WAL 日志模式，文件系统不兼容（NFS/SMB/ZFS/WSL1…）自动降级 DELETE（hermes_state_wal.py `_WAL_INCOMPAT_MARKERS`）；FTS5 + 自研 CJK 分词扩展（native/fts5_cjk）；谱系与跨进程 turn lease 在 hermes_state_compression.py:529-561 |
| 8 | 两条全局不变量：per-conversation prompt caching 不可破坏（唯一例外压缩）；核心窄腰、能力放边缘 | 根 AGENTS.md | ✅ 缓存不变量见 agent-loop 调研 #12（系统 prompt 从会话库逐字节恢复）；窄腰：_HERMES_CORE_TOOLS 硬编码核心表（toolsets.py:12），根 AGENTS.md Footprint Ladder 六级（扩展现有→CLI 命令+skill→服务门控工具→插件→MCP server→新核心工具垫底）；「每个核心工具随每次 API 调用发送」根 AGENTS.md 原文 |
| 9 | 六条设计原则（prompt 稳定 / 可观测 / 可中断 / 平台无关核心 / 松耦合 / profile 隔离） | architecture.md「Design Principles」 | ✅ 各有源码支撑：平台无关=claim1；profile 隔离=claim10；可中断=agent-loop 调研（中断检查/steer/redirect）；松耦合=registry + check_fn 门控 + ABC（memory provider、context engine 均可插拔 ABC） |
| 10 | 每 profile 独立 home/config/memory/会话/gateway PID，多 profile 并发；multiplex 下一个网关进程服务多 profile | architecture.md + 根 AGENTS.md | ✅ profiles 目录 <home>/profiles/<name>（hermes_constants.py:230-246）；gateway.multiplex_profiles 配置键（gateway/config.py:613、hermes_cli/config.py:1016）；profile = home + 密钥域 + 终端域（根 AGENTS.md Code Shape 规则）；token 锁防两 profile 共用一份凭证（gateway/status.py:1681 acquire_scoped_lock / :1729 release） |
| 11 | cron 是一等 agent 任务（不是 shell 任务），jobs.json，多平台投递 | architecture.md + cron/AGENTS.md | ✅ cron/AGENTS.md：agent 用 cronjob 工具排程；用户用 hermes cron 或 /cron；调度格式 4 种；600s 不活跃看门狗；tick 先推进 next_run_at 再派发（at-most-once）；**一个宿主页网关进程 tick 所有 profile 的任务库，不看 multiplex 开关**（run_goals.py:390：CLI/TUI 自己的循环除外） |
| 12 | Hermes 也能当 MCP server | mcp_serve.py | ✅ `hermes mcp serve`：stdio MCP server，把会话暴露成工具（列表/读历史/发消息/轮询事件/管理审批）给任意 MCP 客户端（Claude Code、Cursor…）。注意：是「会话通道桥」，不是把 agent 本身包装成工具 |
| 13 | 桌面 app 的 serve 后端随 app 退出而死，消息网关独立存活 | gateway/AGENTS.md「Gateway lifecycle」 | ✅ serve 后端 spawn 网关用 detached 方式（start_new_session / DETACHED_PROCESS），app 退出的 SIGTERM 到不了网关 |
| 14 | 测试规模 ~25k（architecture.md）/ ~39k（根 AGENTS.md，Sep 2026） | 两处文档 | ⚠️ 数字漂移，文件系统为准：tests/ 5608 个 .py（本 commit 实测）。正文不写具体数 |
| 15 | 代码规模：~45 万行 Python | 本仓库 README（上游 clone 时口径） | ⚠️ wc -l 口径（不含 tests/website）实测 ~89 万行原始行（agent 129k、tools 120k、hermes_cli 253k、gateway 94k、plugins 88k、tui_gateway 42k、cron 17k、根文件 ~120k）；45 万疑似 cloc 纯代码行口径。正文用「几十万行」回避 |

## 二、机制事实清单（正文断言依据）

### 入口层
- 入口分三类：
  - 交互式：CLI（cli.py REPL + 斜杠命令）、TUI（ui-tui，Ink/React 终端界面；tui_gateway/ 是它的 Python JSON-RPC 后端，桌面也走它）、桌面（Electron，apps/desktop，spawn `hermes serve` 后端）、~30 个消息平台适配器、编辑器 ACP（VS Code/Zed/JetBrains，stdio JSON-RPC）。
  - 程序化：HTTP API（api_server 适配器，把 API 当"又一个平台"）、Python 库（直接构造 agent 类）、批处理（batch_runner，多进程池跑 JSONL、生成轨迹）。
  - 自主触发：cron 定时任务；后台进程完成通知（terminal background=true + notify_on_complete → 网关 watcher → 触发新一轮 agent 轮次，gateway/AGENTS.md「Background process notifications」）。
- 所有入口都在**自己进程内** import 同一个 agent 核心类并实例化——没有独立的"agent 服务进程"。
- 网关进程内：每条消息的处理跑在专用线程池上——一轮一线程、无队列无上限（gateway/turn_executor.py `_UnboundedThreadExecutor` docstring：故意不用 ThreadPoolExecutor，因其 max_workers=None 实为 min(32,cpu+4) 的隐形队列）；不同会话的轮次并发，同一会话由跨进程锁串行（见 agent-loop 调研§入口与准入）。
- 网关按会话键缓存 agent 实例（gateway/run_agent_cache.py）：空闲 TTL 回收（:917）+ 逐出前先把记忆刷盘（:851 `_commit_memory_before_soft_evict`）；缓存键含记忆 provider 身份签名（:71-116，Honcho 需要会话初始化参数进键）。
- 会话键：默认 `agent:main`，profile 命名空间 `agent:<profile>`（gateway/session.py:654-662）。平台+聊天 → 会话键的路由。
- CLI/TUI 也可自带 cron 循环（run_goals.py:390 注释）。

### agent 核心（只在架构层点名，内部归第 2 篇）
- 组成：同步主循环（31 个阶段文件）+ 提示词组装（stable→context→volatile 三层，agent/system_prompt.py + prompt_builder.py）+ 上下文压缩 + 工具分发（注册表 import 时自注册）+ provider 解析（三种 API mode：chat_completions/codex_responses/anthropic_messages，统一收敛 OpenAI 内部格式）+ 辅助小模型调用（agent/auxiliary_client.py：视觉/摘要等杂务）+ 记忆管理 + 子代理委派。
- 工具：内置注册 ~64 处 + 15 个 agent 级内联工具（需要活的 agent 状态，绕过注册表）+ MCP 动态挂载（无上限）。41 个工具集（toolset）分组，按平台/会话折叠。

### 状态层
- 每 profile 一个 home 目录；home 内一个 state.db（SQLite）是**所有进程共享的唯一事实源**：会话历史、冻结的系统提示词、跨进程会话锁（turn lease，durable，TTL 300s / 等待上限 1800s）、任务库、用量记账、全文检索索引（FTS5 + native/fts5_cjk 自研中文分词）。
- WAL 模式让多进程读写并发；文件系统不支持 WAL（NFS/SMB/某些 FUSE/WSL1）时自动降级普通日志模式（读阻塞写），hermes_state_wal.py 显式列出三类不兼容标记。
- 库级健康防护：零字节库隔离、WAL 持有者扫描、可移植快照（hermes_state_dbfile.py）。
- 门面 hermes_state.py + 30 个同主题拆分文件（hermes_state_*.py 实测 30 个）。
- kanban 另有自己的库（kanban.db，gateway/kanban_watchers.py）。

### profile 隔离与多路复用
- profile = 独立 home（config/.env/memory/会话/日志）+ 密钥域（secret scope）+ 终端域（terminal scope），根 AGENTS.md Code Shape 规则原话。
- 默认一 profile 一网关（各 PID）；multiplex_profiles=true 时**一个网关进程服务多 profile**：每个入站事件先规范化出路由身份（哪个 bot 收的、在哪个 profile 跑、由哪个 bot 送答，gateway/session_identity.py resolve_identity）。
- 作用域绑定按"活动"不按"轮次"：轮次、回调、逐出、tick 都要绑上 owning profile 的作用域；环境变量在进程级持有的是**启动 profile** 的值——未绑定的读会静默泄漏默认 profile 的数据（根 AGENTS.md 列为 bug 类型，#72348/#86905/#86905 系）。失败语义：scope 已装 + multiplex 激活 → scoped miss 就地失败，绝不回落 os.environ。
- token 锁：独占凭证（bot token/API key）连接时拿作用域锁，两 profile 不能共用一份凭证（gateway/status.py:1681）。
- cron 的 tick 不看 multiplex 开关：宿主页网关 tick 每个 profile 的任务库（cron/AGENTS.md「Cron ownership is not gated on gateway.multiplex_profiles」）。

### 扩展边缘（窄腰的具体形态）
- Footprint Ladder 六级（根 AGENTS.md）：1 扩展现有代码 → 2 CLI 命令+skill → 3 服务门控工具（check_fn：先配置才出现）→ 4 插件（~/.hermes/plugins/ 或 pip 入口点）→ 5 MCP server 进目录 → 6 新核心工具（垫底，仅当 terminal+file 或 MCP 都够不着）。
- 插件三类发现源：用户目录、项目目录、pip 入口点（architecture.md「Plugin System」）。两类单选插件：记忆 provider（7 家：mem0、honcho、supermemory、byterover、holographic、openviking、retaindb）与上下文引擎。
- 模型服务商 38 家插件（plugins/model-providers/ 39 项减 README 实测）。
- skills：内置 + optional-skills（装了才生效）；curator 自动创建/改进（第 11 篇）。
- MCP 双向：client（连外部 server，工具动态挂载）+ server（mcp_serve.py 会话通道桥）。
- 「核心工具随每次 API 调用发送」是窄腰的成本论证：加一个核心工具 = 每次调用都付 schema 与注意力成本（根 AGENTS.md 原话 "every model tool is sent on every API call, so the bar for a new core tool is high"）。

### 代码组织
- 门面 + 同主题拆分：曾经的 god-file 拆成门面 + `<stem>_<topic>.py`。实测规模：hermes_state 30 个、gateway/run 25 个、tools/mcp_tool 17 个、agent/turn 31 个、cli 12 个 mixin。文件 ~2000 行或函数 ~300 行/圈复杂度 30 就该拆（根 AGENTS.md）。
- 每个子系统一份 AGENTS.md（~8k 字上限）+ 根路由表——维护者架构自述内嵌在树里。
- evals/ 69 项离线评测（实测 69 个条目含 __init__）。
- 测试 5608 个文件（实测）；分层：单元（行为契约，禁 change-detector）+ 真实路径 E2E（真 import、临时 home、双 home A→B→A 验 profile 作用域）。
- 依赖链：注册表（零依赖）← 各工具文件（import 时注册）← 工具编排门面 ← agent 门面/CLI/批处理（根 AGENTS.md「Dependency chain」，architecture.md「File Dependency Chain」一致）。

### 规模数字（本 commit 实测，正文用约数）
- 适配器 ≈30（内置 8 + 插件 22）；模型服务商 38；记忆 provider 7；终端后端 7；工具集 41；内置工具注册 64 处 + 15 内联 + MCP 动态；主循环阶段文件 31；hermes_state 拆分 30；evals 69；测试文件 5608；Python 原始行（不含 tests/website）~89 万。

## 三、问题树（正文结构）

1. **Q1 ⭐⭐⭐ 首问**：Hermes 整体是个什么架构？有哪些组成部分？
   - 主答：一句话结论 + 组件布局图（入口层/核心/状态库/边缘）+ 顺着图的概述（分层 + 各层职责一句话）。
   - 追问：为什么说是"个人 agent"架构、和"服务端 agent 平台"架构的差别（单机单用户、无 K8s/消息中间件、状态就地）。
   - 追问：一条平台消息从进来到回答送出的完整路径（把三条数据流中的平台消息流走一遍）。
   - 追问：CLI 直接用和挂网关用，agent 核心有区别吗（无——平台差异留在入口）。
2. **Q2 ⭐⭐⭐**：一份 agent 核心怎么跑遍 CLI、30 个平台、TUI、桌面、编辑器？（平台无关核心；入口在自己的进程里实例化核心；网关一轮一线程 + agent 缓存；交互式/程序化/自主触发三类入口）
   - 追问：入口这么多，怎么都是"调"核心而不是"是"核心（依赖方向：入口依赖核心，核心不依赖入口）。
   - 追问：网关里一轮一线程，同时来 50 条消息会怎样（不同会话并发、同会话排队等锁——细节回指第 2 篇）。
   - 追问：agent 实例是每条消息新建吗（会话级缓存、空闲回收、逐出前刷记忆）。
   - 追问：桌面/TUI 的拓扑（serve 后端随 app 死、消息网关独立存活——为什么这么设计：网关是长驻服务，桌面只是又一个客户端）。
3. **Q3 ⭐⭐**：多个进程共用状态，为什么中心是一块 SQLite 库而不是 Redis/消息队列？
   - 主答：个人 agent 单机单用户 → 不需要网络中间件的运维成本；SQLite WAL 多进程读写并发够用；且缓存不变量恰恰需要"系统提示词和历史有唯一权威存储"。状态库承担的角色清单。
   - 追问：WAL 是什么、为什么需要它（读写不互斥）；网络文件系统上怎么办（自动降级 + 为什么）。
   - 追问：并发写同一会话怎么防（跨进程会话锁——回指第 2 篇，这里只讲"锁在库里不在进程里"的意义：任何入口进程都受同一把锁管）。
   - 追问：全文检索怎么做（FTS5 + 自研中文分词扩展）。
4. **Q4 ⭐⭐**：多身份/多账号怎么隔离？（profile = home+密钥域+终端域；默认一 profile 一网关；多路复用一个网关跑多 profile 的坑：作用域按活动绑定、环境变量是启动 profile 的、泄漏是静默的）
   - 追问：为什么隔离要做成"目录级孤岛"而不是配置继承（官方关闭配置继承 PR 的理由：耦合正是设计要防的）。
   - 追问：一份 bot 凭证两个 profile 抢怎么办（连接时拿作用域锁）。
   - 追问：cron 任务归谁跑（宿主网关 tick 所有 profile 的任务库，与 multiplex 开关无关）。
5. **Q5 ⭐⭐⭐**：这套架构有哪两条贯穿一切的设计不变量？
   - 主答：缓存不可破坏（字节冻结、变更延迟生效、--now 例外）+ 窄腰（核心工具每请求随行的代价 → 能力放边缘，六级阶梯）。
   - 追问：为什么"每个请求都带全部核心工具"（schema 随行是 function calling 机制；换来模型随时可用，代价是 token 与注意力）。
   - 追问：加一个新能力你会怎么加（六级阶梯走一遍 + 案例：订日报 → CLI 命令+skill；Home Assistant 工具 → 服务门控；某 SaaS 集成 → 插件）。
   - 追问：MCP 为什么符合窄腰（零核心占用、按需挂载）。
6. **Q6 ⭐**：几十万行、没有框架，代码库怎么保持可维护？
   - 主答：门面+同主题拆分（含拆分阈值）、树内维护者自述 + 路由表、行为契约测试 + 真实路径 E2E、离线评测。
   - 追问：为什么不用框架（回指第 2 篇 Q1 追问——控制权/同步模型，一句带过）。
7. **Q7 ⭐ 收尾**：这套架构最大的代价是什么？让你改你会改哪？
   - 主答候选：单进程网关的并发上限与故障半径；多路复用的作用域绑定复杂度（官方文档里大量条目在防作用域泄漏——复杂度证据）；同步模型不能横向扩；SQLite 单机写上限。改哪：诚实讨论（如网关进程拆分 vs 保持简单的权衡）。
