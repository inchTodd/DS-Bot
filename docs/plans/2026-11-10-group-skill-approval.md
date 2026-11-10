# 群聊里的技能审批

- 日期：2026-11-10
- 状态：**方案待维护者选择（A / B / B+A），选定前不实现**
- 相关：#1249（群聊展示并处理技能创建审批）、#473（群详情里的审批入口）、#1173（Bot 自己发的卡片，不同来源）

## 为什么

群聊是任务的主要现场，Bot 在群里干活时，用户通常只盯群聊，不会逐个打开折叠成员的私聊。今天有两个和「技能」有关的动作，却都没有在群聊里给出「有事情等你批准」：

- **learning 技能提案**：后台审查 Bot 的工作后提议 New skill / Skill update / AGENTS.md update，卡片只在 Bot 的**私聊**里；群聊没有卡片，也没有入口提示，容易长期无人处理，群里的活就停着。
- **`save_capability` 工具**：Bot 在群聊回合里可以直接创建/修改共享 skill，**不经过任何审批**，下一条消息就生效；没有「等待」这个状态。
- 群里已经有一套审批 UI（DSH 工具调用的审批桥），但它只服务 DSH 的 `approval/request`，不覆盖上面两条。

#1249 期望的是「群聊消息流里能看到并处理技能审批，群入口有待办标记，两边同步，一次生效」。要满足它，得先决定「等待」由哪条路产生，再决定卡片放哪。

## 现在是什么样

### 两条技能路径

| | learning 提案 | `save_capability` 工具 |
|---|---|---|
| 触发 | 长回合后的后台审查 | Bot 在回合里主动调用 |
| 范围 | 只审 Bot **自己聊天**的长回合（`learning.js:345-365` 用 `isBotSession` + `chatOwners`；群成员 Session 是 `groupOwners`，不进 `pending`） | 任意 Session，含群成员会话 |
| 是否需要批准 | 是，卡片上 Approve / Decline（`learning-card.js`） | **否**；skill/pack 直接保存（`tools.js:1126-1154`）；只有 stdio MCP 会 `approved:false`，由用户在设置里允许，不是聊天气泡 |
| 卡片在哪 | 只在 Bot 私聊：`LearningProposals` 要求 `roster.byId[sessionId]` 是 Bot 且 `currentPart(bot) === sessionId`；`LearningDock` 注册在 Bot 会话的 `conversation.input.dock`（`index.js:1133`） | 没有卡片 |
| 已答怎么记 | `learning-answer` 按 proposal id 一次生效，二次回答报 `bot/stale`；已答卡片按私聊当前段展示（`learning.js:657-695, 777-780`） | 无 |

### 群里已有的审批通道

DSH 的敏感工具调用会发 `approval/request`；DS Bot 在 `approvals.js:26` 接住：

- 群成员 Session 的请求带 `roomId`，进 `work.approvals`；
- 群详情里是可直接批准/拒绝的卡片（`member-status.js:145-193`），成员 dock 和会话头胶囊显示「等你批准」，侧栏有提示点（`sidebar.js:316`），还有系统通知（`approval-notices.js`）；
- Bot 私聊的请求与 DSH 原生卡片赛跑，谁先答谁生效，另一张撤回；
- `pending` 只在内存，重启后请求本身也随回合结束，不重建。

这套 UI 和「一次生效」逻辑可以复用，但它的数据源是 DSH 的审批事件，不是 learning 提案。

### DSH 能复用的接口

- `ctx.approval.request({ agent, toolName, callId?, reason?, signal })`（`@deepseek-ai/dsh-user-approval`，`ApprovalService.request`）：要求在**打开的回合**内调用，返回 `allowed-once | rejected | cancelled | unavailable`，并写 `approval/asked` / `approval/decided` 审计事件；`ask` 策略下走答案器链，`never` 策略直接拒绝。
- 没有 DSH 的「技能已创建 / 技能待批准」事件。learning 是 DS Bot 自己的后台审查；`save_capability` 是 DS Bot 自己的工具。所以 A 不能套 `approval/request`，B 可以。

## 方案 A：把 learning 提案带到发起工作的群里

### 拟定的行为

1. learning 审查同时覆盖**群成员 Session**。一个群回合结束后，如果满足现有的「长回合」条件，把该成员 Session 记进 pending；提案额外记录来源 `roomId`、来源 Session 和回合（用于放置和去重）。
2. 从群工作产生的提案，待批准卡片出现在**那个群**：
   - 群聊输入框上方的 dock（复用 `conversation.input.dock` 的 group 会话）显示同一张 Approve / Decline 卡片；
   - 群的入口（侧栏行 / 会话头胶囊）显示待办点；
   - Bot 私聊不再放第二张可操作副本，只显示「在群 X 等你批准」的提示，避免两处都能点。
3. 批准/拒绝仍走 `learning-answer`：
   - 批准按今天的方式应用（`capabilitySave` + 自动挂到该 Bot，`learning.js:707-755`）；
   - `proposal.id` 是唯一入口，二次回答报 stale；两边状态一致；
   - 拒绝写进 `declined`，同一提议不再回来。
4. 已答卡片在**来源群**和 Bot 私聊都可见（只读、折叠），两边读同一份 settled 记录。
5. 群被删除或提案不在成员列表里时，提案回退到 Bot 私聊显示并提示「原群已不可用」。
6. 权限与隐私不变：只有用户能答（`learning-answer` 是客户端端点，Bot 没有这个工具）；learning 审查已有疑似密钥/敏感值过滤，卡片显示 `SKILL.md` 内容与说明，不带密钥。

### 实现位置（选定后）

- `src/host/learning.js`
  - `agent/status` 标记：同时认 `groupOwners`，pending 按 Bot 聚合；提案结构加 `roomId`、`originSessionId`、`originTurn`；
  - `reviewSkill`：群来源的提案在 `bot` 之外带上群上下文（用 `linesSince` 的群记录即可，不改提示词）；
  - `learningView`：按 Bot 返回时带上来源；另出按群的视图（或让客户端按 `roomId` 分组）；
  - `learningSettledView`：settled 记录带 `roomId`，同时按 Bot 私聊和来源群返回；
  - `answer`：settle 时保留 `roomId`；来源群已删时按私聊展示。
- `src/host/channel.js`：`live()` 的 `learning` / `learningSettled` 形状扩展或新增 `learningByRoom`。
- `src/client/learning-card.js`、`learning-card-model.js`：`LearningProposals` 增加按群渲染的入口；放置规则按来源回合的 `at`（群回合可用 `groupProgress` 的回合信息）；私聊侧只显示提示。
- `src/client/index.js`：在群 composer 的 dock 注册一个 group 版 learning dock；侧栏/胶囊复用既有的审批提示点逻辑。
- 群聊消息流里的「回合尾卡片」二期再做：现在 `LearningTurnTail` 挂在 `conversation.chat.turnTail`，群聊的回合由 `GroupReplies` 画，需要另开一个群回合尾槽，风险和改动比 dock 大。

### 风险与边界

- 审查范围从「私聊」扩到「群」：后台模型调用次数和 token 成本上升；需要沿用现有冷却、上限、只审长回合和团队空闲条件。
- 一个 Bot 同时有私聊和多个群的提案：今天 `waitingFor` 只允许一个未决提案，A 保持这个约束，先不并发。
- settled 两边展示要避免重复计数和重复可操作；用 proposal id 去重。
- 群成员变动、群删除、Bot 被删：提案回退或丢弃的规则要写死，并在 UI 说明。
- 群消息流本身不显示卡片（只 dock + 入口点）时，用户可能仍觉得「不够显眼」；这是 A 的取舍，二期再加回合尾卡片。

## 方案 B：让 `save_capability` 先等审批

### 拟定的行为

1. Bot 调用 `save_capability` 创建/修改 skill（范围待定：仅 skill，还是 pack 也含）时，先在**当前回合**里调用 `ctx.approval.request(...)`：
   - `toolName: 'save_capability'`，`reason` 写动作、名字和用途摘要，不带密钥；
   - 群成员回合的请求带 `roomId`，走现有 `approvals.js`，在群详情/dock 出现可批准卡片，侧栏和通知也出现；
   - Bot 私聊的请求与 DSH 原生卡片赛跑（今天已有这个机制）。
2. 结果：
   - `allowed-once`：保存并按今天的方式挂到 Bot，返回「已保存」；
   - `rejected` / `cancelled` / `unavailable`：**不写任何东西**，工具返回「没有保存：用户拒绝/请求已取消/没有可用的审批人」；
   - Session 的 approval policy 是 `never` 时失败关闭（DSH 直接给 `rejected`）。
3. 一次生效：`approvals.js` 的 `pending` 按 id 保证只答一次；两边同时点，后到者无效。
4. learning 的 `applySkill` 不重复问：用户已经在 learning 卡片上批准过，它直接 `capabilitySave`（`learning.js:707-755`），不能因为 B 再弹一次。
5. stdio MCP 的「设置里允许」保持现状；B 先只覆盖 skill（和 pack，若维护者要），不合并两套 MCP 审批。

### 实现位置（选定后）

- `src/host/tools.js:1126-1154`：`save_capability` 在 `capabilitySave` 前请求审批；`exec.agent` 提供 `agent`，`ctx.get('approval')` 调用 `request`；工具结果按 outcome 分支。
- `src/host/approvals.js`：群会话已有 `roomId` 路由；如需要，为 DS Bot 自己的工具请求补一个 `detail`（动作、名字、用途），卡片能显示人类可读的摘要。
- `src/host/capabilities.js:138`：不需要改保存逻辑；如需「只有用户批准过的 skill 才可用」，那是另一个策略，不在 B 内。
- 群聊消息流仍只有「暂停等你批准」的行（`cells.js:956`），完整卡片在群详情/dock；如果维护者要求消息流里也有卡，需另做「工具审批卡片进回合」的改动（见 A 的二期思路）。

### 风险与边界

- **改工具语义**：以前 Bot 在群里能静默建 skill，之后要停下等用户；如果用户长时间不在，回合会挂起，群任务停在审批点。需要说明这是预期。
- 审批卡片在群里走详情/dock，不在消息流正中；#1249 的「消息流里直接看到」仍要靠 A 或二期。
- 重启：`pending` 只在内存；回合随重启结束，审批也消失，不重建。文档要写清。
- 只读进程（另一个 dsh 持有团队）不发保存，也不应发审批；`writeCheck` 在保存时已兜底。
- 模型可能反复重试同一工具：被拒绝后应在工具结果里明确「不要重试」，或者在下一回合的上下文里写明。
- Bot 的 `save_capability` 结果会进入 Bot 的上下文；不要把 `reason` 里的敏感值回显。

## A + B 一起（研发建议的方向）

- B 提供 `save_capability` 的「等待」状态，回答「Bot 不能自批」和「群里有审批点」；
- A 提供 learning 提案的群内可见性，回答「群聊消息流/dock 能看到技能提案」；
- 两者不要对同一个动作出两张卡：B 处理工具调用产生的 skill 保存，A 处理后台 learning 提案；learning 批准后的保存不再走 B。
- 顺序建议：先 B（状态与权限约定），再 A（可见性）；或者 B 与 A 一起做，但测试分开。

## 待定的决定

1. 选 **A / B / B+A**；Issue 的「消息流里直接看到」在 A 里是 dock+入口，在 B 里是群详情/dock，是否接受。
2. B 的范围：只 skill？skill+pack？MCP 是否也纳入（会和设置里的 `approved` 重复）。
3. A 的审查范围：所有群成员会话，还是只有包含 Main Bot/群管理员的群（控制成本）。
4. 已答卡片是否两个地方都显示；A 的私聊侧只显示提示是否可接受。
5. 群删/成员变动/机器人被删时，提案回退还是丢弃。

## 测试计划（选定后实现时）

Host（`ds-bot/test/`）：

- A：群成员长回合被标记；提案带 `roomId`/来源回合；群视图按 room 返回；从群批准后 `capabilitySave` 生效并挂在 Bot；拒绝写 `declined`；二次回答 `bot/stale`；群删除回退私聊；settled 两边各一份、不重复。
- B：`save_capability` 在群回合里等待并在 `allowed-once` 后保存；`rejected`/`cancelled`/`unavailable` 不写；approval policy `never` 时失败关闭；只读 host 不发请求也不保存；learning `applySkill` 不重复请求；`pending` 只答一次。
- 群聊不把审批当成 Bot 的发言或 pass；消息流只多「暂停等你批准」行（若采用）。

Client：

- 群 dock 显示待批准卡片、侧栏/胶囊待办点；已答卡片只读；跨群不串；私聊提示不重复可操作；卡片不显示敏感值。

Bench（mock）：

- B：mock 在群回合里调用 `save_capability`，用户拒绝 → 无 skill、群里卡片消失；批准 → skill 出现并挂到 Bot；两边各点一次只生效一次。
- A：用 `memoryReviewDelayMs`/learning 的测试种子造一条群来源提案，检查群 dock 与入口点；群删除后的回退。

检查：`npm ci`、`npm run harness`、`npm run build`、`npm test`、`node scripts/build-client.mjs --check`、`npm run lint`；改 `index.js`/`src/host/` 必须跑 smoke；推送前泄漏扫描 `must fix: 0`。

## 不做什么

- 不新增角色/权限体系，不引入「群管理员可批准所有人」的规则；审批人仍是用户。
- 不改 DSH，不为审批卡片给 DSH 打补丁。
- 不改 learning 的提示词和模型选择；不在本方案里迁移历史提案。
- 不把审批请求写进群聊的模型上下文（Bot 不需要看到审批本身，除非以后单独设计）。

## 已知限制

- B 的 pending 只在内存，重启后审批随回合消失；A 的提案是持久化的（`learning` 状态），重启后还在。
- 今天一个 Bot 只允许一个未决 learning 提案（`waitingFor`），A 继承这个限制。
- 群被删除、Bot 被删除后，来源信息可能失效，只能回退或丢弃。
- 工具审批（B）在群聊的卡片仍走详情/dock；要进消息流需要另开回合尾槽，属于二期。

## 变更

- 2026-11-10：初稿，A/B 两条方案、风险与测试计划；等维护者选择（#1249）。
