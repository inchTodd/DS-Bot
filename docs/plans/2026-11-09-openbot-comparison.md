# 两个 OpenBot 与 DS Bot 的能力对照

- 日期：2026-11-09
- 状态：调研记录，不是已定的设计
- 对象：[CopilotKit/OpenBot](https://github.com/CopilotKit/OpenBot)、[nightly-labs/openbot](https://github.com/nightly-labs/openbot)（openbot.run）

## 一句话定位

| 项目 | 是什么 | 许可证 |
|---|---|---|
| **DS Bot** | 长驻 Bot 团队，跑在 DSH 上；群聊、记忆、IM、Office，桌面端只连本机 | Apache-2.0 |
| **CopilotKit/OpenBot** | 企业版 AI 同事工作台：每个 Bot 一台自己的电脑，动作过策略网关并留审计，可上 Kubernetes | 应用模板 MIT；CopilotKit Intelligence 单独授权 |
| **nightly-labs/openbot** | 本地优先的桌面工作台：把 Codex/Claude/Grok 等本地 CLI 当 Bot，带队列、文件传输、手机远程和服务器模式 | PolyForm Noncommercial（0.1.11 及以前是 Apache-2.0），贡献要签 CLA |

**合规**：前两个只能学产品思路。CopilotKit 的 MIT 只覆盖应用模板，Intelligence 是专有服务；nightly-labs 是非商业许可。任何一行代码都不抄，除非是 MIT/Apache 且按规则记进 NOTICE。

## 能力对照

| 能力 | DS Bot | CopilotKit/OpenBot | nightly-labs/openbot |
|---|---|---|---|
| Bot 的运行底座 | DSH 会话（每 Bot 一条，可换段） | 内置 Bot 或任意 AG-UI 端点 | 本地 Codex/Claude/Grok/OpenCode/Gemini/Cursor/Cline/Kimi/Devin/Copilot/Mistral/Hermes，走 ACP |
| 隔离 | 每 Bot 一份浏览器 profile；文件和工作区共用本机 | 每人一个容器，Bot 可再独立一个（可选 gVisor），各有 Chromium、登录态、工作区 | 每 Agent 一个工作区 + 共享目录；桌面仍是同一个 |
| 动作审计 | 有审批卡，没有可读的流水 | 网关先决策、写审计再执行；拒绝时给出触发的规则 | 本地命令日志 |
| 远程 / 服务器 | 无：桌面端只连 `127.0.0.1` | Kubernetes / Helm 部署，多端接入 | 一条命令装到 VPS/无屏服务器，Docker 也有 |
| 手机 | 有设计（`docs/plans/2026-10-24-mobile-app.md`），未实现 | iOS / Android（商业版） | 已有：扫码配对、WebRTC、电脑只往外连 |
| IM | 已有：微信、飞书、钉钉、企业微信、QQ、Telegram、Discord、Slack | Slack / Teams | 无 |
| 群聊协作 | 强：群、Goal、派活、群记忆 | 频道、任务负责人、委派 | 频道、单一任务负责人、委派、归档 |
| 记忆 | 已有：长期记忆、学习、记忆面板 | Intelligence（专有） | 每 Agent 记忆；支持从 Grok Bot 导入记忆 |
| Grok Bot 导入 | 开发中（`feat/1126-grok-import`） | 无 | 已有：Grok Bot 里加一个导出 agent，生成 zip 再导入 |
| 生成式界面 | 有：`show_view`、提问卡 | 组件画廊 + 沙箱组件 | UI blocks |
| 市场 | 无 | 组件 / 技能 / MCP 目录 | Marketplace（Agent、技能、应用） |
| 开源程度 | 全量 Apache-2.0，可整份 fork | 应用 MIT，核心 Intelligence 专有 | 非商业，商用要单独授权 |

## DS Bot 的既有优势

- 团队形态最完整：群聊、Goal、派活、群记忆、IM 八个平台。
- 全量 Apache-2.0，改哪都行，不用签 CLA。
- Office 文档、连接器、密钥卡、图片/附件这些日常能力齐。

## 差距里最值得补的三条

1. **每个 Bot 一个隔离环境**（对标 CopilotKit）。这是本机风扇和"抢桌面"的根：现在所有 Bot 共用同一个桌面和同一批应用授权。DSH 已经给了每 Bot 一份浏览器 profile，缺的是文件根与可选的容器隔离，属于 host 层能做的事。
2. **远程运行时 + 手机**（对标 nightly-labs）。桌面端只连本机是硬限制；`docs/plans/2026-11-09-remote-runtime.md` 已经把边界写出来，nightly-labs 的形态证明了这条路可行。
3. **动作审计**（对标 CopilotKit）。现在能看到"批准了"，看不到"这个 Bot 今天被允许/拒绝做了什么"。在 ds-bot 层记一条流水成本不高，价值在信任。

## 结论

- CopilotKit 强在治理与隔离，nightly-labs 强在本地多 CLI、远程/手机、Grok 导入。
- DS Bot 不该抄任何一家的形态，先补第 1 条，因为它同时解决性能、抢占和隐私；第 2 条接现有远程方案；第 3 条做小做早。

## 变更

（暂无）
