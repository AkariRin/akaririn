---
title: "sb6657-mcp"
published: 2026-10-21
description: "玩机器烂梗收集 MCP服务器"
status: "published"
tags:
  - "TypeScript"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/aemeath-projects/sb6657-mcp"
  - label: "主页"
    icon: "material-symbols:link"
    value: "https://sb6657-mcp.aemeath.dev"
---

[玩机器烂梗收集](https://sb6657.cn) 的 MCP Server，为 LLM 客户端提供只读的
烂梗检索、榜单与聚合数据。

![NPM Version](https://img.shields.io/npm/v/%40aemeath-projects%2Fsb6657-mcp?style=for-the-badge)
![CI Status](https://img.shields.io/github/actions/workflow/status/aemeath-projects/sb6657-mcp/publish.yml?style=for-the-badge)
![License](https://img.shields.io/github/license/aemeath-projects/sb6657-mcp?style=for-the-badge)

## 特性

- **只读**：仅查询公开数据，不包含投稿、点赞、投票等写操作。
- **11 个工具**：覆盖搜索、热榜、详情、标签、榜单、生命周期、DNA、赛事、相册、屏蔽词与静态战报。
- **三种运行方式**：stdio、Streamable HTTP、Cloudflare Workers，共用同一套业务代码。
- **双输出**：每次调用同时返回 Markdown 文本与满足 `outputSchema` 的 `structuredContent`。
- **零配置**：无需 API Key，`npx` 即可运行。

## 目录

- [快速开始](#快速开始)
- [工具](#工具)
- [运行方式](#运行方式)
- [配置](#配置)
- [数据来源](#数据来源)
- [开发](#开发)

## 快速开始

直接运行（stdio，默认传输方式）：

```bash
npx -y -p @aemeath-projects/sb6657-mcp sb6657-mcp
```

在 Claude Code 中注册：

```bash
claude mcp add sb6657 -- npx -y -p @aemeath-projects/sb6657-mcp sb6657-mcp
```

Claude Desktop 配置示例：

```json
{
  "mcpServers": {
    "sb6657": {
      "command": "npx",
      "args": ["-y", "-p", "@aemeath-projects/sb6657-mcp", "sb6657-mcp"]
    }
  }
}
```

> Windows 下若客户端找不到 `npx`，可将 `command` 改为 `cmd`，并在 `args` 开头加入 `/c`。

## 工具

所有工具均为只读，并声明 `readOnlyHint`。成功时返回 Markdown 文本与 `structuredContent`；
失败时返回 `isError: true` 与单行错误信息。分页参数 `page` 默认 1，`pageSize` 默认 10、上限 50。

| 工具                 | 用途                                             | 关键参数                                                                   |
| -------------------- | ------------------------------------------------ | -------------------------------------------------------------------------- |
| `search_memes`       | 搜索或分页浏览烂梗                               | `query`、`tags[]`、`startDate`/`endDate`、`sort=default\|copies`、分页     |
| `get_hot_memes`      | 24 小时 / 7 天热榜、随机一条、首页词云           | `scope=24h\|7d\|random\|wordcloud`                                         |
| `get_meme_detail`    | 烂梗详情，可选返回梗 DNA                         | `barrageId`、`includeDna`（默认 `true`）                                   |
| `list_meme_tags`     | 标签字典（编码 ↔ 中文名）                        | 无                                                                         |
| `get_meme_rankings`  | 烂度总榜、实时热度榜、今日 PK、历史周榜          | `scope=stale\|hot\|arena-current\|arena-weekly`、`weekStart`、分页         |
| `get_meme_lifecycle` | 生命周期看板，或按阶段浏览                       | `stage=BIRTH\|BOOM\|STALE\|DEAD`（省略则返回看板）、分页                   |
| `get_meme_dna`       | 梗 DNA v6、进化图谱与旧版关联                    | `memeId`、`mode=v6\|evolution\|legacy`                                     |
| `get_matches`        | 进行中赛事、赛事列表、预测阶段、队伍池、赛事烂梗 | `scope`；`teams` 需 `matchId`+`phase`，`memes` 需 `matchId`；分页          |
| `get_gallery`        | 时光相册图片与评论                               | 分页                                                                       |
| `get_shield_words`   | 屏蔽词字典或分页列表                             | `mode=dict\|list`、分页                                                    |
| `get_static_report`  | 年度 Top20、布雷德十五勇士、超级逮虾户战报       | `report=top20-2025\|top20-2024\|warriors-2026\|warriors-2025\|dejavu-niko` |

调用约定：

- 标签参数既接受编码（如 `06`），也接受中文名（如 `群魔乱舞`），完整字典见 `list_meme_tags`。
- 日期筛选必须同时提供 `startDate`、`endDate` 和 `query`，且起始日期不能晚于结束日期。
- `weekStart` 仅在 `scope=arena-weekly` 时生效。
- 赛事相关接口在非赛事期可能返回空数据。

## 运行方式

### stdio

```bash
npx -y -p @aemeath-projects/sb6657-mcp sb6657-mcp
```

从源码运行：

```bash
pnpm build
pnpm start          # 等价于 node dist/cli.js
```

### Streamable HTTP

```bash
pnpm start:http     # 等价于 node dist/cli.js --transport http
```

默认监听 `http://127.0.0.1:3000/mcp`。健康检查固定为 `GET /health`，不受 `--path` 影响。

```bash
# 健康检查
curl -s http://127.0.0.1:3000/health

# 列出工具
curl -s -X POST http://127.0.0.1:3000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

# 搜索烂梗
curl -s -X POST http://127.0.0.1:3000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"search_memes","arguments":{"query":"原神","pageSize":3}}}'
```

响应为 SSE，`result.structuredContent` 是结构化数据，`result.content` 是对应的 Markdown：

```text
event: message
data: {"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text","text":"# 烂梗搜索：原神\n..."}],"structuredContent":{"page":1,"pageSize":3,"total":42,"hasMore":true,"items":["..."]}}}
```

Node 侧默认绑定 `127.0.0.1`，并启用 localhost 级别的 Host/Origin 校验。监听 `0.0.0.0`
或经反向代理暴露时，必须显式配置 `MCP_ALLOWED_HOSTS` 与 `MCP_ALLOWED_ORIGINS`。

### Cloudflare Workers

```bash
pnpm worker:dev      # http://127.0.0.1:8787/mcp
pnpm worker:check    # 仅打包校验，不部署
pnpm worker:deploy   # 手动部署，仅用于调试
```

日常发布使用 Cloudflare Workers Builds：在 Dashboard 连接 Git 仓库，生产分支设为
`master`，Root directory 为 `/`，Build command 为 `pnpm run build`，Deploy command 为
`npx wrangler deploy`，构建环境变量只需 `NODE_VERSION=26`。运行时变量
`ALLOWED_HOSTS`、`ALLOWED_ORIGINS` 在 Worker 的 Variables 中配置。Worker 端 MCP 端点
固定为 `/mcp`，健康检查为 `/health`。

## 配置

CLI 参数优先级高于环境变量。`MCP_ALLOWED_ORIGINS` 与 `MCP_ALLOWED_HOSTS` 未显式配置时，
绑定 localhost 会自动使用 SDK 的 localhost 白名单。

| CLI 参数                    | 环境变量              | 默认值      | 说明                           |
| --------------------------- | --------------------- | ----------- | ------------------------------ |
| `--transport <stdio\|http>` | `MCP_TRANSPORT`       | `stdio`     | 传输方式                       |
| `--host <host>`             | `MCP_HOST`            | `127.0.0.1` | HTTP 监听地址                  |
| `--port <port>`             | `MCP_PORT`            | `3000`      | HTTP 监听端口                  |
| `--path <path>`             | `MCP_PATH`            | `/mcp`      | MCP 端点路径                   |
| —                           | `MCP_ALLOWED_HOSTS`   | 未设置      | 允许的 Host 主机名，逗号分隔   |
| —                           | `MCP_ALLOWED_ORIGINS` | 未设置      | 允许的 Origin 主机名，逗号分隔 |
| `--help` / `-h`             | —                     | —           | 显示帮助                       |
| `--version` / `-v`          | —                     | —           | 显示版本                       |

Worker 运行时使用 `ALLOWED_HOSTS`、`ALLOWED_ORIGINS`（逗号分隔），空值表示不启用对应校验。

## 数据来源

- 数据来自 sb6657 公开接口 `https://api.hguofichp.cn` 与 OSS 静态 JSON。

## 开发

环境要求：Node.js `>= 24.0.0`，pnpm `12.14.0`（由 `packageManager` 固定）。

```bash
pnpm install
pnpm build
```

构建产物输出到 `dist/`：

- `dist/index.js`：公共库入口
- `dist/http.js`：web-standard HTTP handler
- `dist/cli.js`：命令行入口（含 shebang）
- `dist/chunk-*.js`：代码分割产生的共享 chunk

常用脚本：

| 脚本                | 说明                           |
| ------------------- | ------------------------------ |
| `pnpm dev`          | tsup watch 构建                |
| `pnpm lint`         | ESLint 检查                    |
| `pnpm lint:fix`     | ESLint 自动修复                |
| `pnpm format`       | Prettier 检查                  |
| `pnpm format:fix`   | Prettier 格式化                |
| `pnpm type-check`   | TypeScript 类型检查            |
| `pnpm inspect`      | MCP Inspector 调试（需先构建） |
| `pnpm worker:dev`   | 本地 Worker 开发               |
| `pnpm worker:check` | Worker 打包校验                |
| `pnpm bump`         | 版本号、CHANGELOG、tag 与提交  |
| `pnpm commit`       | Commitizen 交互式提交          |

完整脚本（含 `bump:patch` / `bump:minor` / `bump:major`、`changelog`、`start:http` 等）
见 `package.json`。

目录结构：

```text
src/
├── cli.ts       # bin 入口，stdio/http 分流
├── config.ts    # CLI 与环境变量解析
├── env.ts       # 列表环境变量解析（Worker 可用）
├── http.ts      # 共享 web-standard fetch handler
├── index.ts     # 公共库 barrel
├── node.ts      # node:http + toNodeHandler
├── server.ts    # McpServer 工厂与工具注册
├── version.ts   # 服务名与版本号
├── worker.ts    # Cloudflare Worker 入口
├── sb6657/      # 数据访问层
│   ├── client.ts    # API/OSS 请求、超时、包络解包与错误映射
│   ├── format.ts    # Markdown 渲染与分页摘要
│   ├── tags.ts      # 标签字典缓存与中英文解析
│   └── types.ts     # API 类型、归一化输出与公共 zod schema
└── tools/       # 工具域
    ├── analytics.ts # 榜单、生命周期、DNA
    ├── matches.ts   # 赛事与赛事烂梗
    ├── memes.ts     # 搜索、热门、详情、标签
    ├── misc.ts      # 相册、屏蔽词、静态战报
    └── register.ts  # 统一的成功/失败结果封装
```

`http.ts`、`server.ts`、`worker.ts`、`env.ts`、`version.ts` 保持 Node-free，由 ESLint
`no-restricted-imports` 约束；Node 适配器 `node.ts` 仅在 HTTP 模式下动态加载。
