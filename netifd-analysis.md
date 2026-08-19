# netifd 代码分析

分析对象：`components/netifd`（25762 行 C 代码 + 3 个 shell 脚本库）
分析日期：2026-08-19

> **前提说明**
>
> **本仓库中的 `components/netifd` 就是 upstream 本身。** `origin` 指向官方仓库
> `https://github.com/openwrt/netifd.git`，`master` 与 `origin/master` 同点，
> 工作区干净（`git status --short` 无输出），共 1563 个提交。HEAD 为 `e97e36f`
> （2026-07-17，Felix Fietkau，`config: accept 'true' for the interface disabled
> option`）。因此修复的正确路径是直接改本仓库、按 OpenWrt 格式提交
> （`netifd: fix ...` / `bridge: fix ...` / `interface-ip: fix ...` + `git commit -s`），
> 再提 PR 或发到 openwrt-devel。**报告缺陷前应先 `git fetch`。**
>
> **注意版本落差。** 构建系统实际拉取的不是这个 checkout：
> `package/network/config/netifd/Makefile` 里
> `PKG_SOURCE_VERSION:=cbb83a1857407a28a63dc09412a1f209195914ef`（2026-02-26）。
> 该提交在本 checkout 的历史中，HEAD 比它**领先 125 个提交**。也就是说
> `components/netifd` 是一份比固件里跑的版本更新的源码树，读代码时以本树为准，
> 但排查线上问题时要先确认目标固件用的是哪个版本。
>
> **验证方式说明。** 本文**全部是逐行静态阅读的结论，没有做编译运行复现**，这一点
> 与本仓库的 `procd-analysis.md`（第五章有 ASan PoC）不同。第四章列出的每一条我都
> 回到源码核对过行号与上下文，并标注了确信度：
>
> - **【已核实】**：直接读到代码，结论无歧义（如某个函数只有声明没有定义）。
> - **【推断】**：基于代码结构的性能/设计判断，未做 profiling 佐证。
>
> 分析过程中子任务报告里有若干条经复核为**误报**，已剔除，不在本文中。典型的几条：
> `device.active` 被当成 `active_count`（实际字段名是 `active`）、
> notify action 的编号表（旧文档流传的 `3=available / 4=errors` 与当前源码不符，
> 正确表见 2.8）、`system_get_error`（本树不存在）、
> 订阅了 `RTNLGRP_IPV4_IFADDR`（实际只订阅了 `RTNLGRP_LINK`）。

---

## 目录

1. [代码架构](#一代码架构)
2. [设计原理](#二设计原理)
3. [使用方法](#三使用方法)
4. [可以优化的点](#四可以优化的点)
5. [快速索引](#五快速索引)

---

## 一、代码架构

### 1.1 netifd 是什么

netifd 是 OpenWrt 的网络配置守护进程，**唯一的产物就是 `/sbin/netifd` 一个二进制**
（对比 procd 切出了 11 个产物）。它的职责边界很清楚：

- **它负责**：把 `/etc/config/network` 的声明式配置翻译成内核状态（网卡、桥、VLAN、
  地址、路由、策略路由、邻居表），管理这些状态的生命周期，并把结果通过 ubus 暴露出去。
- **它不负责**：DHCP 客户端本身（udhcpc）、PPP 拨号（pppd）、无线关联（wpad/hostapd）、
  防火墙（fw4）。这些都是外部进程，netifd 只是**编排**它们。

这个边界决定了整个架构：netifd 的核心是一台**状态机 + 事件路由器**，真正干活的逻辑
大量地在进程外（shell 脚本、ucode 脚本、外部 ubus 服务）。

### 1.2 构建与依赖

`CMakeLists.txt` 只产出一个可执行文件：

```21:27:components/netifd/CMakeLists.txt
SET(SOURCES
	main.c utils.c system.c tunnel.c handler.c
	interface.c interface-ip.c interface-event.c
	iprule.c proto.c proto-static.c proto-shell.c proto-ext.c proto-ucode.c
	config.c device.c bridge.c veth.c vlan.c alias.c
	macvlan.c ubus.c vlandev.c extdev.c bonding.c
	vrf.c ucode.c)
```

依赖库（`package/network/config/netifd/Makefile` 的 `DEPENDS`）：

| 库 | 用途 |
|----|------|
| `libubox` | uloop 事件循环、blob/blobmsg 序列化、avl / vlist / safe_list / kvlist 容器 |
| `libubus` | ubus 对象注册与调用 |
| `libuci` | 读 `/etc/config/network` |
| `libnl-tiny` | rtnetlink / genetlink |
| `json-c` + `blobmsg_json` | 解析脚本 handler 的 JSON 元数据、格式化配置传给脚本 |
| `libudebug` | 环形缓冲日志 + netlink 抓包 |
| `ucode` (+ fs/ubus/uloop/uci 模块) | 嵌入式脚本运行时 |

两个编译期开关值得注意：

- **`DUMMY_MODE`**：非 Linux 或显式定义时，用 `system-dummy.c` 替换 `system-linux.c`，
  所有内核操作变成 `D(SYSTEM, "ifconfig ...")` 打印。同时 `netifd.h:38-43` 把路径
  全部改成相对路径（`./examples`、`./config`、`./tmp/resolv.conf`），让 netifd 能在
  开发机上跑完整的配置逻辑而不碰内核。
- **`ethtool-modes.h` 自动生成**：`make_ethtool_modes_h.sh` 用**目标编译器**预处理
  `<linux/ethtool.h>`，grep 出 `ETHTOOL_LINK_MODE_*base*_{Full,Half}_BIT`，生成
  速率/双工映射表。这样内核每加一种 link mode（2.5G/5G/10G/…），netifd 不用改代码。

### 1.3 分层结构

```
                    ┌──────────────────────────────────────┐
   外部世界 ────────►│  ubus.c        network / network.device        │
   (ubus 调用)      │                network.interface[.<name>]       │
                    └──────────────┬───────────────────────┘
                                   │
   /etc/config/network ──► config.c │  UCI section → blob_attr（两阶段加载）
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
  ┌───────────┐            ┌──────────────┐          ┌────────────────┐
  │ device 层 │◄──事件────►│ interface 层 │◄──命令──►│   proto 层     │
  │ device.c  │            │ interface.c  │          │ proto.c        │
  │ bridge/   │            │ interface-ip │          │ proto-static   │
  │ vlan/bond/│            │ interface-   │          │ proto-ext      │
  │ vrf/veth/ │            │   event.c    │          │ proto-shell    │
  │ macvlan/  │            │ iprule.c     │          │ proto-ucode    │
  │ alias/... │            └──────────────┘          └────────────────┘
  └─────┬─────┘                    │                          │
        │                          │                  fork+exec / ubus
        └──────────┬───────────────┘                          ▼
                   ▼                                 /lib/netifd/proto/*.sh
        ┌────────────────────┐                       udhcpc / pppd / ...
        │   system 层        │
        │ system.h 抽象接口  │
        │ system-linux.c     │  netlink + ioctl + sysfs/procfs
        │ system-dummy.c     │
        └─────────┬──────────┘
                  ▼
             Linux 内核
```

**横向的扩展面**（不在主链路上，但是 netifd 灵活性的来源）：

- `handler.c` — 通用脚本 handler 加载框架（扫目录 + `<script> '' dump` 取元数据）
- `extdev.c` — 外部设备类型：通过 ubus 让别的进程实现一种 `device_type`
- `ucode.c` / `proto-ucode.c` — 嵌入 ucode VM，无线管理和新协议走这条路

### 1.4 文件规模分布

| 文件 | 行数 | 角色 |
|------|------|------|
| `system-linux.c` | 5302 | 全部内核交互，占全仓 20% |
| `interface-ip.c` | 2026 | 地址 / 路由 / IPv6 PD / DNS / 邻居 |
| `interface.c` | 1695 | 接口状态机与配置差分 |
| `device.c` | 1659 | 设备抽象、引用计数、事件广播 |
| `bridge.c` | 1478 | 桥 + 桥 VLAN（最复杂的设备类型） |
| `extdev.c` | 1424 | 外部设备处理器代理 |
| `ubus.c` | 1331 | 三个 ubus 对象 + status dump |
| `config.c` | 854 | UCI → blob → vlist |
| `proto-ext.c` | 839 | shell/ucode 协议共用的状态机与 notify 处理 |
| `bonding.c` / `vrf.c` / `proto.c` / `ucode.c` | 760 / 671 / 696 / 605 | |
| 其余 24 个文件 | 各 < 500 | |

一个观察：**`system-linux.c` 一个文件顶得上其余所有设备类型加起来**。这是把"所有
脏活集中在一处"的刻意选择，代价是这个文件本身很难维护（见 4.4）。

### 1.5 三张核心注册表

netifd 的全局状态就是这几张表，理解它们等于理解了数据流：

| 表 | 类型 | 定义位置 | 内容 |
|----|------|----------|------|
| `devtypes` | `list_head` | `device.c:30` | 所有 `device_type` 虚表，`__init` 构造函数注册 |
| `devices` | `avl_tree` | `device.c:31` | 所有 `struct device`，key = ifname |
| `interfaces` | `vlist_tree` | `interface.c` | 所有 `struct interface`，key = 接口名 |
| `handlers` | `avl_tree` | `proto.c:29` | 所有 `proto_handler`，key = 协议名 |

设备类型的注册没有用宏，而是每个文件写一个 `__init` 函数：

| 类型名（UCI `option type`） | 文件 |
|------|------|
| `Network device`（simple） | `device.c` |
| `bridge` | `bridge.c` |
| `8021q` / `8021ad` | `vlandev.c` |
| `bonding` | `bonding.c` |
| `vrf` | `vrf.c` |
| `macvlan` | `macvlan.c` |
| `veth` | `veth.c` |
| `tunnel` | `tunnel.c` |

两个例外值得记住：**`vlan.c` 的点号 VLAN（`eth0.100`）用的是文件内的 `static`
类型，不进 `devtypes`**，只能通过 `__device_get()` 解析名字时隐式创建；
**`alias.c` 的 `@ifname` 别名设备同理**，由 `device_alias_get()` 直接创建。
所以 `device_type_get("VLAN")` 是找不到东西的。

---

## 二、设计原理

### 2.1 device 层：把"存在"和"启用"拆成两个维度

这是 netifd 最核心的抽象，也是它能优雅处理热插拔的原因。一个设备有两组独立状态：

```305:311:components/netifd/device.h
	bool sys_present;
	/* DEV_EVENT_ADD */
	bool present;
	/* DEV_EVENT_UP */
	int active;
```

- **`sys_present`** — 内核里这块网卡在不在。由 netlink `RTM_NEWLINK/DELLINK` 和
  hotplug uevent 驱动，netifd 只是观察者。
- **`present`** — 对上层用户宣告的"可用"。由 `device_refresh_present()` 计算：

```1019:1027:components/netifd/device.c
void device_refresh_present(struct device *dev)
{
	bool state = dev->sys_present;

	if (dev->disabled || dev->deferred)
		state = false;

	__device_set_present(dev, state, false);
}
```

  即 `present = sys_present && !disabled && !deferred`。`disabled` 来自 UCI
  `option enabled '0'`，`deferred` 来自 ubus `network.device set_state`（无线场景：
  wpad 还没准备好时先压住）。

- **`active`** — 这**不是 bool 而是引用计数**。多个 interface 可以同时用一块网卡
  （比如 `lan` 和 `lan6`），谁都可以 claim，最后一个 release 时才真正 down。

`device_claim` / `device_release` 是这套机制的全部：

```773:825:components/netifd/device.c
int device_claim(struct device_user *dep)
{
	...
	dep->claimed = true;
	if (++dev->active != 1)
		return 0;
	device_broadcast_event(dev, DEV_EVENT_SETUP);
	device_fill_default_settings(dev);
	ret = dev->set_state(dev, true);
	if (ret == 0)
		device_broadcast_event(dev, DEV_EVENT_UP);
	else {
		dev->active = 0;
		dep->claimed = false;
	}
	return ret;
}
```

注意 `claim` 失败时会把 `active` 回滚到 0 且不增加引用——`DESIGN` 文件里明确写了
"如果设备起不来，`claim_device()` 返回非零，调用方的引用计数不会增加"。

状态机可以画成：

```
sys_present=0 ──netlink/hotplug──► sys_present=1
                      │
                      ▼
        present = sys_present && !disabled && !deferred
                      │
              present 翻转 ──► DEV_EVENT_ADD / DEV_EVENT_REMOVE
                      │
              users 可以 device_claim()
                      │
   active: 0 ──claim──► 1   SETUP → set_state(true) → UP
   active: 1 ──claim──► N   （无事件，只加计数）
   active: N ─release──► 0   TEARDOWN → set_state(false) → DOWN

sys_present → 0 时：强制广播 REMOVE，并对所有 users 执行 device_release()
```

事件类型一共 13 种（`device.h:187-210`），最常用的是 ADD/REMOVE（present）、
SETUP/UP/TEARDOWN/DOWN（active）、LINK_UP/LINK_DOWN（carrier）、
AUTH_UP（802.1X）、TOPO_CHANGE（桥成员变化）。

### 2.2 device_user + safe_list：迭代中安全删除

任何关心设备状态的东西都注册一个 `device_user`：

```215:226:components/netifd/device.h
struct device_user {
	struct safe_list list;

	bool claimed;
	bool hotplug;
	bool alias;

	uint8_t ev_idx[__DEV_EVENT_MAX];

	struct device *dev;
	void (*cb)(struct device_user *, enum device_event);
};
```

用 libubox 的 `safe_list` 而不是普通 `list_head`，原因很具体：
`__device_broadcast_event()` 遍历 users 调回调时，回调里完全可能触发
`device_remove_user()` 甚至 `device_cleanup()`——普通链表会立刻悬空。
`safe_list` 的实现用迭代器链把"下一个节点"提前移走，保证遍历中删除任意节点都安全。

`device_add_user()` 还有一个"追赶"逻辑：如果设备**已经**是 present 的，新加入的
user 会立刻补发一次 ADD 事件，必要时再补 UP、LINK_UP。这样调用方不用区分
"我是先注册还是设备先出现"，写起来简单很多。

设备还维护了一条独立的 `aliases` 链表，广播事件时**先 aliases 再 users**，
保证 `@ifname` 别名设备先于接口更新。

### 2.3 配置差分：DEV_ATTR / DEV_OPT 位对齐

设备有 48 个可配置属性（`device.c:34-84` 的 `dev_attrs`）。netifd 用了一个很紧凑
的技巧把"哪些属性变了"和"哪些属性要下发"统一成同一个位图：

```125:132:components/netifd/device.h
/*
 * DEV_OPT_X is the bit in device_settings.flags that marks attribute X
 * as configured. Aligned with DEV_ATTR_X so the diff bitmap produced
 * by uci_blob_diff(&device_attr_list, ...) is directly usable as a
 * DEV_OPT_* mask.
 */
```

即 `DEV_OPT_MTU == 1ULL << DEV_ATTR_MTU`。`uci_blob_diff()` 产生的差分位图可以
**直接当作** apply mask 使用，不需要任何转换表。

在此之上还有一层"能否热应用"的划分。`DEV_ATTR_*` 枚举被刻意排序，把纯 per-netdev
sysctl 类属性放在最前面：

```31:58:components/netifd/device.h
enum {
	/*
	 * Live-applicable: pure per-netdev sysctls. A diff limited to these
	 * can be pushed via system_if_apply_settings() without tearing the
	 * device down. Keep this group first so DEV_OPT_LIVE_APPLY_MASK
	 * stays a contiguous low-bit mask.
	 */
	DEV_ATTR_IPV6,
	...
	DEV_ATTR_MULTICAST,
	__DEV_ATTR_LIVE_APPLY_MAX,

	/* Have a DEV_OPT_* flag but require teardown to re-apply. */
	DEV_ATTR_MTU = __DEV_ATTR_LIVE_APPLY_MAX,
	...
```

于是 `DEV_OPT_LIVE_APPLY_MASK` 就是一个连续低位掩码 `(1 << N) - 1`，
`device_diff_live_apply()` 只需一次按位与就能判断"这次改动能不能不断网热应用"。
改 `rp_filter` 不断网，改 `mtu` 要重启设备——这个区别就是这么实现的。

配置变更的处理结果分四级：

| 返回值 | 含义 | 处理动作 |
|--------|------|----------|
| `DEV_CONFIG_NO_CHANGE` | blob 完全相同 | 什么都不做 |
| `DEV_CONFIG_APPLIED` | 只涉及 live-apply 属性 | 设备 active 时直接 `device_apply_live_settings()` |
| `DEV_CONFIG_RESTART` | 需要 down/up | `present=false` → `set_state(false/true)` → `present=true` |
| `DEV_CONFIG_RECREATE` | 类型变了或无法热更新 | `device_create()` 删旧建新并迁移所有 users |

每种设备类型的 `reload()` 回调可以把结果**升级**（比如 bridge 发现 `ports` 变了
就返回 RESTART），也可以覆盖（tunnel 直接对全部属性返回 NO_CHANGE，因为隧道参数
只在创建时有意义）。

### 2.4 interface 层：状态机与 main_dev / l3_dev 分离

接口有两个正交的状态维度：

**运行状态** `enum interface_state`（`interface.h:35-40`）：
`IFS_SETUP` → `IFS_UP` → `IFS_TEARDOWN` → `IFS_DOWN`

**配置状态** `enum interface_config_state`（`interface.h:42-46`）：
`IFC_NORMAL` / `IFC_RELOAD` / `IFC_REMOVE`

裁决全部集中在 `interface_check_state()`：

```394:421:components/netifd/interface.c
interface_check_state(struct interface *iface)
{
	bool link_state = iface->link_state || interface_force_link(iface) ||
			  iface->carrier_loss_timer.pending;

	switch (iface->state) {
	case IFS_UP:
	case IFS_SETUP:
		if (!iface->enabled || !link_state) {
			...
			iface->state = IFS_TEARDOWN;
			...
			interface_proto_event(iface->proto, PROTO_CMD_TEARDOWN, false);
		}
		break;
	case IFS_DOWN:
		if (!iface->available)
			return;
		if (iface->autostart && iface->enabled && link_state &&
		    !config_init && iface->config_state != IFC_REMOVE)
			__interface_set_up(iface);
		break;
```

拉起接口需要**四个条件同时成立**：`available`（设备存在）、`autostart`（允许自启）、
`enabled`（设备已 claim 成功）、`link_state`（有 carrier）。注意
`carrier_loss_timer.pending` 也算有 link——这是 `carrier_loss_delay` 的实现，
网线抖动时不会立刻拆掉整个接口。

**main_dev 与 l3_dev 分离**是 netifd 支持 PPP/VPN 的关键：

```144:149:components/netifd/interface.h
	/* main interface that the interface is bound to */
	struct device_user main_dev;
	struct device_user ext_dev;

	/* interface that layer 3 communication will go through */
	struct device_user l3_dev;
```

默认 `l3_dev == main_dev`。PPPoE 场景下 `main_dev = eth0`（承载 L2 可用性与 carrier
判断），协议脚本通过 notify 把 `l3_dev` 改成 `pppoe-wan`（承载地址和默认路由）：

```380:394:components/netifd/proto-ext.c
	if (!keep) {
		dev = iface->main_dev.dev;
		if (tb[NOTIFY_IFNAME]) {
			...
			dev = device_get(devname, dev_create);
		}
		...
		interface_set_l3_dev(iface, dev);
```

对应地，`interface_l3_dev_cb()` 只在 `l3_dev != main_dev` 时才处理事件，且只关心
`DEV_EVENT_LINK_DOWN`（配合 `PROTO_FLAG_TEARDOWN_ON_L3_LINK_DOWN`）。L2 事件一律
以 main_dev 为准，避免两个设备重复驱动同一个状态机。

### 2.5 配置变更延迟到 DOWN 之后

这是一个很容易被忽略但很重要的设计。UCI reload 时，运行中的接口**不会**被立刻
改配置，而是先记一个 `config_state`，等协议真正 down 了再处理：

```1471:1478:components/netifd/interface.c
set_config_state(struct interface *iface, enum interface_config_state s)
{
	__set_config_state(iface, s);
	if (iface->state == IFS_DOWN)
		interface_handle_config_change(iface);
	else
		__interface_set_down(iface, false);
}
```

`interface_handle_config_change()` 在 `IFPEV_DOWN` 事件里被调用。这样保证了
"改配置"永远不会插到"正在 setup"的中间，避免半配置状态。

删除接口更谨慎，用了一个 1ms 定时器把 free 推迟到当前回调栈之外——因为
`vlist_delete` 很可能是在遍历 users 的回调里触发的。同理，`device_free_unused()`
也是 1ms 定时器批量回收。

配置差分本身很细致（`interface_change_config`，`interface.c:1531-1664`）：

- 触发**整接口 reload**：`device` 变、`proto` 变、协议自己的 config blob 变、
  设备配置变、`auto` 从开变关、`zone` 变、claim 后发现 `main_dev` 换了
- 只触发 **IP 层刷新**（不断协议）：`metric`、`defaultroute`、`ip4table`/`ip6table`
  变 → 走一次 disable/enable IP；`delegate` 变 → 只刷新前缀委派；DNS 列表就地合并

### 2.6 vlist 版本化差分：地址、路由、DNS 的统一模型

`interface-ip.c` 里所有资源（地址、路由、前缀、邻居、DNS）都用同一套模式管理：

```
interface_ip_update_start()
    → vlist_update()           # 版本号 +1，旧节点全部标记为"未更新"
  ... 协议脚本 / 静态配置重新 vlist_add() 所有当前有效的条目 ...
interface_ip_update_complete()
    → vlist_flush()            # 删掉本轮没被重新添加的节点
    → host_routes_refresh()
    → interface_write_resolv_conf()
```

vlist 的 update 回调负责决定"同一个 key 的新旧节点要不要真的重新下发"。以路由为例
（`__interface_update_route`），如果 nexthop / mtu / type / proto 都相同就直接
keep，不产生任何 netlink 消息。地址也一样，只有 flags / broadcast / ptp 变了才
del+add，只是 valid/preferred lifetime 变了就用 replace。

这套机制的好处是**协议脚本可以无脑地每次把完整状态推一遍**，netifd 自动算出最小
增量。DHCP 续租时脚本推的还是那一组地址路由，实际产生的内核操作是零。

### 2.7 IPv6 前缀委派：无类别 first-fit 分配

`interface_update_prefix_assignments()`（`interface-ip.c:1212-1350`）是全仓库算法性
最强的一段。上游接口拿到一个 PD 前缀（比如 `/56`）后，要分给下游各个接口：

1. 清空旧 assignment，放一个 sentinel 节点界定整个地址空间
2. 如果配了 `excl_*`（RFC 6603 排除前缀），插入一个 `!excluded` 占位节点
3. 遍历所有配了 `ip6assign`（取值 48~64）的接口，用 `ip6class` 过滤前缀类别
4. **有 hint 的接口优先**（`ip6hint`）：尝试在指定位置精确插入；失败则退化为自动分配
5. 其余接口进 AVL 排序（按 length 升序、weight 降序、名字），再逐个 first-fit：
   对齐到 `2^(64-len)` 边界找第一个空洞，找不到就 `length++` 重试直到 `/64`
6. `interface_set_prefix_address()` 按 `ip6ifaceid` 生成 IID（固定 / 随机 / EUI-64），
   地址打 `DEVADDR_OFFLINK` 标记，再加 `/64` 路由和源地址策略规则

"无类别"（classless）的意思是它不限于按 `/64` 切分，可以分出 `/60`、`/62` 这种，
所以一个 `/56` 能塞下比 256 个 `/64` 更灵活的组合。

配套的源地址策略路由：地址级规则 priority `10000`（精确源地址）和 `20000`（前缀），
上游接口还会加 `IPRULE_PRIORITY_NW` 的 reject 规则，防止 PD 前缀内的流量在上游
断开后错误地走默认路由漏出去。

### 2.8 proto 层：同步与异步两条路径

协议处理器只需实现三件事：响应 `PROTO_CMD_*`、回送 `IFPEV_*` 事件、清理自己。

**简单协议走同步路径。** 打上 `PROTO_FLAG_IMMEDIATE` 后，核心代码自动发事件：

```667:696:components/netifd/proto.c
	ret = proto->cb(proto, cmd, force);
	if (ret || !(proto->handler->flags & PROTO_FLAG_IMMEDIATE))
		goto out;
	switch(cmd) {
	case PROTO_CMD_SETUP:   ev = IFPEV_UP; break;
	case PROTO_CMD_TEARDOWN: ev = IFPEV_DOWN; break;
	case PROTO_CMD_RENEW:
	case PROTO_CMD_RESTART:  ev = IFPEV_RENEW; break;
	...
	proto->proto_event(proto, ev);
```

`static` 和 `none` 是仅有的两个内建协议，都是 IMMEDIATE 的。`none` 甚至不进 AVL 树，
`get_proto_handler("none")` 直接返回一个静态对象。

**复杂协议走异步路径**，状态机在 `proto-ext.c`，shell 和 ucode 协议**共用**这一套：

| 状态 | 含义 |
|------|------|
| `S_IDLE` | 空闲，或已经 UP |
| `S_SETUP` | setup 脚本运行中 |
| `S_SETUP_ABORT` | setup 途中收到 teardown：SIGTERM + 1 秒超时后转 TEARDOWN |
| `S_TEARDOWN` | teardown 中，5 秒超时；完成后发 `IFPEV_DOWN` |

同时管理**两个进程**：`script_task`（协议脚本本身，短生命周期）和 `proto_task`
（脚本通过 `proto_run_command` 拉起的长驻进程，如 udhcpc / pppd）。

脚本的调用形式是：

```
<script> <proto_name> <action> <iface_name> <config_json> [ifname]
```

`action` ∈ `setup` | `teardown` | `renew` | `restart`。teardown 时若上次有错误，
还会传一个 `ERROR=<exitcode>` 环境变量。

**脚本回调 netifd 的唯一通道是 ubus `notify_proto`**，用一个整数 `action` 区分：

| action | shell API | C 处理函数 | 含义 |
|--------|-----------|-----------|------|
| **0** | `proto_send_update` | `proto_ext_update_link` | 更新 L3 设备、地址、路由、DNS、邻居、隧道；`link-up=1` → `IFPEV_UP`，`link-up=0` → `IFPEV_LINK_LOST` |
| **1** | `proto_run_command` | `proto_ext_run_command` | 启动 `proto_task`（可带 env） |
| **2** | `proto_kill_command` | `proto_ext_kill_command` | 给 `proto_task` 发信号（默认 SIGTERM） |
| **3** | `proto_notify_error` | `proto_ext_notify_error` | 记录错误，status 里的 `errors` 字段 |
| **4** | `proto_block_restart` | `proto_ext_block_restart` | 置 `autostart=false`，阻止自动重试 |
| **5** | `proto_set_available` | `proto_ext_set_available` | 显式设置接口可用性 |
| **6** | `proto_add_host_dependency` | `proto_ext_add_host_dependency` | 注册主机可达性依赖 |
| **7** | `proto_setup_failed` | `proto_ext_setup_failed` | 宣告 setup 失败，触发 teardown |

（这张表以 `proto-ext.c:634-653` 的 `switch` 为准。网上流传的旧版编号表与当前源码
不一致，直接照抄会出错。）

### 2.9 host dependency：让 VPN 跟随底层可达性

`proto_add_host_dependency` 解决的是一个经典问题：VPN 隧道建到 `1.2.3.4`，如果
VPN 自己装了默认路由，到 `1.2.3.4` 的流量就会被自己的隧道吞掉。

netifd 的做法（`proto_ext_add_host_dependency`，`proto-ext.c:564-598`）：

1. 调 `interface_ip_add_target_route()` 在现有路由表里找"谁能到这个 IP"
2. 给那条路由的宿主接口装一个主机路由（`/32` 或 `/128`），nexthop / mtu / metric
   / table 全部继承自匹配到的路由
3. 在依赖接口上注册 user，一旦它 DOWN 就发 `IFPEV_LINK_LOST` 拆掉 VPN
4. 所有依赖都满足才 `interface_set_available(true)`

这条主机路由会随着底层接口的路由变化自动同步（HEAD 附近的提交
`9b62f2a interface-ip: keep host routes in sync with their parent route` 正是修这个）。

### 2.10 system 层：netlink / ioctl / sysfs 三管齐下

`system.h` 定义了约 60 个 `system_*` 抽象接口，Linux 实现全在 `system-linux.c`。
不同操作用不同机制，这是历史演进留下的：

| 操作 | 机制 |
|------|------|
| 建/删桥、macvlan、veth、vlandev、GRE/VTI/XFRM/VXLAN 隧道 | **netlink** `RTM_NEWLINK` |
| 桥成员增删 | **ioctl** `SIOCBRADDIF` / `SIOCBRDELIF` |
| 桥端口参数（learning、isolate…） | **sysfs** `brport/*` |
| bonding 全部操作 | **纯 sysfs** `bonding_masters`、`bonding/*` |
| 旧式 VLAN | **ioctl** `SIOCSIFVLAN` |
| sit / ipip / 6rd 隧道 | **ioctl** `SIOCADDTUNNEL` / `SIOCADD6RD` |
| 接口 up/down | **ioctl** `SIOCSIFFLAGS` |
| 地址、路由、iprule、邻居 | **netlink** |
| per-device sysctl | **procfs** `/proc/sys/net/{ipv4,ipv6}/{conf,neigh}/<if>/*` |
| link speed / duplex / pause / EEE / GRO | **ioctl** ethtool |
| PSE（PoE 供电） | **genetlink** ethtool netlink |
| 接口统计 | **sysfs** `/sys/class/net/<if>/statistics/*` 逐文件读 |

三个 socket 各司其职：`sock_rtnl` 做同步请求-应答，`rtnl_event.sock` 收异步广播，
`sock_ioctl` 做 ioctl。分开的好处是 dump 操作的 ack 序列不会被异步事件打断。

**一个重要的事实：netifd 只订阅了 `RTNLGRP_LINK`**：

```379:379:components/netifd/system-linux.c
	nl_socket_add_membership(rtnl_event.sock, RTNLGRP_LINK);
```

它**不监听地址和路由的变化**。也就是说别人用 `ip addr add` 手工加的地址，netifd
完全不知道。这是刻意的单向模型：netifd 认为自己是配置的唯一权威，内核状态应该
镜像它的内部模型，而不是反过来。启动时的 `system_if_clear_state()` 会把接口上的
残留地址/路由/邻居全部 dump 出来删掉，正是这个假设的体现。

启动清理里有个巧思：dump 出来的 `RTM_NEW*` 消息不能立刻转成 `RTM_DEL*` 发出去，
否则错误应答会打断正在进行的 dump。代码把它们**排进队列**，等 dump 结束再统一
`system_rtnl_call()`。

### 2.11 事件出口：hotplug 串行队列 + ubus 通知

接口状态变化有两个出口，都在 `interface-event.c`：

```126:129:components/netifd/interface-event.c
	if (ev == IFEV_UP || ev == IFEV_DOWN)
		netifd_ubus_interface_event(iface, ev == IFEV_UP);

	netifd_ubus_interface_notify(iface, ev != IFEV_DOWN);
```

- **ubus 广播事件**：`ubus_send_event("network.interface", {action: ifup|ifdown, interface: name})`
- **ubus 对象通知**：向 `network.interface` 和 `network.interface.<name>` 发
  `interface.update` / `interface.down`，payload 是完整 status

- **hotplug 脚本**：fork + `execl("/sbin/hotplug-call", "iface")`，环境变量
  `ACTION` / `INTERFACE` / `DEVICE`（+ ifupdate 时的 `IFUPDATE_ADDRESSES` 等）

hotplug 这条路径是**严格串行**的：全局只有一个 `current` 在跑，每个接口最多再排
一个 pending，并且做事件合并（正在跑同一事件就不重复排队）。这避免了脚本重入
和事件风暴，代价是慢（见 4.3）。

事件类型：`ifdown` / `ifup` / `ifup-failed` / `ifupdate` / `free` / `reload` /
`iflink` / `create`。其中 `iflink` 只发 ubus 不跑 hotplug，`reload` 和 `create`
不进队列。

### 2.12 两阶段配置加载

`config_init_all()`（`config.c:814-853`）的顺序是精心安排的：

```
config_init_package("network")   # 读 UCI
config_init_board()              # 读 /etc/board.json（默认 MAC / GRO / conduit）
vlist_update(&interfaces)        # 打开差分窗口
config_init = true               ← 关键标志
  device_reset_config()          # 所有设备 current_config = false
  config_load_vlans()            # 先收集 bridge-vlan 配置
  config_init_devices(true)      # 先建 bridge 类设备
  config_init_vlans()            # 应用 VLAN 到 bridge
  config_init_devices(false)     # 再建其余设备
  netifd_ucode_config_load(false)# ucode config_init
  config_init_interfaces()       # interface + alias
  config_init_ip()               # route / route6 / neighbor
  config_init_rules()            # rule / rule6
  config_init_globals()          # ula_prefix / tcp_l3mdev / udp_l3mdev
config_init = false
  device_reset_old()             # 清掉不再被引用的设备
  device_init_pending()          # 跑推迟的 config_init 回调
vlist_flush(&interfaces)         # 删掉不再存在的接口
interface_refresh_assignments()  # 重算 IPv6 PD
interface_start_pending()        # 真正拉起接口
netifd_ucode_config_load(true)   # ucode config_start
```

`config_init` 这个全局布尔的作用是**在第一阶段禁止任何接口自动拉起**
（`interface_check_state` 里显式检查 `!config_init`）。这样保证所有设备和接口
都建好、依赖关系都建立完之后，才统一开始拉起，避免"接口 A 起来时接口 B 依赖的
桥还没建好"这类顺序问题。

设备类型的 `config_init()` 回调同样被推迟到 `device_init_pending()`，理由一样。

### 2.13 三种扩展机制

netifd 的可扩展性设计得相当彻底，一共三条路：

**(1) shell 脚本协议** — 最传统。`/lib/netifd/proto/*.sh`，启动时
`netifd_init_script_handlers()` 扫目录，对每个脚本执行 `<script> '' dump` 拿到
JSON 元数据（协议名、配置项列表、各种 flag）。运行时 fork+exec，脚本通过
`netifd-proto.sh` 提供的 `proto_*` 函数调 ubus 回来。

**(2) ucode 协议** — 较新。`ucode.c` 在 netifd 进程内嵌了一个 ucode VM，注册了
一批原生函数（`log`、`process`、`device_set`、`interface_get_enabled`、
`interface_handle_link`、`interface_get_bridge`、`add_proto`）。ucode 协议通过
`add_proto()` 注册，运行时仍走 `proto-ext.c` 那套状态机和 notify 协议——只是
把 shell 换成了 ucode 解释器。无线管理（`/lib/netifd/main.uc`）已经完全迁到这条路。

**(3) 外部设备处理器（extdev）** — 最灵活。`/lib/netifd/extdev-config/*.json`
描述一种设备类型，netifd 据此**动态生成一个 `struct device_type` stub**，把
create / config_init / reload / free / dump_info / dump_stats / hotplug 全部
代理到一个外部 ubus 对象（`ubus_invoke` 超时 3 秒）。

extdev 有一套完整的生命周期管理：外部 handler 可以晚于 netifd 启动（监听
`ubus.object.add` 事件等它出现），出现后 `netifd_reload()` 重放配置；消失后
重新进入等待；重连后 `extdev_dev_resync()` 重放所有 create 并 bounce present 状态。
这是给 DSA / 交换芯片这类需要专门守护进程管理的硬件准备的。

---

## 三、使用方法

### 3.1 UCI 配置参考

配置文件是 `/etc/config/network`。下面的选项表全部从 blob policy 反推，保证与
当前代码一致。

#### `config interface`（`interface.c:83-112`）

| 选项 | 类型 | 说明 |
|------|------|------|
| `device` / `ifname` | string | 绑定的设备名（`ifname` 是旧名，解析后写进同一字段） |
| `proto` | string | 协议名；`none` / `static` 内建，其余来自脚本 |
| `auto` | bool | 是否自动启动，默认 1 |
| `disabled` | bool | 为 1/true 时整个 section 被跳过（解析时特判，不走 blob） |
| `zone` | string | 防火墙区域，只是透传给 status 的 `data` |
| `metric` / `dns_metric` | int | 路由 metric / DNS 服务器排序权重 |
| `defaultroute` | bool | 是否接受协议下发的默认路由 |
| `peerdns` | bool | 是否接受协议下发的 DNS |
| `dns` / `dns_search` | array | 静态 DNS |
| `ip4table` / `ip6table` | string | 路由表（数字或 `/etc/iproute2/rt_tables` 名字） |
| `ip6assign` | int | 从上游 PD 里分配多长的前缀给本接口（48~64） |
| `ip6hint` | string | 期望的子网号（十六进制） |
| `ip6class` | array | 只接受哪些类别的前缀 |
| `ip6ifaceid` | string | IID 生成方式：`eui64` / `random` / 固定地址 |
| `ip6weight` | int | PD 分配优先级 |
| `delegate` | bool | 是否参与前缀委派 |
| `force_link` | bool | 没有 carrier 也强行 setup |
| `carrier_loss_delay` | int | carrier 丢失后的宽限期（毫秒） |
| `renew` | bool | 拓扑变化时触发 `PROTO_CMD_RENEW` |
| `interface` | string | alias 接口指向的父接口 |
| `jail` / `jail_device` / `jail_ifname` / `host_device` | string | 网络命名空间 |
| `tags` | array | 自定义标签，透传到 status |

协议为 `static` 时额外接受（`proto.c:44-54`）：`ipaddr` / `ip6addr` / `ip6prefix`
（array）、`netmask` / `broadcast` / `ptpaddr` / `gateway` / `ip6gw`（string）、
`ip6deprecated`（bool）。

#### `config device`（`device.c:34-84`）

必填 `option name`。通用属性 48 个，按用途分组：

- **基础**：`type`、`mtu`、`mtu6`、`macaddr`、`txqueuelen`、`enabled`、`conduit`
- **IPv6**：`ipv6`、`ip6segmentrouting`、`mldversion`、`dadtransmits`
- **IPv4**：`rpfilter`、`acceptlocal`、`igmpversion`、`sendredirects`、`arp_accept`
- **邻居表**：`neighreachabletime`、`neighgcstaletime`、`neighlocktime`
- **桥端口**：`learning`、`unicast_flood`、`broadcast_flood`、`isolate`、
  `multicast_to_unicast`、`multicast_router`、`multicast_fast_leave`、`multicast`
- **加固**：`drop_v4_unicast_in_l2_multicast`、`drop_v6_unicast_in_l2_multicast`、
  `drop_gratuitous_arp`、`drop_unsolicited_na`、`promisc`
- **物理层**：`speed`、`duplex`、`autoneg`、`pause`、`asym_pause`、`rxpause`、
  `txpause`、`gro`、`eee`
- **PoE**：`pse`、`pse_podl`、`pse_power_limit`、`pse_priority`
- **认证**：`auth`、`auth_vlan`
- **其他**：`vlan`、`tags`

`type='bridge'` 时额外（`bridge.c:49-68`）：`ports`、`stp`、`stp_kernel`、`stp_proto`、
`forward_delay`、`priority`、`ageing_time`、`hello_time`、`max_age`、`igmp_snooping`、
`bridge_empty`、`multicast_querier`、`hash_max`、`robustness`、`query_interval`、
`query_response_interval`、`last_member_interval`、`vlan_filtering`。

#### 其余 section

| section | 选项 |
|---------|------|
| `route` / `route6` | `interface`、`target`、`netmask`、`gateway`、`metric`、`mtu`、`table`、`valid`、`source`、`onlink`、`type`、`proto`、`disabled` |
| `rule` / `rule6` | `in`、`out`、`invert`、`src`、`dest`、`priority`、`tos`、`mark`、`lookup`、`goto`、`action`、`suppress_prefixlength`、`uidrange`、`ipproto`、`sport`、`dport`、`disabled` |
| `neighbor` / `neighbor6` | `interface`、`ipaddr`、`mac`、`proxy`、`router` |
| `bridge-vlan` | `device`、`vlan`、`local`、`ports`、`alias` |
| `globals` | `ula_prefix`、`tcp_l3mdev`、`udp_l3mdev` |

`bridge-vlan` 的 `ports` 支持 `:` 后缀语法（`config.c:465-491`）。解析规则是
**默认 untagged**，后缀里的字符逐个处理：`t` 清掉 untagged 标志（即变成 tagged），
`*` 加上 pvid 标志，其余字符被忽略。所以 `eth0` = untagged、`eth0:t` = tagged、
`eth0:*` = untagged + pvid、`eth0:t*` = tagged + pvid。常见写法里的 `u`
（如 `eth0:u*`）其实不被 switch 处理，只是靠"默认 untagged"碰巧得到期望结果。

### 3.2 ubus 接口

#### `network`

| 方法 | 参数 | 作用 |
|------|------|------|
| `reload` | — | 重新读 UCI 并差分应用（= `netifd_reload()` → `config_init_all()`） |
| `restart` | — | 全部接口 down，1 秒后 `execvp` 自身重启 |
| `add_host_route` | `target`, `v6`, `interface`, `exclude` | 加主机路由，返回选中的接口名 |
| `get_proto_handlers` | — | 列出所有协议及其配置项和 flag |
| `add_dynamic` | `name` + 任意配置 | 创建运行时接口（不写 UCI） |
| `netns_updown` | `jail`, `start` | 启停 netns 内的接口（需要传 fd） |

#### `network.interface` / `network.interface.<name>`

| 方法 | 作用 |
|------|------|
| `up` / `down` / `renew` / `restart` | 接口控制 |
| `status` | 完整状态 JSON |
| `dump` | 所有接口的 status 数组 |
| `add_device` / `remove_device` | 热插拔成员（参数 `name`、`link-ext`、`vlan`） |
| `notify_proto` | 协议脚本回调入口（见 2.8 的 action 表） |
| `prepare` | hotplug 准备，返回可能的 `bridge` |
| `remove` | 删除动态接口（100ms 后 `vlist_delete`） |
| `set_data` | 写自定义 data |

聚合对象 `network.interface` 的每个方法都需要额外传 `interface` 参数；
每接口对象 `network.interface.lan` 则不用。这是 `netifd_add_iface_object()`
在注册时动态改写 handler 和 policy 实现的。

**没有 `del_dynamic` 方法**——删除动态接口用 `remove`。

#### `network.device`

| 方法 | 参数 | 作用 |
|------|------|------|
| `status` | `name`（可选） | 设备状态；不传 name 则 dump 全部 present 设备 |
| `set_alias` | `alias`, `device` | 绑定 `@name` 别名 |
| `set_state` | `name`, `defer`, `auth_status`, `auth_vlans` | 延迟就绪 / 802.1X 认证状态 |
| `stp_init` | — | 通知用户态 STP 初始化完成 |
| `hotplug_event` | `name`, `add` | **仅 DUMMY_MODE**，模拟热插拔 |

### 3.3 命令行工具

这几个工具都是 ubus 的薄封装：

```sh
ifup wan            # ubus call network reload; 然后 down + up
ifdown wan
ifup -a             # 遍历 ubus list 'network.interface.*'
ifstatus wan        # ubus call network.interface status '{"interface":"wan"}'
devstatus eth0      # ubus call network.device status '{"name":"eth0"}'
```

注意 `ifup` **总是先 `network reload` 再 down+up**（`files/sbin/ifup:32`），
所以 `ifup` 会顺带应用你刚改的 UCI 配置。

直接用 ubus：

```sh
ubus call network reload
ubus call network get_proto_handlers
ubus call network add_host_route '{"target":"8.8.8.8","interface":"wan"}'
ubus call network add_dynamic '{"name":"tmp","proto":"static","device":"eth0","ipaddr":["10.0.0.1/24"]}'
ubus call network.interface.tmp remove

ubus call network.interface dump
ubus call network.interface.wan status
ubus call network.device status '{"name":"br-lan"}'

ubus monitor                      # 观察所有事件与 notify
ubus listen network.interface     # 只看接口 ifup/ifdown 事件
```

服务管理走 procd：

```sh
/etc/init.d/network reload    # = ubus call network reload
/etc/init.d/network restart
```

`/etc/init.d/network` 用 `procd_set_param watch network.interface` 让 procd 监视
netifd 的 ubus 对象，netifd 挂掉会被 respawn。

### 3.4 给其他包用的查询 API

`/lib/functions/network.sh` 提供了一层缓存过的 shell 查询接口，底层是
`ubus call network.interface dump` + `jsonfilter`：

```sh
. /lib/functions/network.sh

network_is_up wan              && echo "wan is up"
network_get_ipaddr addr wan    && echo "addr=$addr"
network_get_gateway gw wan
network_get_device dev wan     # l3_device（PPPoE 时是 pppoe-wan）
network_get_physdev pdev wan   # device（PPPoE 时是 eth0）
network_get_dnsserver dns wan
network_get_prefix6 pfx wan6
network_find_wan wan_iface     # 找持有默认路由的接口
network_flush_cache            # 状态可能变了，清缓存
```

### 3.5 hotplug 钩子

在 `/etc/hotplug.d/iface/` 下放脚本，接口状态变化时被调用：

```sh
#!/bin/sh
# /etc/hotplug.d/iface/99-my-hook
case "$ACTION" in
	ifup)
		logger "interface $INTERFACE up on $DEVICE"
	;;
	ifdown)
		logger "interface $INTERFACE down"
	;;
	ifupdate)
		# $IFUPDATE_ADDRESSES / $IFUPDATE_ROUTES / $IFUPDATE_PREFIXES / $IFUPDATE_DATA
		# 标明是哪一类更新
	;;
esac
```

可用变量：`ACTION`（`ifup` / `ifdown` / `ifup-failed` / `ifupdate` / `free` /
`reload` / `create`）、`INTERFACE`、`DEVICE`（ifup/ifupdate 时是 l3_dev 的名字）。

### 3.6 写一个 shell 协议处理器

放到 `/lib/netifd/proto/myproto.sh`，加执行权限。骨架：

```sh
#!/bin/sh
. /lib/functions.sh
. ../netifd-proto.sh
init_proto "$@"

proto_myproto_init_config() {
	no_device=1                  # 本协议不需要绑定物理设备
	available=1                  # attach 后立刻标记 available
	renew_handler=1              # 支持 renew
	proto_config_add_string "server"
	proto_config_add_int "port"
	proto_config_add_defaults    # defaultroute / peerdns / renew / metric
}

proto_myproto_setup() {
	local interface="$1"
	local server port
	json_get_vars server port

	# 依赖底层可达性：底层断了自动拆本接口
	proto_add_host_dependency "$interface" "$server"

	# 拉起一个长驻进程，退出时 netifd 会自动 teardown
	proto_export "INTERFACE=$interface"
	proto_run_command "$interface" /usr/sbin/myclient -s "$server" -p "$port"
}

proto_myproto_teardown() {
	proto_kill_command "$1"
}

proto_myproto_renew() {
	proto_kill_command "$1" SIGUSR1
}

add_protocol myproto
```

真正下发配置一般在外部客户端的回调脚本里（对照 `/lib/netifd/dhcp.script`）：

```sh
proto_init_update "$ifname" 1        # action=0，link-up=1
proto_add_ipv4_address "$ip" "$mask"
proto_add_ipv4_route "0.0.0.0" 0 "$gw"
proto_add_dns_server "$dns"
proto_send_update "$interface"       # 一次性提交
```

改完记得 `/etc/init.d/network restart`——协议元数据只在启动时扫描一次。

### 3.7 调试手段

**debug_mask**（`-d` 参数，位掩码，`netifd.h:65-70`）：

| bit | 值 | 类别 |
|-----|----|------|
| 0 | 1 | SYSTEM（所有内核操作） |
| 1 | 2 | DEVICE（设备事件与状态） |
| 2 | 4 | INTERFACE（接口状态机、协议命令） |
| 3 | 8 | WIRELESS |

```sh
/etc/init.d/network stop
netifd -d 15 -S       # 全部类别，输出到 stderr 而非 syslog
```

注意 `D()` 宏只在 `-DDEBUG` 编译时才真正打印到 stderr。OpenWrt 的包
Makefile 里有 `CMAKE_OPTIONS += -DDEBUG=1`，所以固件里的 netifd 是带 debug 的。

**udebug 环形缓冲**：即使不开 `-d`，`netifd_log_message()` 也总是把消息写进
`netifd_log` ring。另有 `netifd_nl` ring，格式是 `UDEBUG_DLT_NETLINK`，
把所有经 `create_socket()` 创建的 socket 的 netlink 收发**原样抓下来**——
可以直接导出成 pcap 用 Wireshark 看。这是排查"netifd 到底给内核下了什么"最快的路。

（`sock_genl` 和 hotplug socket 没挂 debug 回调，PSE/uevent 抓不到。）

**DUMMY_MODE 本地跑**：

```sh
cd components/netifd
mkdir -p build && cd build
cmake .. -DDUMMY_MODE=1 -DDEBUG=1 -DNO_OPTIMIZE=1
make
cd ..            # 必须在源码根目录跑，因为路径都是 ./examples ./config
./build/netifd -d 15 -S
```

`examples/` 里有 `main.uc`、`proto/*.sh`、`hotplug-cmd`（只 echo 环境变量），
`config/network` 是一份示例配置。所有内核操作变成打印，可以在开发机上完整验证
配置解析、状态机、vlist 差分逻辑。

---

## 四、可以优化的点

> 每条都标了确信度。【已核实】= 直接读到代码，结论无歧义；【推断】= 基于代码结构的
> 判断，未做 profiling 或运行时验证。

### 4.1 死代码与接口不一致

**(1) `interface_ip_update_metric()` 只有声明没有定义**【已核实】

```188:188:components/netifd/interface-ip.h
void interface_ip_update_metric(struct interface_ip_settings *ip, int metric);
```

全仓库搜索只有这一处。没有定义、没有调用者，所以链接不会失败，但它是个遗留残骸。
metric 变更实际走的是 `interface_change_config()` 里的 disable/enable IP 两步，
路由 metric 在 `interface_set_route_info()` 里重新取。建议直接删掉这行声明。

**(2) `device_user.ev_idx[]` 从未被读写**【已核实】

```222:222:components/netifd/device.h
	uint8_t ev_idx[__DEV_EVENT_MAX];
```

全仓库只有这一处出现。看命名像是给事件去重/合并预留的（记录每种事件最后一次的
序号），但从未实现。每个 `device_user` 因此白白多占 13 字节。在有几十个设备和
上百个 user 的场景下不算大，但它更大的问题是**误导读者**：会让人以为存在事件
去重机制，而实际上 `device_broadcast_event()` 是无条件投递的。建议删除，或者
真正实现事件合并。

**(3) `proto_ext_run()` 接受 `force` 参数但完全没用**【已核实】

```719:722:components/netifd/proto-ext.c
int
proto_ext_run(struct proto_ext_state *state,
	      enum interface_proto_cmd cmd, bool force,
	      proto_ext_handler_cb start_cb)
```

函数体内没有任何一处引用 `force`。这与 `DESIGN` 文件的约定直接冲突：

> When a PROTO_SETUP_TEARDOWN call is issued and the 'force' parameter is
> set, the protocol handler needs to clean up immediately as good as possible,
> without waiting for its pending actions to complete.

实际行为是：不管 force 与否，setup 中途被打断都走 SIGTERM + 1 秒超时，teardown
都给 5 秒超时。`force=true` 主要来自 `interface_set_available(false)`
（`interface.c:559`，设备已经消失了），这种情况下等 5 秒是纯浪费——设备都没了，
teardown 脚本大概率也做不了什么。

**影响**：拔掉 USB 网卡或 SFP 模块时，接口要多花最多 5 秒才能进入 DOWN，期间
`ifstatus` 一直显示 `pending`。**建议**：force 时把超时缩到 100~200ms，或者直接
SIGKILL 两个 task 后立刻发 `IFPEV_DOWN`。

**(4) `netifd_ubus_wireless_notify()` 只有声明**【已核实】

声明在 `ubus.h:33`，`ubus.c` 里没有实现。无线通知已经迁到 ucode 侧，这是迁移
留下的残骸。

### 4.2 启动路径的性能

**(5) 协议脚本元数据是串行 `popen` 拿的**【推断】

```150:155:components/netifd/handler.c
#define DUMP_SUFFIX	" '' dump"

	cmd = alloca(strlen(name) + 1 + sizeof(DUMP_SUFFIX));
	sprintf(cmd, "%s" DUMP_SUFFIX, name);

	f = popen(cmd, "r");
```

`netifd_init_script_handlers()` 对 `/lib/netifd/proto/*.sh` 逐个 `popen` 并
**同步等待**。`popen` 本身还要先 fork 一个 `/bin/sh` 再由它 fork 脚本。一个装了
ppp、pppoe、pptp、l2tp、wwan、ncm、qmi、mbim、openvpn、wireguard、gre、vxlan 等
协议包的路由器上，这里是 20~30 次 fork+exec，全部串行，全部在 `main()` 里
`uloop_run()` **之前**完成。

在 400MHz 的 MIPS 上，每次 fork+exec 一个 shell 脚本 + jshn 解析大约几十毫秒，
合计可能有 1~2 秒纯启动延迟。

**建议**：三个方向，从易到难 ——
(a) 用 `posix_spawn` 直接执行脚本，跳过中间的 `/bin/sh`；
(b) 并发启动所有 dump 进程再统一收割（本来就有 uloop，改成异步不难）；
(c) 把 dump 结果缓存到 `/tmp`，按脚本 mtime 失效——启动路径上这类元数据几乎从不变。

**(6) `fork()` 而非 `posix_spawn()`**【推断】

`netifd_start_process()`（`main.c:219`）和 hotplug 的 `run_cmd()` 都用裸 `fork()`。
netifd 本身的 RSS 不大（几百 KB 到几 MB），但在**无 MMU 或内存紧张**的设备上，
fork 的页表复制仍是可观开销。`vfork` + `exec` 或 `posix_spawn` 能省掉这部分。

注意这不是纯理论——OpenWrt 支持的一些低端设备内存只有 32~64MB，网络重启时如果
同时有多个接口在跑协议脚本，fork 风暴是真实存在的。

### 4.3 运行时的性能

**(7) `system_if_dump_stats()` 每次打开 23 个 sysfs 文件**【已核实】

```3716:3742:components/netifd/system-linux.c
	const char *const counters[] = {
		"collisions",     "rx_frame_errors",   "tx_compressed",
		...
	};
	stats_dir = open(dev_sysfs_path(dev->ifname, "statistics"), O_DIRECTORY);
	...
	for (i = 0; i < ARRAY_SIZE(counters); i++)
		if (read_uint64_file(stats_dir, counters[i], &val))
			blobmsg_add_u64(b, counters[i], val);
```

每次 `ubus call network.device status` 都要做 1 次 `open(O_DIRECTORY)` +
23 次 `openat` + `read` + `close` + 字符串转整数。而这些计数器在内核里是同一个
`struct rtnl_link_stats64`，**一条 `RTM_GETLINK` netlink 消息里的 `IFLA_STATS64`
属性就能一次拿全**。

**影响**：LuCI 的实时流量图、collectd、各种监控脚本都在轮询这个接口。一台有
8 个端口的交换机，每秒轮询一次就是每秒 200 次 syscall，纯属浪费。而且 sysfs
读到的是字符串，还要 `strtoull` 一遍。

**建议**：改用 `IFLA_STATS64`。代码里已经有完整的 netlink dump 基础设施
（`system_if_check` 就是发 `RTM_GETLINK` 的），改动量不大。可以保留 sysfs 路径
作为老内核的 fallback。

**(8) hotplug 每个事件 fork 一个 shell**【推断】

`interface-event.c` 的 `run_cmd()` 对每个接口事件 fork + exec
`/sbin/hotplug-call iface`，而 `hotplug-call` 自己又要遍历
`/etc/hotplug.d/iface/*` 逐个 source。而且整条路径是**全局串行**的
（一个 `current` + 每接口一个 pending）。

启动时所有接口同时 ifup，这些事件会排成一队慢慢跑。串行是对的（避免脚本重入），
但每次都重新 fork 一个 shell 解释器去 source 同样几个脚本，重复劳动明显。

**建议**：长期方向是把 hotplug.d/iface 的常见钩子迁到 ucode（netifd 已经内嵌
了 VM，`ucode.c` 里有 `netifd.cb` 的 hotplug 回调机制）。短期可以考虑对同一接口
的连续 `ifupdate` 事件做更激进的合并。

**(9) IPv6 前缀委派每次全量重算**【推断】

`interface_refresh_assignments()` → `interface_update_prefix_assignments()`
会清空所有 assignment 重新跑一遍 first-fit。任何一个接口的 `ip6assign` /
`ip6hint` / `ip6weight` 变化，或者上游前缀 lifetime 刷新，都触发全量重建。

在典型家用路由器（3~5 个下游接口）上无所谓。但在做多 SSID / 多 VLAN 隔离的
企业场景下（几十个接口），每次 DHCPv6 续租都全量重算，而且重算过程中会短暂
删除再重加地址——理论上会让下游主机的地址闪断。

**建议**：至少在"上游前缀内容没变、只是 lifetime 刷新"的路径上跳过重算，
只更新 lifetime。这个判断在 `interface_update_prefix()` 里已经有部分逻辑
（splice assignments 刷新 lifetime），可以扩大适用范围。

**(10) status dump 每次重建完整 blob**【推断】

`netifd_dump_status()` 每次被调用都从零构建整个 blob，包括遍历所有地址、路由、
前缀、DNS、邻居。`ubus call network.interface dump` 会对每个接口做一遍。
LuCI 的状态页轮询频率不低。

**建议**：可以考虑给 status 加一个脏标记 + blob 缓存，只在真正变化时重建。
不过要小心 uptime 这类每次都变的字段。收益有限，优先级低于 (7)。

### 4.4 架构层面

**(11) `system-linux.c` 5302 行，机制混用**【已核实 + 推断】

同样是"配置一个网络设备"，代码里同时存在四种机制（详见 2.10 的表）。这不是
设计缺陷而是历史沉积——桥成员用 ioctl 是因为 `SIOCBRADDIF` 比 netlink 早，
bonding 用 sysfs 是因为内核 bonding 驱动就只暴露了 sysfs。

但它带来了实际问题：
- **错误处理不一致**：netlink 返回 `-errno`，ioctl 返回 -1 + errno，sysfs 写入
  失败被 `if (write(...)) {}` 静默吞掉（`write_file()`，`system-linux.c:384`）
- **没有统一的错误抽象**：调用方拿到一个负数，不知道是 libnl 错误码还是 errno
- **单文件 5302 行**难以审查，编译单元大，改动的爆炸半径不可控

**建议**：至少做两件事 ——
(a) 按子系统拆文件：`system-linux-bridge.c`、`system-linux-tunnel.c`、
   `system-linux-ethtool.c`、`system-linux-addr.c`。纯机械拆分，风险低。
(b) 引入统一的错误返回约定（全部转成负 errno），让上层能区分"设备不存在"、
   "权限不足"、"内核不支持"。目前很多失败路径只是 `D(SYSTEM, ...)` 打一行日志
   就返回 0，上层完全不知道出了问题。

**(12) 只订阅 `RTNLGRP_LINK`，对外部修改无感知**【已核实】

```379:379:components/netifd/system-linux.c
	nl_socket_add_membership(rtnl_event.sock, RTNLGRP_LINK);
```

这是刻意的单向权威模型（见 2.10），但在现实中会踩坑：

- 用户 `ip addr add` 加的地址，netifd 不知道，下次任何一次 IP 更新的
  `vlist_flush()` **不会**删它（因为它根本不在 vlist 里），于是这个地址会一直
  残留，造成"我删了配置但地址还在"的困惑
- 内核自动配置的地址（SLAAC、IPv6 link-local）同理
- 其他守护进程（比如某些 VPN 客户端自己装路由）与 netifd 的路由会互相覆盖，
  没有任何冲突检测

**建议**：不必改成双向同步（那会引入大量复杂度），但可以**订阅
`RTNLGRP_IPV4_IFADDR` / `RTNLGRP_IPV6_IFADDR` 仅用于告警**——检测到自己
不认识的地址出现在被管理的接口上时，打一条 warning 日志。这对现场排障价值很大，
实现成本很低。

**(13) extdev 的 `reload` 是个 TODO**【已核实】

`extdev.c` 里 `__reload` 的实现目前多数情况只是 bounce 一下 present 状态
（down/up），源码里带 TODO 注释。也就是说外部设备处理器实际上不支持真正的
增量配置更新，任何配置变化都是全量重建设备。对 DSA 交换机这类设备，
重建意味着所有端口瞬断。

**(14) shell 协议的进程开销 vs ucode**【推断】

一次 DHCP 续租的完整链路：udhcpc → `dhcp.script`（fork sh）→ 一堆
`json_add_*`（每个都是 shell 函数）→ `ubus call`（再 fork 一个 ubus 进程）。
一次续租至少 2~3 次 fork+exec，加上 jshn 在 shell 里做 JSON 拼接的开销。

netifd 已经有了 ucode 路径（`proto-ucode.c`），且无线管理已经完全迁过去。
协议侧的迁移收益同样明显：ucode VM 在 netifd 进程内，`add_proto` 注册的协议
可以直接调用原生函数，省掉 shell + jshn + ubus CLI 三层。

**建议**：把最热的几个协议（dhcp、dhcpv6）迁到 ucode。注意 `proto-ucode.c` 目前
仍然是 fork 一个 `ucode` 解释器进程去跑 setup（不是进程内回调），所以收益主要来自
省掉 jshn 和 `ubus` CLI，而不是省掉 fork。要拿到全部收益需要进一步做成进程内回调。

### 4.5 可观测性与可测试性

**(15) 没有单元测试**【已核实】

仓库里没有 `tests/` 目录，`CMakeLists.txt` 里没有 `enable_testing()`。
唯一的测试手段是 DUMMY_MODE 手工跑。

考虑到 netifd 里有几段纯算法逻辑（IPv6 PD 的 first-fit 分配、
`parse_ip_and_netmask`、vlist 差分的 keep 判断、bridge vlan 的 pvid 推导），
这些完全可以在不碰内核的情况下做单元测试。

**建议**：优先给 IPv6 PD 分配算法加测试。它是全仓库最复杂、最难手工验证、
出错后果最隐蔽的一段（分错了下游主机拿到重叠前缀，症状是间歇性不通）。

**(16) `-Werror` 让新编译器容易崩构建**【已核实】

```8:13:components/netifd/CMakeLists.txt
ADD_DEFINITIONS(-Wall -Werror)
IF(CMAKE_C_COMPILER_VERSION VERSION_GREATER 6)
	add_definitions(-Wextra -Werror=implicit-function-declaration)
	add_definitions(-Wformat -Werror=format-security -Werror=format-nonliteral)
ENDIF()
```

`-Wall -Werror` + `-Wextra` 意味着任何新版 GCC/Clang 引入的新警告都会直接
让构建失败。对交叉编译的嵌入式项目来说，工具链版本跨度很大（OpenWrt 支持
GCC 8 到 GCC 14+），这个组合很脆弱。

这是 OpenWrt 全线项目的惯例，改动需要社区共识，但至少值得知道：本地用新
编译器编不过时，先看是不是新警告而不是真 bug。

**(17) 抓包覆盖不全**【已核实】

`nl_udebug_cb` 只挂在 `create_socket()` 创建的 socket 上。`sock_genl`
（用 `nl_socket_alloc` + `genl_connect`）和 hotplug socket（走裸 `nl_recv`）
都没有挂。所以 PSE/PoE 相关的 genetlink 交互和 uevent 都抓不到。
既然 udebug 基础设施已经在了，补上这两个是几行代码的事。

### 4.6 与本仓库版本管理相关

**(18) `components/netifd` 与构建系统固定的版本不一致**【已核实】

- `components/netifd` HEAD：`e97e36f`（2026-07-17）
- `package/network/config/netifd/Makefile` 的 `PKG_SOURCE_VERSION`：
  `cbb83a18`（2026-02-26）
- 两者相差 **125 个提交**

这中间的提交里有不少是实打实的修复，从 `git log` 能看到：

```
109610f system-linux: validate FMR ealen and offset before use
b462dea system-linux: fix byte order of multicast GRE tunnel keys
ab060e0 extdev: cancel pending retry timers on device free
362b769 bridge: ignore stale member check timer events
7f6cf98 system-linux: check the nlmsg_parse result in cb_rtnl_event
9b62f2a interface-ip: keep host routes in sync with their parent route
f35f095 iprule: only keep unchanged rules whose installation succeeded
8d77f1b interface-ip: validate the route source mask
c6c2a2b interface-ip: keep addresses that moved to a different list index
```

**影响**：如果按 `components/netifd` 的代码去排查线上问题，可能会发现"代码里
明明有这个检查，为什么还是出问题"——因为固件里跑的是 5 个月前的版本。

**建议**：排查问题前先确认目标固件的 netifd 版本
（`opkg info netifd` 或看 build 日志）。如果需要这些修复，把
`PKG_SOURCE_VERSION` 和 `PKG_MIRROR_HASH` 更新到较新的提交。

---

## 五、快速索引

### 5.1 想改什么就看哪个文件

| 我想…… | 看这里 |
|--------|--------|
| 加一个 UCI 接口选项 | `interface.c:83-112` 的 `iface_attrs` |
| 加一个 UCI 设备选项 | `device.c:34-84` 的 `dev_attrs` + `device.h` 的 `DEV_ATTR_*` / `DEV_OPT_*`（注意位对齐，能热应用的要放在 `__DEV_ATTR_LIVE_APPLY_MAX` 之前） |
| 加一种设备类型 | 抄 `macvlan.c`（最简单的完整例子），写 `__init` 调 `device_type_add()` |
| 加一个协议 | 写 `/lib/netifd/proto/*.sh`，见 3.6 |
| 加一个 ubus 方法 | `ubus.c` 的三个 `*_methods[]` 数组 |
| 改接口起停条件 | `interface.c:394` 的 `interface_check_state()` |
| 改地址/路由下发 | `system-linux.c` 的 `system_addr()` / `system_rt()` |
| 改 status 输出 | `ubus.c:823` 的 `netifd_dump_status()` |
| 理解某个 hotplug 事件从哪来 | `interface-event.c` 的 `interface_queue_event()` |

### 5.2 排障入口

| 症状 | 先看 |
|------|------|
| 接口起不来，`ifstatus` 显示 `pending` | `netifd -d 4 -S` 看协议命令；协议脚本是不是卡住了（`S_SETUP` 5 秒超时） |
| 接口 `available: false` | 设备 `present` 为什么是 false：`devstatus <dev>`，看 `disabled`/`deferred` |
| 配置改了不生效 | 是不是只 `reload` 没 `restart`；协议元数据只在启动时扫描 |
| 地址删了还在 | 见 4.2 的 (12)——可能是外部加的地址，netifd 不管 |
| 想知道 netifd 给内核下了什么 | `netifd_nl` udebug ring，导出成 pcap |
| IPv6 下游拿不到前缀 | `ifstatus <上游>` 看 `ipv6-prefix` 和 `assigned`；检查下游的 `ip6assign` |

### 5.3 关键调用链速查

```
启动
  main() → netifd_ubus_init() → proto_shell_init() → extdev_init()
        → netifd_ucode_init() → system_init() → config_init_all() → uloop_run()

设备出现
  netlink RTM_NEWLINK → cb_rtnl_event() → system_device_update_state()
    → device_set_present(true) → DEV_EVENT_ADD → interface_main_dev_cb()
    → interface_set_available(true) → interface_set_up()

接口拉起
  interface_set_up() → device_claim(main_dev) → DEV_EVENT_UP
    → interface_set_enabled(true) → interface_check_state()
    → __interface_set_up() → interface_proto_event(PROTO_CMD_SETUP)
    → [static] 立即 IFPEV_UP  |  [shell] proto_ext_run() → fork 脚本
                                    → ubus notify_proto action=0 → IFPEV_UP
    → IFS_UP → IFEV_UP → hotplug ifup + ubus event

配置重载
  ubus network reload → config_init_all() → vlist 差分
    → interface_change_config() → set_config_state(IFC_RELOAD)
    → __interface_set_down() → ... → IFPEV_DOWN
    → interface_handle_config_change() → interface_do_reload()
```

---

## 参考

- 源码：`components/netifd`，HEAD `e97e36f`（upstream `openwrt/netifd.git` master）
- 官方设计文档：`components/netifd/DESIGN`（简短，但把 device/interface/proto
  三层的契约说清楚了，值得先读）
- 集成层：`package/network/config/netifd/`
- 同系列分析：`libubox-analysis.md`（netifd 用到的所有容器和事件循环）、
  `ubus-analysis.md`（ubus 协议细节）、`uci-analysis.md`（配置解析）、
  `procd-analysis.md`（netifd 的进程管理者）
