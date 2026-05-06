# Headscale 部署与使用指南

## 目录

- [1. 架构概览](#1-架构概览)
- [2. 启动 Headscale](#2-启动-headscale)
- [3. 域名绑定与 HTTPS](#3-域名绑定与-https)
- [4. 添加 Node（节点）](#4-添加-node节点)
- [5. 添加 Route（路由）](#5-添加-route路由)
- [6. 常见问题](#6-常见问题)
- [7. 常用运维命令](#7-常用运维命令)

---

## 1. 架构概览

本部署由以下服务组成：

| 服务 | 镜像 | 端口 | 说明 |
|------|------|------|------|
| headscale | headscale/headscale:latest | 8080/tcp, 3478/udp | Tailscale 控制服务器 |
| headscale-postgres | postgres:16-alpine | 5432 | PostgreSQL 数据库 |
| headscale-ui | goodieshq/headscale-admin:latest | 8000->80 | Web 管理界面 |
| caddy | caddy:latest | 80, 443 | 反向代理 & 自动 HTTPS |

目录结构：

```
/opt/headscale/
├── config/
│   └── config.yaml           # Headscale 配置文件
├── data/                      # Headscale 数据（密钥、缓存等）
├── run/                       # Unix socket 目录
├── postgressql/
│   ├── init-script/
│   │   └── init-db.sh        # 数据库初始化脚本
│   └── postgres-data/         # PostgreSQL 数据
├── caddy/
│   ├── Caddyfile             # Caddy 反向代理配置
│   ├── caddy_data/           # 证书等数据
│   └── caddy_config/
├── docker-compose.yaml        # Headscale + Postgres
├── headscale-ui.yaml          # Headscale Admin UI
└── caddy.yaml                 # Caddy
```

---

## 2. 启动 Headscale

### 2.1 启动 PostgreSQL + Headscale

```bash
cd /opt/headscale
docker compose -f docker-compose.yaml up -d
```

`docker-compose.yaml` 内容：

```yaml
services:
  headscale:
    image: headscale/headscale:latest
    container_name: headscale
    restart: unless-stopped
    volumes:
      - /opt/headscale/config:/etc/headscale
      - /opt/headscale/data:/var/lib/headscale
      - /opt/headscale/run:/var/run/headscale
    ports:
      - "8080:8080"
      - "3478:3478/udp"
    environment:
      - TZ=Asia/Shanghai
    command: serve
```

> **注意**：首次启动前需确保 `config.yaml` 中 PostgreSQL 连接信息正确：
> ```yaml
> database:
>   type: postgres
>   postgres:
>     host: headscale-postgres
>     port: 5432
>     name: headscale
>     user: root
>     pass: 1a2b3c4d5e6f
> ```

如果 PostgreSQL 容器尚未初始化，需先启动它：

```bash
docker run -d \
  --name headscale-postgres \
  -e POSTGRES_USER=root \
  -e POSTGRES_PASSWORD=1a2b3c4d5e6f \
  -e APP_DB_NAME=headscale \
  -e APP_DB_USER=root \
  -e APP_DB_PASSWORD=1a2b3c4d5e6f \
  -v /opt/headscale/postgressql/init-script:/docker-entrypoint-initdb.d \
  -v /opt/headscale/postgressql/postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  --restart unless-stopped \
  postgres:16-alpine
```

### 2.2 启动 Headscale Admin UI

```bash
docker compose -f headscale-ui.yaml up -d
```

`headscale-ui.yaml` 内容：

```yaml
services:
  headscale-ui:
    image: ghcr.io/gurucomputing/headscale-ui:latest
    restart: unless-stopped
    container_name: headscale-ui
    ports:
      - 8003:8443
      - 8000:8080
```

### 2.3 启动 Caddy 反向代理

```bash
docker compose -f caddy.yaml up -d
```

`caddy.yaml` 内容：

```yaml
version: '3'
services:
  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
      - ./caddy/caddy_data:/data
      - ./caddy/caddy_config:/config
```

### 2.4 一键全部启动

```bash
cd /opt/headscale
docker compose -f docker-compose.yaml up -d
docker compose -f headscale-ui.yaml up -d
docker compose -f caddy.yaml up -d
```

### 2.5 验证服务状态

```bash
docker ps --filter "name=headscale" --filter "name=caddy"
curl http://localhost:8080/health
```

---

## 3. 域名绑定与 HTTPS

### 3.1 DNS 配置

在域名服务商处添加 A 记录：

| 主机记录 | 记录类型 | 记录值 |
|---------|---------|--------|
| headscale | A | 121.40.25.88 |

即 `headscale.hello-gpt.cn` → `121.40.25.88`

### 3.2 Caddy 反向代理配置

Caddy 的核心配置文件为 `/opt/headscale/caddy/Caddyfile`：

```caddyfile
headscale.hello-gpt.cn {
    handle /admin* {
        reverse_proxy http://127.0.0.1:8000
    }
    reverse_proxy * http://127.0.0.1:8080
    log {
        output file /var/log/caddy/caddy.access.log
        level debug
    }
}
```

**路由规则说明**：

- `/admin*` → 转发到 `127.0.0.1:8000`（Headscale Admin UI）
- 其他所有请求 → 转发到 `127.0.0.1:8080`（Headscale API）

### 3.3 自动 HTTPS

Caddy 会自动为 `headscale.hello-gpt.cn` 申请 Let's Encrypt 证书，前提：

1. 域名 DNS 已正确指向服务器 IP
2. 服务器的 80 和 443 端口对外可访问
3. Caddy 容器已映射 80 和 443 端口

### 3.4 访问地址

| 功能 | 地址 |
|------|------|
| Headscale API | `https://headscale.hello-gpt.cn/` |
| Admin 管理界面 | `https://headscale.hello-gpt.cn/admin/settings/` |

### 3.5 Admin 界面登录

1. 访问 `https://headscale.hello-gpt.cn/admin/settings/`
2. 输入 API Key 登录
3. 生成 API Key 的命令：

```bash
docker exec headscale headscale apikeys create
```

---

## 4. 添加 Node（节点）

### 4.1 创建用户

每个节点需要关联到一个用户：

```bash
# 注意：--user 参数在部分命令中需要用户 ID（数字），不是用户名
docker exec headscale headscale users create mengwei968@qq.com
```

查看已有用户：

```bash
docker exec headscale headscale users list
```

### 4.2 生成预认证 Key

使用预认证 key（pre-auth key）可以简化节点加入流程：

```bash
# 注意：--user 参数需要用户 ID（数字），不是用户名
# 先查看用户 ID
docker exec headscale headscale users list

# 生成一次性 key（-u 后跟用户 ID）
docker exec headscale headscale preauthkeys create -u <user-id>

# 生成可重复使用的 key
docker exec headscale headscale preauthkeys create -u <user-id> --reusable

# 设置过期时间
docker exec headscale headscale preauthkeys create -u <user-id> --expiration 24h

# 查看已有 key
docker exec headscale headscale preauthkeys list -u <user-id>
```

### 4.3 在客户端节点上加入网络

#### Linux

```bash
# 安装 tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# 使用预认证 key 加入
tailscale up --login-server https://headscale.hello-gpt.cn --authkey <pre-auth-key>
```

#### macOS

```bash
# 安装 tailscale
brew install tailscale

# 加入网络
tailscale up --login-server https://headscale.hello-gpt.cn --authkey <pre-auth-key>
```

或使用 GUI 版 Tailscale，在设置中自定义控制服务器地址为 `https://headscale.hello-gpt.cn`。

#### Windows

1. 下载安装 [Tailscale Windows 客户端](https://tailscale.com/download/windows)
2. 打开命令行：

```cmd
tailscale up --login-server https://headscale.hello-gpt.cn --authkey <pre-auth-key>
```

#### 不使用预认证 Key（交互式登录）

```bash
tailscale up --login-server https://headscale.hello-gpt.cn
```

命令会输出一个 URL，将该 URL 中的 `https://controlplane.tailscale.com` 替换为 `https://headscale.hello-gpt.cn`，在浏览器中打开完成认证。或者在服务端手动批准：

```bash
# 查看待注册节点
docker exec headscale headscale nodes list

# 注册节点（按 node ID）
docker exec headscale headscale nodes register --id <node-id>
```

### 4.4 查看已加入的节点

```bash
docker exec headscale headscale nodes list
```

### 4.5 删除节点

```bash
docker exec headscale headscale nodes delete --id <node-id>
```

### 4.6 重命名节点

```bash
docker exec headscale headscale nodes rename --id <node-id> <new-name>
```

---

## 5. 添加 Route（路由）

Route 功能允许某个节点充当子网路由器，将局域网内的其他设备暴露给 Tailscale 网络。

### 5.1 开启子网路由广播

在需要充当路由器的节点上执行：

```bash
# 广播单个子网
tailscale up --login-server https://headscale.hello-gpt.cn --advertise-routes=192.168.1.0/24

# 广播多个子网
tailscale up --login-server https://headscale.hello-gpt.cn --advertise-routes=192.168.1.0/24,10.0.0.0/24

# 同时作为 Exit Node（出口节点）
tailscale up --login-server https://headscale.hello-gpt.cn --advertise-routes=192.168.1.0/24 --advertise-exit-node
```

### 5.2 在服务端批准路由

节点广播路由后，需要在 Headscale 服务端批准：

```bash
# 查看所有节点的路由（Approved=已批准，Available=节点广播的，Serving=生效的）
docker exec headscale headscale nodes list-routes

# 批准路由（需指定节点 ID 和路由 CIDR）
docker exec headscale headscale nodes approve-routes -i <node-id> -r 10.0.0.0/16

# 批准多个路由
docker exec headscale headscale nodes approve-routes -i <node-id> -r 10.0.0.0/16,192.168.1.0/24

# 移除已批准的路由（传空字符串）
docker exec headscale headscale nodes approve-routes -i <node-id> -r ""
```

### 5.3 在其他节点上使用路由

批准后，其他节点需要启用路由接收：

```bash
# 查看可用路由
tailscale status

# 接受路由（默认已启用，取决于客户端配置）
tailscale up --login-server https://headscale.hello-gpt.cn --accept-routes
```

### 5.4 使用 Exit Node（出口节点）

#### 广播为 Exit Node

在出口节点上：

```bash
tailscale up --login-server https://headscale.hello-gpt.cn --advertise-exit-node
```

#### 在服务端批准 Exit Node

```bash
# 查看路由
docker exec headscale headscale nodes list-routes

# 批准 Exit Node（0.0.0.0/0 和 ::/0 表示全部流量走该节点）
docker exec headscale headscale nodes approve-routes -i <node-id> -r 0.0.0.0/0,::/0
```

#### 通过 Exit Node 路由流量

在其他节点上：

```bash
# 使用某个 Exit Node
tailscale up --login-server https://headscale.hello-gpt.cn --exit-node=<exit-node-ip-or-name>

# 停止使用 Exit Node
tailscale up --exit-node=
```

### 5.5 查看路由状态

```bash
# 列出所有节点的路由
docker exec headscale headscale nodes list-routes

# 列出某个节点的路由
docker exec headscale headscale nodes list-routes -i <node-id>
```

### 5.6 禁用路由

```bash
# 移除节点的已批准路由（传空字符串即移除所有）
docker exec headscale headscale nodes approve-routes -i <node-id> -r ""
```

---

## 6. 常见问题

### Q: `--user` 参数报错 `strconv.ParseUint: parsing "xxx": invalid syntax`

Headscale 的 `--user` / `-u` 参数在部分命令（如 `preauthkeys create`、`preauthkeys list`）中要求传入**用户 ID（数字）**，而不是用户名。

**错误示例：**
```bash
docker exec headscale headscale preauthkeys create --user mengwei968@qq.com
# Error: invalid argument "mengwei968@qq.com" for "-u, --user" flag: strconv.ParseUint
```

**正确做法：** 先通过 `users list` 查看用户 ID，再用数字 ID 执行命令：

```bash
# 查看用户列表，获取 ID
docker exec headscale headscale users list
# ID | Name               | ...
# 3  | mengwei968@qq.com  | ...

# 使用数字 ID
docker exec headscale headscale preauthkeys create -u 3
```

> **注意**：`users create` 和 `users delete` 命令支持用户名，但 `preauthkeys` 系列命令的 `--user` 只接受数字 ID。

---

## 7. 常用运维命令

### 服务管理

```bash
# 查看所有容器状态
docker ps -a --filter "name=headscale" --filter "name=caddy"

# 重启 Headscale
docker restart headscale

# 查看 Headscale 日志
docker logs headscale --tail 100 -f

# 查看 Caddy 日志
docker logs caddy --tail 100 -f
```

### 用户管理

```bash
# 创建用户
docker exec headscale headscale users create <username>

# 列出用户
docker exec headscale headscale users list

# 删除用户
docker exec headscale headscale users delete <username>
```

### API Key 管理

```bash
# 生成 API Key
docker exec headscale headscale apikeys create

# 列出 API Key
docker exec headscale headscale apikeys list

# 过期 API Key
docker exec headscale headscale apikeys expire --prefix <key-prefix>
```

### 节点管理

```bash
# 列出所有节点
docker exec headscale headscale nodes list

# 注册节点
docker exec headscale headscale nodes register --id <node-id>

# 删除节点
docker exec headscale headscale nodes delete --id <node-id>

# 重命名节点
docker exec headscale headscale nodes rename --id <node-id> <new-name>

# 查看节点详情
docker exec headscale headscale nodes view --id <node-id>
```

### 路由管理

```bash
# 列出所有节点的路由
docker exec headscale headscale nodes list-routes

# 批准路由
docker exec headscale headscale nodes approve-routes -i <node-id> -r 10.0.0.0/16

# 移除已批准的路由
docker exec headscale headscale nodes approve-routes -i <node-id> -r ""
```

### 配置文件

```bash
# 编辑配置文件
vim /opt/headscale/config/config.yaml

# 修改配置后重启服务
docker restart headscale
```

### 备份

```bash
# 备份配置和数据
tar czf headscale-backup-$(date +%Y%m%d).tar.gz /opt/headscale/config /opt/headscale/data

# 备份数据库
docker exec headscale-postgres pg_dump -U root headscale > headscale-db-$(date +%Y%m%d).sql
```
