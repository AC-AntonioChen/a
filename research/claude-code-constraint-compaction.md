# Claude Code 约束规则在上下文压缩中的保真机制调研（基于 `codeaashu/claude-code`）

## 调研结论（先说结论）

1. **你担心的问题在工程上确实存在**：长上下文压缩（compact）会把“历史对话+约束”一起交给模型总结，天然有信息损失风险。
2. 该仓库并不是“完全不压缩约束”，而是用了**多层补偿机制**来降低损失：
   - 压缩后**重新注入关键附件**（plan、plan mode、invoked skills、工具/agent/MCP增量指令）。
   - 对 skill 内容做**单条上限 + 总预算**控制，而不是无限回灌。
   - 保留一段 `messagesToKeep`（部分压缩）以避免全靠摘要。
3. 对你最关心的“约束不被压缩掉”：
   - **项目/用户规则类（CLAUDE.md / rules）**在会话上下文构建时会再次加载（并且有优先级规则），这是最接近“抗压缩丢失”的机制。
   - 但**不是绝对不压缩**：例如 auto memory 的 `MEMORY.md` 入口有明确截断（行数/字节上限），skill 也有截断预算。
4. 因此从 agent 层实现看，它是“**防漂移 + 关键约束再注入**”，不是“**约束永不压缩**”。

---

## 关键源码证据

### 1) 约束来源与优先级（CLAUDE.md / rules）

`src/utils/claudemd.ts` 的文件头注释明确写了加载顺序与优先级：
- managed memory
- user memory
- project memory（含 `.claude/rules/*.md`）
- local memory

并说明“按优先级反向加载，后加载的优先级更高”。这意味着**规则并非只存在于历史对话**，而是通过上下文构建路径被纳入。  
来源：
- https://raw.githubusercontent.com/codeaashu/claude-code/main/src/utils/claudemd.ts

另外，`src/context.ts` 的 `getUserContext()` 会读取 `getClaudeMds(...)` 并把结果放入 `claudeMd` 字段，属于会话上下文的一部分。  
来源：
- https://raw.githubusercontent.com/codeaashu/claude-code/main/src/context.ts

### 2) compact 的本质：摘要 + 保留 + 回灌

`src/services/compact/prompt.ts` 里 compact prompt 的目标是“总结对话”，并要求输出结构化 summary（且分析草稿会被剥离）。说明核心策略仍是**摘要压缩**。  
来源：
- https://raw.githubusercontent.com/codeaashu/claude-code/main/src/services/compact/prompt.ts

`src/services/compact/compact.ts` 显示了压缩后的恢复策略：
- `buildPostCompactMessages` 顺序是：boundary + summary + messagesToKeep + attachments + hookResults。
- 会额外注入 `planAttachment`、`planModeAttachment`、`skillAttachment`。
- 对 deferred tools / agent listing / MCP instructions 做增量再公告。

这说明它并非“只信摘要”，而是**摘要 + 关键状态再注入**。
来源：
- https://github.com/codeaashu/claude-code/blob/main/src/services/compact/compact.ts

### 3) 对“skill 规则”的抗损与限额机制

同文件里有非常直接的注释：
- 过去 skill 回灌是 unlimited，后来改为有预算。
- `POST_COMPACT_MAX_TOKENS_PER_SKILL = 5000`
- `POST_COMPACT_SKILLS_TOKEN_BUDGET = 25000`
- 注释写明“通常顶部指令更关键”，所以倾向截取前部。

这是一种典型工程折中：**尽量保留关键约束，但受 token 预算约束**。
来源：
- https://github.com/codeaashu/claude-code/blob/main/src/services/compact/compact.ts

### 4) auto memory 本身也会被硬截断

`src/memdir/memdir.ts`：
- `MAX_ENTRYPOINT_LINES = 200`
- `MAX_ENTRYPOINT_BYTES = 25_000`
- 超限会截断并附 warning。

所以从实现层面可确认：系统并不存在“规则区完全免压缩/免截断”的统一保障。
来源：
- https://raw.githubusercontent.com/codeaashu/claude-code/main/src/memdir/memdir.ts

---

## 回答你的核心问题

## Q1：如何避免上下文压缩时，约束规则被不合理压缩？

结合这份源码，我建议是“**分层 + 重放 + 可验证**”三件事：

1. **分层放置约束（不要只写在聊天历史里）**
   - 永久高优先级规则放在 `CLAUDE.md` / `.claude/rules/*.md`（或等价系统层文件）。
   - 临时任务约束放在 plan / task 附件，确保 compact 后可回灌。

2. **压缩后重放关键约束（agent层）**
   - 对“模式状态”做显式附件回灌（如 plan mode）。
   - 对“已触发技能”做 invoked_skills 回灌。
   - 对工具清单/MCP 指令采用增量重公告。

3. **引入“约束完整性检查”**
   - 每次 compact 后跑一个 lightweight verifier：检查关键 rule ID 是否仍可见。
   - 若缺失，自动补发约束片段（而不是等模型自己回忆）。

4. **把约束写成“短而可索引”**
   - 把冗长规范拆成索引 + topic 文件（源码里的 memory 体系也是这个方向）。
   - 关键约束放前部（因为存在前部优先截取的现实）。

## Q2：基于该开源代码，是否实现了“agent层面对约束规则的不压缩控制”？

**结论：没有实现“绝对不压缩控制”，实现的是“关键约束保真增强”。**

更准确地说，它做了：
- 摘要压缩时保留部分消息（partial compact / messagesToKeep）。
- 压缩后对 plan/skills/工具增量指令进行再注入。
- 对 memory/skill 施加 token 与尺寸上限。

这是一套“**预算受限下的鲁棒恢复**”设计，不是“**规则区硬保留、永不压缩**”设计。

---

## 我给你的可落地方案（如果你要在自己的 agent 里实现）

1. 给每条高优先级约束分配 `rule_id`（如 `R-ARCH-001`）。
2. compact 后自动运行 `rule_replay_guard`：
   - 检查当前上下文是否仍含所有 `must_keep_rule_ids`；
   - 缺失则从规则仓重新注入最短版本。
3. 在 summary 里强制增加一个 machine-readable 区块：
   - `constraints_preserved: [R-..., R-...]`
   - `constraints_dropped: [...]`
4. 对规则做 two-tier：
   - Tier-0（绝不丢）：永远以附件固定注入，不走摘要。
   - Tier-1（可摘要）：允许进入 compact summary。

这样才能真正解决“token 同权重导致规则漂移”的问题。

---

## 调研范围与可信度说明

- 本次结论依据的是你给定仓库 `codeaashu/claude-code` 的公开源码路径与文件内容。
- 该仓库是泄漏代码的镜像/整理版本，不保证与 Anthropic 当前线上私有版本完全一致。
- 但就“agent层机制设计思路”而言，证据已经足够说明其策略偏向“再注入与限额保真”，非“绝对不压缩”。
