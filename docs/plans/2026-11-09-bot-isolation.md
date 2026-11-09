# 每个 Bot 一个隔离环境：默认最小权限，越界要批准，动作有记录

- 日期：2026-11-09
- 状态：草案，未和维护者确认
- 相关：`ds-bot/src/host/folders.js`、`ds-bot/src/host/computer/leases.js`、`docs/plans/2026-10-10-bot-runtime.md`、`docs/plans/2026-11-04-computer-use.md`、`docs/plans/2026-11-09-openbot-comparison.md`

## 为什么

- 现在所有 Bot 共用同一个桌面和同一套应用授权。一个 Bot 在动浏览器或前台应用时，人就得等它；本机也会同时被多个 Bot 的构建、渲染和索引占满（实测：只要 App 里有 Bot 在 turn，GPU helper 在 20–50%）。
- 默认权限太宽。会话现在跑在 `danger-full-access`，Bot 能改这个用户能改的任何文件。这不是一个团队产品该给的默认值。
- 出了事看不到记录。界面上能看到"批准了"，看不到"这个 Bot 今天被允许/拒绝做了什么"。

## 现状（代码里的事实）

- **每个 Bot 已经有自己的文件夹**：`docs/plans/2026-10-10-bot-runtime.md`；Bot 越过工作文件夹时，DS Bot 已经会用 DSH 的 `sandboxPolicy` 判断并弹批准（`ds-bot/src/host/folders.js` 的 `approveFolder`）。
- **每个 Bot 已经有一份浏览器 profile**：`ds-bot/src/host/browser/session.js`，默认 headless。
- **电脑操控已经有四档**：关 / 只看 / 后台操作 / 可以动前台，还有前后台租约（`computer/leases.js`）。
- **DSH 支持按会话设沙箱档位**：`@deepseek-ai/dsh-sandbox-policy` 的 `session-mode`，用会话日志里的 `sandbox/mode` 事件保存，重启后回放，会话之间互不影响；`sandboxPolicy.resolve({ session })` 读到的就是它。
- **缺的**：没有把这个档位做成 Bot 的设置，默认还是全权；没有审计流水；浏览器 profile 之外没有更强的隔离（可选容器）。

## 已定的行为（草案）

1. **每个 Bot 一个文件权限档**：`只读` / `只能改自己的工作文件夹`（默认） / `可以改任意文件`。存在 Bot 的设置里，改完下一步生效。
2. **默认最小权限**：新 Bot 默认"只能改自己的工作文件夹"，对应 DSH 的 `workspace-write`。老 Bot 升级时不动，界面上标出当前是"全权"。
3. **越界要批准**：Bot 要写工作文件夹以外的地方，按现在的 `approveFolder` 流程弹批准；被拒绝时告诉 Bot 原因，不重试。
4. **群不放大权限**：群会话沿用成员自己的档位；群自己的文件夹不改变单个 Bot 的边界。
5. **审计**：每次工具调用、越界批准/拒绝、电脑操控的前台租约，写一行到 `bot/audit.jsonl`（按天滚动），界面里能按 Bot 看最近记录。
6. **界面**：Bot 详情的"操控"卡片里加一档文件权限；设置里一个总览，列出所有 Bot 当前的档位。
7. **不碰 DSH**：只用 DSH 已导出的 `sandboxPolicy` 和 `sandbox/mode`，不自己写沙箱。

## 不做什么

- 不做内核级隔离；容器化留到远程运行时（`docs/plans/2026-11-09-remote-runtime.md`）。
- 不改 DSH 的沙箱实现，也不把宿主目录映射进任何隔离层。
- 不因为隔离而改 Bot 的工作文件夹规则。

## 分期

- **P0**：每个 Bot 的文件权限档（默认"只能改自己的工作文件夹"），UI 可见，越界走批准。
- **P1**：审计流水与界面查看。
- **P2**：浏览器隔离选项（独立代理、禁网、下载目录）。
- **P3**：可选的容器隔离，作为远程运行时的一部分。

## 已知限制

- `workspace-write` 由 DSH 的沙箱策略执行，不是容器；同用户能改的路径在策略层之外仍然可达。
- 审计只是记录，不是阻断；阻断靠档位和批准。
- 老 Bot 升级后仍是全权，需要用户自己改，默认不替用户收紧。

## 验收（草案）

- 新 Bot 默认写不到工作文件夹以外：越界时报"需要批准"，批准后才写入。
- 把档位改成"只读"后，写入直接失败并说明原因。
- `bot/audit.jsonl` 里能找到这次拒绝，界面里按 Bot 能看到。
- 重启后档位保持；不同 Bot 的档位互不影响。

## 变更

（暂无）
