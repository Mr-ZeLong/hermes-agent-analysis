# hermes-agent-analysis

对 [Nous Research 的 Hermes Agent](https://github.com/NousResearch/hermes-agent) 的源码分析，
产出一套面向工程师面试的中文系列文档：**每个主题写成一场模拟面试，从「它是干什么的」一路问到实现细节。**

## 这是什么

Hermes Agent 是一个约 45 万行 Python 的**自研**个人 Agent：纯同步 while-loop 主循环，
不依赖 LangChain / LangGraph 等框架；同一个 agent 核心跑在 CLI、约 20 个消息平台、TUI 和桌面上；
主打自我改进——skills 自动创建/改进 + 跨会话记忆。

本仓库只分析它 **agent 开发相关的核心实现**（主循环、工具系统、上下文工程、记忆、MCP、
子代理、沙箱、审批安全……），写给准备 Agent 方向面试的后端 / AI 应用工程师：
读完后既讲得清通用原理，又能拿一线工业级源码佐证深度。

- 分析基线：上游 main 分支 commit `8d30c4e`（2026-09-29 clone），文档结论只对该 commit 负责。
- `hermes-agent/` 源码目录不进本仓库（`.gitignore` 排除），只作本地只读参考，文档里用路径引用。
- 上游项目采用 MIT 协议，版权归 Nous Research 所有，本仓库仅为独立分析。

## 文档长什么样

每篇文档就是一场模拟面试，按「主问题 → 精炼主答 → 追问 → 追答」组织：

1. **首问**：「<模块名>是干什么的？核心机制主要流程是什么？」——主答给全局，配一张 mermaid 数据流图。
2. **流程拆解问**：核心流程每一步对应一个主问题，按流程顺序推进，主答精炼到能直接口述。
3. **追问**：细节、边界情况、业界方案对比、设计权衡都放在追问里，像面试官追问一样循序渐进。

所有实现断言都附源码引用（路径 + 类/函数名）。写作规范与工作流程见 [AGENTS.md](AGENTS.md)。

## 文档索引

| # | 主题 | 优先级 | 文档 | 状态 |
|---|------|--------|------|------|
| 1 | Agent 主循环与编排 | P0 | `docs/agent-loop.md` | 未开始 |
| 2 | 工具系统与 function calling | P0 | `docs/tool-system.md` | 未开始 |
| 3 | 上下文工程（prompt 组装 / 缓存 / 压缩） | P0 | `docs/context-engineering.md` | 未开始 |
| 4 | 记忆系统 | P0 | `docs/memory.md` | 未开始 |
| 5 | MCP（client + server） | P1 | `docs/mcp.md` | 未开始 |
| 6 | 子代理与多智能体 | P1 | `docs/subagents.md` | 未开始 |
| 7 | 安全与审批 | P1 | `docs/security-approval.md` | 未开始 |
| 8 | 沙箱与代码执行环境 | P1 | `docs/sandbox.md` | 未开始 |
| 9 | Provider 抽象与流式 | P1 | `docs/provider-streaming.md` | 未开始 |
| 10 | 自我改进闭环 | P2 | `docs/self-improvement.md` | 未开始 |
| 11 | 规划与任务管理 | P2 | `docs/planning.md` | 未开始 |
| 12 | 评估与可观测 | P2 | `docs/evals-observability.md` | 未开始 |

状态取值：未开始 / draft / done。文件名是预定名，开写时按实际调整。

## 如何核对源码引用

本仓库不含上游源码。想跟着文档里的引用读码：

```bash
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent && git checkout 8d30c4e
```

文档里的引用都是相对 `hermes-agent/` 仓库根目录的路径。

## 质量流程

每篇文档定稿前要过**两轮独立审核**，审核员是三个互不通气的 subagent：

- **事实核查员**：逐条打开源码文件验证引用，结论必须有源码依据；
- **面试官**：查问题是不是真实高频问法、答得经不经得起追问；
- **文字编辑**：查语言（无黑话、无废话）和问题树结构是否完整。

主 agent 逐条核实审核意见、属实的改、误判的驳回并说明理由，两轮走完文档才算完成。
每个主题一个 PR，PR 描述里附完整的审核处理记录。
