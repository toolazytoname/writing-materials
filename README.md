# writing-materials

写作素材库：技术方案、部署实录、踩坑记录。每篇素材都是一个可直接改写成文章的完整技术方案。

## 素材索引

| 日期 | 素材 | 一句话 |
|---|---|---|
| 2026-07-21 | [口袋里的 Claude Code：自托管移动开发环境解决方案](docs/happy-claude-mobile-solution.md) | 中继 + Claude Code + Happy + 手机,任意网络用手机遥控多台机器上的 AI 编程;含未备案域名 HTTPS 绕行、v3 兼容补丁等干货 |
| 2026-07-21 | [Tailnet 脑洞玩法大全 — 从自嗨到挣钱](docs/tailnet-brainstorm.html) | 单页 HTML:Tailscale 网络的玩法合集,从个人自用到变现思路 |
| 2026-07-21 | [个人云笔记系统方案](docs/个人笔记系统方案.md) | 以本地 Markdown 为核心的多端同步笔记方案:手机流畅查看/搜索 + 移动端 AI,含已拍板决策与待办清单 |
| 2026-07-22 | [无浏览器服务器如何完成 Codex OAuth 登录](docs/cliproxyapi-codex-oauth-flow-zh.md) | CLIProxyAPI 远程认证全流程图解:OAuth 回调接收器 + SSH 本地端口转发,浏览器登录与服务器令牌分离,基于终端日志 + 源码分析 |

## 配套仓库

- [happy-relay-deploy](https://github.com/toolazytoname/happy-relay-deploy) —— 上述方案沉淀的可复用 skill
