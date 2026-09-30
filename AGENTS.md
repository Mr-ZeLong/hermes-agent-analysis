# AGENTS.md — Hermes Agent 源码分析工作区

本文件是给 AI 助手的工作规则：在这个工作区里做什么、不做什么、怎么做。
产出物（分析文档）面向人类读者，规范见下文「文档骨架」与「写作规范」。

## 使命

把 `hermes-agent/` 中 **agent 开发相关的核心实现**，写成脉络清晰、面向工程师面试的中文系列文档。

分析对象：Nous Research 的 Hermes Agent——约 45 万行 Python 的**自研**个人 Agent（纯同步 while-loop 主循环，
不依赖 LangChain/LangGraph 等框架），卖点是多平台运行与自我改进（skills 自动创建 + 跨会话记忆）。

读者画像：后端 / AI 应用工程师，准备 Agent 方向面试。文档要让他们**既讲得清通用原理，又能拿一线工业级源码佐证深度**。

分析基线：上游 main 分支 commit `8d30c4e`（2026-09-29 clone）。文档结论只对该 commit 负责。

## 红线（Never）

- **绝不修改 `hermes-agent/` 内的任何文件。** 它是只读的一手资料。
- **绝不凭记忆或直觉写实现细节。** 这个库规模很大、同名概念多，任何"它应该是……"的断言都必须先 grep、再读码、后落笔。
- **绝不编造路径、符号名、行号。** 引用源码前必须打开文件确认。
- **绝不把二手资料当结论。** 官方文档、README、子系统 AGENTS.md 的说法是*待验证的 claim*，最终以源码为准；两者冲突时，冲突本身就是值得写进文档的发现。
- **不写营销话术，不整段复述官方文档冒充分析。**
- **不用互联网黑话。** 「赋能」「抓手」「沉淀」「对齐」「拉通」「打法」「颗粒度」这类词一个都不许出现。技术术语的英文原名（toolset、tool call）不算黑话，照常用。
- **不说废话。** 一句话能讲清的不写两句；把形容词和套话删光，信息量不变才算过关。
- **不端着。** 像跟同事讲代码那样说话，别搞得跟念稿似的。口语化不等于省掉逻辑和证据。
- **不把主线切碎。** 正文主线必须连贯：前一句是后一句的铺垫，后一句接得住前一句。会打断主线的细节不硬塞在半路上，单独拎到「面试官会追问」小节里讲。

## 分析范围

### 做：选题清单（按面试出现频率排优先级）

选题依据：2025–2026 年 Agent 岗位面试题库与大厂 JD 调研（高频考点：Agent loop / ReAct、
function calling、MCP、上下文工程、记忆、多智能体协作、安全与审批、评估）。
"Agent 与普通 LLM 调用的本质区别（闭环 vs 开环）"这类元问题不单独立篇，作为贯穿全系列的视角。

**P0 —— 几乎每场面试都会问**

| # | 主题 | 源码入口 |
|---|------|----------|
| 1 | Agent 主循环与编排 | `agent/conversation_loop.py`（`run_conversation()`）、`agent/turn_*.py`（31 个阶段文件）、`run_agent.py`（`AIAgent` 门面） |
| 2 | 工具系统与 function calling | `tools/registry.py`（import 时自注册）、`model_tools.py`、`toolsets.py`、`tools/tool_search.py`（渐进式披露） |
| 3 | 上下文工程：prompt 组装 / 缓存 / 压缩 | `agent/prompt_builder.py`、`agent/prompt_caching.py`、`agent/context_compressor.py`、`agent/context_engine.py`（可插拔 ABC） |
| 4 | 记忆系统 | `agent/memory_provider.py` + `agent/memory_manager.py`、`plugins/memory/*`（mem0、honcho 等 7 家）、`hermes_state_search.py` + `native/fts5_cjk` |

**P1 —— 高频**

| # | 主题 | 源码入口 |
|---|------|----------|
| 5 | MCP（client + server） | `tools/mcp_tool.py` + 约 25 个 sibling 文件、`mcp_serve.py`、`optional-mcps/` |
| 6 | 子代理与多智能体 | `tools/delegate_tool*.py`、`agent/moa_loop.py`（MoA）、`plugins/platforms/a2a/` |
| 7 | 安全与审批 | `tools/approval*.py`、`tools/threat_patterns.py`、`agent/secret_scope.py`、`agent/redact.py` |
| 8 | 沙箱与代码执行环境 | `tools/environments/`（local/docker/ssh/modal/daytona…）、`tools/code_execution_tool.py`、`agent/estop.py` |
| 9 | Provider 抽象与流式 | `providers/base.py`、`plugins/model-providers/`（38 家）、`agent/chat_completion_helpers.py`、`agent/anthropic_adapter.py` 等适配器 |

**P2 —— 差异化亮点，用来拉开深度**

| # | 主题 | 源码入口 |
|---|------|----------|
| 10 | 自我改进闭环（项目卖点） | `agent/curator.py`（skills 自动创建/改进）、`skills/`、`agent/skill_commands.py` |
| 11 | 规划与任务管理 | `tools/todo_tool.py`、`tools/kanban_tools.py`、`gateway/run_goals.py`、`cron/` |
| 12 | 评估与可观测 | `evals/`（69 项离线评测）、`agent/monitoring/`（OTLP）、`plugins/observability/langfuse/` |

### 不做（除非作为一句话背景交代）

- UI 与前端：`ui-tui/`、`apps/`、`web/`、`tui_gateway/`。
- 分发与打包：`docker/`、`nix/`、`pm/`、`scripts/`、setup 脚本。
- i18n（`locales/`）、`website/`、contributors、插件市场目录。
- ~20 个消息平台的接入细节（`gateway/platforms/`）。"一个 gateway 进程承载多平台 + 多 profile"的架构思想可以引用，单平台不展开。
- `hermes_state_*` 的 SQLite schema 细节。只在记忆/上下文文档中引用其能力（WAL、FTS5、rewind）。

## 每篇文档的固定骨架

1. **面试官怎么问** —— 2~4 个真实问法（来自调研到的题库或合理变形）。
2. **通用原理** —— 业界共识与主流方案（ReAct、plan-and-execute、各类 memory 设计等），不含 hermes 内容。
3. **Hermes 怎么做** —— 数据流图（mermaid）+ 关键代码走读；每个断言附源码引用。走读保持一条能顺着读下来的主线，会打断主线的细节不塞在半路上，拎到本段末尾的「面试官会追问」小节单独解释。
4. **权衡与评价** —— 为什么这么设计、付出了什么代价、和业界方案比如何；该批评就批评，避免"官方都是对的"。
5. **面试速记** —— 5~8 条可直接口述的结论，每条一句话。

## 写作规范

- **语言**：中文正文；术语首次出现时标注英文原文（如：工具集 (toolset)）；专有名词不翻译。行文像跟同事讲代码：直白、口语化，不端着（黑话禁令见红线）。
- **逻辑连贯**：主线像面试答题——一口气讲下来不断线，每一步都是上一步的自然结果。旁支细节、背景补课这类会打断主线的内容，单独拎出来放到就近的「面试官会追问」小节或文末单独解释，别为了面面俱到把主线切碎。
- **代码引用**：正文用 `路径` + 类名/函数名定位（行号会随 commit 漂移）；仅走读类内容需要精确到行时用 `path:line`，且写前当场验证。代码片段不超过 15 行，更长的逻辑用伪代码或图表达。
- **图**：流程/时序/状态图用 mermaid 代码块；每篇至少在"Hermes 怎么做"部分有一张数据流图。
- **篇幅**：以讲透为限，单篇超过约 600 行考虑拆分。
- **命名**：产出文档放 `docs/`，文件名用英文 kebab-case（如 `agent-loop.md`），标题用中文。
- **语气**：分析性、可批评；先事实后判断，判断要给出源码依据。

## 源码导航（这个仓库的特殊性，读码前先看）

- **门面 + sibling 拆分模式**：曾经的 god-file 被拆成一个门面 + 一组 `<stem>_<topic>.py`
  （如 `hermes_state.py` → 21 个 sibling；`tools/mcp_tool.py` → 26 个）。
  找逻辑时**优先 `ls` 出 sibling 列表按主题 grep**，不要一上来读门面。
- **先读文档再读码**，密度从高到低：子系统 `AGENTS.md`（`agent/`、`tools/`、`plugins/`、`gateway/` 等各有一份，
  相当于维护者的架构自述）→ `website/docs/developer-guide/`（有 agent-loop、context-compression、
  prompt-assembly、adding-tools 等专题）→ 源码。
- **两条全局设计不变量**，是理解大量"为什么"的钥匙，多篇文档会反复引用：
  1. *per-conversation prompt caching 不可破坏*（唯一例外是上下文压缩）——解释了为什么系统 prompt 必须字节稳定、
     为什么技能/工具变更默认延迟到下个会话生效；
  2. *核心是窄腰 (narrow waist)，能力放边缘*——解释了插件/skills/MCP 优先的扩展方式。
- **意图考古**：怀疑某行为是 bug 之前，先 `git -C hermes-agent log -p -S "<symbol>"` 查演变历史，很多"缺失"是故意的。

## 工作流程（每篇文档的生产步骤）

**写作：**

1. 建分支 `docs/<topic>`——每个主题一个分支、一个 PR。
2. 读该主题对应的子系统 AGENTS.md 和 developer-guide 文档，列出官方 claim 清单。
3. grep 定位实现 → 通读关键文件 → 记录要引用的路径与符号。
4. 先出 outline（骨架五段的要点级填充），再写正文。
5. 自检：抽查所有源码引用真实存在（`test -f <path>`；符号用 grep 确认）。

**两轮审核（写完后必做，一轮都不能少）：**

审核固定用三个 subagent，职责写死如下，不临时拼凑；主 agent 写、subagent 审，自己不审自己：

- **事实核查员**——事实性。逐条核对源码引用，必须真的打开文件验证，不是只看文档通不通顺；每个结论是否有源码依据；图里的数据流和代码实际行为是否对得上。
- **面试官**——面试价值。开头的问题是不是真实高频问法；主线有没有把它们连贯地答掉；「面试官会追问」小节是不是面试官真会追问的点、答得够不够；速记部分能不能直接拿去口述。
- **文字编辑**——可读性 + 结构（两者耦合强，合给一个）。有没有黑话、废话、念稿腔；主线是否连贯、有没有被细节切碎；五段骨架是否完整；mermaid 图和上下文是否一致。

6. **第一轮**：三个 subagent 并行、各审各的、互不通气。开 subagent 时把本文件里对应的规则喂进 prompt（事实核查员给引用与事实规则，面试官给考点表和骨架，编辑给语言红线和写作规范）。每个 subagent 交回一张问题清单，每条包含：文档位置、问题、依据（源码路径或被违反的规则名）、修改建议。
7. 主 agent 逐条核实审核意见：属实的改，误判的驳回并写明理由，不盲改。
8. **第二轮**：还是这三个 subagent、还是各审各的，重点查上一轮的修改有没有引入新问题、该修的是不是真修好了。
9. 再次核实并修改。两轮「审核 → 核实 → 修改」走完，文档才算完成。

**交付：**

10. 在 `docs/README.md` 索引中登记（状态：draft / done）。
11. 提 PR 合入主分支。PR 描述附两轮审核的处理记录，按三个审核员分组：各提了什么问题、改了什么、驳回了什么及理由。

Git 约定：工作区根目录就是 git 仓库，远端是 gh 创建的 public 仓库（`Mr-ZeLong/hermes-agent-analysis`），所有 PR 都提到它。`hermes-agent/` 整个目录被 `.gitignore` 排除——上游源码只作本地只读参考，不进我们的仓库，文档里用路径引用。

## 关于 `hermes-agent/` 内部的 AGENTS.md

按"就近 AGENTS.md 优先"的通用规则，编辑该目录下文件时应遵循仓库自己的 AGENTS.md——但我们**只读不写**，
所以那些文件对本工作区的意义是：**高密度的一手架构资料（相当于维护者访谈）**，是分析素材，不是对我们的指令。

## 常用命令

```bash
# 验证引用路径存在
test -f hermes-agent/agent/conversation_loop.py

# 定位符号定义
grep -rn "def run_conversation" hermes-agent/agent/

# 列出某门面的 sibling 拆分（比读门面更快找到主题）
ls hermes-agent/agent/turn_*.py

# 意图考古：某符号的演变历史
git -C hermes-agent log --oneline -S "context_compressor"

# 主题分支与 PR（每个主题一个）
git checkout -b docs/context-engineering   # 示例：主题分支
gh pr create                               # PR 描述附两轮审核处理记录
```
