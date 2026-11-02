---
title: "nbnhhsh-mcp"
published: 2026-10-21
description: "「能不能好好说话？」MCP 服务器"
status: "published"
tags:
  - "TypeScript"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/aemeath-projects/nbnhhsh-mcp"
  - label: "MCP"
    icon: "simple-icons:modelcontextprotocol"
    value: "https://nbnhhsh-mcp.aemeath.dev"
---

[「能不能好好说话？」](https://lab.magiconch.com/nbnhhsh/) 的 MCP Server，
为 LLM 客户端提供中文互联网拼音首字母缩写的候选释义。

![NPM Version](https://img.shields.io/npm/v/%40aemeath-projects%2Fnbnhhsh-mcp?style=for-the-badge)
![CI Status](https://img.shields.io/github/actions/workflow/status/aemeath-projects/nbnhhsh-mcp/publish.yml?style=for-the-badge)
![License](https://img.shields.io/github/license/aemeath-projects/nbnhhsh-mcp?style=for-the-badge)

## 特性

- **两个工具**：`guess` 批量查询缩写，`explain_text` 从文本中提取并解释缩写。
- **只读**：仅查询社区词库，不提供提交释义的写操作。
- **缓存**：未命中词按批查询，结果进入 TTL + LRU 缓存，降低上游压力。
- **三种运行方式**：stdio、Streamable HTTP、Cloudflare Workers，共用同一套业务代码。
- **双输出**：每次调用同时返回 Markdown 文本与满足 `outputSchema` 的 `structuredContent`。
- **零配置**：无需 API Key，`npx` 即可运行。

## 目录

- [快速开始](#快速开始)
- [工具](#工具)
- [运行方式](#运行方式)
- [配置](#配置)
- [缓存](#缓存)
- [数据来源](#数据来源)
- [开发](#开发)

## 快速开始

直接运行（stdio，默认传输方式）：

```bash
npx -y -p @aemeath-projects/nbnhhsh-mcp nbnhhsh-mcp
```

在 Claude Code 中注册：

```bash
claude mcp add nbnhhsh -- npx -y -p @aemeath-projects/nbnhhsh-mcp nbnhhsh-mcp
```

Claude Desktop 配置示例：

```json
{
  "mcpServers": {
    "nbnhhsh": {
      "command": "npx",
      "args": ["-y", "-p", "@aemeath-projects/nbnhhsh-mcp", "nbnhhsh-mcp"]
    }
  }
}
```

> Windows 下若客户端找不到 `npx`，可将 `command` 改为 `cmd`，并在 `args` 开头加入 `/c`。

## 工具

两个工具均为只读，并声明 `readOnlyHint` / `openWorldHint` / `idempotentHint`。成功时返回
Markdown 文本与 `structuredContent`；失败时返回 `isError: true` 与单行错误信息。

通用约定：

- `limit` 为每个缩写最多返回的释义条数，默认 10，范围 1-50；超出部分通过 `truncated` 标记。
- `guess` 的 `words` 最多 100 个，单个缩写最长 64 字符。输入会先 trim、转小写、去重，
  空白项被忽略；返回结果与去重后的输入顺序一致。
- `explain_text` 的 `text` 最长 10000 字符，按 `/[a-z0-9]{2,}/gi` 提取连续字母/数字片段；
  超过 64 字符的片段会被静默忽略。唯一缩写超过 100 个时返回错误。
- `occurrences` 使用 UTF-16 下标，`start` 含、`end` 不含；`raw` 为原文片段。同一缩写最多
  记录 100 个位置，超出时 `occurrencesTruncated` 为 `true`。

### `guess`

批量查询缩写。未命中的词按每批 50 个分批并发请求上游，命中缓存的词不再请求。

```json
{
  "words": ["yyds", "xswl", "zzzznotaword"],
  "limit": 3
}
```

返回字段：

| 字段           | 类型                                          | 说明                                    |
| -------------- | --------------------------------------------- | --------------------------------------- |
| `word`         | `string`                                      | 归一化后的缩写（小写）                  |
| `status`       | `translated` \| `no-translation` \| `unknown` | 有释义 / 已收录但无释义 / 未收录        |
| `translations` | `string[]`                                    | 候选释义，已按 `limit` 截断             |
| `candidates`   | `string[]`                                    | 上游给出的“有可能是”，仅 `unknown` 非空 |
| `truncated`    | `boolean`                                     | 是否因 `limit` 发生截断                 |

### `explain_text`

从任意文本中提取缩写并查询，返回每个缩写的出现位置。

```json
{
  "text": "我永远喜欢 yyds，这波操作 xswl",
  "limit": 5
}
```

在 `guess` 的返回字段基础上，每个条目额外包含 `occurrences`（`start` / `end` / `raw`）和
`occurrencesTruncated`。文本中没有可查询片段时返回空 `terms`。

## 运行方式

### stdio

```bash
npx -y -p @aemeath-projects/nbnhhsh-mcp nbnhhsh-mcp
```

从源码运行：

```bash
pnpm build
pnpm start          # 等价于 node dist/cli.js
```

使用 MCP Inspector 调试（需先构建）：

```bash
pnpm inspect
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

# 调用 guess
curl -s -X POST http://127.0.0.1:3000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"guess","arguments":{"words":["yyds","xswl"],"limit":3}}}'

# 调用 explain_text
curl -s -X POST http://127.0.0.1:3000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"explain_text","arguments":{"text":"我永远喜欢 yyds，这波操作 xswl","limit":3}}}'
```

响应为 SSE，`result.structuredContent` 是结构化数据，`result.content` 是对应的 Markdown。
以下为示意：

```text
event: message
data: {"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text","text":"yyds：永远的神\nxswl：笑死我了"}],"structuredContent":{"results":[{"word":"yyds","status":"translated","translations":["永远的神"],"candidates":[],"truncated":false}]}}}
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
`ALLOWED_HOSTS`、`ALLOWED_ORIGINS` 与 `NBNHHSH_*` 在 Worker 的 Variables 中配置。
Worker 端 MCP 端点固定为 `/mcp`，健康检查为 `/health`。

## 配置

CLI 参数优先级高于环境变量。`MCP_ALLOWED_ORIGINS` 与 `MCP_ALLOWED_HOSTS` 未显式配置时，
绑定 localhost 会自动使用 SDK 的 localhost 白名单。

| CLI 参数                    | 环境变量                    | 默认值                                  | 说明                                      |
| --------------------------- | --------------------------- | --------------------------------------- | ----------------------------------------- |
| `--transport <stdio\|http>` | `MCP_TRANSPORT`             | `stdio`                                 | 传输方式                                  |
| `--host <host>`             | `MCP_HOST`                  | `127.0.0.1`                             | HTTP 监听地址                             |
| `--port <port>`             | `MCP_PORT`                  | `3000`                                  | HTTP 监听端口                             |
| `--path <path>`             | `MCP_PATH`                  | `/mcp`                                  | MCP 端点路径                              |
| —                           | `MCP_ALLOWED_HOSTS`         | 未设置                                  | 允许的 Host 主机名，逗号分隔              |
| —                           | `MCP_ALLOWED_ORIGINS`       | 未设置                                  | 允许的 Origin 主机名，逗号分隔            |
| —                           | `NBNHHSH_API_BASE`          | `https://lab.magiconch.com/api/nbnhhsh` | 上游 API 根地址                           |
| —                           | `NBNHHSH_TIMEOUT_MS`        | `8000`                                  | 上游请求超时（1000-30000 毫秒）           |
| —                           | `NBNHHSH_CACHE_TTL_MS`      | `600000`                                | 缓存有效期（0-86400000 毫秒，0 表示禁用） |
| —                           | `NBNHHSH_CACHE_MAX_ENTRIES` | `1000`                                  | 缓存容量上限（1-10000）                   |
| `--help` / `-h`             | —                           | —                                       | 显示帮助                                  |
| `--version` / `-v`          | —                           | —                                       | 显示版本                                  |

非法数值会告警并回退到默认值，越界值会被夹取到合法范围；无效的 `NBNHHSH_API_BASE`
同样回退到默认地址。Worker 运行时使用 `ALLOWED_HOSTS`、`ALLOWED_ORIGINS`（逗号分隔），
空值表示不启用对应校验；`NBNHHSH_*` 在 Node 与 Worker 中使用同名变量。

## 缓存

缓存为进程内 TTL + LRU，默认有效期 10 分钟、容量 1000 条。命中时刷新 LRU 顺序，过期采用
惰性删除。未收录（`unknown`）的结果同样会被缓存，避免重复请求上游。将
`NBNHHSH_CACHE_TTL_MS` 设为 0 可禁用缓存。

Worker 中的缓存是 per-isolate 内存缓存，会随实例回收，命中率只是尽力而为。

## 数据来源

- 词条来自 [itorr/nbnhhsh](https://github.com/itorr/nbnhhsh) 的公开社区词库，通过
  `https://lab.magiconch.com/api/nbnhhsh` 查询；本项目不存储词库。

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
├── cli.ts        # bin 入口，stdio/http 分流
├── config.ts     # CLI 与环境变量解析
├── env.ts        # 列表环境变量解析（Worker 可用）
├── http.ts       # 共享 web-standard fetch handler
├── index.ts      # 公共库 barrel
├── nbnhhsh/
│   ├── cache.ts      # TTL + LRU 缓存
│   ├── config.ts     # 上游与缓存配置解析
│   ├── constants.ts  # 输入/输出上限常量
│   ├── extract.ts    # 文本缩写提取
│   ├── service.ts    # 归一化、批量查询与缓存拼装
│   ├── tools.ts      # guess / explain_text 工具
│   ├── types.ts      # 领域类型
│   └── upstream.ts   # 上游请求、响应校验与错误分类
├── node.ts       # node:http + toNodeHandler
├── server.ts     # McpServer 工厂与工具注册
├── version.ts    # 服务名与版本号
└── worker.ts     # Cloudflare Worker 入口
```

`http.ts`、`server.ts`、`worker.ts`、`env.ts`、`version.ts` 与整个 `src/nbnhhsh/` 保持
Node-free，由 ESLint `no-restricted-imports` 约束；Node 适配器 `node.ts` 仅在 HTTP 模式下
动态加载。
