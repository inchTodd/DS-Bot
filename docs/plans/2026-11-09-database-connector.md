# 数据库连接：PostgreSQL / MySQL 的只读接入卡片

- 日期：2026-11-09
- 状态：草案，未和维护者确认
- 相关：`ds-bot/src/host/connectors/`（`catalog.js` 目录、`index.js` 引擎）、`docs/plans/2026-11-04-connectors.md`

## 为什么

- 用户要 Bot 能查自己的库（PostgreSQL / MySQL）。现有的连接器目录只收「有 OAuth 的远程 MCP」，自建库没有 OAuth，硬塞进目录不合适。
- DSH 已经支持 stdio / 回环 http 的自定义 MCP，DS Bot 也已经支持「自定义服务器」（`state.connectors[id].custom` 带 `url` 或 `command`）。缺的不是协议，是**引导**：现在要用户自己写命令、自己想密钥变量名。
- 安全边界必须写死：只读、最小权限账号、密码只进密钥库、错误信息不带连接串。

## 已定的行为（草案）

1. **连接器页多一张「Database」卡片**，选 PostgreSQL 或 MySQL；它不是 OAuth 连接器，点开是一个表单。
2. **表单**：host、port、database、user、password、SSL（默认开）、只读（默认开且不可关，见第 6 条）。
3. **密码只进密钥库**：存为 `CONNECTOR_<id>_PASSWORD`，不进 `state.json`、不进日志、不进 Bot shell 的默认环境。
4. **生成一个自定义连接器**：`id` 由 host+port+database 派生并加随机后缀，名称可改；底层是 stdio 命令 + 环境变量，环境变量的值在挂载时从密钥库注入。
5. **不新增运行时依赖**：v1 让用户选一个 MCP 服务器命令（文档给官方参考实现与自建方式），DS Bot 只负责生成配置、注入密钥、校验连通。要是以后决定随 App 打包只读 SQL MCP，再单独评估依赖许可并写进 NOTICE。
6. **只读**：表单强制只读；生成的环境变量明确写 `READ_ONLY=1`；同时要求在数据库侧用只读账号（文档给 `CREATE ROLE ... LOGIN` + `GRANT SELECT` 的最小 SQL）。DS Bot 不假装 SQL 解析能替代账号权限。
7. **地址**：允许 loopback、私网；公网地址给出警告并确认一次。非 TLS 连接默认拒绝，除非是 loopback。
8. **错误**：连接失败只说主机/端口/数据库和原因，不回显密码或完整连接串。
9. **按 Bot 关闭**：沿用现有 `offFor`，关掉后该 Bot 看不到工具。

## 合规落点（评审给的四条，采纳）

1. **只读账号是唯一安全边界**。界面和文档都写死「必须用只读账号」；插件侧的 SQL 关键字过滤只做提示，不当防护。
2. **密码只走密钥引用，不放 `args`**：stdio MCP 的 `env` 支持 `secret:NAME`（`src/host/capabilities.js` 的 `resolveValues`），DSN 放 `env` 的密钥引用；`args` 会被 `ps` 看到，一律不用。
3. **所有出口都 scrub**：错误、日志、诊断包、feed、IM 转发统一走 `rt.redactSecrets`（`capabilities.js` 现有写法），host/user/password 片段在拼接 cause 前后都清。
4. **连接串只在 host 内存里拼**：不写工作目录、不进 Bot 卡片、不进团队导出；默认 `sslmode=require`，不提供「跳过证书校验」开关。
5. **资源边界有默认值**：连接 5s、查询 30s、最大返回 1000 行、连接池 4；超限报错而不是截断后继续。
6. **默认网络范围**：只允许 loopback 和私网；公网地址需要显式确认一次，且必须 TLS（非 TLS 一律拒）。
7. **结果外发要说清**：卡片上写明「查询结果会进入对话并发送给模型」，默认先拉元数据（表/列），查询默认限制行数。
8. **目录注释与测试**：`catalog.js` 头注释补一句「本目录只收 remote MCP（oauth/key）；本地数据库是 stdio MCP，走新的卡片入口」；fixtures 用假 DSN（`postgres://u:fake@127.0.0.1/db`），断言日志、错误和 `state.json` 里没有密码；合并前 `node scripts/scan.mjs` 的 must-fix 必须为 0。
9. **不内置驱动**：v1 不把 `pg`/`mysql2` 塞进依赖（也就暂时不涉及 NOTICE）；用户选 MCP 服务器。

## 不做什么

- 不做 ORM、不缓存数据、不代理写语句。
- 不自动在数据库里建账号——需要用户在自己的库里执行一段最小权限 SQL。
- 不做 SSH 隧道；需要隧道的场景先让用户在本地把端口转出来。
- 不把 MySQL/PostgreSQL 的驱动打进 App（v1）。

## 分期

- **P0**：数据库卡片 + 表单 + 密钥注入 + 连通校验 + 文档（含最小权限 SQL）。
- **P1**：表单里「测试连接」并列出前 20 张表（只读），方便确认挂对了库。
- **P2**（要单独评审）：随 App 打包一个只读 SQL MCP，去掉对第三方命令的依赖。

## 已知限制

- v1 的查询能力取决于用户选的 MCP 服务器；DS Bot 只保证配置、密钥和地址策略。
- 「只读」是三层叠加：账号权限（最硬）、服务器只读标志、界面默认。任何一层被绕过都还有另外两层，但都不是数据库权限本身。
- 私有数据库通常在内网，远程 runtime（另一台机器）连不进来，这是预期行为。

## 验收（草案）

- 在本机 `.dsh-test-*` 起一个 PostgreSQL，用只读账号接入；Bot 能 `SELECT`，`INSERT` 被数据库拒绝。
- 密码不出现在 `state.json`、连接器日志、Bot 的工具输出里。
- 断开重连后不需要重新输入密码（密钥库里有）。
- 公网 + 非 TLS 的组合被拒绝并给出原因。

## 变更

（暂无）
