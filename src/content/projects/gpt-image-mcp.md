---
title: "gpt-image-mcp"
published: 2026-10-29
description: "GPT Image系列生图模型 MCP服务器"
status: "published"
tags:
  - "TypeScript"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/aemeath-projects/gpt-image-mcp"
  - label: "MCP"
    icon: "simple-icons:modelcontextprotocol"
    value: "https://gpt-image-mcp.aemeath.dev"
---

GPT Image 图像生成与编辑的 MCP Server，为 LLM 客户端提供标准化的工具调用能力。

<div style="display:flex;flex-wrap:wrap;align-items:center;gap:0.5rem;margin:1rem 0">
  <img src="https://img.shields.io/npm/v/%40aemeath-projects%2Fgpt-image-mcp?style=for-the-badge" alt="NPM Version" style="margin:0" />
  <img src="https://img.shields.io/github/actions/workflow/status/aemeath-projects/gpt-image-mcp/publish.yml?style=for-the-badge" alt="CI Status" style="margin:0" />
  <img src="https://img.shields.io/github/license/aemeath-projects/gpt-image-mcp?style=for-the-badge" alt="License" style="margin:0" />
</div>

## 特性

- **Images + Responses 双接口**：支持 `/v1/images/generations`、`/v1/images/edits` 与 Responses API 的 `image_generation` 工具。
- **GPT Image 2 / 2.5**：默认使用 `gpt-image-2.5-flare` 生成、`gpt-image-2.5-sunburst` 编辑，兼容 `gpt-image-2` 及日期快照。
- **自定义 endpoint**：通过 `GPT_IMAGE_BASE_URL` 接入 OpenAI 兼容中转站，自动补 `/v1`。
- **两端鉴权**：入站 HTTP 支持 `Authorization: Bearer` / `X-API-Key`；出站自动注入上游 API Key。
- **内联 + 落盘**：默认返回 MCP image content；Node 下可传 `output_path` 写入本地文件。
- **三种运行方式**：stdio、Streamable HTTP、Cloudflare Workers，共用同一套工具代码。
- **统一结果约定**：成功返回 Markdown + `structuredContent`（+ 可选 image content），失败走 `isError` 通道。

## 目录

- [快速开始](#快速开始)
- [工具](#工具)
- [运行方式](#运行方式)
- [配置](#配置)
- [模型与尺寸](#模型与尺寸)
- [数据来源](#数据来源)
- [开发](#开发)

## 快速开始

必须配置上游 API Key。以下示例同时适用于 OpenAI 官方端点与中转站：

```bash
export GPT_IMAGE_API_KEY=sk-...
# 可选：中转站地址，服务会自动补 /v1
export GPT_IMAGE_BASE_URL=https://relay.example.com
```

直接运行（stdio，默认传输方式）：

```bash
npx -y -p @aemeath-projects/gpt-image-mcp gpt-image-mcp
```

在 Claude Code 中注册：

```bash
claude mcp add gpt-image \
  -e GPT_IMAGE_API_KEY=sk-... \
  -e GPT_IMAGE_BASE_URL=https://api.openai.com \
  -- npx -y -p @aemeath-projects/gpt-image-mcp gpt-image-mcp
```

Claude Desktop 配置示例：

```json
{
  "mcpServers": {
    "gpt-image": {
      "command": "npx",
      "args": ["-y", "-p", "@aemeath-projects/gpt-image-mcp", "gpt-image-mcp"],
      "env": {
        "GPT_IMAGE_API_KEY": "sk-...",
        "GPT_IMAGE_BASE_URL": "https://api.openai.com"
      }
    }
  }
}
```

> 本地 mock 或无需鉴权的中转站可显式设置 `GPT_IMAGE_API_KEY=""`，此时不会发送
> `Authorization` 头。Windows 下若客户端找不到 `npx`，可将 `command` 改为 `cmd`，
> 并在 `args` 开头加入 `/c`。

## 工具

所有工具都声明 `annotations`。成功时返回 Markdown 文本与 `structuredContent`；
默认还会附带 MCP image content。失败时返回 `isError: true` 与单行中文错误信息。

| 工具                           | 用途                                      |
| ------------------------------ | ----------------------------------------- |
| `generate_image`               | 通过 Images API 生成图片                  |
| `edit_image`                   | 通过 Images API 编辑图片，支持多图与 mask |
| `generate_image_via_responses` | 通过 Responses API 生成/编辑图片          |

### `generate_image`

| 参数                 | 类型/默认                                       | 说明                                |
| -------------------- | ----------------------------------------------- | ----------------------------------- |
| `prompt`             | string，1–32000，必填                           | 提示词                              |
| `model`              | string，默认 `GPT_IMAGE_MODEL`                  | 必须以 `gpt-image-` 开头            |
| `n`                  | integer 1–10，默认 1                            | 生成数量                            |
| `size`               | string，默认 `auto`                             | `auto` 或 `宽x高`，按模型能力校验   |
| `quality`            | `low/medium/high/xhigh/max/auto`，默认 `medium` | `xhigh/max` 仅 2.5 系列支持         |
| `background`         | `transparent/opaque/auto`，默认 `auto`          | `transparent` 只能配 `png/webp`     |
| `output_format`      | `png/jpeg/webp`，默认 `png`                     | 输出格式                            |
| `output_compression` | integer 0–100，可选                             | 仅 `jpeg/webp`                      |
| `moderation`         | `auto/low`，默认 `auto`                         | 内容审核级别                        |
| `output_path`        | string，可选                                    | Node 下落盘路径                     |
| `overwrite`          | boolean，默认 `false`                           | 是否覆盖已存在文件                  |
| `inline`             | boolean，默认 `true`                            | 为 `false` 时必须提供 `output_path` |

### `edit_image`

在 `generate_image` 参数基础上增加：

| 参数             | 类型/默认            | 说明                                                             |
| ---------------- | -------------------- | ---------------------------------------------------------------- |
| `images`         | string[]，1–16，必填 | 支持 `http(s)://`、`data:`、本地路径（本地路径仅 Node）          |
| `mask`           | string，可选         | 局部重绘遮罩，来源同 `images`                                    |
| `input_fidelity` | `low/high`，可选     | 仅 `gpt-image-1.5` / `gpt-image-1-mini` 支持，2 / 2.5 会显式报错 |

单图使用 multipart 字段 `image`，多图默认使用 `image[]`。中转站不兼容时可通过
`GPT_IMAGE_MULTI_IMAGE_FIELD` 改为 `image`。

### `generate_image_via_responses`

| 参数                                   | 类型/默认                              | 说明                                        |
| -------------------------------------- | -------------------------------------- | ------------------------------------------- |
| `prompt`                               | string，1–32000，必填                  | 提示词                                      |
| `images`                               | string[]，0–16，可选                   | 作为 `input_image` 传入                     |
| `model`                                | string，默认 `GPT_IMAGE_MODEL`         | `image_generation` 工具的图像模型           |
| `text_model`                           | string，默认 `gpt-6-astra`             | Responses 顶层文本模型                      |
| `size` / `quality` / `background`      | 同上                                   | 传给 `image_generation` 工具                |
| `action`                               | `auto/generate/edit`，默认 `auto`      | 工具动作                                    |
| `effort`                               | `minimal/low/medium/high`，默认 `high` | 文本模型推理强度，映射到 `reasoning.effort` |
| `previous_response_id`                 | string，可选                           | 多轮编辑                                    |
| `output_path` / `overwrite` / `inline` | 同上                                   | 落盘与内联控制                              |

成功结果的 `structuredContent` 包含 `model`、`images[]`（`mimeType`、`bytes`、
可选 `path` / `revisedPrompt`）、`saved_paths`、`usage` 等字段；Responses 工具额外返回
`response_id`。

### 落盘命名

- `output_path` 以 `/` 或 `\` 结尾，或指向已存在目录：`image_1.png`、`image_2.png`……
- `n > 1` 且指向文件：`stem-1.png`、`stem-2.png`……
- 未提供扩展名时按 `output_format` 补全。
- 默认不覆盖；存在同名文件时整体失败。内联返回模式下会保留图片并给出 `warning`，
  `inline: false` 时直接报错。

## 运行方式

### stdio

```bash
pnpm build
GPT_IMAGE_API_KEY=sk-... pnpm start    # 等价于 node dist/cli.js
```

从 npm 运行：

```bash
GPT_IMAGE_API_KEY=sk-... npx -y -p @aemeath-projects/gpt-image-mcp gpt-image-mcp
```

### Streamable HTTP

```bash
MCP_API_KEY=local-dev GPT_IMAGE_API_KEY=sk-... pnpm start:http
```

默认监听 `http://127.0.0.1:3000/mcp`。健康检查固定为 `GET /health`，不受 `--path`
影响且不鉴权。配置 `MCP_API_KEY` 后，`/mcp` 需要携带 `Authorization: Bearer <key>`
或 `X-API-Key: <key>`。

```bash
# 健康检查（公开）
curl -s http://127.0.0.1:3000/health

# 列出工具（需要鉴权）
curl -s -X POST http://127.0.0.1:3000/mcp \
  -H 'Authorization: Bearer local-dev' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

# 生成图片
curl -s -X POST http://127.0.0.1:3000/mcp \
  -H 'Authorization: Bearer local-dev' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"generate_image","arguments":{"prompt":"a red ceramic cube","size":"1024x1024","quality":"low"}}}'
```

响应为 SSE，`result.content` 同时包含 Markdown 与 image content，
`result.structuredContent` 是不含 base64 的元数据。

Node 侧默认绑定 `127.0.0.1`，并启用 localhost 级别的 Host/Origin 校验。监听 `0.0.0.0`
或经反向代理暴露时，必须显式配置 `MCP_ALLOWED_HOSTS`、`MCP_ALLOWED_ORIGINS` 与
`MCP_API_KEY`；未配置 `MCP_API_KEY` 时启动会打印安全警告，端点不启用鉴权。

### Cloudflare Workers

```bash
pnpm worker:dev      # http://127.0.0.1:8787/mcp
pnpm worker:check    # 仅打包校验，不部署
pnpm worker:deploy   # 手动部署，仅用于调试
```

密钥使用 secret，不要写入 `wrangler.jsonc`：

```bash
wrangler secret put GPT_IMAGE_API_KEY
wrangler secret put MCP_API_KEY
```

非敏感配置（`ALLOWED_HOSTS`、`ALLOWED_ORIGINS`、`GPT_IMAGE_BASE_URL`、`GPT_IMAGE_MODEL`
等）可放在 Worker 的 Variables 中。日常发布使用 Cloudflare Workers Builds：连接 Git
仓库，生产分支设为 `master`，Build command 为 `pnpm run build`，Deploy command 为
`npx wrangler deploy`，构建环境变量只需 `NODE_VERSION=26`。

Worker 未启用 `nodejs_compat`，因此：

- 图片默认以 base64 内联返回；
- `output_path` 与本地文件路径会在请求上游之前报错；
- 编辑图片请使用 `http(s)` URL 或 base64 data URI。

## 配置

CLI 参数优先级高于环境变量。所有 `GPT_IMAGE_*` 变量由服务启动时解析；非法值会直接阻止
启动并给出中文错误。

| CLI 参数                    | 环境变量                         | 默认值                   | 说明                                       |
| --------------------------- | -------------------------------- | ------------------------ | ------------------------------------------ |
| `--transport <stdio\|http>` | `MCP_TRANSPORT`                  | `stdio`                  | 传输方式                                   |
| `--host <host>`             | `MCP_HOST`                       | `127.0.0.1`              | HTTP 监听地址                              |
| `--port <port>`             | `MCP_PORT`                       | `3000`                   | HTTP 监听端口                              |
| `--path <path>`             | `MCP_PATH`                       | `/mcp`                   | MCP 端点路径                               |
| —                           | `MCP_ALLOWED_HOSTS`              | 未设置                   | 允许的 Host 主机名，逗号分隔               |
| —                           | `MCP_ALLOWED_ORIGINS`            | 未设置                   | 允许的 Origin 主机名，逗号分隔             |
| —                           | `MCP_API_KEY`                    | 未设置                   | 入站鉴权 key，逗号分隔支持多 key           |
| —                           | `GPT_IMAGE_BASE_URL`             | `https://api.openai.com` | 自动补 `/v1`；拒绝带账号密码的 URL         |
| —                           | `GPT_IMAGE_API_KEY`              | 回退 `OPENAI_API_KEY`    | 空字符串表示不发送 Authorization           |
| —                           | `GPT_IMAGE_MODEL`                | `gpt-image-2.5-flare`    | 生成默认模型                               |
| —                           | `GPT_IMAGE_EDIT_MODEL`           | `gpt-image-2.5-sunburst` | 编辑默认模型                               |
| —                           | `GPT_IMAGE_RESPONSE_MODEL`       | `gpt-6-astra`            | Responses 顶层文本模型                     |
| —                           | `GPT_IMAGE_REASONING_EFFORT`     | `high`                   | Responses 推理强度；`none` 表示不发送      |
| —                           | `GPT_IMAGE_TIMEOUT_MS`           | `120000`                 | Images 请求超时（1000–900000）             |
| —                           | `GPT_IMAGE_RESPONSES_TIMEOUT_MS` | `300000`                 | Responses 请求超时（1000–900000）          |
| —                           | `GPT_IMAGE_OUTPUT_ROOT`          | 未设置                   | 设置后 `output_path` 必须位于该目录        |
| —                           | `GPT_IMAGE_MULTI_IMAGE_FIELD`    | `image[]`                | 多图编辑 multipart 字段名                  |
| —                           | `GPT_IMAGE_RESPONSE_FORMAT`      | 未设置                   | 附加 `response_format`（`url`/`b64_json`） |
| `--help` / `-h`             | —                                | —                        | 显示帮助                                   |
| `--version` / `-v`          | —                                | —                        | 显示版本                                   |

`MCP_API_KEY` 使用逗号分隔，因此 key 本身不能包含逗号。Worker 侧对应
`ALLOWED_HOSTS` / `ALLOWED_ORIGINS` 变量与 `MCP_API_KEY` / `GPT_IMAGE_*` secret。

## 模型与尺寸

| 模型                                      | quality                        | 任意尺寸 | transparent | input_fidelity |
| ----------------------------------------- | ------------------------------ | -------- | ----------- | -------------- |
| `gpt-image-2` 及日期快照                  | low/medium/high/auto           | 是       | 支持        | 不支持         |
| `gpt-image-2.5-flare` / `sunburst` 及快照 | low/medium/high/xhigh/max/auto | 是       | 支持        | 不支持         |
| `gpt-image-1.5`                           | low/medium/high/auto           | 否       | 支持        | 支持           |
| `gpt-image-1-mini`                        | low/medium/high/auto           | 否       | 支持        | 仅 `low`       |

任意尺寸需满足：宽高均为 16 的倍数、比例 1:3–3:1、最长边 ≤ 3840、总像素
655,360–8,294,400；高于 2560×1440 属于实验性范围。未知的 `gpt-image-*` 模型只做
前缀与语法校验，便于新模型无需改代码即可透传。

## 数据来源

- 请求发送到 `GPT_IMAGE_BASE_URL` 指向的 OpenAI 兼容服务：
  `/v1/images/generations`、`/v1/images/edits`、`/v1/responses`。
- 图片以 base64 返回并转换为 MCP image content；只有传 `output_path` 时才写入本地。
- `previous_response_id` 的多轮上下文由上游保存，数据保留策略由上游决定。
- 上游 API Key 只写入请求头，不写日志、不拼进 URL；错误信息会对 key 脱敏。
- 上游返回 `data[].url` 时，服务会下载并转为 base64；下载第三方存储地址时不携带 key。

## 开发

环境要求：Node.js `>= 24.22.1`，pnpm `12.15.0`（由 `packageManager` 固定）。

```bash
pnpm install
pnpm build
pnpm lint
pnpm type-check
pnpm worker:check
```

构建后可用 MCP Inspector 手动调试（需要配置可用的 API Key 与 endpoint）：

```bash
pnpm inspect
```

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

目录结构：

```text
src/
├── auth.ts          # 入站 API Key 校验（常量时间比较）
├── cli.ts           # bin 入口，stdio/http 分流并注入 Node 文件能力
├── config.ts        # MCP 传输层 CLI 与环境变量解析
├── env.ts           # 列表环境变量解析（Worker 可用）
├── errors.ts        # ToolError 与错误消息工具
├── http.ts          # 共享 web-standard fetch handler（含鉴权）
├── index.ts         # 公共库 barrel
├── node.ts          # node:http + toNodeHandler
├── node-files.ts    # Node 文件读写实现（仅 cli.ts 引用）
├── server.ts        # McpServer 工厂与工具注册
├── version.ts       # 服务名与版本号
├── worker.ts        # Cloudflare Worker 入口
├── gpt-image/       # 领域层
│   ├── base64.ts    # base64 / data URI / MIME 工具
│   ├── client.ts    # GptImageApiError
│   ├── config.ts    # GPT_IMAGE_* 配置与 base URL 归一化
│   ├── files.ts     # ImageIO 接口、MIME 魔数识别
│   ├── images.ts    # Images API 生成与编辑
│   ├── inputs.ts    # URL / data URI / 本地路径解析
│   ├── models.ts    # 模型能力表与参数校验
│   ├── request.ts   # fetch 封装、超时、错误映射
│   └── responses.ts # Responses API image_generation
└── tools/
    ├── images.ts    # generate_image / edit_image
    ├── output.ts    # 落盘与结果组装
    ├── register.ts  # 统一的成功/失败结果封装
    ├── responses.ts # generate_image_via_responses
    └── schemas.ts   # 共享 zod 输入/输出 schema
```

新增工具时，按 `src/tools/images.ts` 的模式实现 `registerXxxTools(server, config)`，在
`src/server.ts` 中注册，领域逻辑放入 `src/gpt-image/`。

`http.ts`、`server.ts`、`worker.ts`、`env.ts`、`version.ts`、`auth.ts`、`tools/**`、
`gpt-image/**` 保持 Node-free，由 ESLint `no-restricted-imports` 约束；Node 文件能力
`node-files.ts` 只由 `cli.ts` 注入，不会进入 Worker bundle。
