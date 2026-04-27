# Learning Journal

跟踪通过 Claude Code（尤其是 /teach-me skill）学习的知识，以及未来的学习计划。

## 已完成的学习

<!-- 格式：
### [日期] 主题名称
- **方式**: teach-me / 对话 / 代码探索
- **要点**: 学到的核心概念（2-3 句）
- **相关文件/资源**: （可选）
-->

### [2026-04-17] Feature Flag 系统对比
- **方式**: 代码探索 + 对话
- **要点**: 对比了三种项目（Bun/TS、Rust、Python+Rust）的编译时/运行时 feature flag 实现差异。理解了 `bun:bundle` 的 `feature()` 函数在编译时被替换为布尔值的机制。
- **相关文件**: `src/types/internal-modules.d.ts`, `scripts/defines.ts`, `build.ts`

### [2026-04-17] learn-claude-python 项目架构
- **方式**: 代码探索
- **要点**: 了解了 Python+Rust 重写版本的 Claude Code 架构，包括入口点、Provider 层、Tool 系统等与 TypeScript 版本的对应关系。

### [2026-04-14] Agent Tracing & Observability
- **方式**: teach-me
- **进度**: 9/10 概念掌握
- **要点**: Agent 可观测性三层模型（运维/决策/推理层）。`decision_context` 状态快照是核心洞察。LLM 解释是事后合理化，不是真实推理记录。分析工作流：统计 → diff → 回溯。
- **笔记**: `.claude/skills/teach-me/records/agent-tracing-observability/session.md`

### [2026-04-27] MCP 协议设计思想
- **方式**: teach-me
- **进度**: 7/8 概念掌握（跳过方案对比章节）
- **要点**: MCP 不只是语义描述 — 是有状态长连接 + push-on-change 的交互模型变革。三大原语按控制权分层（Prompts/user、Resources/app、Tools/model）。"Progressive complexity" 贯穿协议设计。Claude Code 中 MCP tools 与内置 tools 统一抽象，对 LLM 完全透明且缓存友好。
- **笔记**: `.claude/skills/teach-me/records/mcp-protocol/session.md`

## 学习计划

<!-- 格式：
- [ ] 主题名称 — 简述为什么想学 / 预期收获
-->

- [ ] Claude Code 核心循环（query.ts + QueryEngine.ts）— 理解 API 调用、流式响应、tool call 的完整流程
- [ ] Tool 系统设计（Tool.ts + tools.ts）— 理解 tool 注册、发现、权限控制的架构
- [x] MCP 协议与集成 — ✅ 2026-04-27 完成
- [ ] Ink UI 框架 — 理解终端 UI 的 React 渲染模型和自定义 Ink 框架
- [ ] 系统提示词组装（context.ts）— 理解 system prompt 是如何动态构建的
- [ ] Agent 与 Swarm 模式 — 理解多 agent 协作的实现
- [ ] Provider 兼容层 — 深入理解 OpenAI/Gemini/Grok 适配器的流转换设计
- [ ] 权限与安全模型 — 理解 permission mode、sandbox、auto-mode 的设计

## 学习笔记索引

<!-- 如果有详细的学习笔记，在这里记录指向具体笔记文件的链接 -->

- [Agent Tracing session](.claude/skills/teach-me/records/agent-tracing-observability/session.md)
- [MCP Protocol session](.claude/skills/teach-me/records/mcp-protocol/session.md)
