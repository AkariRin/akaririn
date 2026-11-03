---
title: "Bookshelf"
published: 2026-07-24
description: "Bookshelf：跑团日志阅读器"
status: "published"
tags:
  - "TypeScript"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/aemeath-projects/bookshelf"
  - label: "主页"
    icon: "material-symbols:link"
    value: "https://bookshelf.aemeath.dev"
---

跑团日志阅读器。读取 [Aemeath TRPG Engine](https://github.com/aemeath-projects/trpg/) 生成的 `.story` 归档，在浏览器中渲染发言时间线、检定卡片与人物卡快照。

<div style="display:flex;flex-wrap:wrap;align-items:center;gap:0.5rem;margin:1rem 0">
  <img src="https://img.shields.io/github/license/AkariRin/bookshelf?style=for-the-badge" alt="License" style="margin:0" />
</div>

## 特性

- **离线优先**：PWA 应用，安装后可离线使用
- **隐私保护**：默认不加载远程头像，避免向第三方 CDN 暴露阅读行为
- **加密支持**：自动识别加密归档并提示输入口令
- **大归档优化**：虚拟滚动处理数千条目，worker 线程解包避免主线程掉帧
- **无 HTML 注入风险**：正文一律作纯文本渲染，不解释任何标记

## 使用方式

### 本地文件

拖入或选择 `.story` 文件即可打开。文件只在本次浏览中保留在内存中，不会被保存或上传。

### 远程存储（可选）

在项目根目录创建 `.env` 文件，配置 S3 兼容存储的公开读取端点：

```bash
cp .env.example .env
# 编辑 .env，填入 VITE_S3_ENDPOINT 的值
```

> **注意**：变量名必须以 `VITE_` 开头，Vite 构建时才会将其注入到客户端代码。

配置后可在首页输入日志名称（不含 `.story` 后缀）直接打开。需在 S3 侧配置 CORS 允许本站域名。

## 开发

```bash
pnpm install
pnpm dev          # 启动开发服务器
pnpm build        # 构建生产版本
pnpm preview      # 预览生产构建
pnpm test         # 运行单元测试
pnpm lint         # 代码检查
pnpm type-check   # 类型检查
```

## 部署

站点以 Cloudflare Workers 静态资源（assets-only，无 Worker 脚本）形式托管，由 Workers Builds 从 `master` 分支构建。

控制台配置：

- Build command：`pnpm build`
- Deploy command：`pnpm worker:deploy`
- 构建变量：`VITE_S3_ENDPOINT`（与 `.env` 中同名，构建期注入；未配置时首页不提供远程日志入口）

本地验证生产构建与路由回退：

```bash
pnpm build          # 先生成 dist/
pnpm worker:check   # 校验 Wrangler 配置与资源目录
pnpm worker:dev     # http://localhost:8787，按生产构建预览
```

`worker:dev` / `worker:check` 只读取 `dist/`，不会启动 Vite 开发服务器；日常开发仍用 `pnpm dev`。

## 技术栈

- **框架**：Vue 3 + TypeScript + Vite
- **UI**：Vuetify 4 + Material Design Icons
- **状态**：Pinia + pinia-plugin-persistedstate
- **路由**：Vue Router
- **测试**：Vitest + @vue/test-utils
- **PWA**：vite-plugin-pwa + Workbox
- **解析**：[@aemeath-projects/trpg/story](https://github.com/aemeath-projects/trpg/tree/master/story)，msgpack + gzip 容器，Web Worker 解包

## 设计决策

### 为何不内置归档管理

本应用刻意不提供「归档库」功能（上传、列表、收藏等）：

1. **隐私**：日志内容敏感，托管在中心化服务会产生额外的信任要求与合规义务
2. **职责分离**：管理归档是存储层的事，阅读器只负责渲染已有字节
3. **灵活性**：用户可自行选择存储方式（本地文件、自建 S3、私有 NAS 等），阅读器通过统一的 `.story` 格式对接

### 为何默认不加载远程头像

`actor.avatar` 字段存放的是指向聊天平台 CDN 的 URL。加载它会：

- 向第三方暴露「谁在什么时间阅读了这份日志」
- 离线时必然失败

故默认只渲染名字首字作为占位，远程加载需用户在设置中显式开启。

### 为何正文不支持 Markdown / HTML

归档格式规范明确「只存纯文本」。引入任何标记渲染都会成为注入通道，且无对应收益——跑团日志的正文就是玩家原始发言，不需要也不该支持富文本。

### 为何用 Worker 解包

`unpackStory` 内部是 gunzipSync + msgpack 同步解码，在主线程直接调用会在大归档（数千条目）上造成掉帧。加密路径还要跑 600k 次 PBKDF2 迭代，更不能阻塞 UI。此外，解析不可信字节流天然适合隔离在独立上下文中。

## 许可

MIT
