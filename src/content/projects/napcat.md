---
title: "Aemeath NapCat SDK"
published: 2026-05-26
description: "Aemeath NapCat SDK"
status: "published"
tags:
  - "TypeScript"
  - "napcat"
  - "onebot"
  - "qq"
  - "qqbot"
  - "sdk"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/aemeath-projects/napcat"
---

NapCat / OneBot 11 TypeScript SDK. 提供对 NapCat 的完整 TypeScript 封装，支持四种连接方式，支持 tree-shaking

![NPM Version](https://img.shields.io/npm/v/%40aemeath-projects%2Fnapcat?style=for-the-badge)
![NapCat Version](https://img.shields.io/badge/napcat-4.18.41-pink?style=for-the-badge)
![CI Status](https://img.shields.io/github/actions/workflow/status/aemeath-projects/napcat/publish.yml?style=for-the-badge)
![Codecov](https://img.shields.io/codecov/c/github/aemeath-projects/napcat?style=for-the-badge)
![License](https://img.shields.io/github/license/aemeath-projects/napcat?style=for-the-badge)

## 安装

```bash
pnpm add @aemeath-projects/napcat
```

## NapCat 版本兼容性

本 SDK 的版本号与 NapCat 版本解耦。下表列出每个 release 发布时对齐的 NapCat 版本，以及通过 [NapCatDocs](https://github.com/NapNeko/NapCatDocs) 的 OpenAPI 快照逐版本比对得到的端点层面可运行范围。

| SDK 版本 | 发布时对齐的 NapCat | 端点层面可运行范围    |
|----------|---------------------|-----------------------|
| `v1.0.0` | 未标注              | `4.12.0` – `4.18.41`  |
| `v1.1.0` | `4.18.6`            | `4.18.5` – `4.18.41`  |
| `v1.1.1` | `4.18.6`            | `4.18.5` – `4.18.41`  |
| `v1.1.2` | `4.18.6`            | `4.18.5` – `4.18.41`  |
| `v1.2.0` | `4.18.6`            | `4.18.5` – `4.18.41`  |
| `v1.2.1` | `4.18.7`            | `4.18.5` – `4.18.41`  |
| `v1.2.2` | `4.18.9`            | `4.18.5` – `4.18.41`  |
| `v1.2.3` | `4.18.9`            | `4.18.5` – `4.18.41`  |
| `v1.2.4` | `4.18.9`            | `4.18.5` – `4.18.41`  |
| `v1.2.5` | `4.18.9`            | `4.18.5` – `4.18.41`  |
| `v1.2.6` | `4.18.9`            | `4.18.5` – `4.18.41`  |
| `v1.2.7` | `4.18.9`            | `4.18.5` – `4.18.41`  |
| `v1.3.0` | `4.18.13`           | `4.18.5` – `4.18.41`  |
| `v1.3.1` | `4.18.13`           | `4.18.5` – `4.18.41`  |
| `v1.4.0` | `4.18.37`           | `4.18.30` – `4.18.41` |
| `v1.4.1` | `4.18.38`           | `4.18.30` – `4.18.41` |
| `v1.5.0` | `4.18.41`           | `4.18.30` – `4.18.41` |

- 范围表示该 release 调用的全部 action 在对应 NapCat 版本中均存在，不代表参数、返回值或实际行为完全一致。
- 发布时对齐与端点下界可能不同，例如 `v1.1.0` 标注 `4.18.6`，但其 168 个 action 在 `4.18.5` 已齐全。

**如何选择**

- 新项目：`v1.5.0` + NapCat `4.18.41`；需要整批相册评论或 `deleteGroupFile` 业务 ID 时必须使用 `4.18.41`。
- NapCat `4.18.30` – `4.18.40`：`v1.5.0` 的端点齐全，但 `4.18.41` 新增的可选参数会被忽略；`4.18.41` 以下官方不保留旧版契约兼容，生产环境建议使用 `4.18.41`。
- NapCat `4.18.5` – `4.18.13`：使用 `v1.3.1`，该代 168 个 action 完全相同。
- NapCat `4.12.0` – `4.18.4`：只有 `v1.0.0` 的 118 个 action 全部存在；需要 4.18.5 新增端点时请升级 NapCat。
- 早于 `4.12.0`：未核对。

**版本门槛**

| NapCat 版本 | 影响                                                                                                                                                                                  |
|-------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `4.18.5`    | 新增自定义表情 4 个端点与 `get_group_signed_list`，是 `v1.1.0` – `v1.3.1` 的下界。`set_group_album_media_like` 在 4.12.0 快照中为 `id` + `set`，在 4.18.x 快照中改为必填 `batch_id`。 |
| `4.18.9`    | `v1.2.2` 起 `uploadGroupFile` / `uploadPrivateFile` 适配 `uploadFile` 参数；`upload_file` 属性自 4.12.0 即存在，因此这是行为适配。                                                    |
| `4.18.14`   | 新增 `send_qzone_msg`、`delete_qzone_msg`、`set_group_member_invite_policy`、`set_group_member_permissions`、`set_group_new_member_history_visibility`。                              |
| `4.18.26`   | 快照中首次出现 `get_group_share_link`。                                                                                                                                               |
| `4.18.30`   | 新增 `upload_images_to_qun_album`、`upload_video_to_qun_album`，是 `v1.4.x` 的下界。                                                                                                  |
| `4.18.36`   | `create_flash_task` 新增可选参数 `validitySeconds`。                                                                                                                                  |
| `4.18.37`   | `v1.4.0` / `v1.4.1` 的官方契约基线；SDK 类型标注群公告 `sendToNewMember` / `pinned` 为 4.18.37+。                                                                                     |
| `4.18.38`   | `v1.4.1` 的目标版本；与 4.18.37 相比未发现路径变化。                                                                                                                                  |
| `4.18.41`   | 当前最新快照、`v1.5.0` 的目标版本。`do_group_album_comment` 的 `lloc` 改为可选并新增 `batch_id`（二者至少提供一项）；`delete_group_file` 新增可选 `bus_id` / `busid`。               |

**判定依据**

基于每个 release tag 的 `src/api/*.ts` 中 `invoke()` 的 action 集合，与 NapCatDocs 的 95 个 OpenAPI 快照（`4.12.0` – `4.18.41`）比对。该结果为静态分析，不等同于逐版本运行验证；事件推送、返回值与权限行为未覆盖。

## 快速开始

```ts
import {
  NapCatClient,
  WebSocketTransport,
  MessageApi,
  GroupApi,
  Seg,
} from '@aemeath-projects/napcat'

// 正向 WS（SDK 连接到 NapCat）
const transport = new WebSocketTransport({
  url: 'ws://localhost:3001',
  token: 'your-access-token',
  reconnect: { initialDelay: 1000, maxDelay: 30000 },
})

const client = new NapCatClient(transport)
const msg = new MessageApi(client)
const group = new GroupApi(client)

// 监听群消息
client.on('message.group', async (event) => {
  if (event.raw_message === 'ping') {
    await msg.sendGroupMsg(event.group_id, [Seg.text('pong')])
  }
})

client.on('error', (err) => console.error('连接异常:', err.message))
client.on('reconnecting', (attempt, delay) =>
  console.log(`第 ${attempt} 次重连，${delay}ms 后`)
)

await client.connect()
```

## 连接方式

| Transport                   | 用途                       |
|-----------------------------|----------------------------|
| `WebSocketTransport`        | 正向 WS：SDK 连接到 NapCat |
| `ReverseWebSocketTransport` | 反向 WS：NapCat 连接到 SDK |
| `HttpTransport`             | HTTP + 事件上报            |
| `SseTransport`              | HTTP-SSE 扩展模式          |

## API 模块

```ts
const msg = new MessageApi(client)    // 消息收发
const group = new GroupApi(client)    // 群管理
const friend = new FriendApi(client)  // 好友操作
const file = new FileApi(client)      // 文件上传/下载
const system = new SystemApi(client)  // 系统信息
const ext = new ExtensionApi(client)  // NapCat 扩展
```

所有 API 方法返回 `Result<T>` 类型：

```ts
const result = await msg.sendGroupMsg(groupId, [Seg.text('hello')])
if (result.ok) {
  console.log('发送成功，message_id:', result.data.message_id)
} else {
  console.error('发送失败，code:', result.error.code)
}
```

## 消息段构建

```ts
import { Seg } from '@aemeath-projects/napcat'

Seg.text('Hello')
Seg.at(123456)
Seg.image('http://example.com/img.png')
Seg.reply(messageId)
Seg.face(1)
// ...更多见 Seg 类型定义
```
