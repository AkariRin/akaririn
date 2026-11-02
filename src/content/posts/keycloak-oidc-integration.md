---
title: "把自建服务接上 Keycloak：OIDC 客户端配置清单"
published: 2026-08-22T23:52:27+08:00
description: "自建服务多了以后，每个服务一套账号很难管理。本文讲清 Keycloak 里 OIDC client 的每个关键字段，以及回调地址、组映射、token 校验、登出这些最容易踩坑的地方，并以一个真实服务为例走完整流程。"
tags: [Keycloak, OIDC, SSO, Docker]
category: "教程"
image: ""
series: "Keycloak 实战"
seriesOrder: 2
draft: true
comment: true
---

上一篇文章把 Keycloak 跑起来了，但它本身只是个登录页。真正的价值发生在「下游服务把登录交给它」之后：用户只输一次密码，Grafana、文档站、代码托管各自拿到身份，管理员只需要在一个地方加组。

一次 OIDC 登录的骨架并不复杂。用户访问服务，服务发现没有会话，于是把浏览器重定向到 Keycloak 的授权端点；用户在 Keycloak 登录后，浏览器带着一次性授权码回到服务登记过的回调地址；服务在后台用授权码换 token；最后服务校验 token 的签名、issuer 和 audience，建立自己的会话。后面所有配置项都在回答两个问题：**这次跳转允许回到哪里**，以及 **token 里应该带什么**。

## OIDC 里的四个关键地址

每个 realm 都有一份 discovery 文档，所有端点都从它派生。先把它取下来：

```bash
curl -s https://auth.example.com/realms/homelab/.well-known/openid-configuration \
  | jq '{issuer, authorization_endpoint, token_endpoint, jwks_uri, userinfo_endpoint, end_session_endpoint}'
```

四个核心地址的职责：

| 地址 | 作用 | 谁访问 |
|---|---|---|
| `issuer` | 身份提供方标识，token 里的 `iss` 必须等于它 | 服务端校验 |
| `authorization_endpoint` | 浏览器跳转登录的入口 | 浏览器 |
| `token_endpoint` | 用授权码换 access/id token | 服务端 |
| `jwks_uri` | 签名公钥集合 | 服务端 |

`issuer` 是最容易被忽视、又最容易出事的一个。它不是「大概指向 Keycloak」的字符串，而是 token 校验的硬约束：下游会拿 token 里的 `iss` 和 discovery 返回的 `issuer` 做**逐字比较**。只要 `KC_HOSTNAME` 配错、反代没传 `X-Forwarded-Proto`，或者服务端从容器内网地址拉 discovery，`issuer` 就会变成 `http://keycloak:8080/...` 之类的内部地址，浏览器侧登录一切正常，服务端却拒绝 token。

判断方法只有一条：从**下游服务所在的网络位置**执行上面的 curl，看 `issuer` 是不是浏览器实际访问的那个地址。在 Docker Compose 里，这意味着要区分两个视角：浏览器用 `https://auth.example.com`，容器之间可以用 `http://keycloak:8080` 换 token，但 discovery 返回的 `issuer` 必须仍然是外部地址。这靠的是 Keycloak 侧的 `KC_HOSTNAME` 固定值，而不是请求头。

还有一个容易被忽略的细节：下游服务通常会在启动时拉一次 discovery，并把 JWKS 缓存起来。换域名、换证书或重建 realm 之后，旧进程里缓存的公钥和 issuer 可能仍然有效，表现为「配置已经改对了，服务还是报签名错误」。遇到这种情况先重启下游，再判断是不是 Keycloak 的问题。反过来，如果你在调试 issuer，也不要只改浏览器地址——服务端的 discovery 结果才是它实际信任的那一份。

## 创建 Client：先选对类型

进入 Admin Console → Clients → Create client。Keycloak 会先问 client type，这个选择决定了后面哪些开关可用。

**传统 Web 应用（有后端）**：选 OpenID Connect，Client authentication 打开，得到 client ID 和 client secret。授权码在后端交换，secret 不出现在浏览器里。绝大多数自建服务属于这一类，Grafana、GitLab、Outline 都是。

**单页应用或移动端**：Client authentication 关闭，成为 public client，必须使用 PKCE。没有 secret 可保密，安全性来自一次性 code verifier。

**机器对机器**：打开 Service accounts roles，用 client credentials 模式直接拿 token。这类 client 不需要浏览器，也不该出现在登录页上。

创建页面上还有一组能力开关，默认值值得保留：

- **Standard flow**：授权码模式，Web 应用必须开。
- **Direct access grants**：用户名密码直接换 token，等于绕过登录页。除了迁移期做脚本测试，生产环境应关闭。
- **Implicit flow**：已过时，不要开。
- **Service accounts roles**：只有机器账号才开。

client ID 一旦被下游写进配置，就不要改。它同时出现在授权请求、token 的 `azp`/`aud` 声明和下游的环境变量里，改名意味着两边一起改；部分服务还会把 client ID 当作账号关联的稳定标识。取一个全小写、能看出用途的名字，例如 `grafana`、`docs`、`gitlab`，比 `my-app-2` 这类名字省事得多。

建好之后，Clients 列表里会出现新条目。下图是接入 Grafana 之后的列表，`grafana` 与内建的 `account`、`admin-cli` 并列：

![](./images/keycloak-oidc-integration-1.png)

## 回调地址与 Web Origins

这是配置里最容易返工的一节。两个字段名字相近，作用完全不同。

**Valid redirect URIs** 控制授权成功后允许跳回的地址，防止攻击者把授权码送到自己的站点。Keycloak 的匹配规则是精确匹配，只允许末尾一个 `*` 作为通配：

- `https://grafana.example.com/login/generic_oauth` — 只允许这一个完整地址。
- `https://grafana.example.com/*` — 允许该站点下任意路径，但仍限制主机和协议。
- `https://grafana.example.com` — **不匹配**带路径的回调，也不匹配 `http://` 或别的端口。

**Web Origins** 控制浏览器跨域请求。纯后端换 token 的服务不需要它；前端直接调 userinfo 或 token 端点的 SPA 才需要。填 `+` 表示继承 redirect URI 里的 origin，这是最省事的写法。

最常见的三个错误：

1. **协议不一致。** 反代已经上了 HTTPS，client 里还写着 `http://`。浏览器地址栏是 `https://`，回调却是 `http://`，Keycloak 直接拒绝。
2. **端口不一致。** 本地测试时服务跑在 `13001`，client 里填了 `3000`。容器内外的端口是两回事，回调地址以**浏览器地址栏**为准。
3. **结尾斜杠。** `/callback` 和 `/callback/` 是两个不同的 URI，不会自动归一化。

最可靠的做法不是照文档抄，而是打开下游服务的登录页，点一次 OIDC 登录，从地址栏或开发者工具的网络面板里复制 `redirect_uri` 参数，逐字填进 Keycloak。各服务的文档经常落后于实现，浏览器实际发出的那一跳才是真相。

Grafana 的 client 详情如下，Root URL、Home URL 和回调都指向 `http://localhost:13001`，与浏览器访问地址一致：

![](./images/keycloak-oidc-integration-2.png)

## 声明（Claims）与组映射

Keycloak 默认**不发** `groups` claim。这不是配置漏了，而是设计如此：组信息可能很大，不该默认塞进每个 token。要让它出现，需要三步都做对。

**第一步，建一个 client scope 并加 Group Membership mapper。** 可以建 realm 级的共享 scope（推荐，多个 client 复用），也可以在每个 client 的 Dedicated scopes 里单独加。共享 scope 的做法是 Client scopes → Create client scope，名字取 `groups`，类型 OpenID Connect，然后进 Mappers → Add mapper → Group Membership。

关键字段：

| 字段 | 建议值 | 说明 |
|---|---|---|
| Token Claim Name | `groups` | 必须与下游期望的 claim 名一致 |
| Full group path | Off | 输出裸组名 `admins`，而不是 `/admins` |
| Add to ID token | On | Grafana 等从前端 id_token 读取 |
| Add to access token | On | 资源服务器从 access token 读取 |
| Add to userinfo | On | 走 userinfo 端点的服务需要 |

`full.path` 是最隐蔽的一项。设成 On 时 claim 里是 `/admins`，带前导斜杠；很多下游不做归一化，直接按 `admins` 匹配就会失败。更糟的是部分系统（例如 Proxmox VE）会丢弃含 `/` 的组名，表现为「登录成功但没有任何权限」，日志里一行错误都没有。统一输出裸组名，让下游按原样匹配，是维护成本最低的选择。

三个 token 出口也建议全开。不同服务读取的位置不一样：Grafana 读 id_token，部分 Go 服务读 userinfo，资源服务器读 access token。只开一个，必然有一半服务拿不到组。

**第二步，把 scope 挂到 client 上。** 在 client 的 Client scopes 页添加 `groups` 为 Default，或设为 realm 的 default client scope。注意：**realm 默认 scope 不回溯**，设置之前已经存在的 client 不会自动带上，必须逐个补挂。漏挂的症状非常安静——登录正常、用户能进，只是所有组都是空数组。

下图是 `groups` scope 的 Mappers 页，`full.path` 已设为 false：

![](./images/keycloak-oidc-integration-3.png)

**第三步，处理 audience。** 有些下游会校验 `aud` 必须包含自己的 client ID。Keycloak 默认把 `aud` 设成 `account` 或请求方，未必符合预期。在 client 的 Dedicated scopes → Mappers 里加一个 Audience mapper，把目标 client ID 写进 Included Client Audience；或者在共享 scope 上做同样的事。只有下游明确报 `invalid audience` 时才需要，不要预先给所有 client 加。

验证不要靠猜。Admin Console → Clients → `grafana` → Client scopes → Evaluate，选一个用户，点 Generated ID token，应该看到：

```json
"groups": ["admins"]
```

不在组里的用户应该得到 `"groups": []`，而不是字段缺失。字段缺失说明 mapper 或 scope 没生效；空数组说明链路是通的，只是用户没进组。

## 登出与会话

OIDC 的登出比登录更容易被忽略。Keycloak 的 `end_session_endpoint` 接受 `post_logout_redirect_uri`，但这个地址必须登记在 client 的 **Valid post logout redirect URIs** 里，否则登出后只会停在 Keycloak 的提示页。

一个完整的登出地址形如：

```
https://auth.example.com/realms/homelab/protocol/openid-connect/logout
  ?client_id=grafana
  &post_logout_redirect_uri=https://grafana.example.com/login
```

还要区分两种「登出」。**本地登出**只清掉下游服务自己的会话，Keycloak 的 SSO 会话还在，下次点登录会直接放行，用户会觉得「怎么没退出去」。**单点登出**要由下游调用 end_session_endpoint，或者由 Keycloak 通过 back-channel/front-channel logout 通知各服务。后者需要下游支持，不是所有服务都实现了。实际使用中，Grafana 这类服务配置 `signout_redirect_url` 指向 Keycloak 的登出端点即可；不支持 back-channel 的服务，只能接受「本地登出、SSO 会话仍在」的行为。

会话时长在 Realm settings → Sessions 里配置。默认空闲 30 分钟、最长 10 小时；`accessTokenLifespan` 默认 5 分钟。小团队可以把空闲调到 8 小时、最长 7 天，减少重复登录；token 寿命不要跟着放大，access token 短、refresh token 长才是正常组合。

## 一个完整的接入实例

用 Grafana 走一遍完整流程。它的 OIDC 配置全在环境变量里，最适合演示「哪些地址给浏览器、哪些给容器」。

```yaml
services:
  grafana:
    image: grafana/grafana:13.3.0
    container_name: grafana
    environment:
      GF_SERVER_ROOT_URL: https://grafana.example.com
      GF_AUTH_GENERIC_OAUTH_ENABLED: "true"
      GF_AUTH_GENERIC_OAUTH_NAME: Homelab
      GF_AUTH_GENERIC_OAUTH_CLIENT_ID: grafana
      GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET: "<your-secret>"
      GF_AUTH_GENERIC_OAUTH_SCOPES: openid profile email groups
      # 浏览器跳转用外部地址
      GF_AUTH_GENERIC_OAUTH_AUTH_URL: https://auth.example.com/realms/homelab/protocol/openid-connect/auth
      # 服务端换 token 走容器网络，避免绕回公网
      GF_AUTH_GENERIC_OAUTH_TOKEN_URL: http://keycloak:8080/realms/homelab/protocol/openid-connect/token
      GF_AUTH_GENERIC_OAUTH_API_URL: http://keycloak:8080/realms/homelab/protocol/openid-connect/userinfo
      GF_AUTH_GENERIC_OAUTH_LOGIN_ATTRIBUTE_PATH: preferred_username
      GF_AUTH_GENERIC_OAUTH_NAME_ATTRIBUTE_PATH: name
      GF_AUTH_GENERIC_OAUTH_EMAIL_ATTRIBUTE_PATH: email
      GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH: "contains(groups[*], 'admins') && 'Admin' || 'Viewer'"
      GF_AUTH_SIGNOUT_REDIRECT_URL: https://auth.example.com/realms/homelab/protocol/openid-connect/logout?client_id=grafana&post_logout_redirect_uri=https://grafana.example.com/login
    ports:
      - "3000:3000"
```

Keycloak 侧对应四项配置：client ID `grafana`、Client authentication 打开、回调 `https://grafana.example.com/login/generic_oauth`、Web Origins `https://grafana.example.com`。secret 从 Credentials 页复制，填进 Grafana 的环境变量。

`ROLE_ATTRIBUTE_PATH` 是 Grafana 自己的表达式：从 `groups` 数组里找 `admins`，命中就是 Admin，否则 Viewer。它依赖的正是前面那个 mapper。如果组名带了斜杠，这里必须写成 `/admins`，否则永远匹配不上——所以 `full.path=false` 不只是「好看」。

保存后重启 Grafana，打开登录页，应出现 `Sign in with Homelab`。登录成功后进入首页：

![](./images/keycloak-oidc-integration-4.png)

如果右上角显示的是用户名而不是邮箱，说明 `LOGIN_ATTRIBUTE_PATH` 生效了；如果能看到 Admin 菜单，说明组映射也通了。这两个检查比看日志快得多。

## 排错：从错误码倒推

| 现象 | 最可能的原因 | 先看哪里 |
|---|---|---|
| `Invalid parameter: redirect_uri` | 回调地址与登记值不一致 | 浏览器地址栏里的 `redirect_uri` 原文，逐字比对协议、端口、斜杠 |
| `invalid_client` / `unauthorized_client` | client 类型或 secret 错 | Client authentication 是否打开；Grafana 里的 secret 是否与 Credentials 页一致 |
| 登录成功但下游 401 | issuer、audience 或时钟 | discovery 的 `issuer`；token 的 `aud`；服务器时间是否同步 |
| 下游报 `x509: certificate signed by unknown authority` | 容器不信任你的 CA | 从容器内 curl discovery；Go 用 `SSL_CERT_FILE`，Node 用 `NODE_EXTRA_CA_CERTS` |
| `groups` 为空或缺失 | scope 没挂、用户没进组、mapper 出口没开 | Client scopes → Evaluate 生成 token，逐项核对 |
| 登录后立刻跳回登录页 | 回调地址对但 cookie/secure 配置不匹配 | 反代是否传 `X-Forwarded-Proto`；服务是否要求 HTTPS cookie |
| `Account is not fully set up` | 用户缺少必填 profile 字段 | Users → 该用户 → 补 email 或姓名 |

这张表的用法是从上往下缩小范围，而不是一次查所有项。`redirect_uri` 和 `invalid_client` 属于「配置写错」，通常在 Keycloak 侧就能解决；401 和 x509 属于「下游不认」，要看服务端日志；`groups` 为空属于「链路通了但内容不对」，去 Evaluate 页看 claim 最快。

## 小结

接一个服务，实际只需要确认四件事：回调地址逐字一致、client 类型选对、`groups` scope 挂上且 `full.path` 关闭、登出地址登记。剩下的错误大多可以归到 issuer 或信任库这两个根因上。

下一篇处理长期运行才会暴露的问题：Passkey 怎么开、realm 邮件怎么配、登录页怎么改，以及升级和安全加固时该守住哪些底线。
