# FreeLLMAPI Docker 镜像构建与运行说明

本文说明三件事：

1. 这个项目的 Docker 镜像里有什么、怎么构建；
2. 用官方镜像或本地镜像把服务跑起来的三种方式；
3. 首次启动后的配置、持久化、升级与排障。

> 所有命令默认在仓库根目录执行。仓库中的参考文件：
> [Dockerfile](Dockerfile)、[docker-compose.yml](docker-compose.yml)、
> [docker-entrypoint.sh](docker-entrypoint.sh)、[.dockerignore](.dockerignore)、
> [.env.example](.env.example)、[docker/README.md](docker/README.md)。

---

## 目录

- [1. 镜像概览](#1-镜像概览)
- [2. 前置条件](#2-前置条件)
- [3. 构建镜像](#3-构建镜像)
- [4. 运行方式](#4-运行方式)
- [5. 首次启动与配置](#5-首次启动与配置)
- [6. 环境变量](#6-环境变量)
- [7. 数据持久化与备份](#7-数据持久化与备份)
- [8. 升级、停止与卸载](#8-升级停止与卸载)
- [9. 健康检查与排障](#9-健康检查与排障)
- [10. 安全建议](#10-安全建议)

---

## 1. 镜像概览

一个容器同时提供 **Express API** 和 **构建好的 React 仪表盘**，单进程监听 `3001`，数据落在容器内 `/app/server/data` 的 SQLite 文件里。

| 项目 | 说明 |
|---|---|
| Node 版本 | `node:24.22.1-bookworm-slim`（可用 `NODE_IMAGE` 构建参数覆盖） |
| 构建方式 | 多阶段：`deps` → `build` → `runtime` |
| 依赖安装 | `npm ci`，构建阶段安装 `python3 make g++` 以编译 `better-sqlite3` |
| 产物 | `server/dist` + `client/dist` + 运行时 `node_modules` |
| 运行用户 | 入口脚本以 root 修正数据目录属主，随后降权到 `node` 用户运行 |
| 暴露端口 | `3001`（`EXPOSE 3001`） |
| 健康检查 | 每 30s 请求 `http://127.0.0.1:3001/api/ping`，启动宽限 15s |
| 启动命令 | `node server/dist/index.js` |
| 内置环境变量 | `NODE_ENV=production`、`PORT=3001`、`FREELLMAPI_INSTALL_METHOD=docker` |
| 数据目录 | `/app/server/data` |
| 镜像标签 | `latest`、`v*.*.*`、`sha-<commit>`，多架构 `linux/amd64` + `linux/arm64` |

两个容易忽略的设计：

- **镜像故意不声明 `VOLUME`**。持久化由部署方负责：Compose 挂载具名卷，`docker run` 使用 `-v`。这样在 Railway / Render 等 PaaS 上重建时不会拿到一个匿名空卷。
- **入口脚本会尝试 `chown -R node:node` 数据目录**，以兼容 PaaS 上 root 属主的挂载卷；chown 失败只告警，不阻断启动。

---

## 2. 前置条件

- Docker Engine 20.10+ 与 Compose v2（`docker compose` 子命令）。
- 生成加密密钥需要 `openssl`（macOS/Linux 自带）。
- 构建镜像需要能访问 npm registry；拉取镜像需要能访问 GHCR。
- macOS/Windows 用户需先启动 Docker Desktop；确认守护进程可用：

```bash
docker info > /dev/null && echo "docker ready"
```

---

## 3. 构建镜像

### 3.1 基础构建

```bash
cd /path/to/freellmapi
docker build -t freellmapi:local .
```

构建参数：

| 参数 | 默认值 | 用途 |
|---|---|---|
| `NODE_IMAGE` | `node:24.22.1-bookworm-slim` | 覆盖基础镜像 |
| `FREELLMAPI_COMMIT_SHA` | 空 | 写入镜像，仪表盘展示当前版本对应的提交 |

```bash
docker build \
  --build-arg FREELLMAPI_COMMIT_SHA="$(git rev-parse --short HEAD)" \
  -t freellmapi:local .
```

### 3.2 多架构构建（推送到自己的 Registry）

`docker build` 一次只能产出当前平台。需要同时支持 Intel 与 ARM 时用 Buildx，并且必须 `--push`（多平台清单无法 `--load` 到本地）：

```bash
docker buildx create --name freellmapi-builder --use   # 只需执行一次

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --build-arg FREELLMAPI_COMMIT_SHA="$(git rev-parse --short HEAD)" \
  -t ghcr.io/<你的命名空间>/freellmapi:local \
  --push .
```

> 在 Apple Silicon 上构建 `linux/amd64` 会走 QEMU 模拟，`better-sqlite3` 需要现场编译，速度明显更慢。日常本地使用直接构建原生架构即可；需要 amd64 时优先拉官方多架构镜像。

### 3.3 构建上下文说明

[.dockerignore](.dockerignore) 已排除 `.git`、`node_modules`、`server/dist`、`client/dist`、`server/data`、`.env*`、`*.db*`、`.encryption-key` 等。

**不要**把包含真实提供方密钥的 `.env` 打进镜像；它只在运行时由 `--env-file` / Compose `env_file` 注入。

---

## 4. 运行方式

无论用哪种方式，都必须提供 `ENCRYPTION_KEY`（64 位十六进制，AES-256-GCM 主密钥）。先生成并保存：

```bash
ENCRYPTION_KEY="$(openssl rand -hex 32)"
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$ENCRYPTION_KEY" > .env
```

> `.env` 一旦丢失或更换，已加密存储的上游 provider key 将无法解密，没有恢复途径。请像对待密码一样备份它。

### 4.1 方式一：官方镜像 + docker run

```bash
docker pull ghcr.io/tashfeenahmed/freellmapi:latest

docker run -d \
  --name freellmapi \
  --restart unless-stopped \
  --env-file .env \
  -p 127.0.0.1:3001:3001 \
  -v freellmapi-data:/app/server/data \
  ghcr.io/tashfeenahmed/freellmapi:latest

docker logs -f freellmapi
```

- `-p 127.0.0.1:3001:3001` 只绑定本机。要局域网访问改成 `-p 3001:3001` 或 `-p 192.168.1.50:3001:3001`。
- 想换宿主端口：`-p 127.0.0.1:8080:3001`，容器内仍监听 3001。
- 需要访问宿主机上的 Ollama / 代理时，追加 `--add-host=host.docker.internal:host-gateway`。

### 4.2 方式二：Docker Compose（推荐）

仓库自带 [docker-compose.yml](docker-compose.yml)，包含具名卷、健康检查、默认本机绑定，以及可选的 Watchtower 自动更新 profile。

```bash
docker compose up -d
docker compose logs -f freellmapi
```

Compose 中同时声明了 `image:` 和 `build:`：

- 本地没有该镜像时会**从源码构建**；
- 想用官方预构建镜像，先 `docker compose pull` 再 `up -d`；
- 想强制重建，用 `docker compose up -d --build`。

端口绑定由两个变量控制：

| 变量 | 作用 | 默认 |
|---|---|---|
| `HOST_BIND` | Compose 端口映射的宿主绑定地址 | `127.0.0.1` |
| `PORT` | **宿主**映射端口；容器内始终是 `3001` | `3001` |

```bash
# 局域网可访问
HOST_BIND=0.0.0.0 docker compose up -d

# 宿主端口改为 8080（容器内仍是 3001）
PORT=8080 docker compose up -d
```

### 4.3 方式三：本地源码运行（非 Docker，对照参考）

容器内使用的是 Node 24，本机运行要求 Node `>=20.18 <25`（见 `.nvmrc` 的 22）：

```bash
npm install
npm run dev          # 仪表盘 :5173，API :3001
# 或生产构建
npm run build && node server/dist/index.js
```

---

## 5. 首次启动与配置

1. 打开 `http://localhost:3001`。
2. 创建第一个仪表盘账号（邮箱 + 密码）。
   - 与服务器同机的浏览器可直接创建；
   - 如果通过局域网/远程访问且尚无账号，启动日志会打印一次性安装码，用它完成创建。
3. 进入 **密钥 / Keys** 页，添加至少一家上游提供方的 API Key：
   - 内置平台：Groq、Google Gemini、Cerebras、OpenRouter、NVIDIA、Mistral、Cloudflare、智谱等；
   - 自定义端点：任意 OpenAI 兼容地址，如 `http://host.docker.internal:11434/v1`（宿主机 Ollama）。
4. 在 **密钥** 页顶部复制统一密钥 `freellmapi-…`，这就是给 OpenAI SDK / CLI 使用的 Key。

验证：

```bash
curl http://localhost:3001/v1/chat/completions \
  -H "Authorization: Bearer freellmapi-your-unified-key" \
  -H "Content-Type: application/json" \
  -d '{"model":"auto","messages":[{"role":"user","content":"hi"}]}'
```

> 没有配置任何上游 Key 时，请求会返回 “All models exhausted / no key configured”——这是预期行为，统一密钥不能替代上游 Key。

批量配置可用 `.env` 或声明式 JSON：

```env
FREEAPI_CONFIG_JSON={"keys":[{"platform":"groq","key":"gsk_...","label":"main"}],"routing":{"strategy":"balanced"}}
```

字段说明见 [.env.example](.env.example) 与 [安装文档](docs/zh-cn/install/01-install.md#声明式启动配置)。

---

## 6. 环境变量

容器相关的核心变量：

| 变量 | 必填 | 默认 | 说明 |
|---|---|---|---|
| `ENCRYPTION_KEY` | **是** | 无 | 64 位 hex，加密存储上游 Key。生产环境不设置会启动失败 |
| `PORT` | 否 | `3001` | 容器内监听端口；Compose 已固定为 3001 |
| `HOST` | 否 | `::` | 容器内绑定地址 |
| `HOST_BIND` | 否 | `127.0.0.1` | **仅 Compose**：宿主端口绑定地址 |
| `FREEAPI_DB_PATH` | 否 | `/app/server/data/freeapi.db` | SQLite 路径（代码默认文件名是 `freeapi.db`，部分文档写作 `freellmapi.db`，以本变量为准） |
| `FREEAPI_DB_BACKUP_PATH` | 否 | 无 | 本地加密备份文件，缺失时启动恢复 |
| `FREEAPI_DB_BACKUP_URL` | 否 | 无 | HTTP(S) 加密备份目标（GET 恢复 / PUT 上传） |
| `FREEAPI_DB_BACKUP_TOKEN` | 否 | 无 | 备份 URL 的 Bearer Token |
| `FREEAPI_DB_BACKUP_KEY` | 否 | `ENCRYPTION_KEY` | 备份信封密钥 |
| `FREEAPI_CONFIG_PATH` | 否 | 无 | 启动时幂等应用的 JSON 配置 |
| `FREEAPI_CONFIG_JSON` | 否 | 无 | 内联 JSON 配置，优先于 PATH |
| `PROXY_URL` | 否 | 无 | 出站代理，如 `socks5h://host.docker.internal:7890` |
| `TRUST_PROXY` | 否 | `false` | 反向代理后面时信任的地址/CIDR，或 `true` |
| `ALLOWED_HOSTS` | 否 | 空 | 开启 DNS rebinding 防护的主机白名单 |
| `DASHBOARD_ORIGINS` | 否 | 空 | 额外允许的仪表盘跨域来源 |
| `FREEAPI_BLOCK_PRIVATE_PROVIDER_URLS` | 否 | 未设置 | 设为 `true` 时禁止自定义端点指向 loopback/私网 |
| `REQUEST_ANALYTICS_RETENTION_DAYS` | 否 | `90` | 请求分析保留天数，`0` 关闭 |
| `REQUEST_ANALYTICS_MAX_ROWS` | 否 | `100000` | 请求记录行数上限，`0` 关闭 |
| `WATCHTOWER_TOKEN` | 否 | 无 | 开启 `autoupdate` profile 时用于仪表盘内一键更新 |

完整变量清单（含路由、额度、压缩、探测等）见 [.env.example](.env.example)。

---

## 7. 数据持久化与备份

数据库和全部配置都在 SQLite 中：

```
/app/server/data/freeapi.db      # 默认；可用 FREEAPI_DB_PATH 覆盖
```

必须保留两样东西：

1. **数据卷**：`freellmapi-data`（或你自己 `-v` 挂载的目录）；
2. **同一把 `ENCRYPTION_KEY`**。

升级、重建容器、换机器时两者缺一不可。只拷数据库而不拷密钥，上游 Key 全部不可解。

备份方式：

```bash
# 简单快照（停服后拷贝卷）
docker run --rm -v freellmapi-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/freellmapi-data-$(date +%F).tar.gz -C /data .

# 或使用内置加密备份
# .env 中配置
FREEAPI_DB_BACKUP_PATH=/app/server/data/freeapi.db.backup
FREEAPI_DB_BACKUP_KEY=<64 位 hex>
FREEAPI_DB_BACKUP_INTERVAL_MS=300000
```

启动时若数据库文件不存在，会先尝试从备份恢复，再执行迁移。

---

## 8. 升级、停止与卸载

```bash
# 升级到最新发布镜像（保留卷与 .env）
docker compose pull
docker compose up -d

# 可选：启用 Watchtower，之后可在仪表盘「设置 → 检查更新 → 立即更新」一键升级
# （需在 .env 设置 WATCHTOWER_TOKEN；会挂载 docker.sock，等效 root 权限）
docker compose --profile autoupdate up -d

# 查看状态 / 日志
docker compose ps
docker compose logs -f freellmapi

# 停止（保留数据）
docker compose down

# 停止并删除数据卷（不可恢复，先备份）
docker compose down -v
```

用 `docker run` 部署时：

```bash
docker pull ghcr.io/tashfeenahmed/freellmapi:latest
docker rm -f freellmapi
# 用与首次启动完全相同的 -v、--env-file 重新 docker run
```

> 升级前建议先做一次数据库备份，尤其是跨越多个版本时。

---

## 9. 健康检查与排障

### 9.1 健康状态

```bash
docker inspect --format '{{.State.Health.Status}}' freellmapi
curl -fsS http://localhost:3001/api/ping
```

镜像健康检查每 30s 请求 `/api/ping`，失败 3 次判为 unhealthy，启动后有 15s 宽限。

### 9.2 常见问题

| 现象 | 原因 / 处理 |
|---|---|
| 容器启动即退出，日志提示 `ENCRYPTION_KEY` | 未设置或仍是占位值；生成 64 位 hex 并重启 |
| `docker compose` 报 `env file .env not found` | 先创建 `.env`（见第 4 节） |
| 端口被占用 | 改宿主端口：`PORT=8080 docker compose up -d` |
| 本机可访问、局域网打不开 | 默认只绑 `127.0.0.1`；用 `HOST_BIND=0.0.0.0` |
| 容器内访问不到宿主机 Ollama | Base URL 用 `http://host.docker.internal:11434/v1`；`docker run` 需加 `--add-host=host.docker.internal:host-gateway`；仍 connection refused 时让 Ollama 监听 `0.0.0.0`（见下文） |
| 容器内访问不了外网/提供方 | 配置 `PROXY_URL=socks5h://host.docker.internal:7890`，并让代理允许非 loopback 连接（Clash 的 `allow-lan: true`） |
| 请求返回 “All models exhausted / no key configured” | 仪表盘密钥页没有任何可用上游 Key，先添加 |
| 自定义端点返回 `404 model not found` | 登记的模型 id 与端点 `/v1/models` 返回不一致，逐字符对齐 |
| PaaS 上 SQLite 无法写入 | 入口脚本已尝试 chown；若用了 `--cap-drop=CHOWN` 或 NFS/CIFS，可能只告警，需自行保证目录可写 |
| 构建时 `better-sqlite3` 编译失败 | 构建阶段已装 `python3 make g++`；ARM 跨架构构建建议用 Buildx 原生平台或官方镜像 |
| 更新后密钥全部失效 | `ENCRYPTION_KEY` 被替换；只能恢复旧密钥或重新录入所有上游 Key |

### 9.3 在 Docker 里接入宿主机 Ollama

仪表盘「密钥 → 添加自定义端点」填写：

| 字段 | 值 |
|---|---|
| Base URL | `http://host.docker.internal:11434/v1`（必须带 `/v1`） |
| API Key | 留空即可（Ollama 不校验；表单若强制要求可填任意占位符） |
| 模型 | 与 `curl http://host.docker.internal:11434/v1/models` 返回的 id 逐字符一致，如 `qwen3.5:0.8b` |

不要填 `127.0.0.1` 或 `localhost`——在容器里它们指向容器自身。

如果容器侧报 `ECONNREFUSED`，说明 Ollama 只监听了宿主机的 loopback：

```bash
# macOS：退出 Ollama，设置监听地址后重新打开
launchctl setenv OLLAMA_HOST "0.0.0.0:11434"
open -a Ollama
# 或直接前台运行
OLLAMA_HOST=0.0.0.0 ollama serve
```

`0.0.0.0` 会让 Ollama 对局域网开放且它没有鉴权，建议只在需要时临时开启，或在防火墙/路由器层限制来源。若 Ollama 本身也跑在 Compose 里，则改用 `http://ollama:11434/v1`（同一 Docker 网络的服务名，无需 `host.docker.internal`）。

容器内自检：

```bash
docker compose exec freellmapi node -e \
  "fetch('http://host.docker.internal:11434/v1/models').then(r=>r.text()).then(console.log)"
```

另外确认没有开启 `FREEAPI_BLOCK_PRIVATE_PROVIDER_URLS=true`——它会把 `host.docker.internal` 这类私有地址一并拦掉（见 [变量说明](docs/zh-cn/env/01-variables.md)）。

更多问题见 [docs/zh-cn/troubleshooting/01-common-issues.md](docs/zh-cn/troubleshooting/01-common-issues.md)。

---

## 10. 安全建议

- **默认只绑定 `127.0.0.1`**。这是单用户产品，唯一防护就是统一密钥，不要直接暴露到公网。
- 需要远程访问时，放在带 TLS 的反向代理后面，并正确设置 `TRUST_PROXY`（例如 `TRUST_PROXY=192.168.1.10` 或 CIDR）。
- 暴露到可被他人访问的环境时，建议：
  - 设置 `ALLOWED_HOSTS` 防 DNS rebinding；
  - 设置 `FREEAPI_BLOCK_PRIVATE_PROVIDER_URLS=true`，禁止自定义端点访问 loopback/私网/云元数据地址；
  - 限制 `DASHBOARD_ORIGINS`。
- `.env` 与 `FREEAPI_DB_BACKUP_KEY` 按密码级别保管，不要提交到 Git，也不要打进镜像。
- 上游免费额度受各提供方 ToS 约束：单账号、不转售、不共享端点、不当作付费生产后端。详见 [提供方文档](docs/zh-cn/providers/01-supported-platforms.md)。
