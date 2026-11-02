---
title: "Kite：用 Docker 部署一个轻量的 Kubernetes 管理面板"
published: 2026-10-22T03:58:18+08:00
description: "Kite 是一个单二进制、界面现代的 Kubernetes 管理面板。本文用 Docker Compose 部署 Kite 与它的 PostgreSQL，接入一个已有集群，并整理 OIDC 登录、监控接入和插件这些实际会用到的配置。"
tags: [Kite, Kubernetes, Docker, 运维]
category: "自托管"
image: ""
draft: false
comment: true
---

homelab 里只跑一两个集群时，`kubectl` 仍然是最快的工具。但有几类操作在终端里始终别扭：翻某个 Namespace 下的工作负载、按关键字过滤容器日志、临时进容器执行一条命令、看一眼节点还剩多少资源。每次都要回忆 `-n`、label selector 和 `kubectl logs -f` 的参数，时间久了对集群的直观感受也会变淡。

Kite 解决的就是这部分需求。它是一个用 Go 写的单进程 Web 服务，自己跑在集群外面，靠一份 kubeconfig（或集群内的 ServiceAccount）连接已有的 Kubernetes 集群，把工作负载、日志、事件、终端和资源关系放进浏览器。和 Rancher 这类平台相比，它不需要往被管集群里装一整套控制器；和 Lens、Headlamp 这类桌面客户端相比，它开一个浏览器就能用，适合放在内网当常驻面板。

::github{repo="kite-org/kite"}

## Kite 是什么、不是什么

Kite 的定位是「多集群操作台」。它把每个接入的集群当成一个工作区，提供资源列表、YAML 编辑、Pod 日志、节点与 Pod 终端、RBAC、OAuth/LDAP 登录、Prometheus 曲线和插件扩展。数据面很薄：一个二进制，加上一个 PostgreSQL（也支持 SQLite）存用户、权限、集群清单、审计和插件安装记录。

它不是一个声明式部署工具。Kite 不接管你的应用发布，不维护期望状态，也不替代 `kubectl`。面板挂掉只影响你自己看不到界面，不会影响集群里任何工作负载；同样，你在 Kite 里做的修改就是直接调用 Kubernetes API，不存在「等面板同步」这一步。

这决定了它的部署方式可以很随意：一台能访问集群 API Server 的机器，一个 Docker Compose 文件，两个容器就够了。本文只讲这条路径。

## 准备：一个集群与一份 kubeconfig

Kite 需要一个能访问的 Kubernetes 集群，以及一份有足够权限的 kubeconfig。集群可以是 k3s、kubeadm，也可以是本机用 kind 起的测试集群。kind 把节点跑在 Docker 里，适合验证接入流程：

```bash
kind create cluster --name kite-demo
kubectl config get-contexts
```

创建完成后，kubeconfig 里的 `server` 通常长这样：

```text
server: https://127.0.0.1:54955
```

这是整篇文章最容易踩的坑。`127.0.0.1` 是相对「谁在发起连接」而言的：在宿主机上执行 `kubectl` 时它指向宿主机，而 Kite 跑在容器里，容器内的 `127.0.0.1` 指向容器自己。把这份 kubeconfig 原样贴进 Kite，表现就是连接超时或 `connection refused`，但你在宿主机上 `kubectl get nodes` 一切正常。

对 kind 来说，控制平面容器的名字是固定的。先把 kubeconfig 的 `server` 改成控制平面容器名，并顺手导出 CA，后面验 TLS 用：

```bash
kubectl config view --raw --minify --context kind-kite-demo > kubeconfig.yaml
sed -i 's#https://127.0.0.1:[0-9]*#https://kite-demo-control-plane:6443#' kubeconfig.yaml

grep certificate-authority-data kubeconfig.yaml \
  | sed 's/.*certificate-authority-data: //' | tr -d ' ' | base64 -d > ca.crt
```

Kite 容器起来后还要把它接到 kind 的 Docker 网络，这一步放在下一节部署完成后执行。对真实集群也是同样的道理：`server` 必须是 Kite 容器能解析、能路由到，并且被 API Server 证书 SAN 覆盖的地址。用内网域名或固定 IP 都行，唯独不能用只对宿主机有意义的 `localhost`。

## 用 Compose 部署

Kite 的官方镜像发布在 `ghcr.io/kite-org/kite`，发布流水线同时把同一份构建推到 Docker Hub 的 `zzde/kite`。写作时从当前网络匿名拉 ghcr 返回了 `denied`，改用 Docker Hub 的 `zzde/kite:v0.16.0` 完成实测；你可以在自己环境里先试 ghcr，拉不动再换 Docker Hub，tag 对齐即可。不要用 `latest`，Kite 还在 0.x，升级镜像 tag 往往伴随数据库迁移。

工作目录里放一个 `compose.yaml`：

```yaml
name: kite-demo

services:
  postgres:
    image: postgres:17-alpine
    container_name: kite-demo-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: kite
      POSTGRES_USER: kite
      POSTGRES_PASSWORD: change-me-please
    ports:
      - "15434:5432"
    volumes:
      - kite-pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kite -d kite"]
      interval: 5s
      timeout: 3s
      retries: 20

  kite:
    image: zzde/kite:v0.16.0
    container_name: kite-demo-kite
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "19000:8080"
    environment:
      DB_TYPE: postgres
      DB_DSN: "postgres://kite:change-me-please@postgres:5432/kite?sslmode=disable"
      JWT_SECRET: "change-me-jwt-secret"
      KITE_ENCRYPT_KEY: "change-me-encrypt-key"
      HOST: "http://localhost:19000"
      PLUGIN_DIR: /data/plugins
    volumes:
      - kite-data:/data

volumes:
  kite-pgdata:
  kite-data:
```

几个参数值得解释：

`DB_TYPE` 必须显式写 `postgres`。Kite 不写时默认用 SQLite，数据会落在容器内的 SQLite 文件里，表象是「PostgreSQL 起来了但库一直是空的」。`DB_DSN` 用的是标准的 PostgreSQL URL 形式，同一 Docker 网络内可以 `sslmode=disable`；如果数据库在另一台机器上，应按实际情况改成 `require` 或 `verify-full`。

`JWT_SECRET` 和 `KITE_ENCRYPT_KEY` 都要换成自己的随机值。前者用于签发登录态，后者用于加密落库的敏感字段（例如 kubeconfig、OAuth client secret）。`KITE_ENCRYPT_KEY` 一旦有数据就不要再改，改了以后旧密文读不出来，只能清库重来。

`HOST` 决定 Kite 生成 OAuth 回调地址时用的外部地址。只在本地试用时写 `http://localhost:19000`；挂到反向代理后面时，要写成用户浏览器实际访问的 `https://kite.example.com`，否则 OIDC 回调会对不上。

`PLUGIN_DIR` 和 `/data` 卷放在一起，插件文件才会在容器重建后保留。PostgreSQL 的 `15434` 端口映射只是为了从宿主机排查，正式使用时可以删掉，让数据库只暴露在 Compose 网络里。

Kite 镜像里没有 shell、`curl`、`ls`，所以没法写容器级 healthcheck。判活从宿主机做：

```bash
docker compose up -d
docker compose ps
curl -fsS http://localhost:19000/healthz
```

预期是 PostgreSQL 显示 `healthy`，Kite 显示 `Up`，`/healthz` 返回 200。首次打开 `http://localhost:19000` 会被重定向到 `/setup`。

```text
$ docker compose ps
NAME                 IMAGE                COMMAND                  SERVICE   CREATED             STATUS                   PORTS
kite-demo-kite       zzde/kite:v0.16.0    "./kite"                 kite      About an hour ago   Up 4 minutes             0.0.0.0:19000->8080/tcp, [::]:19000->8080/tcp
kite-demo-postgres   postgres:17-alpine   "docker-entrypoint.s…"   postgres  About an hour ago   Up 4 minutes (healthy)   0.0.0.0:15434->5432/tcp, [::]:15434->5432/tcp
```

如果用 kind 验证，现在把 Kite 容器接到 kind 网络，并确认容器内的地址、DNS 和证书 SAN 能对上：

```bash
docker network connect kind kite-demo-kite

docker run --rm --network kind \
  -v "$PWD/ca.crt:/ca.crt:ro" curlimages/curl:8.20.0 \
  --cacert=/ca.crt -s https://kite-demo-control-plane:6443/version
```

返回 `gitVersion` 就说明 TLS 链路没问题。真实集群不需要这一步，只要 `server` 从容器内可达且证书匹配即可。

## 首次登录与安全设置

第一次访问是两步式初始化：先创建本地管理员，再接入第一个集群。这个管理员不是从环境变量读的，而是在页面上现场创建；`KITE_USERNAME`、`KITE_PASSWORD` 在当前版本里只用于旧数据迁移，不要指望用它们预置账号。

![Kite 首次启动的初始化页面](./images/kite-docker-deploy-5.png)

填好用户名和密码后进入第二步。创建完成后，后续再访问会看到正常的登录页：

![Kite 登录页](./images/kite-docker-deploy-1.png)

登录后第一件事是在右上角用户菜单的 **Account Settings** 里改掉初始化密码。然后确认两件事：

- `ANONYMOUS_USER_ENABLED` 保持默认的 `false`。这个开关打开后，所有访问都不需要登录且默认拥有最高权限，只适合完全隔离的演示环境。
- 不要把 `19000` 直接暴露到公网。前面应该有一层 TLS 反向代理；同时把 `HOST` 改成外部域名，并确认代理转发了 WebSocket 升级头，否则日志和终端会一直转圈。

初始化页面里的 analytics 开关可以关掉，不影响功能。

## 接入集群

如果刚走完初始化向导，第二步就是粘贴 kubeconfig；也可以之后从 **Settings → Cluster → Add Cluster** 补加。把上一节改好 `server` 的 kubeconfig 整个粘进去，Kite 会自动识别出集群名。本次实测里，它识别成了 `kind-kite-demo`。

集群名不是纯粹显示用的。Kite 的集群级接口路径是 `/api/v1/_clusters/<name>/…`，日志里能看到实际请求；名字频繁改，书签和脚本都会跟着失效。起一个稳定、简短的名字。

接入类型会显示为 `External`，因为是从集群外部用 kubeconfig 连进去的。只有 Kite 自己跑在被管集群内部时，`In-Cluster` 才有意义，这不是本文的 Compose 场景。

权限上的限制要单独说清楚：Kite 对集群的访问能力，最终取决于这份 kubeconfig 对应的用户或 ServiceAccount；面板内的 RBAC 只能在这个上限内再收窄，不能凭空放大。测试时用的是 kind 的管理员 kubeconfig，所以面板里什么都能改；如果只想看图不改资源，应该单独生成一份只读凭据再接入，而不是在面板里靠「小心操作」来兜底。

接入成功后，概览页会显示节点、Pod、Namespace、Service 的数量和最近事件：

![Kite 集群概览](./images/kite-docker-deploy-2.png)

工作负载列表可以直接按 Namespace 过滤。默认只显示 `default`，切到 `All Namespaces` 才能看到系统组件：

![Kite 的 Deployments 列表](./images/kite-docker-deploy-3.png)

节点页会列出角色、Pod 数量、CPU/内存请求和版本。kind 单节点集群只有一行，真实集群会按节点展开：

![Kite 的节点列表](./images/kite-docker-deploy-4.png)

到这一步，「接入集群 → 看到节点和 Pod → 打开工作负载」这条主链路就闭环了。

## 接入监控

Kite 的监控是「按集群配置一个 Prometheus 地址」，不是内置时序库。入口在 **Settings → Cluster → Actions → Edit**，弹窗里有一个 `Prometheus URL` 字段：

![编辑集群时的 Prometheus URL 字段](./images/kite-docker-deploy-7.png)

地址必须从 Kite 容器内部可达。常见错误是填 `http://localhost:9090`：在宿主机上浏览器能打开，但 Kite 容器里的 `localhost` 是它自己。Prometheus 和 Kite 在同一个 Compose 项目里时，用服务名即可，例如 `http://prometheus:9090`；在另一台机器上就用那台机器的内网地址。

本次环境里没有 Prometheus，只验证了字段保存：保存后集群列表的 Prometheus 列变成 `Yes`，随后概览页立刻出现查询失败日志：

```text
Failed to get resource usage history: ... Post "http://prometheus:9090/api/v1/query_range":
dial tcp: lookup prometheus ... no such host
```

这条报错同时说明两件事：配置确实生效了，以及地址不可达时面板的其余功能不受影响，只是曲线缺失。文章不把「曲线可见」算作已验证项。

另外，概览页的 CPU/内存「用量」还需要集群里有 metrics-server。没有它时，Kite 日志会出现 `no matches for kind "NodeMetrics"`，页面上的用量保持 0，只显示 requests/limits。Prometheus 负责历史曲线，metrics-server 负责即时用量，两者不是一回事。

还有一个字段限制：Kite 的集群模型只有 `prometheusURL`，没有自定义请求头。如果你的 Prometheus 兼容服务是多租户的、查询必须带 `X-Scope-OrgID` 之类的头，填进去也不会生效，只能换一个不需要额外头的查询端点。

## 插件

插件入口在右上角用户菜单的 **Plugin management**，也可以直接访问 `/plugins`。官方目录从 `plugins.kitehq.dev` 拉取，页面会列出可安装项和版本。本次实测安装了 `cert-manager` 插件 `0.1.0`：安装记录写入数据库，侧栏随即出现 `CERT-MANAGER` 分组，下面有 Certificates、Issuers、Orders 等入口。

![Kite 的插件管理页面](./images/kite-docker-deploy-6.png)

插件文件落在容器的 `/data/plugins`，也就是 `kite-data` 卷里。卷还在，容器重建后插件仍然可用；卷丢了，官方目录里的插件会按数据库里的安装记录重新下载，手工上传的插件则必须重传。离线环境可以用页面上的 **Install from file** 上传打包好的插件，或者在设置里把 Plugin catalog 换成本地目录地址。

插件是纯前端模块，运行在浏览器里，使用的是当前登录用户的 Kite 权限。页面上的那句提示不是客套：只安装你信任来源的插件，它和后端服务拥有同一套访问能力。

## OIDC 登录（可选）

如果已经有 Keycloak 之类的 OIDC 提供方，可以在 **Settings → Authentication → Add Provider** 里接入。表单里需要填的字段是：

| 字段 | 说明 |
|---|---|
| Name | 提供方标识，不是显示名；会出现在登录按钮和 `?provider=` 参数里 |
| Issuer | OIDC issuer，例如 `https://auth.example.com/realms/homelab`，Kite 用它做 discovery |
| Client ID / Client Secret | 在 IdP 侧创建的客户端凭据 |
| Scopes | 默认 `openid,profile,email` |
| Username Claim | 可选，指定用哪个 claim 当用户名 |
| Groups Claim | 可选，指定组信息的 claim，例如 `groups` |
| Allowed Groups | 可选，逗号分隔；配置后只有属于这些组的用户能登录 |

回调地址由 Kite 按 `HOST` 生成，路径固定为：

```text
https://kite.example.com/api/auth/callback
```

IdP 侧的 redirect URI 必须与它逐字一致。改域名、改 `HOST`、改反向代理的外部地址，三处要一起改。登录按钮不出现或点登录返回 500，先看 Kite 日志里的 OIDC discovery 请求；如果 IdP 使用内网 CA，还要把 CA 挂进容器并设置 `SSL_CERT_DIR`。

这里有一个 Go 运行时的细节：`SSL_CERT_DIR` 一旦设置，是**替换**默认证书目录而不是追加。只写内网 CA 目录会把公共根证书一起丢掉，症状是 SSO 能通、插件目录却拉不下来，或者反过来。正确写法是两段：

```yaml
volumes:
  - ./internal-ca:/etc/ssl/certs/internal:ro
environment:
  SSL_CERT_DIR: /etc/ssl/certs:/etc/ssl/certs/internal
```

改完要重启容器，Go 只在启动时读一次信任库。

OIDC 的完整登录链路（含 IdP 侧客户端配置）不在本次实测范围内，上面的字段和回调路径来自实际页面与官方文档；如果你照着配，先用一个测试账号走一遍，确认组映射和 RBAC 都符合预期再放开。

## 常见问题

**连接集群超时，但宿主机 `kubectl` 正常。** 先看 kubeconfig 的 `server` 是不是 `127.0.0.1` 或 `localhost`。容器里的回环地址指向容器自己，必须换成 Kite 容器能访问的地址，这是本文最集中的坑。

**报 `x509: certificate is valid for ...`。** 地址能通，但证书 SAN 不含你用的主机名。要么改用证书里已有的域名，要么给 API Server 证书补 SAN；不要长期用 `insecure-skip-tls-verify` 绕过，那等于把中间人防护关掉。

**Kite 起来了，但 PostgreSQL 里没有表。** 检查 `DB_TYPE` 是否写成 `postgres`，以及 `DB_DSN` 是否指向正确的库。Kite 默认回落 SQLite，这种情况下容器本身完全正常，只是数据写进了容器内的文件。

**概览页 CPU/内存一直是 0。** 集群没有 metrics-server，或者 Kite 没有权限读 `metrics.k8s.io`。这和 Prometheus 是两套数据源，不要混在一起排查。

**保存 Prometheus 地址后曲线仍然没有。** 先看 Kite 日志里的 `query_range` 请求是否报 DNS 或连接错误，再确认地址从容器内可达。面板能打开、列表能刷新，说明集群连接本身没问题。

**宿主机或 Docker 重启后，Kite 里一个集群都没有了，但数据库里还有记录。** Kite 启动时如果连不上集群，就不会把它注册进运行时列表。等集群恢复后执行 `docker compose restart kite` 即可，kubeconfig 不需要重贴。

**升级镜像后启动失败或卡住。** Kite 在启动时执行数据库迁移，迁移不可回退。升级前先备份：

```bash
docker exec kite-demo-postgres pg_dump -U kite -d kite > kite-backup.sql
```

然后改镜像 tag、`docker compose up -d`。不要跨多个大版本直接跳，先读对应版本的 release notes。

**日志和终端一直转圈，列表却正常。** 多半是反向代理没有转发 WebSocket 的 `Upgrade` / `Connection` 头。Kite 的实时日志、watch 和终端都走长连接，普通的 HTTP 反代配置不够。

## 适用场景

Kite 适合「集群不多、想要一个浏览器入口、愿意自己管一份 kubeconfig」的场景。单集群 homelab、几台机器的测试环境、需要给不熟悉 kubectl 的同事看资源，这些都很顺手。

如果你需要的是声明式发布和漂移检测，应该继续用 GitOps 工具；如果你要管几十个集群、需要细粒度的多租户隔离和统一策略，Rancher 这类平台更合适；如果你只在本地偶尔看看，Lens 或 Headlamp 的桌面体验更轻。

还要接受它的版本节奏：Kite 仍在 0.x，数据库迁移和配置项会变。镜像 tag 钉死，升级前备份，这两条比选哪个面板更重要。
