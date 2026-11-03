---
title: "用 Docker Compose 部署 Vaultwarden 密码库"
published: 2026-11-03T22:20:00+08:00
description: "Vaultwarden 是 Bitwarden 服务端的轻量实现。本文用 Docker Compose 搭一套带 PostgreSQL、Nginx 反向代理和邮件邀请的实例，并处理备份、升级与常见故障。"
tags: [Vaultwarden, Docker, 密码管理, 自托管]
category: "自托管"
image: ""
draft: false
comment: true
---

密码库是所有账号的最后一道门。放在别人的服务器上，服务商封号、改条款或出故障时，受影响的不只是数据，还有恢复账号的入口。自建的差别在于控制权：密文、备份、服务存续都归自己。

Vaultwarden 是 Bitwarden 服务端的 Rust 实现，官方浏览器扩展、桌面端、iOS、Android 和 CLI 都能直接连。它把官方那套多服务架构压成单个进程，资源占用低，适合一台小机器或 homelab。代价也很明确：它是非官方实现，安全审计和合规背书不如官方服务端；密码库又是零知识架构，主密码丢失后没有任何找回路径。

这篇用 Docker Compose 部署一套可长期运行的实例：PostgreSQL 存结构化数据，Nginx 负责 HTTPS，Vaultwarden 只监听内网；注册关闭，账号只能从管理面板邀请。文中的截图来自本地实例，域名和邮箱都换成了示例值。

::github{repo="dani-garcia/vaultwarden"}

## 这套实例由什么组成

生产环境是三个常驻容器，本地测试再加一个 Mailpit：

```mermaid
flowchart LR
    C["浏览器 / 扩展 / 手机"] -->|HTTPS| P["Nginx :443"]
    P -->|HTTP| V["Vaultwarden :80"]
    V --> DB[("PostgreSQL")]
    V --> D[("/data：附件、Send、rsa_key")]
    V -.->|SMTP| M["邮件服务"]
```

- **vaultwarden**：应用本体。账号、密文条目、组织关系在数据库里；附件、Send 上传件、图标缓存和 `rsa_key` 在 `/data`。
- **postgres**：存用户、密文条目、组织、附件元数据。这里用 PostgreSQL 17。
- **nginx**：终止 TLS，反代到 `vaultwarden:80`，转发 WebSocket Upgrade。
- **mailpit**：只在本地测试时启用，接住邀请邮件和验证邮件。生产环境换成真实 SMTP。

有两份不可再生的状态：数据库和 `/data`。数据库丢了，账号和条目全没；`/data` 丢了，附件和 Send 取不回来，`rsa_key.pem` 也会重新生成——密文条目还在，但所有客户端会被强制登出。备份必须同时覆盖这两处。

Vaultwarden 本身不做多副本。多实例要共享同一份 `rsa_key`，WebSocket 还要粘性会话；单机场景没必要引入这层复杂度。下面按单实例写。

## Compose 编排

目录里放 `compose.yaml`、`.env`、`nginx.conf`，再加一个存证书的 `certs/`。先看完整的 compose：

```yaml
name: vaultwarden

services:
  postgres:
    image: postgres:17-alpine
    container_name: vaultwarden-postgres
    environment:
      POSTGRES_DB: vaultwarden
      POSTGRES_USER: vaultwarden
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U vaultwarden -d vaultwarden"]
      interval: 5s
      timeout: 3s
      retries: 20
    restart: unless-stopped
    networks: [internal]

  vaultwarden:
    image: vaultwarden/server:1.37.4
    container_name: vaultwarden
    environment:
      DOMAIN: ${DOMAIN}
      DATABASE_URL: postgresql://vaultwarden:${DB_PASSWORD}@postgres:5432/vaultwarden
      SIGNUPS_ALLOWED: "false"
      INVITATIONS_ALLOWED: "true"
      INVITATION_ORG_NAME: ${INVITATION_ORG_NAME}
      SHOW_PASSWORD_HINT: "false"
      ADMIN_TOKEN: ${ADMIN_TOKEN}
      ICON_SERVICE: internal
      ENABLE_WEBSOCKET: "true"
      PUSH_ENABLED: "false"
      SMTP_HOST: ${SMTP_HOST}
      SMTP_PORT: ${SMTP_PORT}
      SMTP_SECURITY: ${SMTP_SECURITY}
      SMTP_FROM: ${SMTP_FROM}
      SMTP_FROM_NAME: ${SMTP_FROM_NAME}
      SMTP_USERNAME: ${SMTP_USERNAME}
      SMTP_PASSWORD: ${SMTP_PASSWORD}
    volumes:
      - vwdata:/data
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost/alive"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped
    networks: [internal]

  nginx:
    image: nginx:1.29.8-alpine
    container_name: vaultwarden-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
      - ./certbot/www:/var/www/certbot:ro
    depends_on:
      vaultwarden:
        condition: service_healthy
    restart: unless-stopped
    networks: [internal]

  mailpit:
    image: axllent/mailpit:v1.33.0
    container_name: vaultwarden-mailpit
    profiles: ["local"]
    environment:
      # 本地测试用：让 Mailpit 宣告 AUTH 并接受任意凭据
      MP_SMTP_AUTH_ACCEPT_ANY: 1
      MP_SMTP_AUTH_ALLOW_INSECURE: 1
    ports:
      - "127.0.0.1:8025:8025"
    restart: unless-stopped
    networks: [internal]

volumes:
  pgdata:
  vwdata:

networks:
  internal:
    driver: bridge
```

几个配置上的取舍：

- **Vaultwarden 不映射端口**。它只在内网被 Nginx 访问，应用端口和数据库端口都没有理由直接出现在公网。
- **PostgreSQL 也不映射端口**。数据库只在 compose 网络里可达。
- **Mailpit 只绑定 `127.0.0.1`**，并且放在 `local` profile 里，默认不启动。
- **镜像全部固定版本**。Vaultwarden 上游大约每月发一个次版本，`:latest` 会让升级不可审计；它的 `:alpine` 变体更新滞后，也不要用。

`.env` 至少包含这些值：

```dotenv
DOMAIN=https://vault.example.com
DB_PASSWORD=REPLACE_WITH_HEX
ADMIN_TOKEN='REPLACE_WITH_ARGON2_PHC'

INVITATION_ORG_NAME=Home Vault

SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURITY=starttls
SMTP_FROM=noreply@example.com
SMTP_FROM_NAME=Home Vault
SMTP_USERNAME=replace-me
SMTP_PASSWORD=replace-me
```

数据库口令用 hex，不用 base64：

```bash
openssl rand -hex 24
```

原因是 `DB_PASSWORD` 会被拼进 `DATABASE_URL` 的 userinfo 段，而 Vaultwarden 只认这一条完整连接串，没有分项的 `DB_HOST` / `DB_USER` / `DB_PASSWORD`。口令里只要出现 `@`、`:`、`/`、`?`、`#`、`&`，连接串就会截断或解析到错误的主机名，报错通常是 `invalid connection string`。hex 字符集不含 URL 保留字符，从生成口令这一步就把问题绕开。

管理令牌用 Argon2，不用明文：

```bash
docker run --rm -it vaultwarden/server:1.37.4 /vaultwarden hash
```

命令会要求输入两次口令，输出形如 `$argon2id$v=19$m=65540,t=3,p=4$...` 的 PHC 字符串。把它写进 `.env` 时**用单引号包住**，Compose 才不会把 `$argon2id`、`$v` 当成变量插值：

```dotenv
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...'
```

容易混淆的是：`.env` 里存的是 hash，`/admin` 登录时填的是生成 hash 之前的**原始口令**。hash 只用于服务端校验，不能拿来登录。原始口令要单独存进密码管理器；丢了它不会丢数据，但只能重新生成一组 `ADMIN_TOKEN` 覆盖，原来那组口令同时作废。

## Nginx 与 HTTPS

生产环境的 `nginx.conf` 挂在容器的 `/etc/nginx/conf.d/default.conf`：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name vault.example.com;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    http2 on;
    server_name vault.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # 附件从这里上传，nginx 默认只放行 1 MB
    client_max_body_size 525M;

    location / {
        proxy_pass http://vaultwarden:80;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        # WebSocket 空闲超过 60 秒会被 nginx 断开
        proxy_read_timeout 3600s;
    }
}
```

几行配置对应的实际故障：

- `X-Forwarded-Proto $scheme` 把浏览器侧的协议告诉 Vaultwarden。TLS 在 nginx 终结，上游只看到 HTTP，这个头是它判断外部协议的唯一样本；漏掉时容易出现协议判断错误和混合内容。
- `Host $host` 保留浏览器请求的域名。改写成容器名或内网地址后，邀请链接和附件下载会指向客户端不可达的主机。
- `map` 让普通请求发 `Connection: close`，只有带 `Upgrade` 的请求发 `Connection: upgrade`。把 `Connection` 写死成 `upgrade` 也能跑，但普通请求的 keep-alive 会被一起关掉。
- `client_max_body_size` 默认是 1 MB，Bitwarden 客户端允许的附件远大于这个值。不调的话大附件上传直接 413，日志里只有一条 `client intended to send too large body`。
- `proxy_read_timeout` 默认 60 秒。WebSocket 空闲一分钟就被断开，客户端表现为反复重连、通知时有时无。
- `http2 on` 需要 nginx 1.25.1 及以上；`listen 443 ssl http2` 是旧写法，新版本会提示指令已废弃。

证书用 certbot 在宿主机签发，走 webroot 模式，把 challenge 文件写到 nginx 也能读到的目录：

```bash
mkdir -p certbot/www
sudo certbot certonly --webroot -w "$PWD/certbot/www" -d vault.example.com
```

compose 里已经把 `./certbot/www` 挂到容器的 `/var/www/certbot`，80 端口那段配置就是给 ACME challenge 用的，首次签发和续期都不需要停 nginx。签发后把 compose 里 nginx 的 `./certs` 这一行换成 Let's Encrypt 的 live 目录：

```yaml
    volumes:
      - /etc/letsencrypt/live/vault.example.com:/etc/nginx/certs:ro
```

`fullchain.pem` 包含站点证书和中间 CA，`privkey.pem` 是私钥。只挂 `cert.pem` 会让 iOS 和部分客户端报证书链不完整。

续期由 certbot 的定时器负责，但 nginx 读的是容器启动时那份文件，续期后要让它重新加载。加一个 deploy hook：

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh >/dev/null <<'EOF'
#!/bin/sh
cd /opt/vaultwarden
docker compose exec -T nginx nginx -s reload
EOF
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

`cd` 的路径换成 compose 文件所在目录。这是 nginx 和 Caddy 最大的差别：Caddy 自己申请、续期、热加载证书；nginx 只负责用证书，签发归 certbot，续期后还得有人通知它重新加载。

Vaultwarden 的 Web Vault 对协议有硬性要求。当前版本（Web Vault 2026.7.0）在 `http://` 下会直接拒绝 API 请求，控制台报：

```text
Insecure URL not allowed. All URLs must use HTTPS.
```

所以 `DOMAIN` 必须是 `https://` 开头，本地测试也不能用 `http://localhost` 代替。本地验证时用自签证书：

```bash
mkdir -p certs
openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout certs/privkey.pem -out certs/fullchain.pem \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

`nginx.conf` 与生产相同，只改三处：`server_name` 换成 `localhost`、删掉整个 80 端口的 `server` 块、compose 的端口映射改成 `8443:443`。`.env` 的 `DOMAIN` 改成 `https://localhost:8443`，邮件指向 Mailpit：

```dotenv
SMTP_HOST=mailpit
SMTP_PORT=1025
SMTP_SECURITY=off
SMTP_USERNAME=demo
SMTP_PASSWORD=demo
```

Mailpit 默认不宣告 AUTH。如果 Vaultwarden 配了 `SMTP_USERNAME`，它会尝试认证，然后在发邀请时报 `No compatible authentication mechanism was found`。compose 里给 Mailpit 的 `MP_SMTP_AUTH_ACCEPT_ANY` 和 `MP_SMTP_AUTH_ALLOW_INSECURE` 就是为了让它接受任意凭据；本地账号密码随便填，不会真的校验。

然后带上 `local` profile 启动，Mailpit 才会被拉起：

```bash
docker compose --profile local up -d
```

自签证书不在任何客户端的信任库里，截图和自动化测试需要忽略证书错误，或者把 `certs/fullchain.pem` 装进客户端信任库。这套配置只用于本地验证。

换成 Traefik 或其他反代时，同样要满足三件事：转发 `Host`、带上 `X-Forwarded-Proto`、允许 WebSocket Upgrade。`DOMAIN` 仍然写外部 HTTPS 地址，不要写容器名或内网 IP。证书文件要给完整链（站点证书 + 中间 CA），只给叶证书时，iOS 和部分客户端会报链不完整。

## 启动与首个账号

```bash
docker compose up -d
docker compose ps
docker compose logs -f vaultwarden
```

日志里看到 `Rocket has launched from http://0.0.0.0:80`，说明应用起来了；数据库迁移在启动时自动执行，不需要手工建表。首次访问 `https://vault.example.com` 会看到登录页：

![](./images/vaultwarden-docker-compose-1.png)

注册是关闭的，第一个账号要从管理面板邀请。打开 `https://vault.example.com/admin`，填原始管理口令，进入配置页：

![](./images/vaultwarden-docker-compose-2.png)

管理面板的登录方式是 `POST /admin` 表单加会话 cookie（`VW_ADMIN` JWT），不是每个请求都走 HTTP Basic。用 curl 验收时不能写 `-u admin:<token>`，要先拿 cookie：

```bash
curl -c ck -d "token=$ADMIN_TOKEN" https://vault.example.com/admin
curl -b ck https://vault.example.com/admin/users
```

这里的 `$ADMIN_TOKEN` 是原始口令，不是 `.env` 里的 Argon2 hash。

在 `Users` 页填邮箱发邀请，邮件走 `SMTP_*` 配置的服务。本地用 Mailpit 时，打开 `http://localhost:8025` 就能看到信；生产环境则直接进收件箱。邀请发出后，用户列表里会出现一条 `Invited` 记录：

![](./images/vaultwarden-docker-compose-3.png)

收件人点邮件里的链接，设置主密码，账号就建好了。登录后进入密码库：

![](./images/vaultwarden-docker-compose-4.png)

第一次登录后，可以先做三件事：

1. **设置 → 安全 → 两步登录**，绑定 TOTP 认证器，并把恢复码抄到离线介质；
2. 在密码库里新建一条「Vaultwarden 主密码」记录，把主密码和恢复码存进去——这是这个服务唯一的自举依赖；
3. 如果需要多人共用，再考虑组织或紧急访问；个人使用可以跳过。

主密码不可找回。Vaultwarden 是零知识架构，服务端只有密文；`/admin` 面板的 `Remove 2FA` 只能解决双因素丢失，解决不了主密码遗忘。这是零知识架构的结果，不是部署缺陷。

## 客户端接入

所有客户端都把「自托管服务器 URL」填成 `https://vault.example.com`，不要带路径，不要带端口。

- **浏览器扩展**：登录页左下角齿轮 → 自托管环境 → 服务器 URL → 保存 → 登录。
- **iOS / Android**：登录页齿轮 → 自托管环境 → 填同一个地址 → 登录后下拉同步。
- **桌面端 / CLI**：同样在设置里改服务器地址。

`PUSH_ENABLED=false` 表示不接 Bitwarden 官方推送中继。改密码不会实时推到手机，打开 App 或手动下拉时才同步；已缓存的条目离线仍然可读，自动填充不依赖网络。想要实时推送，就得接受官方中继，这和自建的隐私目标冲突，按需取舍。

WebSocket 走同一域名的 `/notifications/hub`。上面的 nginx 配置已经带了 `Upgrade` 和 `Connection` 两个头；漏掉时客户端仍能登录和同步，但新条目不会实时出现，只能手动下拉。

## 可选：数据泄露检测

Web Vault 的「报告」页有一项「数据泄露报告」，输入邮箱后查这个地址出现在哪些已知泄露事件里。Vaultwarden 把查询转给 Have I Been Pwned 的 `breachedaccount` 接口，这个接口需要付费 API key。

开启要改两处配置。`.env` 里加：

```dotenv
HIBP_API_KEY=你的key
```

compose 的 vaultwarden 服务加一行：

```yaml
HIBP_API_KEY: ${HIBP_API_KEY}
```

重启应用即可：

```bash
docker compose up -d vaultwarden
```

没配 key 时，报告页不会报错，而是返回一条「Manual HIBP Check」提示，让你去 haveibeenpwned.com 手动查，并附上购买 key 的链接。功能不可用，密码库的其他部分不受影响。

几点实际影响：

- key 按 HIBP 的订阅档位限速，多人同时查会撞上 429；Vaultwarden 只把失败转成一条错误，页面上不一定看得到 429 原文。
- 查询时邮箱会以明文出现在请求 URL 里。介意这一点的话，不配这个功能、用官网手动查即可。
- key 存在服务端配置里，由 Vaultwarden 代所有用户查询。它和密码库数据无关，泄露的后果主要是配额被别人消耗。单独申请一个，不用和其他服务抢额度。
- 「已泄露的密码」检查用的是 Pwned Passwords 的 k-anonymity 接口，免费，和这个 key 无关。

费用和档位见 HIBP 的 API 页面，个人用量买最低档就够。

## 备份与升级

备份分两块，缺一不可。

数据库用自定义格式导出，恢复时可以并行、可以选择性还原：

```bash
mkdir -p backup
docker compose exec -T postgres pg_dump -U vaultwarden -Fc vaultwarden \
  > backup/vaultwarden-$(date +%F).dump
```

`/data` 建议停掉应用再打包，避免附件写到一半：

```bash
docker compose stop vaultwarden
docker run --rm \
  -v vaultwarden_vwdata:/data \
  -v "$PWD/backup":/backup \
  alpine tar czf /backup/vwdata-$(date +%F).tgz -C /data .
docker compose start vaultwarden
```

卷名前缀是 compose 项目名。项目名固定为 `vaultwarden` 时，卷就是 `vaultwarden_vwdata`；改过项目名就用 `docker volume ls` 确认。

恢复数据库时先停应用，避免恢复过程中有写入：

```bash
docker compose stop vaultwarden
cat backup/vaultwarden-2026-11-03.dump \
  | docker compose exec -T postgres pg_restore -U vaultwarden -d vaultwarden --clean --if-exists
docker compose start vaultwarden
```

升级前**同时**做数据库和 `/data` 的备份：

```bash
docker compose pull vaultwarden
docker compose up -d vaultwarden
docker compose logs -f vaultwarden
```

Vaultwarden 发版会跑数据库迁移。镜像回滚不代表 schema 回滚，备份只能放在拉新镜像之前；升级失败后再补，表结构已经变了。`/data` 里的附件和 `rsa_key` 也不会跟着数据库迁移回滚。

## 常见故障

### Web Vault 打不开，控制台报 Insecure URL not allowed

`DOMAIN` 还是 `http://`，或者前面没有 TLS。Vaultwarden Web 当前版本要求 API 走 HTTPS，本地也不例外。本地用自签证书，生产用真实证书；改成 HTTP 不会让它工作。

### 容器起来就退出，日志报 invalid connection string

`DB_PASSWORD` 含 URL 保留字符。重新生成 hex 口令，改 PostgreSQL 角色口令，再更新 `.env`：

```bash
docker compose exec -T postgres psql -U vaultwarden -d postgres \
  -c "ALTER ROLE vaultwarden PASSWORD '新的hex口令';"
docker compose up -d vaultwarden
```

### `/admin` 返回 404

`ADMIN_TOKEN` 没注入或为空。先确认容器里非空：

```bash
docker compose exec -T vaultwarden printenv ADMIN_TOKEN
```

如果值存在但还是 404，检查 `.env` 里的单引号是否把整段 Argon2 包住了；`$` 被 Compose 提前展开时，值会变成半截。

### 邀请邮件里的链接指向旧域名

改过 `DOMAIN` 之后，已经发出但尚未接受的邀请令牌仍然带着旧 issuer，重发也不会改。把待邀请用户删掉，用新域名重新邀请一次。

### 邀请邮件收不到

按顺序查：`SMTP_HOST` / `SMTP_PORT` 是否可达，`SMTP_SECURITY` 是否和端口匹配（587 用 `starttls`，465 用 `force_tls`），发件地址是否已在邮件服务商侧验证，日志里有没有 `SMTP` 报错。邮件服务商的沙箱模式通常只能发给已验证地址。

如果只是想把第一个账号建起来，可以临时把 `SIGNUPS_ALLOWED` 改成 `"true"`，创建完账号再改回 `"false"` 并 `docker compose up -d vaultwarden`。这只适合初始化，用完就改回来。

### 发邀请返回 500，日志报 No compatible authentication mechanism

SMTP 服务没有宣告 AUTH，但 Vaultwarden 配了 `SMTP_USERNAME`，于是它在认证阶段找不到可用机制。两条路：SMTP 不需要认证时把 `SMTP_USERNAME` / `SMTP_PASSWORD` 从环境变量里去掉；本地 Mailpit 则按上文本地配置打开 `MP_SMTP_AUTH_ACCEPT_ANY`，并填一组随便的凭据。

### 注册接口返回 404，是没关掉注册吗

不是。1.37 起注册流程迁到了 `POST /identity/accounts/register/send-verification-email` 和 `/identity/accounts/register/finish`，旧的 `/api/accounts/register` 恒为 404。判断注册是否关闭，看新接口是否返回 `400 Registration not allowed`：

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST \
  -H 'Content-Type: application/json' \
  -d '{"email":"probe@example.com"}' \
  https://vault.example.com/identity/accounts/register/send-verification-email
```

期望 `400`。

### 附件没有落到 S3

Vaultwarden 的 S3 后端是编译期 feature。官方镜像的构建命令只编了 `sqlite,mysql,postgresql`，用 `s3://` 路径会直接报 `S3 support is not enabled`。要走 S3，必须自己构建带 `s3` feature 的镜像，并自行验证 S3 兼容存储的 endpoint 和 path-style。当前这套配置用本地卷，零额外依赖，代价是附件必须自己备份。

### `/admin` 诊断页报 Internet access Error

诊断页会去请求 `github.com` 和 `cloudflare.com`。如果容器所在网络把这些域名解析到保留地址或做了透明代理，Vaultwarden 的自定义 DNS 校验会拒绝请求，进而报错。这不影响 Web Vault、客户端同步、SMTP 和图标服务；要消掉这两行，得调整出口 DNS 策略，通常不值得为诊断页外观改全局配置。

## 配置参考

| 键 | 值 | 说明 |
|---|---|---|
| `DOMAIN` | `https://vault.example.com` | 生成邀请、附件、邮件链接的基址；必须是外部 HTTPS 地址 |
| `DATABASE_URL` | `postgresql://vaultwarden:${DB_PASSWORD}@postgres:5432/vaultwarden` | 单条连接串，口令限 hex |
| `SIGNUPS_ALLOWED` | `false` | 关闭自助注册 |
| `INVITATIONS_ALLOWED` | `true` | 账号只能从 `/admin` 邀请 |
| `ADMIN_TOKEN` | Argon2 PHC 字符串 | 不设置时 `/admin` 返回 404；登录用原始口令 |
| `ICON_SERVICE` | `internal` | 服务端自抓图标；改成第三方会把站点域名外泄 |
| `ENABLE_WEBSOCKET` | `true` | 走同域名的 `/notifications/hub` |
| `PUSH_ENABLED` | `false` | 不接官方推送中继，移动端手动同步 |
| `HIBP_API_KEY` | 空 | 可选；启用 Web Vault 的「数据泄露报告」，需要 HIBP 付费 key |
| `SMTP_SECURITY` | `starttls` | 587 端口；465 直连用 `force_tls` |

镜像版本：`vaultwarden/server:1.37.4`、`postgres:17-alpine`、`nginx:1.29.8-alpine`、`axllent/mailpit:v1.33.0`。Vaultwarden 上游约每月一个次版本，升级前先看 release notes 里的数据库迁移说明。

## 最后

这套实例的边界很清楚：单机、单副本、本地附件卷，适合个人或小团队。它不解决高可用，也不替代托管密码库的合规能力。实际要维持的只有三件事：`/data` 和数据库一起备份、主密码和 TOTP 恢复码离线保存、`ADMIN_TOKEN` 的原始口令单独留存。

公网部署还可以补两件事：在反向代理或 WAF 层给 `/identity/connect/token` 加限速，避免登录接口被无限次尝试；把容器日志接入集中采集，出问题时能查登录来源。nginx 用 `limit_req` 就能做这层限速，不需要额外组件。

升级、迁移和故障处理都建立在这三件事之上。

## 参考

- [Vaultwarden 项目主页](https://github.com/dani-garcia/vaultwarden)
- [配置项全表（.env.template）](https://github.com/dani-garcia/vaultwarden/blob/main/.env.template)
- [Vaultwarden Wiki](https://github.com/dani-garcia/vaultwarden/wiki)
- [反向代理与 WebSocket 说明](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples)
- [官方镜像的 S3 支持讨论](https://github.com/dani-garcia/vaultwarden/discussions/7900)
