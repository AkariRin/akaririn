---
title: "Aemeath Exostrider"
published: 2026-06-18
description: "隧者：TypeScript 事件驱动框架，内置装饰器路由、服务生命周期编排、会话状态机、客户端池管理、消息与命令解析、env 配置中心与结构化日志"
status: "archived"
tags:
  - "TypeScript"
  - "decorator"
  - "dependency-injection"
  - "event-dispatcher"
  - "event-driven"
  - "message-routing"
  - "session-management"
  - "state-machine"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/aemeath-projects/exostrider"
  - label: "npm"
    icon: "simple-icons:npm"
    value: "https://www.npmjs.com/package/@aemeath-projects/exostrider"
---

隧者：TypeScript 事件驱动框架，内置装饰器路由、服务生命周期编排、会话状态机、客户端池管理、消息与命令解析、env 配置中心与结构化日志

![NPM Version](https://img.shields.io/npm/v/%40aemeath-projects%2Fexostrider?style=for-the-badge)
![CI Status](https://img.shields.io/github/actions/workflow/status/aemeath-projects/exostrider/publish.yml?style=for-the-badge)
![Codecov](https://img.shields.io/codecov/c/github/aemeath-projects/exostrider?style=for-the-badge)
![License](https://img.shields.io/github/license/aemeath-projects/exostrider?style=for-the-badge)

**Exostrider** | 隧者 是一个平台无关的事件驱动框架库，专为构建基于装饰器的消息处理系统而设计。

框架由八个解耦模块组成：

| 模块            | 功能                                             |
|---------------|------------------------------------------------|
| **Echo**      | 扫描指定目录，自动 import 触发装饰器副作用注册                    |
| **Lifecycle** | 拓扑排序服务依赖，提供 `@Startup`/`@Shutdown` 生命周期钩子与依赖注入 |
| **Dispatch**  | 七种路由策略（命令/正则/关键词等）的复合映射，支持拦截器链；`@OnCommand` 支持 `take` 截断声明 |
| **Session**   | 基于有限状态机的会话管理，支持超时自动取消与并发锁                      |
| **Logger**    | pino 封装，全局注入，支持外部订阅日志事件                        |
| **Pool**      | 客户端连接池管理，支持多角色、负载均衡、健康检测与事件去重                |
| **Message**   | 平台无关消息视图与命令解析：token 化、头部识别（`head.prefix`/`head.word`）、`take` 截断、`enums` 候选集（独立工具模块） |
| **Config**    | 基于 TypeBox 的 env 配置中心，零缓存加载器与领域配置访问器（独立工具模块）  |

通过 `Exostrider` 门面类一行完成 bootstrap，泛型 `TEvent`/`TApis` 由宿主（调用方）传入，实现零耦合接入任意平台。

```ts
const ex = new Exostrider({ echo: { ... }, dispatch: { ... } })
await ex.bootstrap()
await ex.dispatch(event, apis)
await ex.shutdown()
```

## 安装

```bash
pnpm add @aemeath-projects/exostrider
```
