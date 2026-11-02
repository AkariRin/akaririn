---
title: "用 Docker Compose 部署 Outline 知识库"
published: 2026-09-02T23:41:34+08:00
description: "Outline 是一个界面干净、支持 Markdown 的团队知识库。本文用 Docker Compose 搭一套完整实例：PostgreSQL、Redis、S3 兼容存储、邮件与可选的 OIDC 登录，并整理附件上传、数据库迁移这些容易卡住的地方。"
tags: [Outline, Docker, Wiki, 知识库]
category: "教程"
image: ""
draft: true
comment: true
---

团队文档最常见的归宿是聊天记录和网盘：找起来靠关键词，权限靠自觉，版本靠文件名。自建知识库的价值不在于「有一个 Wiki」，而在于把文档、附件、搜索、权限和备份放进同一套可掌控的边界里。

Outline 的定位是团队 Wiki。它的编辑器支持 Markdown 语法和 `/` 快捷插入，支持实时协作、层级文档、集合与权限，界面比 Wiki.js 更接近现代 SaaS，部署复杂度又比 Confluence 低得多。和 Docmost 相比，Outline 的生态更成熟，S3 附件、OIDC、邮件这些外围能力都有明确的环境变量入口。本文用 Docker Compose 搭一套能真正跑起来的实例，重点处理三件容易出问题的事：数据库迁移、第一个管理员、附件直传 S3。

## 依赖与整体结构

整套服务是五个容器：

```mermaid
flowchart LR
    B["浏览器"]
    O["outline"]
    B -->|"http://localhost:13000"| O
    B -.->|"附件上传时直接 POST"| M["S3 (MinIO)"]
    O -->|"结构化数据、会话、权限"| P["PostgreSQL"]
    O -->|"缓存、队列、协作状态"| R["Redis"]
    O -->|"附件对象"| M
```

- **outline**：应用本体，本身不存文件，重启不会丢数据。
- **postgres**：文档、用户、权限、附件元数据都在这里。本文用 16。
- **redis**：缓存和队列，不承担持久化职责。本文关闭 RDB/AOF，重启后缓存丢失可以接受；正在编辑的协作状态会中断，但文档内容不会丢。
- **minio**：S3 兼容对象存储，存附件。桶名 `outline`。
- **mailpit**：仅用于本地测试邮件，不接真实服务商。生产环境删掉这个服务，换成真实 SMTP 参数即可。

MinIO 在 2025 年调整了镜像分发，`minio/minio` 在 Docker Hub 上已经无法直接拉取。我这次用的是 Bitnami 归档的 `bitnamilegacy/minio:2025.4.22`，功能等价，而且支持用 `MINIO_DEFAULT_BUCKETS` 在启动时自动建桶。如果你从其他镜像源拿到官方镜像，把镜像行换掉，再手动执行 `mc mb` 建桶即可。S3 端也可以换成 SeaweedFS，只要保持 path-style 和浏览器可达的 bucket URL，本文不展开。

端口方面，为了不和本机其他服务冲突，示例统一做了偏移：Outline `13000`、PostgreSQL `15433`、Redis `16379`、MinIO `19011/19012`、Mailpit `18035/11035`。生产环境通常只需要暴露 Outline 的 3000（或反代后的 443），数据库和对象存储走内网。

## Compose 编排

完整文件如下，项目名固定为 `ol-demo`。代码里的 `SECRET_KEY`、`UTILS_SECRET` 和所有 `change-me-please` 都是本地示例值，照抄前请重新生成并替换。

```yaml
name: ol-demo

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: outline
      POSTGRES_PASSWORD: change-me-please
      POSTGRES_DB: outline
    ports:
      - "15433:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U outline -d outline"]
      interval: 5s
      timeout: 3s
      retries: 20

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--save", "", "--appendonly", "no"]
    ports:
      - "16379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20

  minio:
    image: bitnamilegacy/minio:2025.4.22
    environment:
      MINIO_ROOT_USER: outline-s3
      MINIO_ROOT_PASSWORD: change-me-please
      MINIO_DEFAULT_BUCKETS: outline
      MINIO_API_CORS_ALLOW_ORIGIN: http://localhost:13000
    ports:
      - "19011:9000"
      - "19012:9001"
    volumes:
      - miniodata:/bitnami/minio/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 5s
      timeout: 3s
      retries: 20

  mailpit:
    image: axllent/mailpit:v1.33.0
    ports:
      - "18035:8025"
      - "11035:1025"

  outline:
    image: outlinewiki/outline:1.12.0
    command:
      - sh
      - -c
      - node_modules/.bin/sequelize db:migrate --env=production-ssl-disabled && node build/server/index.js
    environment:
      NODE_ENV: production
      URL: http://localhost:13000
      PORT: 3000
      FORCE_HTTPS: "false"
      SECRET_KEY: ef2b241a0ec4c3333100ebfe6a46103a3bba06d8256ba8df9b880e82a26db0c5
      UTILS_SECRET: 7e20fb8c8e7bc38e597bd79448289a455b0366688c7e60fe10c4557d96da5dfa
      DATABASE_URL: postgres://outline:change-me-please@postgres:5432/outline
      PGSSLMODE: disable
      REDIS_URL: redis://redis:6379
      FILE_STORAGE: s3
      AWS_ACCESS_KEY_ID: outline-s3
      AWS_SECRET_ACCESS_KEY: change-me-please
      AWS_REGION: us-east-1
      AWS_S3_UPLOAD_BUCKET_NAME: outline
      AWS_S3_UPLOAD_BUCKET_URL: http://localhost:19011
      AWS_S3_FORCE_PATH_STYLE: "true"
      AWS_S3_ACL: private
      SMTP_HOST: mailpit
      SMTP_PORT: 1025
      SMTP_SECURE: "false"
      SMTP_FROM_EMAIL: noreply@example.com
      ENABLE_UPDATES: "false"
    ports:
      - "13000:3000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      minio:
        condition: service_healthy
      mailpit:
        condition: service_started

volumes:
  pgdata:
  miniodata:
```

几个配置项值得单独说明：

- `SECRET_KEY` 必须是 64 个十六进制字符，生成方式是 `openssl rand -hex 32`。它用于加密数据库里的 token 和会话，**上线后不要再改**，否则已有登录态和集成会全部失效。`UTILS_SECRET` 同样要持久化。
- `DATABASE_URL` 用容器名 `postgres`，不是 `localhost`。`PGSSLMODE=disable` 是给应用进程用的，避免它对本地 PostgreSQL 发起 SSL 握手。
- `URL` 必须是浏览器最终访问的地址，它决定 cookie 域、邮件链接和 OIDC 回调。本地是 `http://localhost:13000`，生产应改成 `https://wiki.example.com`。`FORCE_HTTPS` 在本地必须设为 `false`，否则页面会被强制跳转到 https。
- `AWS_S3_UPLOAD_BUCKET_URL` 是**浏览器**访问 S3 的地址，不是容器内地址。这是后面附件一节的核心。
- `AWS_S3_FORCE_PATH_STYLE=true` 对 MinIO 是必需项，它使用 `host/bucket/key` 的路径风格，开启后签名和寻址才一致。
- `AWS_S3_ACL=private` 会进入上传表单。MinIO 接受这个值；如果你换成 SeaweedFS，需要在身份策略里补 `s3:PutObjectAcl`，否则上传会 403。

`docker compose up -d` 之后，用 `docker compose ps` 应该看到五个服务，其中数据库、Redis、MinIO 为 healthy，Outline 在完成迁移后也会转为 healthy。

```text
$ docker compose ps
NAME                 IMAGE                           COMMAND                  SERVICE    CREATED             STATUS                   PORTS
ol-demo-mailpit-1    axllent/mailpit:v1.33.0         "/mailpit"               mailpit    About an hour ago   Up 4 minutes (healthy)   0.0.0.0:11035->1025/tcp, [::]:11035->1025/tcp, 0.0.0.0:18035->8025/tcp, [::]:18035->8025/tcp
ol-demo-minio-1      bitnamilegacy/minio:2025.4.22   "/opt/bitnami/script…"   minio      About an hour ago   Up 4 minutes (healthy)   0.0.0.0:19011->9000/tcp, [::]:19011->9000/tcp, 0.0.0.0:19012->9001/tcp, [::]:19012->9001/tcp
ol-demo-outline-1    outlinewiki/outline:1.12.0      "docker-entrypoint.s…"   outline    About an hour ago   Up 4 minutes (healthy)   0.0.0.0:13000->3000/tcp, [::]:13000->3000/tcp
ol-demo-postgres-1   postgres:16-alpine              "docker-entrypoint.s…"   postgres   About an hour ago   Up 4 minutes (healthy)   0.0.0.0:15433->5432/tcp, [::]:15433->5432/tcp
ol-demo-redis-1      redis:7-alpine                  "docker-entrypoint.s…"   redis      About an hour ago   Up 4 minutes (healthy)   0.0.0.0:16379->6379/tcp, [::]:16379->6379/tcp
```

## 首次启动与数据库迁移

Outline 官方镜像的入口脚本只做了一件事：如果第一个参数不是系统命令，就补上 `node`。它**不会**自动执行数据库迁移。镜像默认的启动命令是 `node build/server/index.js`，直接启动的话，应用面对的是空库或旧表结构，通常会报错退出。

所以 compose 里的 `command` 把迁移和应用启动串起来：

```bash
node_modules/.bin/sequelize db:migrate --env=production-ssl-disabled && node build/server/index.js
```

这里有两个实测出来的细节。

第一，1.12.0 镜像里没有 `yarn`。执行 `yarn --version` 会得到 `yarn: not found`，照抄旧文档里的 `yarn sequelize db:migrate` 会直接失败。镜像里已经带了 `sequelize` 的可执行文件，直接调用 `node_modules/.bin/sequelize` 即可。

第二，迁移必须显式指定 `--env=production-ssl-disabled`。Outline 的 `server/config/database.js` 在 `production` 环境下强制给 PostgreSQL 加 SSL 参数，而本地容器里的 PostgreSQL 没有开启 SSL。即使应用侧已经设置了 `PGSSLMODE=disable`，迁移 CLI 仍然会按 `production` 配置走，报错：

```text
ERROR: The server does not support SSL connections
```

`--env=production-ssl-disabled` 是 Outline 镜像里预置的另一个配置项，内容和 `production` 相同，但不加 SSL。应用进程启动后则读取 `PGSSLMODE=disable`，两边各管各的。

首次启动会执行数据库迁移。以 1.12.0 为例，日志里是一串 `== 2026...: migrated`；其中 `baseline-schema` 一次性建立基础表结构，并把历史迁移记录写进 `SequelizeMeta`，跑完这张表共 312 条记录。迁移结束后会进入 lifecycle 日志：

```text
$ docker compose logs outline | head -10
outline-1  | ☠ [MISSING_ENV_FILE] missing file (.env). fix: [https://github.com/dotenvx/dotenvx/issues/484]
outline-1  |
outline-1  | Sequelize CLI [Node: 26.12.1, CLI: 6.6.5, ORM: 6.37.8]
outline-1  |
outline-1  | Loaded configuration file "server/config/database.js".
outline-1  | Using environment "production-ssl-disabled".
outline-1  | == 20260104155140-baseline-schema: migrating =======
outline-1  | == 20260104155140-baseline-schema: migrated (2.164s)
outline-1  |
outline-1  | == 20260107213946-create-access-requests: migrating =======

... 中间省略 40 余条迁移 ...

outline-1  | {"label":"lifecycle","level":"info","message":"Note: Restricting process count to 1 due to use of collaborative service without REDIS_COLLABORATION_URL"}

$ docker compose logs outline | tail -2
outline-1  | {"level":"warn","message":"Enforced https was disabled with FORCE_HTTPS env variable"}
outline-1  | {"label":"lifecycle","level":"info","message":"Listening on http://localhost:3000 / http://localhost:13000"}
```

启动需要十几到二十秒，`docker compose logs -f outline` 看到 lifecycle 日志后再访问页面。想确认迁移落库，可以查：

```bash
docker compose exec -T postgres psql -U outline -d outline \
  -c 'SELECT count(*) FROM "SequelizeMeta";'
```

如果容器反复重启，先看日志里是连接失败还是迁移失败。数据库没起来时，`depends_on` 的 healthcheck 已经挡住了大部分情况；剩下的多半是密码、库名或 SSL 参数。

## 创建管理员与基础设置

数据库为空时，访问 `http://localhost:13000` 会自动进入创建工作区页面，只要求三项：工作区名称、管理员名称、管理员邮箱。没有密码输入框，提交后直接进入 `/home`，第一个用户自动成为管理员。

![Outline 首次创建工作区页面](./images/outline-docker-deploy-1.png)

这个设计对应 Outline 的认证模型：它没有传统的用户名密码登录，内置的 `email` 认证走一次性登录链接，另外支持 passkey 和外部 OIDC。首次 setup 之所以能直接进入，是因为服务端在创建用户时同时签发了会话；一旦登出或会话过期，就要靠邮件里的链接重新登录。

这也意味着 **SMTP 不是可选项**。没有可用的 SMTP，邀请成员、找回密码、邮件登录链接都发不出去。首次 setup 仍然能完成，但管理员登出后就很难再回到系统。可以先用本地 Mailpit 把链路跑通，再换成真实服务商。

进入工作区后，左侧是集合与个人文档，中间是首页的最近浏览、热门、最近更新。第一个用户的角色可以直接在数据库里确认：

```bash
docker compose exec -T postgres psql -U outline -d outline \
  -c "SELECT name, email, role FROM users;"
```

输出里 `role` 为 `admin`。如果后续需要把另一个已存在账号提为管理员：

```sql
UPDATE users SET role = 'admin' WHERE email = 'admin@example.com';
```

改完后让该用户重新登录即可生效，不需要重启容器。

![Outline 知识库首页](./images/outline-docker-deploy-2.png)

## 附件上传：S3 直传的坑

这是整套部署里最容易误判的一段。Outline 1.12 默认的 `AWS_S3_UPLOAD_METHOD` 是 `post`，附件上传分成两段完全独立的链路：

1. 编辑器调用 `POST /api/attachments.create`。服务端在数据库里建一条附件记录，并用 S3 凭据生成一个预签名 POST 表单，返回给浏览器。
2. 浏览器拿表单**直接 POST 到 `AWS_S3_UPLOAD_BUCKET_URL/<bucket>`**，不经过 Outline 服务端。

第二段是浏览器发起的，所以服务端日志里不会出现上传失败的错误。附件表有记录、桶里却没有对象，通常就是这一段断了。

配置上有四个硬要求：

- **`AWS_S3_UPLOAD_BUCKET_URL` 必须是浏览器可达的地址。** 本地填 `http://localhost:19011`；生产填 `https://s3.example.com`。如果填成容器内的 `http://minio:9000`，浏览器 DNS 解析失败，请求根本到不了 MinIO，而 Outline 侧一条 error 都不会有。
- **`AWS_S3_FORCE_PATH_STYLE=true`。** MinIO 使用 path-style 寻址，不开启时预签名 URL 的路径和实际桶地址对不上，常见表现是 404 或签名不匹配。
- **CORS 要放行 Outline 的源。** MinIO 通过 `MINIO_API_CORS_ALLOW_ORIGIN` 控制，本文显式设为 `http://localhost:13000`。生产环境改成 `https://wiki.example.com`。如果页面是 HTTPS 而桶地址是 HTTP，浏览器会按混合内容拦截，表现为上传按钮无反应。
- **桶必须存在。** 本文用 Bitnami 镜像的 `MINIO_DEFAULT_BUCKETS=outline` 在启动时自动建桶。换成官方镜像或其他 S3 实现时，先确认桶已创建。

还有一个容易忽略的点：Outline 默认 `AWS_S3_ACL=private`，上传表单会带 `ACL` 字段。MinIO 接受这个值；部分 S3 兼容实现（例如 SeaweedFS）会把 ACL 当作独立授权动作，需要在身份策略里显式允许 `s3:PutObjectAcl`，否则带 ACL 的上传一律 403。

判断「上传真的成功」不能看页面有没有报错，也不能只看数据库。实测判据有三个：

1. 浏览器 Network 里 `POST /api/attachments.create` 返回 200，紧接着有一条 `POST http://localhost:19011/outline` 返回 **204**。
2. MinIO 桶里出现对象。路径形如 `uploads/<userId>/<attachmentId>/<filename>`：

```bash
docker exec ol-demo-minio-1 /opt/bitnami/minio-client/bin/mc \
  alias set local http://127.0.0.1:9000 outline-s3 change-me-please
docker exec ol-demo-minio-1 /opt/bitnami/minio-client/bin/mc \
  ls -r local/outline
```

```text
Added `local` successfully.
[2026-11-02 11:54:44 UTC]  42KiB STANDARD uploads/30f90c4f-d755-44d1-8c03-fd1a56d80581/60114ac6-fe5a-4266-8807-158f8f829528/outline-docker-deploy-2.png
```

3. 刷新文档后，图片仍然能显示。此时 Network 里会先请求 `GET /api/attachments.redirect?id=...`，返回 302，再跳转到 MinIO 的预签名 GET 地址并返回 200。这说明读取也走通了，而不只是浏览器内存里的临时预览。

![在 Outline 编辑器中插入图片](./images/outline-docker-deploy-3.png)

![刷新后附件从 MinIO 正常读取](./images/outline-docker-deploy-4.png)

排查时按现象分流：服务端无报错、桶为空，先查 `AWS_S3_UPLOAD_BUCKET_URL`；上传返回 403，查 CORS 和 ACL；返回 404，查 path-style 和桶名；页面是 HTTPS 但桶地址是 HTTP，先解决协议一致性。

建议每篇文章都用文档编辑器实测一次图片上传和刷新后读取，不要只测头像。头像和文档附件走的是同一套 S3 逻辑，但文档附件的路径、权限和展示链路更完整。

## 邮件通知

Outline 通过 SMTP 发信。只要设置了 `SMTP_HOST` 或 `SMTP_SERVICE`，`EMAIL_ENABLED` 就为真，不需要额外的开关。本文接本地 Mailpit，配置是：

```yaml
SMTP_HOST: mailpit
SMTP_PORT: 1025
SMTP_SECURE: "false"
SMTP_FROM_EMAIL: noreply@example.com
```

`SMTP_SECURE` 的默认值是 `true`，表示一开始就建立 TLS 连接，对应 465 端口。Mailpit 的 1025 是明文端口，必须显式设为 `false`。同样的规则适用于真实服务商的 587 端口：587 使用 STARTTLS，先把 `SMTP_SECURE` 设为 `false`，让 nodemailer 先明文连接再升级；如果服务商要求 465 隐式 TLS，才设为 `true`。

`SMTP_FROM_EMAIL` 必须是合法邮箱格式，否则应用启动时的环境变量校验就会失败。发件人最好和 SMTP 账号的授权域一致，否则真实服务商可能拒信。

本地验证很简单：完成 setup 后，Mailpit 的收件箱里会出现一封主题为 `Welcome to Outline!` 的邮件，发件人是 `noreply@example.com`，正文里带工作区地址。也可以直接查 API：

```bash
curl -s http://localhost:18035/api/v1/messages
```

返回 JSON 里的 `total` 大于 0，说明 Outline 到 SMTP 的链路是通的。Mailpit 的 Web 界面在 `http://localhost:18035`，不需要登录。

生产环境把 `SMTP_HOST`、`SMTP_PORT`、`SMTP_USERNAME`、`SMTP_PASSWORD`、`SMTP_FROM_EMAIL` 换成服务商提供的值，删掉 mailpit 服务。不要把真实密码写进 compose 文件，至少用 `.env` 或容器编排平台注入的 secret。

## 可选：接入 Keycloak OIDC

如果已经有一套 Keycloak，可以让 Outline 走 OIDC。本文的 compose 没有启用，这一节只给接入点，不展开 Keycloak 侧的 client 配置。

Outline 支持两种写法：只给 `OIDC_ISSUER_URL`，让它自动发现端点；或者手动给三个地址。常用变量：

```yaml
OIDC_CLIENT_ID: outline
OIDC_CLIENT_SECRET: <your-secret>
OIDC_ISSUER_URL: https://auth.example.com/realms/homelab
OIDC_DISPLAY_NAME: Keycloak
OIDC_USERNAME_CLAIM: preferred_username
OIDC_SCOPES: openid profile email
```

回调地址由 Outline 的 `URL` 拼出来，固定为 `${URL}/auth/oidc.callback`。本地是 `http://localhost:13000/auth/oidc.callback`，生产是 `https://wiki.example.com/auth/oidc.callback`。Keycloak 的 client 里必须登记这个地址，协议、域名、端口、路径任何一处不一致都会在回调阶段失败。

两个实用开关：

- `OIDC_DISABLE_REDIRECT=true`：当 OIDC 是唯一登录方式时，Outline 默认会自动跳转到身份提供商。设成 true 可以先显示登录页，方便同时保留邮件登录或排查问题。
- `NODE_EXTRA_CA_CERTS`：如果身份提供商使用私有 CA 签发的证书，Node 默认不信任。把 CA 证书挂进容器，并设置这个变量指向证书路径，否则回调时会在 TLS 校验阶段失败。

OIDC 部分没有在这次 Compose 环境里实测，接入前建议先用 `curl -I` 确认 Outline 容器能访问 issuer 地址，再走一遍浏览器登录。

## 升级、备份与迁移

Outline 每次发版都可能带数据库迁移，而且迁移不保证向后兼容。升级的稳妥顺序是：先备份数据库和桶，再拉新镜像，最后让迁移自动执行。

```bash
# 1. 备份数据库
docker compose exec -T postgres pg_dump -U outline -Fc outline \
  > backup/outline-$(date +%F).dump

# 2. 备份附件桶
docker run --rm --network ol-demo_default \
  -v "$PWD/backup:/backup" \
  --entrypoint /bin/sh bitnamilegacy/minio:2025.4.22 -c \
  '/opt/bitnami/minio-client/bin/mc alias set local http://minio:9000 outline-s3 change-me-please >/dev/null && /opt/bitnami/minio-client/bin/mc mirror --overwrite local/outline /backup/outline'

# 3. 升级
docker compose pull outline
docker compose up -d outline
docker compose logs -f outline
```

升级后重点看三件事：迁移是否全部完成、页面能否登录、附件能否上传并刷新显示。只回滚镜像 tag 是不够的，因为新版本可能已经改过表结构；回滚需要同时恢复数据库备份，必要时再恢复桶。`SECRET_KEY` 和 `UTILS_SECRET` 在升级、迁移、换机器时都必须保持不变。

换域名时，至少同步改四处：`URL`、`AWS_S3_UPLOAD_BUCKET_URL`、OIDC 回调地址、反向代理的转发规则。旧域名签发的 cookie 会失效，用户需要重新登录；如果附件桶地址也变了，旧文档里的附件链接需要重新生成或做重定向。

`docker compose down -v` 会删除 `pgdata` 和 `miniodata` 两个卷，附件和文档一起消失。验证环境可以这么做，生产环境不要带 `-v`。

## 常见问题

**容器启动即退出，日志报 SSL 错误。** 迁移命令没有加 `--env=production-ssl-disabled`，或者 `DATABASE_URL` 指向了要求 SSL 的数据库。先看日志里的 `ERROR: The server does not support SSL connections`，再对照 compose 的 command。

**页面能开，但上传附件 403。** 先看浏览器 Network 里直传 S3 的那条请求。403 通常是 CORS 没放行 Outline 的源，或者 S3 侧缺少 ACL 授权；如果请求根本没发出去，检查 `AWS_S3_UPLOAD_BUCKET_URL` 是否浏览器可达、协议是否和页面一致。

**登录后立刻跳回登录页。** 多数是 `URL` 和实际访问地址不一致，导致 cookie 域对不上；也可能是 `FORCE_HTTPS=true` 但实际用 HTTP 访问。把浏览器地址栏的 scheme、host、port 和 `URL` 对齐后再试。

**邮件发送超时。** 先确认 `SMTP_SECURE` 和端口匹配：465 用 `true`，587 和 1025 用 `false`。其次是 `SMTP_FROM_EMAIL` 格式和发件域授权。

**附件在编辑器里能看，刷新后裂图。** 说明上传成功但读取地址不可达。检查 `AWS_S3_UPLOAD_BUCKET_URL` 是否仍能被浏览器访问，以及预签名 GET 是否返回 200。

**升级后页面白屏。** 先强刷浏览器缓存，再确认反向代理没有缓存旧的静态资源。Outline 的静态文件带 hash，但 HTML 入口被缓存时仍可能引用旧 chunk。

## 小结

Outline 适合小团队或 homelab：一台 2C4G 的机器、一块 SSD、PostgreSQL 和 MinIO 各一份持久卷就能跑起来。它的资源开销主要来自 Node 进程和 PostgreSQL，附件多了之后要给对象存储留足空间。

这套部署的难点不在容器数量，而在三个边界：迁移必须手动串在启动命令里，并且要绕开 production 的 SSL 配置；登录依赖邮件，SMTP 不能省；附件是浏览器直传，S3 地址必须从浏览器视角配置。把这三处验证清楚，剩下的就是常规的备份和升级。

---

## 验证记录

- 验证日期：2026-09-02
- 实际镜像：`outlinewiki/outline:1.12.0`、`postgres:16-alpine`（16.15）、`redis:7-alpine`（7.4.12）、`bitnamilegacy/minio:2025.4.22`、`axllent/mailpit:v1.33.0`
- Compose 路径：本地验证目录下的 `compose.yaml`
- 已跑通：五个容器启动并 healthy；312 条迁移记录（`SequelizeMeta`）；首次 setup 创建管理员（角色 `admin`）；Mailpit 收到 `Welcome to Outline!`；编辑器上传图片时 `POST http://localhost:19011/outline` 返回 204，桶内出现对象；刷新后 `/api/attachments.redirect` 302 到 MinIO 预签名 GET 并返回 200。
- 未实测/降级：OIDC 接入未在本环境实测；HTTPS 反向代理未实测，文中协议与回调相关建议按环境变量语义给出；MinIO 官方镜像当前无法直接拉取，改用 Bitnami 归档镜像。
- 截图：`outline-docker-deploy-1.png` 至 `-4.png` 为自动截图；终端输出已以文本形式附在正文。
