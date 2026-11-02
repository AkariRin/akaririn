---
title: "Keycloak 进阶：Passkey、邮件与长期运维"
published: 2026-08-29T23:52:27+08:00
description: "Keycloak 跑起来只是开始。这篇文章整理几件迟早要做的事：用 Passkey 替代密码、配好 realm 邮件、定制登录页、调整会话与 token 生命周期，以及升级和安全加固时该注意什么。"
tags: [Keycloak, Passkey, WebAuthn, Docker]
category: "教程"
image: ""
series: "Keycloak 实战"
seriesOrder: 3
draft: true
comment: true
---

认证系统上线后的前两周，注意力都在「能不能登录」；半年后真正消耗时间的，是另外三件事：用户不想再记密码、密码重置邮件发不出去、升级时担心把数据库改坏。这三件事都不会在部署当天暴露，但迟早要处理。

这篇按「少用密码 → 邮件可用 → 长期可维护」的顺序展开。前提是前两篇已经完成：实例在跑，至少有一个 realm 和一个能登录的下游服务。

## 用 Passkey 替代密码

Passkey 在 Keycloak 里有两套策略，容易混淆：

- **WebAuthn Policy**：把安全密钥当作**第二因子**，登录仍然先输密码。
- **WebAuthn Passwordless Policy**：允许**无密码**登录，凭据本身就是身份。

本文讲后者。它对应 Authentication → Policies → WebAuthn Passwordless Policy，也可以直接在 realm 的 JSON 里改。26.8 中 Passkeys 已是默认特性（`serverinfo` 里类型为 `DEFAULT`），不需要额外开 feature flag。

关键字段和我的实测值：

| 字段 | 值 | 说明 |
|---|---|---|
| RP ID | `auth.example.com` | 凭据绑定的域名，必须是稳定的 |
| RP Entity Name | `Homelab` | 认证器弹窗里显示的服务名 |
| Signature Algorithms | `ES256`, `RS256` | 覆盖 Apple 与 Windows/Android |
| Attestation | `none` | 不上传证明链，自用场景足够 |
| Resident Key | `Yes` | 无用户名登录的前提 |
| User Verification | `required` | 必须验证指纹/面容/PIN |
| Avoid Same Authenticator | `false` | 允许同一设备登记多条 |

本地测试时 RP ID 是 `localhost`，因为浏览器访问的就是 `http://localhost:18080`。这个值不是随手填的：浏览器只在**当前 origin 与 RP ID 匹配**时才出示凭据。RP ID 必须是 origin 的可注册域后缀；`localhost` 是特例，允许 HTTP。生产环境用 `https://auth.example.com`，RP ID 就填 `auth.example.com`，不要带协议、端口或路径。

**换域名等于所有 Passkey 作废。** 凭据仍然留在用户的认证器里，但浏览器不会再为新的 origin 提供它，服务端也不会有任何日志。用户看到的现象只是「我的 passkey 不见了」。所以 RP ID、realm 名、`KC_HOSTNAME`、证书 SAN 应该在第一次上线前一次定死，不要留过渡域名。

开放自助登记需要打开 required action，但**不要设为默认强制**：

```bash
docker exec kc-demo-keycloak /opt/keycloak/bin/kcadm.sh \
  update authentication/required-actions/webauthn-register-passwordless \
  -r homelab -s enabled=true -s defaultAction=false
```

我实测的登记路径是：用户登录 Account Console → Account security → Signing in → Passwordless → Set up Passkey → 浏览器弹出认证器 → Register。登记成功后，同一页会出现凭据条目：

![](./images/keycloak-advanced-1.png)

Keycloak 26 的登录页还有一个「Sign in with Passkey」入口，点击后不需要先输入用户名。未登记的设备仍然回落到用户名加密码，这条回退路径不能被切断——Passkey 是加法，不是替换。

还有一条恢复路径要提前准备：注册了 Passkey 的用户如果换了设备、又忘了密码，就会同时失去两种登录方式。建议至少保留一种可自助恢复的凭据（例如设置密码或 TOTP），或者由管理员在 Users → Credentials 里重置。不要为了「纯无密码」把密码和恢复选项全部关掉，否则一次换机就会变成人工工单。

策略页在 Authentication → Policies，可以同时看到 Password、OTP、WebAuthn、WebAuthn Passwordless 和 CIBA 五组：

![](./images/keycloak-advanced-3.png)

回滚分两种：单个用户在 Account Console 里删除该凭据；整体停用则把 required action 的 `enabled` 改回 false。已登记的凭据会保留，只是不再被触发。

## 配好 Realm 邮件

Keycloak 26 起，SMTP 是 **per-realm** 配置，位置在 Realm settings → Email。网上很多老教程还在让你改启动参数或全局环境变量，那些做法对应的是旧版本，照抄不会生效。

需要填的核心字段：

| 字段 | 示例 | 注意 |
|---|---|---|
| From | `keycloak@example.com` | 必须与 SMTP 账号允许的发信地址一致 |
| From display name | `Homelab` | 收件人看到的发信人名称 |
| Host | `mailpit` | 容器间用服务名，不要写 `localhost` |
| Port | `1025` | 465 是隐式 TLS，587 是 STARTTLS，别混 |
| Authentication | Off / On | 本地 Mailpit 关闭；真实服务商一般开启 |
| StartTLS | Off / On | 与端口匹配 |
| SSL | Off / On | 465 端口才开 |

我本地用 Mailpit 做闭环测试：compose 里加一个 `axllent/mailpit` 容器，realm 的 Host 填 `mailpit`、Port 填 `1025`、From 填 `keycloak@example.com`，认证和 TLS 都关闭。配置页如下：

![](./images/keycloak-advanced-2.png)

然后打开「忘记密码」，输入一个已存在的邮箱。Keycloak 会走 SMTP 发信，Mailpit 的 Web 界面立即能看到：

![](./images/keycloak-advanced-4.png)

这封邮件的 From、To、Subject 和正文链接都正常，说明 per-realm SMTP 链路是通的。真实环境把 Host 换成邮件服务商地址、补上用户名密码、按端口选择 STARTTLS 或 SSL 即可。

常见失败按概率排：

1. **465 与 587 混淆。** 465 要求从一开始就 TLS（SSL 开关），587 要求先明文连接再 STARTTLS。选错的表现通常是连接超时或 `SSLHandshakeException`。
2. **25 端口被云厂商封锁。** 很多主机商默认封 25，改用 587 提交端口。
3. **From 与账号不一致。** 服务商校验发信地址，`from` 必须是已验证的域名或账号。
4. **容器内 DNS 解析失败。** Host 写 `localhost` 会指向 Keycloak 容器自己；同一 compose 网络里用服务名。
5. **26.8 的 STARTTLS 收紧。** 一旦开启 StartTLS，服务器必须真的协商成功；对方不宣告或中间设备剥离 TLS 时，发信会直接失败，不再退回明文。这是升级后「以前能发、现在发不出」的一个新原因。

改完配置后，用页面上的 Test connection 发一封测试信，比等到用户点「忘记密码」才发现要快得多。

## 登录页与品牌

品牌化有两条路径，按投入选。

**第一条：只换主题名。** Realm settings → Themes → Login theme，留空表示跟随版本默认。26.8 的内置登录主题是 `keycloak.v2`（对应 `LOGIN_V2` 特性），账户控制台是 `keycloak.v3`；具体可选值以当前镜像为准，不要照抄旧版本教程里的 `keycloak`。这条路径零维护，升级不会坏。

**第二条：最小覆盖。** 不要复制整个内置主题，只覆盖需要改的文件。目录结构大致是：

```
themes/homelab/login/theme.properties
themes/homelab/login/resources/css/custom.css
```

`theme.properties` 里继承内置主题：

```properties
parent=keycloak.v2
styles=css/login.css css/custom.css
```

然后把整个 `themes/homelab` 挂进容器：

```yaml
volumes:
  - ./themes/homelab:/opt/keycloak/themes/homelab:ro
```

再在 Realm settings → Themes 里把 Login theme 设为 `homelab`。改完要重启容器，或者至少让主题缓存失效；Keycloak 会缓存主题资源，改完不重启经常看不到变化。

不要改 `login.ftl` 里的结构，除非你准备在每次大版本升级时重新对齐模板。绝大多数需求（logo、主色、页脚文字、背景图）都能靠 CSS 变量和 `theme.properties` 完成。把改动限制在 CSS，升级成本最低。

## 会话、Token 与并发

Keycloak 26.8 的 realm 默认值如下，我逐项读过：

| 配置 | 默认值 | 含义 |
|---|---|---|
| Access Token Lifespan | 5 分钟 | access token 有效期 |
| SSO Session Idle | 30 分钟 | 空闲多久后 SSO 会话失效 |
| SSO Session Max | 10 小时 | SSO 会话的绝对上限 |
| Offline Session Idle | 30 天 | 离线会话空闲上限 |
| Refresh Token Max Reuse | 0 | 刷新令牌可复用次数；0 表示不限制，仅在轮换开启时有意义 |
| Revoke Refresh Token | false | 是否启用 refresh token 轮换 |

小团队或家庭场景，我建议把 **SSO Session Idle 调到 8 小时、Max 调到 7 天**，access token 保持 5–15 分钟。理由很实际：一天里反复登录很烦，但 access token 是 bearer token，泄露即生效，寿命越短越好。refresh token 负责在后台续期，用户感知不到。

**打开 refresh token 轮换**（Revoke Refresh Token = true）能降低 refresh token 被复用的风险，代价是每次刷新都会生成新 token，个别老服务可能不适应。先在一个 client 上验证，再全局打开。

client 级可以覆盖 realm 级。Clients → `<client>` → Advanced settings 里有 Access Token Lifespan、Client Session Idle、Client Session Max、Client Offline Session Idle/Max。只给某个长任务服务放宽，不要为了一个服务改整个 realm。

**offline session 要克制。** 它用于「用户不在场也要长期访问」的场景，比如定时任务。普通 Web 应用开了只会积累无用会话，最后表现为内存和数据库连接缓慢上涨。遇到「内存持续走高」，先看会话数量，再考虑加内存。

## 升级与回滚

升级流程和第一篇一致，但长期运行后多了两个验证点：Passkey 和邮件。

```yaml
# compose.yaml
services:
  keycloak:
    image: kc-demo/keycloak:<newer-tag>   # 替换为目标版本，例如 26.8.2
```

```dockerfile
FROM quay.io/keycloak/keycloak:<newer-tag> AS builder
# 其余不变
```

步骤：备份数据库 → 读 release notes（重点看数据库迁移、配置项弃用、SMTP 行为变化）→ 同时改 Dockerfile 和 compose 的 tag → `docker compose up -d --build` → 观察日志里的 Liquibase 迁移 → 验证登录、一个下游服务、一封测试邮件、一条 Passkey。

**升级是单向的。** Keycloak 启动时会自动迁移 schema，降回旧镜像不会回滚表结构。所谓回滚，实际是「恢复升级前的数据库 dump + 换回旧镜像」，不是把 tag 改回去就完事。升级前那份 dump 是唯一的安全网。

不要跨多个大版本一次跳。26.0 → 26.1 → 26.2 逐级升，每一级都留一份可恢复的备份。小版本之间通常只需几分钟，大版本之间的迁移可能需要停服更久。

26.8 有两条行为收紧值得提前知道：开启 StartTLS 后必须真的协商成功；只有 `view-clients` 权限的调用方拿不到 client secret（返回掩码），需要读 secret 的自动化要补 `manage-clients`。如果你的备份脚本或监控依赖读 secret，升级前先检查。

## 安全加固清单

下面每一条都能在十分钟内完成，收益明确：

1. **关闭所有 client 的 Direct Access Grants。** 它允许用用户名密码直接换 token，绕过登录页和 MFA。迁移脚本除外，生产环境不应保留。
2. **强制 HTTPS。** 反代层做 301 + HSTS，realm 的 Require SSL 设为 `all requests`。本文测试环境用 `external requests`，是因为本地是 HTTP。
3. **打开暴力破解检测。** Realm settings → Security defenses → Brute force detection。建议阈值 10 次、初始等待 60 秒、最长 900 秒。默认是关闭的。
4. **管理员账号独立。** 不要用日常账号兼任管理员；单独邮箱、强口令，最好只从内网或 VPN 访问 `/admin/`。
5. **关闭自助注册。** Realm settings → Login → User registration，除非你确实要开放注册。开放时至少打开 Email verification。
6. **定期导出 realm。** 导出配置（不含用户）作为数据库备份之外的第二层，便于比对「谁改了哪个 client」。
7. **限制会话。** 按上一节的建议设置 Idle/Max，不要让会话无限续期。
8. **跟进镜像更新。** 订阅 Keycloak 的安全公告，至少每个小版本评估一次；不要长期停在某个旧 tag 上。

## 备份与恢复演练

备份分两层，缺一不可。

**第一层是数据库。** 它包含全部状态：realm、client、用户、组、Passkey 凭据、会话索引。

```bash
docker exec kc-demo-postgres pg_dump -U keycloak -Fc keycloak \
  > keycloak-$(date +%F).dump
```

**第二层是 realm 配置导出。** Admin Console → Realm settings → Action → Partial export，勾选 clients、groups、roles，不勾 users。它适合做配置审计和跨环境迁移，但不能替代数据库备份——用户和凭据不在里面。

恢复顺序：停 Keycloak → 重建空库 → `pg_restore` → 启动 → 验证登录和一个下游服务。如果只恢复了 realm 配置而数据库是新的，**client secret 会重新生成**，所有下游都要更新 secret；这是恢复演练里最容易漏的一步。恢复后先核对一个 client 的 secret 是否与下游配置一致，再逐个检查。

建议每季度做一次恢复演练，至少在测试环境跑一遍。备份文件存在不等于能恢复，只有真正 `pg_restore` 过一次，才知道它是否完整。

## 小结

长期维护 Keycloak，实际只需要守住几条线：

- Passkey 的 RP ID 与访问域名绑定，上线前一次定死。
- 邮件是 per-realm 配置，端口与 TLS 模式必须匹配。
- 主题只覆盖 CSS，不改模板。
- access token 短、SSO 会话按使用习惯放宽。
- 升级前备份，回滚靠数据库 dump，不靠改 tag。

做到这些，这套认证服务可以安静地跑很久；剩下的工作，基本只是偶尔加一个 client、加一个用户。
