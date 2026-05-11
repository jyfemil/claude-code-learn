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

## 学习计划（2026-05-11 调整）

结合 agent-docs 理论库 + sf-audit-agent 实战项目，删除与实际工作无关的主题（Ink UI、Bridge、多模型接入），聚焦核心 agent 工程能力。

**可视化知识图**: [docs/knowledge-graph.html](knowledge-graph.html)

### Phase 1 — 核心机制（当前优先）

- [ ] Agent 核心循环 — ReAct 循环、状态流转、流式处理、中断恢复、双层循环架构
- [ ] Tool 系统设计 — 注册与发现、Handler 模式、权限控制、延迟加载、MCP 集成
- [ ] Prompt 工程与组装 — 系统提示词动态构建、上下文工程、版本管理、Few-shot、动态注入

### Phase 2 — 智能层（紧随其后）

- [ ] Skill 系统 — 两阶段加载、目录扫描优先级、系统提示注入、防循环设计、渐进式知识 L1-L4
- [ ] Memory 系统 — 短期/长期、会话状态、压缩策略、TTL+LRU 淘汰
- [ ] Planning 模式 — 计划生成、确认门控、动态调整、任务分解、Sequential vs Parallel

### Phase 3 — 生产强化（按需拓展）

- [ ] 错误恢复与韧性 — 重试阶梯、降级策略、故障分类、回滚、人工升级
- [ ] 权限与安全 — 多层守卫、实体敏感度、Policy Engine、输入验证、审计日志
- [ ] Multi-Agent 协作 — 协作模型、Agent Spawning、任务分配、A2A 协议、Swarm
- [ ] Reflection — 自我纠错、Follow-up 生成、Producer-Reviewer、错误计数
- [ ] RAG — Standard/Graph/Agentic RAG、混合检索
- [ ] 评估与监控 — LLM-as-Judge、语义相似度、漂移检测、AI Contract

### 已删除（与实际工作无关）

- ~~Ink UI 框架~~ — 终端 UI 渲染与实际 agent 工程无关
- ~~Bridge / Remote Control~~ — 远程控制场景不适用
- ~~Provider 兼容层~~ — 多模型接入非当前重点

## 学习笔记索引

<!-- 如果有详细的学习笔记，在这里记录指向具体笔记文件的链接 -->

- [Agent Tracing session](.claude/skills/teach-me/records/agent-tracing-observability/session.md)
- [MCP Protocol session](.claude/skills/teach-me/records/mcp-protocol/session.md)
