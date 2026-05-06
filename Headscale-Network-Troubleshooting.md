# Headscale 网络连接排查指南

## 目录

- [1. 检查节点连接方式](#1-检查节点连接方式)
- [2. 点对点直连失败排查](#2-点对点直连失败排查)
- [3. NAT 类型详解](#3-nat-类型详解)
- [4. 优化中继延迟](#4-优化中继延迟)
- [5. 常见问题汇总](#5-常见问题汇总)

---

## 1. 检查节点连接方式

### 1.1 查看节点状态

```bash
tailscale status
```

输出示例：

```
100.64.0.2  goodeep-ijfxgrpp  mengwei968@  linux    -
100.64.0.6  goodeep           mengwei968@  linux    offline, last seen 201d ago
100.64.0.4  hasee-pc          mengwei968@  windows  offline, last seen 1d ago
100.64.0.1  work-mac          mengwei968@  macOS    active; relay "tok", tx 183576 rx 34736
```

**关键字段解读**：

| 状态 | 含义 |
|------|------|
| `active; direct` | 点对点直连 |
| `active; relay "xxx"` | 通过 DERP 中继转发（xxx 为中继节点名） |
| `-` | 当前无活跃连接 |
| `offline` | 节点离线 |

上例中 `work-mac` 显示 `relay "tok"`，说明走的是东京中继，不是点对点直连。

### 1.2 检测网络连通性

```bash
tailscale netcheck
```

输出示例：

```
Report:
        * Time: 2026-05-06T09:53:09.732426799Z
        * UDP: true
        * IPv4: yes, 125.121.239.18:35849
        * IPv6: no, but OS has support
        * MappingVariesByDestIP: false
        * PortMapping:
        * CaptivePortal: false
        * Nearest DERP: San Francisco
        * DERP latency:
                - sfo: 159ms   (San Francisco)
                - lax: 163.5ms (Los Angeles)
                - tok: 271.8ms (Tokyo)
                - hkg: 307ms   (Hong Kong)
                - headscale:         (Headscale Embedded DERP)
```

**关键字段解读**：

| 字段 | 含义 |
|------|------|
| `UDP: true/false` | 是否能发送 UDP 包，false 则无法直连 |
| `MappingVariesByDestIP` | **true = 对称型 NAT，无法打洞；false = 锥型 NAT，可打洞** |
| `Nearest DERP` | 当前最近的中继节点 |
| `headscale: (空)` | 内置 DERP 不可用 |

### 1.3 实时检测与对端的连接方式

```bash
tailscale ping <对端节点名或IP>
```

输出会明确显示走中继还是直连，以及延迟。多次执行可观察是否从 `via DERP` 变为直连。

---

## 2. 点对点直连失败排查

### 2.1 排查流程

```
两端都执行 tailscale netcheck
        │
        ├── MappingVariesByDestIP: true（任一端）──→ 对称型 NAT，无法打洞
        │       │
        │       ├── 换网络（家庭宽带）
        │       ├── 路由器端口映射 UDP 41641
        │       └── 只能走中继，优化中继延迟
        │
        └── MappingVariesByDestIP: false（两端）──→ 检查防火墙
                │
                ├── UDP: false ──→ 防火墙阻止了 UDP，开放 UDP 出站
                └── UDP: true ──→ 等待几分钟，Tailscale 后台打洞需要时间
```

### 2.2 实际案例

**问题**：goodeep（Linux）与 work-mac（macOS）之间无法建立点对点直连，走了东京中继。

**goodeep 端 netcheck**：

```
* UDP: true
* IPv4: yes, 125.121.239.18:35849
* MappingVariesByDestIP: false          ← 锥型 NAT，可以打洞
* Nearest DERP: San Francisco
```

**work-mac 端 netcheck**：

```
* UDP: true
* IPv4: yes, 39.144.124.175:50830
* MappingVariesByDestIP: true           ← 对称型 NAT，无法打洞！
* Nearest DERP: Hong Kong
```

**结论**：work-mac 的 IP `39.144.x.x` 属于中国移动网络，运营商采用对称型 NAT，导致无法打洞。两端只要有一端是对称型 NAT，就无法建立点对点直连。

---

## 3. NAT 类型详解

### 3.1 锥型 NAT（可打洞）

内部程序用同一个内网端口发往**不同目标**时，NAT 映射出的**公网端口固定不变**：

```
内网 10.0.0.x:12345 → 服务器A  → 映射为 公网IP:50830
内网 10.0.0.x:12345 → 服务器B  → 映射为 公网IP:50830  ← 端口不变
```

对方知道你的公网端口后，直接发包过来就能收到，打洞成功。

### 3.2 对称型 NAT（不可打洞）

内部程序用同一个内网端口发往**不同目标**时，NAT 会映射出**不同的公网端口**：

```
内网 10.0.0.x:12345 → 服务器A  → 映射为 公网IP:50830
内网 10.0.0.x:12345 → 服务器B  → 映射为 公网IP:50831  ← 端口变了！
```

打洞时，A 通过中继观察到 B 映射给自己的端口是 `50830`，但 A 发包给 B 时，B 的 NAT 认为这是"新目标"，会映射成另一个端口，A 发往 `50830` 的包被丢弃。双方互相猜不到对方的端口，打洞必然失败。

### 3.3 常见场景 NAT 类型

| 场景 | 大概率 NAT 类型 | MappingVariesByDestIP |
|------|---------------|----------------------|
| 家庭宽带 + 公网 IP | 锥型 | false |
| 家庭宽带 + CGNAT | 锥型偏多 | false |
| 手机热点 / 4G/5G | 对称型 | true |
| 公司/学校 Wi-Fi | 对称型偏多 | true |
| 云服务器（有公网IP） | 不经过 NAT，直连 | N/A |

---

## 4. 优化中继延迟

当无法建立点对点直连时，流量走 DERP 中继。优化目标是让中继延迟尽可能低。

### 4.1 方案对比

| 方案 | 难度 | 效果 | 说明 |
|------|------|------|------|
| 修复内置 DERP | 简单 | 中继走国内服务器，延迟 17ms | 阿里云服务器自带 DERP，需开放端口 |
| 换家庭宽带网络 | 中 | 可能点对点直连 | 脱离对称型 NAT |
| 路由器端口映射 | 中 | 可能点对点直连 | 映射 UDP 41641 到目标机器 |

### 4.2 修复内置 DERP

Headscale 配置了内置 DERP 服务器（`config.yaml` 中 `derp.server.enabled: true`），但客户端可能连不上。

#### 步骤一：确认服务端 DERP 正常

```bash
curl -s -o /dev/null -w "HTTP %{http_code}, time: %{time_total}s\n" https://headscale.hello-gpt.cn/derp/probe
curl -s -o /dev/null -w "HTTP %{http_code}, time: %{time_total}s\n" https://headscale.hello-gpt.cn/derp/latency-check
```

正常输出示例：

```
HTTP 200, time: 0.038367s
HTTP 200, time: 0.074077s
```

#### 步骤二：开放防火墙端口

内置 DERP 需要以下端口：

| 端口 | 协议 | 用途 |
|------|------|------|
| 443 | TCP | HTTPS + DERP（通过 Caddy 反向代理） |
| 80 | TCP | HTTP（ACME 证书申请） |
| 3478 | UDP | STUN（NAT 穿透探测） |

**阿里云**需要在控制台安全组规则中放行以上端口，服务器本机防火墙也需确认：

```bash
# 检查防火墙状态
ufw status

# 检查端口监听
ss -tlnp | grep -E ":(443|80|8080|3478) "
ss -ulnp | grep -E ":(3478|41641) "
```

正常输出示例：

```
LISTEN  0  4096  *:80    *:*  users:(("caddy",...))
LISTEN  0  4096  *:443   *:*  users:(("caddy",...))
UNCONN  0  0     0.0.0.0:3478  0.0.0.0:*  users:(("docker-proxy",...))
```

#### 步骤三：客户端重新检测

开放端口后，在两端重启 Tailscale 并重新 netcheck：

```bash
# goodeep 端
sudo tailscale down && sudo tailscale up --login-server https://headscale.hello-gpt.cn --accept-routes --advertise-routes=10.0.0.0/16
tailscale netcheck

# work-mac 端
sudo tailscale down && sudo tailscale up --login-server https://headscale.hello-gpt.cn
tailscale netcheck
```

#### 修复前后对比

**修复前**（netcheck 输出，内置 DERP 不可用）：

```
* Nearest DERP: San Francisco
* DERP latency:
        - sfo: 159ms   (San Francisco)
        - tok: 271.8ms (Tokyo)
        - hkg: 307ms   (Hong Kong)
        - headscale:         (Headscale Embedded DERP)  ← 延迟为空，不可用
```

**修复后**（阿里云安全组放行 443/3478 后）：

```
* Nearest DERP: Headscale Embedded DERP
* DERP latency:
        - headscale: 17ms    (Headscale Embedded DERP)  ← 生效，延迟仅 17ms
        - lax: 174.5ms (Los Angeles)
        - sfo: 184.2ms (San Francisco)
        - hkg: 245.6ms (Hong Kong)
        - tok: 265.6ms (Tokyo)
```

### 4.3 重启连接使中继生效

修复后如果 ping 延迟仍然很高，说明旧连接还在走之前的东京中继，需要重启 Tailscale 重新建立连接：

```bash
# goodeep 端
sudo tailscale down && sudo tailscale up --login-server https://headscale.hello-gpt.cn --accept-routes --advertise-routes=10.0.0.0/16

# work-mac 端
sudo tailscale down && sudo tailscale up --login-server https://headscale.hello-gpt.cn
```

重启后延迟从 300ms+ 降到约 30-40ms（两端到服务器各 17ms）。

---

## 5. 常见问题汇总

### Q1: tailscale status 显示 relay，为什么不是 direct？

说明两端之间无法建立点对点直连，常见原因：

1. **对称型 NAT**：任一端 `MappingVariesByDestIP: true` 则无法打洞
2. **防火墙阻止 UDP**：`netcheck` 显示 `UDP: false`
3. **打洞尚未完成**：Tailscale 先走中继，后台尝试打洞，可能需要几分钟

### Q2: 内置 DERP 显示延迟为空，如何修复？

通常是防火墙/安全组未放行 443 端口。参考 [4.2 修复内置 DERP](#42-修复内置-derp)。

### Q3: 修复 DERP 后 ping 延迟仍然很高？

旧连接仍走之前的中继，需重启 Tailscale 重新建立连接。参考 [4.3 重启连接使中继生效](#43-重启连接使中继生效)。

### Q4: 如何让对称型 NAT 的节点建立直连？

- **方案一**：换到家庭宽带网络（大概率变成锥型 NAT）
- **方案二**：在路由器上做端口映射，将 UDP 41641 转发到该节点内网 IP
- **方案三**：无法直连时，优化中继延迟（修复内置 DERP）

### Q5: macOS 上 tailscale 命令在哪里？

```bash
# GUI 版安装的
/Applications/Tailscale.app/Contents/MacOS/Tailscale

# 命令行版本
which tailscale
# 或
/usr/local/bin/tailscale
```

### Q6: DERP 中继需要开放哪些端口？

| 端口 | 协议 | 用途 |
|------|------|------|
| 443 | TCP | HTTPS + DERP |
| 80 | TCP | HTTP（ACME 证书申请） |
| 3478 | UDP | STUN（NAT 穿透探测） |

云服务器需同时检查**安全组规则**和**系统防火墙**。
