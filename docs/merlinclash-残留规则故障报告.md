# MerlinClash 插件残留 iptables 规则导致境外网络不通 — 故障报告

> 排查日期：2026-07-23
> 设备：ASUS RT-AC66U B1（Koolshare 梅林固件，SSH 别名 router_66ub1，LAN 网关 192.168.50.1）

---

## 一、故障现象

- 家里 192.168.50.x 网段下的所有设备（Mac、NAS 等）**访问境外 IP/网站失败**，典型表现为 `connection refused`（连接被拒绝）或超时。
- 具体受影响案例：
  - Tailscale 控制平面 `controlplane.tailscale.com` / `login.tailscale.com`（192.200.0.x:443）连接被拒，导致 Mac 上 Tailscale 长期 offline，NAS 无法安装登录 Tailscale。
  - 其他境外站点同样不通；改用上一级运营商路由器的网络则一切正常。
- 插件管理页面（`http://192.168.50.1/Module_merlinclash.asp`）显示 MerlinClash **已是关闭状态**，但拦截仍在发生 —— 困扰用户很久。

## 二、根本原因

**MerlinClash 插件在关闭时没有清理它运行期间注入的 iptables 流量劫持规则（插件 bug）。**

插件工作时通过透明代理方式劫持流量：

1. 在 nat 表 `PREROUTING` 链插入跳转规则，把局域网设备的**全部 TCP 流量**送入自定义链 `merlinclash`；
2. 在 nat 表 `OUTPUT` 链插入 3 条按 fwmark 匹配的跳转规则，送入 `merlinclash_EXT`；
3. 这些自定义链按 ipset（`direct_list` 直连白名单 / `ipset_proxyarround` / `merlinclash_CHN` 大陆路由表等）分流，命中"需要代理"的流量被 `REDIRECT` 到本机透明代理端口，由 Clash 核心进程接管。

用户后来在插件页面点击了"关闭"，插件配置标志已变为 `merlinclash_enable=0`：

```
# dbus get merlinclash_enable
0
```

**但上述 iptables 规则和自定义链没有被删除。** 而 Clash 核心进程已停止运行，透明代理端口没有任何进程监听。于是：

```
局域网设备 → TCP 连接境外 IP → 被残留规则 REDIRECT 到本机代理端口 → 无人监听 → 内核直接回 RST → 客户端收到 "connection refused"
```

这解释了为什么故障表现为"拒绝"而非"超时"，也解释了为什么插件界面显示关闭、拦截却依然存在。

## 三、诊断过程（证据链）

1. **NAS 无法安装 Tailscale**：`curl https://pkgs.tailscale.com` / `login.tailscale.com` 均失败。
2. **排除 DNS 污染**：`dig A login.tailscale.com` 经系统 DNS / 阿里 DNS / DoH 均正常返回 192.200.0.x。
3. **排除 GFW / 运营商封锁**：上一级路由器网络下访问相同站点正常；且失败表现为 RST（refused），不像 GFW 常见的丢包/重置特征。
4. **排除 VPS 问题**：自购 VPS（67.216.196.122）全端口 refused 是独立事件（VPS 本身宕机），与此故障无关。
5. **登路由器检查**：
   - 发现 koolshare 插件目录存在 `N99merlinclash.sh` / `S99merlinclash.sh`；
   - `iptables -t nat -L -n` 发现 `PREROUTING` 第 6 条：`merlinclash  tcp -- anywhere anywhere` —— **全部 TCP 被劫持**；
   - `merlinclash` 链 → `merlinclash_CHN` 继续分流；
   - 但 `ps` 中**没有任何 Clash 核心进程**，`netstat -tlnp` 也**没有透明代理端口在监听**；
   - `dbus get merlinclash_enable` = `0`（插件确实已关）；
   - 无看门狗 cron（`cru l` 无 clash 相关任务），规则不会自动重建。
6. **结论**：这是插件停用流程未清理防火墙规则的残留 bug。规则是"僵尸"状态——插件已死，劫持仍在。

## 四、修复方法（已执行）

在路由器上执行（SSH）：

```bash
# 1. 删除 nat PREROUTING 中的总劫持跳转（当时是第 6 条，删前请用 -L --line-numbers 确认）
iptables -t nat -D PREROUTING 6

# 2. 删除 nat OUTPUT 中 3 条 fwmark 跳转（每删一条序号前移，连续删 3 次第 1 条）
iptables -t nat -D OUTPUT 1
iptables -t nat -D OUTPUT 1
iptables -t nat -D OUTPUT 1

# 3. 清空并删除 4 条自定义链
iptables -t nat -F merlinclash
iptables -t nat -F merlinclash_CHN
iptables -t nat -F merlinclash_EXT
iptables -t nat -F merlinclash_NOR
iptables -t nat -X merlinclash
iptables -t nat -X merlinclash_CHN
iptables -t nat -X merlinclash_EXT
iptables -t nat -X merlinclash_NOR
```

> 注意：只删了插件注入的劫持规则，未改动路由器其他防火墙/NAT 配置，也未卸载插件本身。

## 五、验证结果

- NAS → `https://controlplane.tailscale.com/derpmap/default`：**HTTP 200** ✅
- Mac → `https://login.tailscale.com`：**HTTP 302**（正常跳转）✅
- NAS 上 `tailscale up` 成功拿到认证链接，可正常登录 ✅

## 六、如何检查是否复发

如果以后再次出现"境外网站打不开、但插件显示已关闭"，SSH 登路由器执行：

```bash
# 快速自检：有输出说明残留规则又出现了
iptables -t nat -L -n | grep merlinclash

# 确认插件状态
dbus get merlinclash_enable      # 0 = 已关闭；若为 0 但上一条有输出，即为本 bug 复发

# 确认代理核心是否在跑（没有进程却有劫持规则 = 故障状态）
ps w | grep -i clash | grep -v grep
```

若复发，重复第四节命令清理即可；也可以考虑在插件页面重新开启再正常关闭一次，观察它是否能正确自清（不同版本行为可能不同），或彻底卸载插件：

```bash
/koolshare/scripts/uninstall_merlinclash.sh   # 如需彻底卸载（会删除配置，谨慎）
```

## 七、附：本次排查发现的家庭网络拓扑（备忘）

```
互联网
  └── 运营商光猫/路由器（顶层，192.168.1.1，日常主力网络）
        └── RT-AC66U B1（Koolshare 梅林，66UB1，WAN 侧 192.168.1.2，LAN 192.168.50.1）
              ├── WiFi：66UB1 / 66UB1_5G（Mac、手机等直连）
              └── RT-AC66U-B6D8（第二台，192.168.50.50，无线桥接/MAC-NAT 模式）
                    └── HP 台式机 NAS（Debian 12，主机名 hp，192.168.50.225）
```

- NAS 的有线网卡正常，它接在**第二台桥接路由器**下面；桥接器做 MAC 地址转换，所以主路由设备列表里看不到 NAS 的真身 MAC，DHCP 租约（hp）与 ARP 表 MAC 不一致属该模式的正常现象。
- NAS SSH（已在 NAS 上装好 Mac 的公钥，免密）：

```bash
ssh -o ProxyCommand="ssh router_66ub1 nc %h %p" snail@192.168.50.225
```

- 已安装 Tailscale 1.98.9（静态包手动安装，systemd 服务已启用）。

## 八、遗留事项（截至 2026-07-23）

1. **Mac 到同网段设备单向不通**：Mac（192.168.50.58，5GHz）能通网关，但 ARP 广播始终发不到任何内网客户端（反向 NAS→Mac 正常）。重启路由器无线、关闭 Mac 上 Tailscale 均无效，怀疑 macOS 网络栈异常，建议重启 Mac 后再测。
2. **VPS 宕机**：67.216.196.122 的 22/443/8443 全部 connection refused，Reality 节点不可用，需去服务商后台处理。
