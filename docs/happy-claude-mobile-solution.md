# 口袋里的 Claude Code：自托管移动开发环境解决方案

> 一套「中继服务 + Claude Code + Happy + 手机」的完整方案，让你在任何网络环境下用手机实时指挥任意多台机器上的 Claude Code。
> 部署日期：2026-07-21 ｜ 状态：生产运行中

---

## 一、方案概览

### 核心理念

Happy 的架构是「星型」的：**一个中继服务（Relay）作为中心，所有跑 Claude Code 的工作机和所有手机/平板客户端都连到它**。中继只转发端到端加密的消息（零知识设计，服务器读不到内容），因此它可以放在任何一台 7×24 在线的机器上。

```
                        ┌─────────────────────────────┐
                        │   公网 VPS (腾讯云·北京)       │
                        │                             │
   手机 (4G/WiFi)  ─────┤  Caddy :8443 (HTTPS 入口)    ├─────  Mac (家里)
   Happy App            │    │                        │       happy CLI + Claude Code
                        │    ├─ happy.* → 中继:3005    │
   Linux 服务器  ───────┤    └─ opencode.* → :8000    ├─────  Linux 服务器
   happy CLI            │                             │       happy CLI + Claude Code
                        └─────────────────────────────┘
```

### 为什么优雅

- **手机无需 VPN**：中继在公网，4G/任何 WiFi 直连，iOS 不会杀后台断连
- **工作机零网络配置**：家里/公司 NAT 后的机器主动 outbound 连中继，不需要公网 IP、不需要端口转发
- **隐私不打折**：端到端加密（Signal 同款思路），中继被攻破也读不到内容
- **全开源、零订阅费**：Happy（MIT）+ happy-server-light，服务器是自己的
- **一域名管所有服务**：泛域名证书 + Caddy 反代，加新服务只需加一段配置

---

## 二、组件清单

| 组件 | 角色 | 位置 |
|---|---|---|
| **happy-server-light** | 中继服务（SQLite 单文件，无 Postgres/Redis） | VPS `/opt/happy-server-light` |
| **Caddy** | HTTPS 入口 + 按域名分流反代 | VPS Docker 容器 |
| **acme.sh** | Let's Encrypt 泛域名证书签发与自动续期 | VPS `/root/.acme.sh` |
| **systemd** | 中继进程守护（开机自启、崩溃拉起） | VPS |
| **ufw + fail2ban** | 防火墙 + 防爆破 | VPS |
| **happy CLI** | 工作机上的 Claude Code 包装器 | 各工作机（`npm i -g happy`） |
| **Happy App** | 手机客户端（iOS/Android） | 手机 |

### 关键资源

- 中继地址：`https://happy.子域.你的域名:8443`
- 证书：泛域名证书（Let's Encrypt，acme.sh 每天检查，到期前 30 天自动续期）
- 服务器：任意 7×24 在线的 VPS（1C/1G 即过剩）

### 轻量级 vs 重量级中继

| | 官方 happy-server（重量级） | happy-server-light（轻量级） |
|---|---|---|
| 依赖 | PostgreSQL + Redis（+ S3/MinIO） | SQLite 单文件，零依赖 |
| 设计目标 | 多人/团队/公有云（官方公共服务器跑的就是它） | 个人/小团队自托管 |
| 扩展性 | 可水平扩展 | 单实例 |

个人使用选轻量级；给几十上百人的团队提供服务、需要多实例高可用时选重量级。

---

## 三、网络与安全设计（重要决策记录）

### 为什么是 8443 而不是 443？

服务器在**中国大陆机房**，域名**未做 ICP 备案**。云厂商会在机房入口检查 80/443 端口 Web 流量中的域名，未备案即拦截。**非标端口（8443）不受此限制**。

### 为什么证书用 DNS 验证？

常规 HTTP-01 验证需要 Let's Encrypt 访问服务器的 80 端口——会被备案拦截。DNS-01 验证只要求证明「你能修改域名的 DNS 记录」，Let's Encrypt 直接查询公共 DNS，**完全不接触你的服务器**，绕行拦截。同时 DNS-01 是签发**泛域名证书**的唯一方式。

实现：acme.sh + DNS 服务商 API Key（如阿里云 RAM 子账号，仅 DNS 权限）。

### 中继暴露公网安全吗？

- **内容安全**：端到端加密，服务器只存加密数据块，任何人（包括服务器管理员）读不到
- **控制安全**：配对需扫码 + 手机密钥批准，攻击者无法冒充你的设备
- **残留风险**：Happy 采用密码学认证（无密码、无白名单），理论上任何人发现地址都能注册自己的账号「蹭中继」。缓解措施：非标端口 + 未公开域名（隐蔽性）、ufw 最小放行、fail2ban、日志审计
- **后备方案**：如需更高安全，可改 tailnet-only 模式（`tailscale serve`），代价是手机需挂 Tailscale

---

## 四、完整部署步骤（可复刻）

### 4.1 部署中继（含 v3 兼容性补丁）

> ⚠️ 重要：Happy App v1.2+ 使用 `/v3/sessions/:id/messages` 接口，上游 happy-server-light 只实现了 v1，会导致「会话列表正常、点进去无限 loading」。补丁已提交上游 PR（leeroybrun/happy-server-light#2），合并前请使用补丁分支或手动移植（见配套 skill 仓库）。

```bash
cd /opt && git clone --depth 1 https://github.com/leeroybrun/happy-server-light.git
cd happy-server-light && yarn install
node ./scripts/dev.mjs   # 首跑初始化 SQLite + 生成主密钥
```

systemd 守护（`/etc/systemd/system/happy-server.service`）：

```ini
[Unit]
Description=Happy Server Light (relay for Claude Code mobile)
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/happy-server-light
Environment=PATH=/opt/node/bin:/usr/local/bin:/usr/bin:/bin
Environment=NODE_ENV=production
Environment=PORT=3005
ExecStart=/opt/node/bin/node /opt/happy-server-light/node_modules/.bin/tsx /opt/happy-server-light/sources/main.ts
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 4.2 泛域名证书（acme.sh + DNS API）

```bash
curl https://get.acme.sh | sh -s email=你的邮箱
export Ali_Key="RAM子账号AccessKeyID" Ali_Secret="..."
~/.acme.sh/acme.sh --issue --dns dns_ali -d 子域.域名 -d "*.子域.域名" --server letsencrypt
# 自动装 cron 续期任务,无需人工干预
```

### 4.3 Caddy HTTPS 入口

```caddy
{
    admin off
}
happy.子域.域名:8443 {
    tls /etc/caddy/certs/happy.pem /etc/caddy/certs/happy.key
    reverse_proxy host.docker.internal:3005
}
# 其他自托管服务照此加块,如 opencode → :8000
```

### 4.4 防火墙（三层门）

1. **云平台安全组**：放行 TCP 8443（控制台操作，ufw 管不到这层）
2. **ufw**：`allow 22/tcp`、`allow 8443/tcp`、`allow in on docker0`（漏了 docker0 这条 Caddy 反代必 502）
3. **fail2ban**：默认 sshd jail

### 4.5 DNS 记录

| 主机记录 | 类型 | 值 |
|---|---|---|
| `子域` | A | 服务器 IP |
| `*.子域` | A | 服务器 IP（泛解析,新服务免加记录） |

---

## 五、客户端配置

### 工作机（Mac/Linux）

```bash
npm install -g happy
echo 'export HAPPY_SERVER_URL="https://happy.子域.域名:8443"' >> ~/.zshrc
source ~/.zshrc && happy auth    # 出二维码,手机扫码
cd 项目目录 && happy              # 代替 claude 启动
```

新增工作机：同样两条命令，手机 App 里自动出现新机器的会话。

### 手机 App

1. 设置/登录页的服务器地址（右上角数据库图标）填中继地址
2. 扫工作机 `happy auth` 的二维码配对
3. 无需 VPN，任意网络可用

---

## 六、日常维护

```bash
systemctl restart happy-server              # 重启中继
journalctl -u happy-server -f               # 实时日志
curl https://happy.子域.域名:8443/health     # 健康检查
~/.acme.sh/acme.sh --list                   # 证书续期状态
tar -czf happy-backup-$(date +%F).tar.gz /root/.happy/server-light/   # 备份
```

---

## 七、故障排查

| 症状 | 排查 |
|---|---|
| 手机 App 主界面红色错误 | ① 健康检查 ② `systemctl status happy-server` ③ 安全组 |
| `happy auth` 报 Failed to create authentication request | 终端 `echo $HAPPY_SERVER_URL` 是否为空 |
| 手机一直 Waiting for authentication | 手机 App 服务器地址没改成自建中继,批准发去了官方服务器 |
| 会话列表正常,点进去无限 loading | 服务端缺 v3 接口（见 4.1） |
| Caddy 502 | `ufw allow in on docker0` |
| 语音功能不可用 | 已知限制：App 写死官方 ElevenLabs agent ID（slopus/happy#472） |

---

## 八、已知限制

1. **语音功能不可用**，文字/推送/审批全部正常
2. **单点**：中继宕机 = 手机端失联（工作机本地 Claude Code 不受影响）
3. **中继公开注册**（见第三节）
4. **历史数据不迁移**：切换中继 = 全新账号命名空间

## 九、成本

| 项目 | 费用 |
|---|---|
| VPS | 约 ¥99/年档次即可 |
| 域名 | 约 ¥几十/年 |
| Happy 全家桶 + Let's Encrypt | 免费开源 |

---

*配套资源：[happy-relay-deploy skill](https://github.com/toolazytoname/happy-relay-deploy)（本方案的可复用 agent skill，含 v3 补丁源码）｜ 上游补丁 PR：[leeroybrun/happy-server-light#2](https://github.com/leeroybrun/happy-server-light/pull/2)*
