# 调研记录：插件体系（docs/plugins.md 的事实依据）

事实基线：上游 main `8d30c4e`（2026-09-29）。本文档是写作依据，不进正文。
审核时拿正文断言对照本记录逐条核对。

## 一、官方 claim 清单与验证结果

来源：`plugins/AGENTS.md`、`website/docs/developer-guide/plugins/index.md`（下称 guide）、
`website/docs/developer-guide/plugins/application-declarations.md`。

| # | 官方说法 | 来源 | 验证结果 |
|---|---------|------|---------|
| 1 | 四级来源 later-wins：bundled → `~/.hermes/plugins/` → `./.hermes/plugins/`（需 HERMES_ENABLE_PROJECT_PLUGINS）→ pip entry points | plugins/AGENTS.md + guide | ✅ plugins_discovery.py:189-218 扫描顺序；resolve_manifest_winners(:221) later-wins。注意 pip entry point 不参与目录 later-wins：目录插件同 key 时 entry point 被剔除（plugins.py:1451-1455） |
| 2 | 「Plugins never touch core」— 插件不许改 run_agent.py/cli.py/gateway 等；缺能力就加宽通用插件面 | plugins/AGENTS.md | 政策性 claim（PR #5295 例证），无法从代码直接验证；正文作为设计原则引用 |
| 3 | memory/model-providers 目录关闭新增（2026-05/06），新后端独立仓库 | plugins/AGENTS.md | 政策，引用 |
| 4 | plugin catalog 是唯一 out-of-tree 发现系统，YAML+40-hex SHA pin，removed.yaml kill list，`--allow-removed` 唯一豁免且记录在安装记录 | plugins/AGENTS.md | 待 subagent B 核实（plugin_catalog.py、plugins_cmd_catalog.py） |
| 5 | 通用插件注册 hooks：pre/post_tool_call、pre/post_llm_call、on_session_start/end | plugins/AGENTS.md | ✅ 但实际 VALID_HOOKS 远多于此：plugins.py:109-205 共 40+ 个（transform_*、pre_verify、api_request 三件套、auxiliary 两件套、kanban 系列、approval 观察者、pre_gateway_dispatch、pre_command 等） |
| 6 | discover_plugins 是 model_tools.py import 的副作用；中途装载路径都 force | plugins/AGENTS.md | 待 subagent A 核实 |
| 7 | 生命周期 hook 在 owning profile 的 scope 里跑；一个进程服务多个 profile | plugins/AGENTS.md | ✅ PluginManager 按 hermes home 键控（plugins.py:1637 `_plugin_managers_by_home`）；scope_key 贯穿注册表（tools/registry.py overlay） |
| 8 | 兼容契约是行为契约不是版本号：additive、签名反射、未知字段忽略、弃走两版窗口 | plugins/AGENTS.md + guide | ✅ 契约文本见 guide；实现佐证：hook 回调签名反射（developer-guide 记载，dispatch 层实现待核） |
| 9 | 内部 import 路径不是 API；2026-09-14 兼容层移除，旧路径 ImportError | plugins/AGENTS.md + guide | 引用 |
| 10 | manifest v2 additive：manifest_version/api_version 双轴、requires_plugins（advisory+拓扑序）、python_dependencies（PM 准入）、config_schema | guide | 待 subagent B/C 核实 admission；拓扑序见 manifest（待核） |
| 11 | capabilities 声明 + consent：7 个 id，consent 一次、更新只问新增；fail-closed；pip 插件用 entry-point group 声明 | guide | ✅ plugin_capabilities.py:28-47 七个 id：tools.override、llm.provider/model/agent_id/profile/task_override、gateway.platform_actions；record_consent(:161) 记 hash+时间戳并镜像 legacy 键；pending_capabilities(:196) 只问新增；entry-point capabilities 见 plugins_discovery.py:72-100 |
| 12 | override 内置工具需 operator 授权（allow_tool_override / capabilities），bundled 豁免；deregister 同样受门 | guide | ✅ plugins.py:456-499 register_tool 的 override 检查；:576-599 bundled 豁免 + fail-closed |
| 13 | pre_llm_call 注入贴 user message 不动 system prompt（缓存保持）；单 hook 上限 10k 字符，超出落盘 spill | guide | 待 subagent A 核实注入点；spill 配置见 guide（hooks.output_spill） |
| 14 | middleware 四种：tool_request/llm_request（改请求）、tool_execution/llm_execution（包执行，next_call 单次） | guide | ✅ middleware.py:24-26 VALID_MIDDLEWARE；:161-214 执行链 next_call 单次调用 raise；失败跳过继续链 |
| 15 | 中途装载拆分：gateway 命令/平台 handler 立即重接线；工具/prompt 段/MCP 延迟到下个会话/mcp.reload；禁用无 un-wire | guide + plugins/AGENTS.md | 待 subagent C 核实 |
| 16 | 便携式 Agent Plugins v1（plugin.json）：不导入 Python、skills 只读命名空间、mcp.json 按名合并、config 冲突 config 赢 | guide + plugins/AGENTS.md | 待 subagent C 核实 |
| 17 | 插件状态存储：settings 在 config.yaml（plugins.entries.\<id\>.settings）、state 在 plugin-data/（profile 隔离、10MiB、原子替换） | guide | ✅ plugins.py:270-289（get/set_config、PluginState）；上限 10MiB 见 guide（PluginState 实现待核 plugins_state.py） |
| 18 | 内树插件政策：不收第三方产品插件；example 系列移到 hermes-example-plugins | plugins/AGENTS.md | 政策，引用 |

## 二、机制事实清单（正文断言依据）

### 发现（plugins_discovery.py）
- 扫描顺序（collect_directory_manifests :189-218）：bundled 顶层（排除 memory/context_engine/model-providers/cron_providers 四类独立发现）→ user（HERMES_HOME/plugins）→ project（cwd/.hermes/plugins，需 HERMES_ENABLE_PROJECT_PLUGINS）。目录深度封顶 2：平铺 `<name>/` 或分类 `<cat>/<name>/`；dunder 目录（__pycache__ 等）和别家 harness 的 manifest 目录（.claude-plugin 等 5 个）跳过（:35-37, :150-158）。
- later-wins（resolve_manifest_winners :221-244）：user/project 同名盖 bundled 是**文档支持的覆盖方式**；但「冒名」检测——用户/项目目录的 manifest 名字撞 bundled key 而目录名不同 → 跳过+警告（防 enable kanban 激活无关代码）。
- 门控顺序（gate_manifest :257-311）：旧 relay 拒绝 → disabled（deny-list 优先）→ exclusive（memory provider 类，走 memory.provider 单选，只登记不装载）→ model-provider（走 providers 独立发现，避免双实例）→ bundled backend 自动装载 / bundled platform **defer 延迟装载**（急切导入 ~20 个重 SDK 给每次启动加秒级延迟）→ plugins.enabled opt-in（None=没启用过任何；首次 migrate 会 grandfathering）→ 非 bundled 复查 catalog kill list（installed_plugin_removal，离线检查，尊重安装时的 --allow-removed）。
- entry points（discover_entrypoint_manifests :72-100）：hermes_agent.plugins group；capability 从 hermes_agent.plugin_capabilities group 读（不导入插件代码）；kind 靠无导入的源码扫描分类。
- HERMES_SAFE_MODE 跳过全部插件发现（plugins.py:1364-1367）。
- discovery 可被抑制（contextvar，plugins_discovery.py:44-60）：多路复用网关读 PARKED profile 配置时不导入那个 profile 的插件。

### 装载（PluginManager.discover_and_load，plugins.py:1346-1394）
- 幂等短路 `_discovered`；force=True 先 unload 再全量重扫（中途装载路径都用 force）。
- 失败不缓存：扫描抛异常时 `_discovered` 复位（:1390-1392），避免「发现了 0 个」被当成功缓存。
- force 后恢复 config 拥有的 shell hooks / outbound webhooks（:1384-1389）。
- 装载顺序：resolve_plugin_load_order 按 requires_plugins 拓扑序（待核 manifest 细节）。
- on_plugin_loaded 监听器在锁外通知（:1393-1394），报告「这次扫描新装入的」（boot=全部，中途=新来者）。

### PluginContext 注册面（plugins.py:231-1223，正文「受控注册接口」的依据）
工具 register_tool（override 门）、钩子 register_hook、中间件 register_middleware、
CLI 子命令 register_cli_command、会话内斜杠命令 register_command（撞内置命令静默拒绝 :689-694）、
system prompt 段落 register_system_prompt_section（冻结进新会话 prompt，量有上限）、
技能 register_skill（只读、命名空间 `plugin:skill`）、语言包 register_locale(_dir)、
平台适配器 register_platform、平台原生事件 handler register_platform_handler（connect 时接线，先于核心 handler）、
Slack action handler、审批通道 register_approval_transport、辅助 LLM 任务 register_auxiliary_task、
脱敏正则 register_redaction_patterns、记忆 provider（inert，激活归 memory.provider）、context engine（单选）、
dashboard auth、secret source、call_mcp（按服务器 allowlist，默认拒绝 :510-574）、
inject_message（网关注入需 allow_gateway_injection :604-661）、dispatch_tool（走正规审批/脱敏管道 :705-714）、
ctx.llm（宿主模型面，override 类能力 fail-closed :382-390）、emit/subscribe 插件间事件总线（命名空间强制）。
配置与状态：get/set_config 只进自己命名空间（plugins.entries.\<id\>.settings）；ctx.state profile 隔离持久状态。

### 能力授权（plugin_capabilities.py）
- 7 个 capability 全部映射到既有强制门（:27-47）——「没有强制面的 id 不铸造」。
- 检查 fail-closed：读不到 consent = 拒绝（:99-109）；判定走 granted_capabilities 或 legacy allow_* 键（:130-147），留审计日志。
- record_consent（:161-183）：granted 并集合并 + 声明集合的 hash + UTC 时间戳 + 镜像 legacy 键。
- 更新时 declared_set_changed/pending_capabilities：只对新增能力重新征求。
- 未知 capability id 从 manifest 丢弃+警告（:76-81）——hide from consent screen 是 fail-closed 选择。

### 中间件（middleware.py）
- 请求型：每个回调看到前一个改写后的 payload；original_args/original_request 永远带改写前副本；trace 记录（source/reason/name）给下游观察 hook（middleware_trace kwarg）。
- 执行型：next_call 单次（二次调用 raise，:179-195）；回调抛异常 → warn-once 报告 + 跳过该帧继续下游（next 未调用则直接 call 下一层）；下游异常包装透传不吞（:200-212）。
- 工具请求链前面有个内建 relay_runtime intercept（nemo_relay，:113-119）——不是插件，但同一条链。
- 「observer hook 报告发生了什么，middleware 改变会发生什么」是模块自己的总结（:1-5）。

### 插件间/宿主隔离要点
- 工具覆盖门：bundled 豁免（维护者决定 vs 第三方提权，plugins.py:576-599）。
- call_mcp 默认拒绝、按服务器授权（:510-528）。
- ctx.llm 的模型/服务商覆盖全部 fail-closed 门控（:382-390 docstring）。
- 平台动作 platform_actions 默认 OFF、每次调用复查（:292-298）。
- 注册账本（PluginLedgerMixin）：每次注册登记 handle，卸载按逆序 teardown、identity-conditional（一个 profile 的卸载不会清掉别的 profile 的注册，plugins.py:1285-1301 注释）。

### 多 profile 共存
- manager 按 resolved hermes home 键控（:1637）；网关消息注入 host 进程级单例（:1642-1649 注释：多路复用网关一个调度器，插件隔离在 per-profile manager）。
- registry overlay 按 scope_key 分层（tools/registry.py）——工具注册表带 profile 维度。

### 装载顺序与 kind 检测（plugins_manifest.py，已核实）
- resolve_plugin_load_order（:219-265）：graphlib 拓扑排序，A requires B 则 B 先装；同级字母序；环→警告+全部退回字母序；缺依赖→警告一次但照装（advisory，运行时 ctx.has_plugin 探测）；自依赖忽略。
- _detect_kind_from_source（:268-278）：无导入的源码文本扫描——memory/cron provider 标记 → exclusive；register_provider+ProviderProfile → model-provider；用于 entry-point 和没写 kind 的 manifest 自动归类。
- PluginManifest 字段（:341-394）：name/version/description/requires_env/provides_tools/provides_hooks/source/path/kind/key/requires_hermes/portable/skill_namespace/capabilities/manifest_version(默认1,支持2)/api_version/requires_plugins/python_dependencies/config_schema/license/homepage/tags/emits/listens/provides_locales。
- manifest_version 超前（>2）：警告但照装、未知字段忽略（:137-140）——前向兼容。

### 配置与状态存储（plugins_state.py，已核实）
- settings（ctx.get/set_config）：只接受插件相对键；全局键、路径遍历、保留根（model/plugins/security/settings）在读写前拒绝（:24-39）。写入原子（合并读+原子写，跨线程/进程文件锁串行化）；托管安装（is_managed）改设置直接 PermissionError；管理员管理键拒绝（:66-90）。
- state（ctx.state，PluginState :134-195）：profile 隔离目录 plugin-data/\<namespace\>/state.json；10MiB/插件配额（超出抛错不截断）；键校验 1-128 字符禁 `..`；读改写全程文件锁（fcntl/msvcrt 双平台）；原子写（临时文件替换 inode，锁放兄弟文件就是为此）；目录名走摘要命名空间（防 Windows 保留设备名、防折叠碰撞，:93-101）。

### 与缓存不变量的关系（待 subagent A/B 补充触发链细节）
- guide 明说：注入贴 user message 是为保 system prompt 字节稳定（prompt cache）；中途装载的工具/prompt 段延迟到下个会话，同 /skills install 规则。
- plugin toolset keys 持久化缓存（plugins.py:1845-1890）：启动早期（插件还没装完）就要知道插件 toolset 集合的消费者服务——待核。

## 三、subagent 调研回填

### A. 发现触发链路（已回填，均经 subagent 打开文件核实）

**发现时机——三条触发路径收敛到同一个幂等 discover_and_load()：**
- CLI 启动：`hermes_cli/main.py:3025` 起 daemon 线程预热（manifest 扫描约 150ms，与启动其余部分重叠），线程体=发现+落盘 toolset keys 缓存（plugins.py:1813-1833）。
- import 副作用兜底：`model_tools.py:154-164` 模块体调 discover_plugins()——import model_tools 的进程都会触发；内置工具同文件上方也是 import 副作用注册。MCP 发现已从模块级移除（曾阻塞 event loop 120s，#16856）。
- 各入口显式：agent_init（抓工具快照前）、gateway 适配器启动、cron 调度器、TUI server 等（agent_init.py:1116、run_adapters.py:1028、run_startup.py:1041、cron/scheduler.py:3738、tui_gateway/server.py:1972）。
- 同步消费者先 join 后台线程（30s 超时，plugins.py:1836-1842）；register() 里再入发现的 worker 直接短路（plugins.py:1351-1355）。

**hook 触发点（按主循环时序，全部经 plugins_dispatch.py:209-243 逐回调隔离分发）：**
- on_session_start：conversation_loop.py:861-869，新会话首个 turn。
- pre_llm_call：turn_context.py:754-808，每轮 turn 组装时、压缩之后。返回 context 拼接注入**当轮用户消息**（content sidecar；多模态 append）——docstring 明确 never the system prompt。单 hook 超长 context 落盘溢写（tools/hook_output_spill，turn_context.py:778-803）。
- pre/post_api_request、api_request_error：turn_api_request.py:158-164，载荷脱敏截断（api_request_hooks.py:118-132），观察者。
- pre/post_auxiliary_call：auxiliary_hooks.py:222-236，辅助 LLM 调用每次物理尝试一对，与主循环 api_request 事件刻意分开（#79733）。
- post_llm_call：turn_finalizer.py:437-464，每 turn 一次，观察者。
- transform_llm_output：turn_finalizer.py:467+，第一个非空返回值替换最终回复；在 assistant 行首次持久化**之前**（#44239）；只动当前 turn 不碰历史与 system prompt（prompt-cache invariant）。
- on_session_end：turn_finalizer.py:770-782（每 turn 尾部都发，带 completed/failed/interrupted）；Ctrl-C 路径 cli_shutdown.py:184-192 补发。
- pre_tool_call（唯一策略 hook、唯一 fail-closed）：model_tools.py:766-801 → plugins.py:2016-2136。三种指令：block（工具不执行，消息变工具结果 status=blocked）、approve（升级人工审批门；审批门自身错/拒/超时按 block，fail-closed）、modify（浅合并参数，多个叠加）。优先级 block > approve > none，**不是注册顺序**（#87420）。skip_pre_tool_call_hook 保证每次执行恰好发一次。
- post_tool_call：model_tools.py:685-704，包装所有返回路径（成功/桥/block/异常），带 status/error_type/duration_ms。
- transform_tool_result：model_tools.py:849-866，结果进上下文前替换。
- hook 超时：`plugins.hook_callback_timeout` 默认 30s，超时放弃 worker 不 join（#6622 挂死教训）；**唯一 fail-closed 的是 pre_tool_call**（超时/异常合成 block），其余 fail-open。
- 一次工具执行顺序：pre_tool_call（可拦/改/升级审批）→ ACP 编辑审批 → 执行（经 tool-execution middleware）→ post_tool_call → transform_tool_result（可替换）→ 结果进上下文。

**工具进主循环通路：**
- 插件工具与内置工具进**同一个进程级注册表**，无合并层；差别只在注册时机（内置 import 副作用 / 插件 register）与写权限（覆盖要授权）。注册带 profile scope overlay（registry.py:433-473）；跨 toolset 遮蔽无 override 直接拒；schema 注册时校验；每次注册 bump generation（工具清单缓存失效钥匙，registry.py:743）。
- agent init 先 discover_plugins 再抓 generation 快照再取工具清单；quiet 模式 memo 的 cache key 含 generation+config 签名+scope（model_tools.py:256-279）。
- 合并发生在 toolsets 解析层：静态 TOOLSETS 与注册表按 toolset 取并集（toolsets.py:286-327）；纯插件 toolset 直接从注册表生成；enabled_toolsets=None（all）时插件工具默认进清单。
- 渐进披露：核心工具和 GUI 直连永不延迟；**插件工具默认可延迟**（tool_search.py:150-162）——超份额时 schema 换成三个桥工具，桥拆包重派发、下游 hook 看到真名（model_tools.py:903-923）。

**system prompt 段落（纠偏：不在 prompt_builder.py，在 agent/system_prompt.py + plugins_dispatch.py）：**
- 注册只允许 after_memory 一个锚点；三重预算：单段默认 4000 字符、最多 32 段、合计 8000 字符（plugins_dispatch.py:69-72）。
- 每会话渲染一次**冻结成字节**；重启恢复时从落盘 prompt 里按 `<!-- hermes-plugin-sections:start/end -->` 标记+每段字符数注释精确切出原文，重组结果必须与原帧一致才接受——**永不重跑插件代码**（system_prompt.py:113-162）。
- 位置在 system prompt 的 volatile 尾层（stable→context→volatile 三层，system_prompt.py:735-809）——插段落只许锚尾部一个粗位置，恢复进程才能不跑插件代码重建稳定前缀。
- 压缩边界失效重渲，重渲失败回退上一份字节（段落消失而非变错，system_prompt.py:812-829）。

**toolset keys 持久化（重要边界）：**
- 后台发现线程跑完把插件 toolset 名+portable MCP 名原子写 cache/plugin_toolset_keys.json（按 profile 分文件，0600）。
- 启动早期三个消费者（CLI --toolsets 校验、保存的 platform_toolsets 校验、tools 配置界面）读它，不阻塞 join 发现线程；三级取值：注册表就绪→实时、扫描中→上次缓存、都没有→阻塞发现。
- 契约：caller 只拿它**排除/判非 typo**，过期集合无害且自愈（#71650）。**不要把它说成提示词缓存机制的一部分**——它是启动体验优化。

**横向：**
- hook 投递走 _delivery_manager() 懒发现（plugins.py:1892-1906）——不 import model_tools 的 surface（dashboard/TUI/cron）hook 也不再哑（#50776）。
- 插件表面变化允许落后到下个会话生效，但会话内 prompt 与工具字节绝不中途变形——这套哲学统一了 toolset keys 缓存、工具清单 pin、prompt 段冻结三个机制。

### B. 安装/目录/准入/更新（已回填）

**安装（plugins_cmd_install.py / plugins_cmd_git.py）——一个原子事务：**
- 三种参数形态：裸名（走 catalog）、Git URL、owner/repo[/subdir] 简写；非 catalog 来源打 unreviewed 警告。浅克隆（--depth 1）到临时目录，全部检查过了才发布落盘，失败不留半成品。装完默认禁用，TTY 问 Enable now。
- --ref 只收完整 40 位 SHA；monorepo 子目录用 sparse-checkout（例：Hindsight 170MB→2MB）。clone 后立即洗掉 .git/config 里的凭证 URL；git 报错脱敏输出；克隆超时默认 300s。匿名优先、私有库才挂凭证。
- 安装账本 ~/.hermes/plugins/.install-metadata.json（树外！repo 伪造不了）：pinned/revision/source(洗过)/update_url 安装时拷贝/catalog 块(sha=实际 checkout，pin=目录 pin)/allow_removed。跨进程文件锁读改写。重装自动沿用旧 pin。
- requires_env 缺失变量交互式问、密钥掩码、写 .env。能力 consent 非交互 fail-closed 不授予。

**catalog（plugin_catalog.py / plugins_cmd_catalog.py）：**
- in-tree plugin-catalog/ 每 YAML 一条，40-hex SHA 强制、repo 必须 https、tier official/community；「进目录=人审通过，SHA bump=新的重审 PR」。
- 在线刷新：docs/api/plugin-catalog.json，TTL 6h、缓存 2MB 上限、断网旧缓存 24h、**removals 永不过期**；live 与 in-tree 双源合并，pin 不同取较新（防 hermes update 刚升的 pin 被旧缓存投票回去）。
- kill list removed.yaml：in-tree ∪ live 并集；repo 归一化成 host/path（scheme/用户名/www./.git/大小写/尾斜杠/scp 风格全抹平）——kill 写 https、装 git@ 也挡。比对点四个：install/update/enable/**load 时复查**（gate_manifest，离线检查不阻塞网络）。已装插件的召回判定五路比对（manifest 名/source/catalog 名/repo/目录名）。
- --allow-removed 唯一豁免，记录在安装账本（树外）；dashboard/TUI 硬编码不提供绕过；树内 .hermes-catalog.json 只是便利拷贝，Python 从不信任。

**enable/disable 与 admission 事务（plugins_admission.py / plugins_cmd.py / pm/）：**
- enable 写 plugins.enabled 允许表；disable 写 plugins.disabled 拒绝表（**disabled 压过 enabled**）。写 canonical key 不写裸名（loader 只认 canonical）。disable 只写配置，打印 Takes effect on next session，不触发卸载。
- 所有激活面（CLI/界面/dashboard）提交「候选并集」给唯一准入函数：PM 安装锁下先验证并集能在活动环境构建，同一事务里写 config；拒绝/写失败→config 字节与依赖环境都原地不动（「env + config move together or not at all」）。config 写乐观并发控制（digest 前后校验+journal+崩溃恢复回滚）。
- requires_hermes 不满足 load-blocking；python_dependencies 声明者成为 workspace member 参与统一解析；python_runtime: external 自管运行时不进共享环境。

**PM 环境代际（generation）与原子发布：**
- 启用带依赖的插件：核心+所有启用插件依赖作整体，uv 在**全新代际目录**解析构建；通过→指针原子指向新代。旧代完整保留（「prior generation survives both successful replacement and failed builds」）；运行中进程 boot 时 lease 钉住所跑代，收集器只删无 lease 旧代；中途换代 import 冲突→提示 restart。
- workspace member 键=名字+路径 hash 前 16 位（同 profile 同名不撞）；成员集字节哈进 venv stamp，一变就重建。
- 冲突→DependencyConflict「config and environment unchanged」；hermes update 拖垮环境的插件被禁用（二分逐出）而不是让更新失败。
- 插件代码发布同样 staged：临时树→journal→os.replace 两次换目录→durable 写账本；崩溃恢复按 facts 变没变决定删 backup 还是回滚。

**update（plugins_cmd_update.py / plugins_transaction.py）：**
- catalog 安装→重新 clone 到目录新 pin（永不 git pull）；自定义 git→autostash 的 pull --ff-only（用户常原地小改插件，纯 ff 会永久卡死）。pinned 无 --ref 拒绝更新。
- re-pin 前**先在预览临时目录 clone 新 pin、算 surface delta、要 consent**（五类面：capabilities/tools/hooks/Python 依赖/Desktop；新增才要同意）；本地 tracked 修改备份到 plugins-backup/。
- 自动更新默认只查不更（gateway 周期 tick 只读检查+receipt+一行 log，「a log, not a system-prompt mutation — cache safe」）；plugins.auto_apply 才动 git 类。
- update_url 信任模型：feed 只许 https（重定向也必须 https）；「拉来的 commit 引入安装时没存过的 url」与「换了 url」同罪拒 fetch；移动 saved tag 唯一路径是显式 trust-update-url。
- provenance 2x2：账本行×.git = git 安装/DRIFT(重装)/SELF_CLONED(可 adopt)/MANUAL(不可自动更)。
- 运行时注册所有权账本（plugins_ledger.py，与安装账本同名不同物）：每次注册记带逆操作的 handle，unload 按逆序 dispose、身份条件恢复；persistent 注册（进程级设施）常规卸载不拆、force 重扫后清理没回来的。

**validate / doctor：**
- validate=catalog 准入 CI：manifest 严格校验（requires_hermes 严格——load 时宽松、验收时 typo 必须失败）、可加载性（光 manifest 无代码=fail）、PEP 508、能力探针（子进程裸解释器+录音 stub 跑 register，30s 超时，**不 import 任何 hermes**；录得与 provides_* 双向 diff：未声明就注册=fail，声明没注册=warning）、内置工具撞名、安全扫描（dangerous=拒）。
- doctor=运行时诊断：临时 HERMES_HOME+进程内 monkeypatch 封锁 socket.create_connection/connect（软封锁，拦 Python 层出站，非沙箱）、真 PluginManager 真发现真导入；finally 全量还原（先 on_unload 关句柄再删临时目录，Windows #99918）。

### C. 中途装载与插件安全（已回填）

**中途装载激活拆分（plugins_activation.py:31-74）：**
- 判据是「插件实际注册了什么」（读 ledger），不是 manifest 承诺。
- 立即生效：gateway slash 命令（每条消息现查 dict，写入即活）、transform 类 hook、平台 handler 回调（网关 rewire 重接线）、语言包（注册即重置 i18n 缓存）。
- 延迟：tools/prompt 段（**改模型可见的工具数组/缓存前缀会破坏 per-conversation prompt caching**，同 /skills install 规则）；MCP servers 第三条路——当下连接、对打开会话可用，但工具藏在 tool_search 桥后不改前缀字节（plugins_activation_live.py docstring）。
- 通知链：activate_plugin_now（install/enable/update 统一激活点）→ 本进程 force 重扫+连 MCP → 控制套接字发 reload-plugins verb → 网关在 socket 线程 force 重扫+各 adapter 重接线；没人应答→restart_required。
- rewire 幂等键 (plugin, qualname)——force 重扫产出新函数对象，按 identity 判重会双重注册；抛过异常的工厂也记入已接线防反复重抛。
- **disable 无 un-wire**：只写配置；摘 handler 需要每家平台的反注册 API，不存在。兜底=任何 force 重扫先 unload() 全量拆卸（ledger 逆序 dispose+没回来的插件 persistent 注册被清扫）。期间已接线 handler 继续活。
- 提示词：**「立即/延迟」的判据是「改不改模型可见的 prompt 前缀」，不是「改不改进程状态」**。

**装载健壮性（plugins_loader.py）：**
- 命名空间：合成父包 hermes_plugins.，spec_from_file_location 按路径导入，**不碰 sys.path**（插件目录顶层模块名会和 site-packages 互相遮蔽；前缀给运行时归因——工具注册 trust gate 靠模块名前缀判「这是插件代码」，registry.py:617-629）；__path__ 钉死插件目录；相对导入照常。多 profile 同 slug：首个认领 bare 名，其他加 home hash 后缀。
- 导入前驱逐 sys.modules 同名旧条目（含子模块——只换顶层模块不够，陈旧子模块会继续服务上一次的代码）。
- 超时：plugins.load_timeout_seconds 默认 10s（0 关、上限 600s）；import+register 在 daemon worker 跑（Python 杀不了线程只能遗弃）；超时→ctx._abandon_load 打标→**超时后所有 register_*/subscribe/on_unload 一律 no-op+warning**（遗弃 worker 可能还在跑 register，迟到注册会落进刚清扫过的注册表 #108139）；abandoned worker 槽上限 8 个，到顶**拒绝装载**而不是 inline 跑。超时前的部分注册由失败路径统一拆（「a failed plugin must leave no callable reachable」）。
- register 抛异常（含 SystemExit）：记 loaded.error、插件占位进列表 enabled=False、**agent 继续跑**。
- 卸载：驱逐模块+逆序 dispose 注册；定向 unload 连 persistent 拆，force sweep 保留 persistent 后按「插件是否回来重新注册」裁决。

**capabilities 强制点：**
- 7 个 id 一比一映射既有强制门；「没有强制面的 id 故意不铸造」。
- consent 屏任何非 TTY 场景直接不授权（fail closed）；「Consent + audit, NOT a sandbox」；enable 本身不是授权请求（Enabling is not a request for undeclared privileges）；首次全量、更新只问新增（hash 检出声明集变化）。
- 工具覆盖强制点在 tools/registry.py register()：无授权抛 PermissionError；loader 每次装载前以 lease 装入 override policy（unload 能撤销过期授权不丢归因）；**归因靠 handler 定义模块解析，lambda/嵌套函数继承命名空间——「插件不能经回调洗白一次覆盖」**。

**plugin_guard（tools/plugin_guard*.py）——安装时静态扫描器，不是运行时审批一环：**
- skills_guard 引擎用于 install/update：代码运行前扫临时克隆。结构检查（符号链接逃逸=critical、捆绑二进制=high、400 文件/10MB/单文件 1MB 上限）+全树威胁正则。
- 判定：safe 装；caution 确认（--force 或 reviewed catalog pin 可过）；dangerous 阻断且 **--force 不可覆盖**。
- plugin_guard_context.py 是误报治理：威胁正则原为「agent 逐字执行的 SKILL.md」设计，插件仓库是代码库（README/test/正则字面量会有同样文本）；纯谓词**只降级不删除**；test 树仍扫（插件根是 submodule_search_locations，from .tests import evil 会执行）。
- 运行时：插件工具与内置工具**同一审批管线**，无专属策略；插件 pre_tool_call 只能把调用**升级**人工审批（pattern 加 plugin_rule: 前缀，fail-closed），不能绕过/放松。

**加载顺序与互斥（见前文 manifest 节）；portable v1（agent_plugins.py）：**
- plugin.json schema 精确匹配（未知顶层字段剔除）；**不导入任何 Python**——装载只做注册 skills+登记 mcp.json；无 kind 永远 opt-in。
- skills：frontmatter name 必须等于目录名；命名空间 agent-plugin-\<slug>-\<hash8\>；不复制进 ~/.hermes/skills/、**不进系统 prompt 的 available_skills 索引**、只能按限定名 skill_view 读。
- mcp.json 翻译：stdio command 单 token 或 ./ 相对；PLUGIN_ROOT/PLUGIN_DATA 保留 env 自动注入；cwd 展开逃出根即报错（**discovery 只在 data_root 下建目录，包根永不因 discovery 可写**）；remote 非 loopback 必须 https、禁 userinfo/fragment、跨源重定向丢 header。sse 显式拒绝。
- 合并冲突：**native config 赢**；插件间先装赢；故意不加插件前缀（40 字符命名空间会顶破 mcp__\<server\>__\<tool\> 64 字符上限），重复名装载时报错点名属主。

### D. 插件样例与扩展阶梯（已回填）

**bundled 盘点（104 个 plugin.yaml）：** platforms 22 个适配器（irc 为标准例：manifest+3 行 __init__+adapter.py，一次 ctx.register_platform 全元数据）；model-providers 38 家；memory 7 家（已封盘；honcho 17 文件、register 一行 register_memory_provider）；web 搜索 11、image_gen 8、video_gen 4、browser 3、dashboard_auth 4、observability 1（langfuse，8 个观察 hook）；cron_providers 1；顶层 standalone 若干（google_meet/spotify/security-guidance/disk-cleanup）。**context_engine bundled 引擎为零**（默认是内置压缩器，插件位留空）。
- kind 词表 5 种：standalone（默认，需 opt-in）/backend（bundled 自动载）/exclusive（单选）/platform/model-provider；无 kind 按源码标记自动判定。
- spotify 放 plugins/ 不放 tools/ 的理由（docstring 原文）：tools/ 留给基础能力，第三方服务集成归 plugins/；backend 自动加载、check_fn 把调度挡到授权之后。

**user-guide pluggable interfaces 表（17 行）+ 有意混搭三种风格：** Python 插件（ctx.register_*）、config-driven（TTS/STT 命令模板、MCP mcp_servers、shell hooks）、drop-in 目录（gateway hooks 放 ~/.hermes/hooks/ 即 opt-in）。「并非一切都是 Python 插件」。
- 文档漂移：user-guide 写 27 个 hook，代码 41 个；architecture.md 列 21 平台（实际 22，漏 a2a）。

**application-declarations.md：** 给「MCP server 背后是桌面应用」的插件声明应用清单（per-OS 怎么找到应用+版本+liveness），核心宿主机评估，按结果 gate 该 server 的工具和相关 skill；只 locate+读版本，**绝不探测 server、不启动应用**；六态评估结果；两道 gate：MCP 工具 check_fn（连接活 AND 声明可提供）、skill 的 requires_apps frontmatter（未注册名 fail-closed 隐藏）。

**Footprint Ladder 六级（根 AGENTS.md:133-152 唯一完整出处）：** 1) 扩展现有代码 → 2) CLI 命令+skill → 3) 服务门控工具 check_fn → 4) 插件 → 5) catalog 里的 MCP server → 6) 新核心工具（最后手段：基础性+几乎人人有用+终端和文件都够不着）。偏好顺序 :56-58。「The core is a narrow waist; capability lives at the edges.」
- （第 1 篇 architecture.md Q5 已把六级阶梯完整写给读者——本篇开头回指，不重讲。）

**middleware 实际使用：bundled 零**。全仓 register_middleware 引用只有定义/校验 stub/测试。langfuse 用的是观察 hook。「有能力但 bundled 没人用」——写文档定位为「独立于 hook 之外的另一注册面」比列举使用者重要。

**横向主线（B/C/D subagent 总结）：**
- 「一切都在树外」：pin、豁免、catalog 凭证、安装账本全在插件树外，repo 无法伪造自己的凭证。
- 「一个事务」：enable 的 config 写、依赖环境构建、插件代码发布共享 PM 锁+journal+前像校验。
- 「fail closed」：非交互场景遇到 consent 一律拒绝不代答。
- 安全四道门时间轴：安装前静态扫描（plugin_guard）→ 启用前 consent（capabilities）→ 注册时 override policy（PermissionError+模块归因）→ 运行时与内置同轨审批+插件只能升级不能放松。
