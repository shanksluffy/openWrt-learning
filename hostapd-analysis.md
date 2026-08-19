# hostapd 代码分析

分析对象：`package/network/services/hostapd`（OpenWrt 集成层）+ 上游 `hostap.git`（约 59 万行 C 代码）
分析日期：2026-08-19

> **前提说明**
>
> 和本仓库里 `components/rpcd`、`components/netifd` 那几篇不一样，**hostapd 的上游代码并不在
> 这个仓库里**。`package/network/services/hostapd/` 只有三样东西：
>
> 1. `Makefile`（870 行）——一个把上游 hostapd 切成 38 个 opkg 包的构建器；
> 2. `patches/`——69 个补丁，合计 16705 行；
> 3. `src/` + `files/`——OpenWrt 自己写的、要拷进上游源码树一起编的**集成层**
>    （`ubus.c` / `ucode.c` / `radius.c` 共 5363 行 C，加 `hostapd.uc` / `wpa_supplicant.uc`
>    共 2297 行 ucode）。
>
> 上游源码由 `Makefile` 在构建时按 `PKG_SOURCE_VERSION` 拉取：
>
> ```10:14:package/network/services/hostapd/Makefile
> PKG_SOURCE_URL:=https://w1.fi/hostap.git
> PKG_SOURCE_PROTO:=git
> PKG_SOURCE_DATE:=2025-08-26
> PKG_SOURCE_VERSION:=ca266cc24d8705eb1a2a0857ad326e48b1408b20
> PKG_MIRROR_HASH:=59ac677093f524ff98588abd9f33805a336a6e929d6814222f0d784c854f2343
> ```
>
> 为了让本文的每一条结论都能落到具体行号上，我把这个 pin 住的 commit 拉到了本地
> `/tmp/hostap`（`git fetch --depth 1 origin ca266cc24…`，HEAD 提交是
> `ca266cc nl80211: Fix crash by setting the drv->ctx properly`）。下文凡是写
> `src/ap/...`、`src/drivers/...`、`hostapd/...` 的路径，指的都是**上游树**；写
> `package/network/services/hostapd/...` 的，指的是**本仓库**。
>
> **本文所有结论都是【静态】的**——逐行读源码 + 用 `grep`/`awk` 统计得出，**没有做运行时
> 复现**。原因是 hostapd 的运行需要真实无线网卡（或 mac80211_hwsim + 内核模块）、netifd、
> ubusd、ucode 运行时一整套环境，在这台机器上搭起来的成本远高于收益。文中出现的每一个数字
> （行数、字段数、函数指针个数、超时毫秒数）都是我实际跑命令数出来的，可以复算，方法见第六章。
> 但**性能类结论（第四章）是从代码结构推导的，没有 profile 数据支撑**，请按"值得测量的怀疑
> 点"而不是"已确诊的瓶颈"来读。
>
> 另外提醒一句：hostapd 的上游是 Jouni Malinen 的 hostap.git，**不接受 GitHub PR**，补丁要
> 发到 `hostap@lists.infradead.org`；而 OpenWrt 侧的补丁走 openwrt/openwrt 的正常流程。改
> 之前先想清楚这个 bug 该修在哪一边——修错地方的话，下次 OpenWrt bump `PKG_SOURCE_VERSION`
> 你的补丁就冲突了。

---

## 目录

1. [代码架构](#一代码架构)
2. [设计原理](#二设计原理)
3. [使用方法](#三使用方法)
4. [可以优化的点](#四可以优化的点)
5. [几个容易踩的坑](#五几个容易踩的坑)
6. [数据来源与复算方法](#六数据来源与复算方法)

---

## 一、代码架构

### 1.1 hostapd 到底是什么

一句话：**hostapd 是 802.11 AP 的用户态 MLME + 认证器**。

这句话里有两个独立的职责，理解它们的分界是读懂整个代码库的前提：

- **MLME（MAC 子层管理实体）**——决定"谁能连上来"。收发 Beacon、Probe Response、
  Authentication、Association 这些管理帧，维护 STA 状态机。这部分**只在 softMAC 驱动上
  由 hostapd 做**（Linux mac80211/nl80211 走这条路），fullMAC 驱动自己在固件里做完，
  hostapd 只收结果通知。代码里这个分界叫 `NEED_AP_MLME`。
- **认证器（Authenticator）**——决定"连上来之后能不能发数据"。跑 802.1X/EAPOL 端口控制、
  WPA/WPA2/WPA3 的四次握手、SAE、PMKSA 缓存、802.11r 快速漫游，必要时通过 RADIUS 去问
  AAA 服务器。**这部分和驱动类型无关，永远由 hostapd 做**。

所以 hostapd 也可以完全不管无线，纯粹给有线交换机做 802.1X 认证器（`CONFIG_DRIVER_WIRED`，
OpenWrt 是编进去的）。

它**不**负责的事同样重要：不发数据帧（那是内核的事）、不做 DHCP、不做 NAT、不管 IP、
不管桥接（`bridge=` 只是告诉它把 VLAN 口挂哪儿）。

### 1.2 上游代码规模

按目录统计（`*.c` + `*.h`，pin 住的 ca266cc）：

| 目录 | 文件数 | 行数 | 干什么 |
|------|-------:|-----:|--------|
| `wpa_supplicant/` | 90 | 122252 | STA 侧（AP 侧用不到，但 wpad 里要一起编） |
| `src/ap/` | 107 | **83252** | **AP 核心**：MLME、认证器、Beacon、ACS/DFS |
| `src/common/` | 53 | 63531 | 帧格式定义、SAE、DPP、共享协议逻辑 |
| `src/drivers/` | 40 | 59107 | 驱动抽象层 + nl80211/wired/bsd/… 后端 |
| `src/crypto/` | 87 | 40201 | 对 OpenSSL/wolfSSL/mbedTLS/内置实现的统一封装 |
| `src/eap_peer/` | 40 | 28811 | EAP 客户端方法 |
| `src/eap_server/` | 34 | 24152 | EAP 服务端方法（内置 EAP server） |
| `src/wps/` | 35 | 22027 | WPS |
| `src/p2p/` | 12 | 21203 | Wi-Fi Direct |
| `src/tls/` | 32 | 19001 | 内置 TLS 实现（`CONFIG_TLS=internal` 时用） |
| `src/utils/` | 62 | 19209 | eloop、wpabuf、os 抽象 |
| `src/rsn_supp/` | 11 | 15420 | STA 侧 RSN |
| `hostapd/` | 12 | 16506 | `main.c` + 配置解析 + 控制接口 + Makefile |
| `src/radius/` | 8 | 8230 | RADIUS 客户端/服务端 |
| 其它 | — | — | eapol_auth、l2_packet、pae(MACsec)、eap_common |

单文件榜（这几个是读代码时绕不开的）：

| 文件 | 行数 | 说明 |
|------|-----:|------|
| `src/drivers/driver_nl80211.c` | 15114 | nl80211 后端主体 |
| `src/ap/ieee802_11.c` | 8914 | 管理帧处理总入口 |
| `src/ap/wpa_auth.c` | 7698 | RSN 认证器状态机 |
| `src/drivers/driver.h` | 7313 | 驱动抽象接口定义 |
| `hostapd/ctrl_iface.c` | 6322 | 文本控制接口 |
| `src/ap/hostapd.c` | 5238 | 接口生命周期与启动状态机 |
| `hostapd/config_file.c` | 5075 | 配置文件解析 |
| `src/ap/wpa_auth_ft.c` | 4984 | 802.11r |
| `src/ap/beacon.c` | 3329 | Beacon/Probe Response 构造 |

### 1.3 三层对象模型

这是 hostapd 最核心的抽象，看懂它，后面的代码就顺了：

```
struct hapd_interfaces          ← 进程级，全局唯一（main.c 里是个栈变量）
   │   iface[]  count
   │   global_ctrl_sock         ← -g 参数创建的全局控制 socket
   │   mld[]  mld_count         ← 802.11be AP MLD 注册表
   ▼
struct hostapd_iface            ← 一个物理射频（PHY / radio）
   │   conf  → struct hostapd_config       （109 个字段，射频级配置）
   │   state → enum hostapd_iface_state    （启动状态机，见 2.2）
   │   bss[] num_bss
   │   current_mode / hw_features / freq   （硬件能力与当前信道）
   ▼
struct hostapd_data             ← 一个 BSS（一个 SSID / 一个网络接口）
       conf     → struct hostapd_bss_config （约 400 个字段，BSS 级配置）
       drv_priv → 驱动私有数据（注意：所有 BSS 共享 bss[0] 的那一份）
       sta_list / sta_hash[256]            （关联的 STA）
       wpa_auth / eapol_auth / radius      （认证器实例）
```

字段数是我数出来的：`hostapd_bss_config` 约 400 个成员，`hostapd_config` 109 个。这两个巨型
struct 是 hostapd 配置模型的全部——**没有插件化、没有分模块的配置注册，所有特性的配置项都平铺
在这两个结构体里**，靠 `#ifdef CONFIG_XXX` 裁剪。这个设计在第四章会再提。

**"第一个 BSS 拥有驱动"这个约定必须记住。** `hostapd_driver_init()` 只对 `iface->bss[0]`
调用驱动的 `hapd_init()`，拿到 `drv_priv` 之后，`setup_interface()` 把这个指针复制给同一射频上
的所有其它 BSS：

```2100:2107:/tmp/hostap/src/ap/hostapd.c
	/*
	 * Make sure that all BSSes get configured with a pointer to the same
	 * driver interface.
	 */
	for (i = 1; i < iface->num_bss; i++) {
		iface->bss[i]->driver = hapd->driver;
		iface->bss[i]->drv_priv = hapd->drv_priv;
	}
```

后果是：凡是射频级的操作（扫描、survey、设信道、DFS）都必须通过 `bss[0]` 发起；`bss[3]` 上
的代码想扫描，得写 `hapd->iface->bss[0]`。这个约定在代码里到处出现，第一次读会觉得莫名其妙。
MBSSID 的发送 BSS（TX BSS）也永远是 `bss[0]`（`hostapd_mbssid_get_tx_bss()`，`hostapd.c:96`）。

### 1.4 驱动抽象层

`src/drivers/driver.h`（7313 行）定义了 `struct wpa_driver_ops`——**187 个函数指针**。这是
hostapd 和 wpa_supplicant 共用的接口，所以里面既有 AP 侧的（`set_ap`、`sta_add`、`sta_remove`、
`send_mlme`、`hapd_send_eapol`），也有 STA 侧的（`associate`、`scan2`），还有 MACsec 的
23 个 `macsec_*`。AP 侧真正用到的是其中一个子集。

反向通道是**一个统一的事件回调**：驱动调 `wpa_supplicant_event(ctx, EVENT_XXX, &data)`，
`enum wpa_event_type` 有 **74 个事件**。注意这个函数名——它在 hostapd 里的实现在
`src/ap/drv_callbacks.c:2564`，名字带 "supplicant" 纯粹是历史遗留（也正因为两边同名，
OpenWrt 的 multicall 补丁才必须把它改成函数指针，见 1.5）。

驱动列表在 `src/drivers/drivers.c`（53 行）里是一个被 `#ifdef` 包起来的静态数组：

```14:52:/tmp/hostap/src/drivers/drivers.c
const struct wpa_driver_ops *const wpa_drivers[] =
{
#ifdef CONFIG_DRIVER_NL80211
	&wpa_driver_nl80211_ops,
#endif
	/* ... WEXT, HOSTAP, BSD, OPENBSD, NDIS, WIRED, MACSEC_*, ATHEROS, NONE ... */
	NULL
};
```

**OpenWrt 只编 nl80211 + wired**（`hostapd-*.config` 里 `CONFIG_DRIVER_NONE` 是注释掉的），
事件循环选 epoll（`CONFIG_ELOOP_EPOLL=y`）。

nl80211 后端内部还有一层"每 PHY vs 每 netdev"的划分：

- `struct wpa_driver_nl80211_data`——每个 PHY 一个，持有能力集、扫描状态、
  `first_bss` 链表头；
- `struct i802_bss`——每个网络接口（BSS）一个，持有 ifindex、MAC、以及自己的
  `nl_mgmt`（管理帧订阅）、`nl_connect`（beacon 下发）、`nl_preq` 三个 genl socket。

Beacon 下发路径是 `wpa_driver_nl80211_set_ap()`（`driver_nl80211.c:5222`），一次调用会
往 `NL80211_CMD_NEW_BEACON`/`SET_BEACON` 里塞 30~40 个 netlink 属性，取决于开了多少特性。

### 1.5 OpenWrt 的改造

69 个补丁按性质分成六类。**这个分类比逐个看补丁重要得多**，因为它揭示了 OpenWrt 对上游的
真实态度：

| 类别 | 数量 | 代表 | 行数 |
|------|-----:|------|-----:|
| **mbedTLS 移植** | 6 | `110-mbedtls-TLS-crypto-option-initial-port.patch` | **8058** |
| **OpenWrt 控制平面** | 3 | `601-ucode_support`、`600-ubus_support`、`220-indicate-features` | 1743 |
| **构建改造** | 2 | `200-multicall`、`201-lto-jobserver-support` | 426 |
| **行为策略分歧** | 15 | `300-noscan`、`310-rescan_immediately`、`701-reload_config_inline`… | ~950 |
| **上游安全/缺陷回合** | ~20 | `001~007`（MLD 链路 ID 越界）、`190`、`192` | ~800 |
| **体积削减** | 6 | `410-limit_debug_messages`、`252-disable_ctrl_iface_mib` | ~600 |

几个值得单独说的：

**`110-mbedtls-*.patch`（8058 行）**——单个补丁就占了整个补丁集的一半。它新增了
`src/crypto/crypto_mbedtls.c`（4043 行）和 `src/crypto/tls_mbedtls.c`（3313 行），
让 hostapd 能用 mbedTLS 而不是 OpenSSL。这对 OpenWrt 是刚需：mbedTLS 的 flash 占用比
OpenSSL 小一个数量级，而且系统里的 `wget`/`uhttpd` 本来就用 mbedTLS，共用一份库。
代价是**这 8000 行完全由 OpenWrt 维护，上游从没合过**——这是整个包最大的长期负债。

**`200-multicall.patch`（362 行）**——把 hostapd 和 wpa_supplicant 合并成一个 `wpad`
二进制。技术难点是两边都定义了 `wpa_supplicant_event()`，补丁把它改成函数指针，
hostapd 侧赋值为 `hostapd_wpa_event`，supplicant 侧赋值为 `supplicant_event`。然后各自
用 `-Dmain=hostapd_main` / `-Dmain=wpa_supplicant_main` 编成静态库，最后由 28 行的
`files/multicall.c` 按 `argv[0]` 分发。省下的是一份 libcrypto、一份驱动层、一份 EAP 代码——
对 8MB flash 的路由器来说是决定性的。

**`600-ubus_support.patch`（659 行）+ `601-ucode_support.patch`（1015 行）**——这两个补丁
本身只负责**插桩**，真正的实现在本仓库的 `src/src/ap/ubus.c`（2146 行）和
`src/src/ap/ucode.c`（1059 行）里。这是 OpenWrt 和上游最本质的架构分歧，2.11 节详述。

**`701-reload_config_inline.patch`（76 行）**——小补丁，作用大。它让配置文件路径可以写成
`data:<配置内容>`，用 `fmemopen` 直接从内存读：

```c
if (!strncmp(fname, "data:", 5)) {
    f = fmemopen((void *)(fname + 5), strlen(fname + 5), "r");
    fname = "<inline>";
} else {
    f = fopen(fname, "r");
}
```

有了它，`hostapd.uc` 就能把生成好的配置直接塞进 hostapd，不用落盘到 `/var/run/`——省一次
flash/tmpfs 写、省一次读、也避免了配置文件里的明文密码留在文件系统上。

### 1.6 构建变体矩阵

`Makefile`（870 行）从**同一份源码**切出 **39 个包**——其中 36 个是守护进程本身的编译变体，
另外 3 个是辅助包（`hostapd-common` 公共文件、`hostapd-utils` 的 `hostapd_cli`、`wpa-cli`）。
变体维度是三个正交轴的笛卡尔积：

- **角色**：`hostapd`（纯 AP）× `wpa-supplicant`（纯 STA）× `wpad`（合体，multicall）
- **功能档**：`mini` × `basic` × `full` × `mesh` × `p2p`
- **TLS 后端**：`internal` × `openssl` × `wolfssl` × `mbedtls`

功能档的差异（来自 `files/hostapd-{mini,basic,full}.config` + Makefile 的 `DRIVER_MAKEOPTS` 覆盖）：

| 特性 | mini | basic | full |
|------|:----:|:-----:|:----:|
| nl80211 / 11n / 11ac / WPA2-PSK / ubus / ucode | ✓ | ✓ | ✓ |
| SAE (WPA3-PSK)¹ | ✓ | ✓ | ✓ |
| OWE、802.11r、OCV、airtime policy | — | ✓ | ✓ |
| EAP server、WPS、RADIUS 客户端、HS20、WNM、动态 VLAN、MIB | — | — | ✓ |
| 802.11be (MLO) | — | ✓² | ✓² |

¹ SAE 不在 `.config` 里，是 Makefile 按 TLS 后端加的（`CONFIG_SAE=y`，见 `Makefile:109/124/139`），
所以 `*-internal` 变体没有 SAE。
² 需要 `CONFIG_DRIVER_11BE_SUPPORT`，且 mini 档被显式排除（`Makefile:85-89`）。

**默认变体是 `wpad-basic-mbedtls`**（`Makefile:251` 的 `DEFAULT_VARIANT:=1`）——这个选择很能
说明 OpenWrt 的取舍：合体二进制省空间，basic 档给 WPA3 + 802.11r 但不给企业级 EAP，
mbedTLS 而不是 OpenSSL。

---

## 二、设计原理

### 2.1 单线程事件循环——一切设计的根源

hostapd **全程单线程**。没有 worker 线程池，没有 io_uring，加密运算和帧处理都在同一个
`eloop_run()` 循环里跑完。理解这一点，第四章一半的优化点就自然浮出来了。

`src/utils/eloop.c`（1359 行）提供四种后端（select/poll/epoll/kqueue），OpenWrt 选 epoll。
定时器实现值得看一眼：

```804:811:/tmp/hostap/src/utils/eloop.c
	/* Maintain timeouts in order of increasing time */
	dl_list_for_each(tmp, &eloop.timeout, struct eloop_timeout, list) {
		if (os_reltime_before(&timeout->time, &tmp->time)) {
			dl_list_add(tmp->list.prev, &timeout->list);
			return 0;
		}
	}
	dl_list_add_tail(&eloop.timeout, &timeout->list);
```

**一个按到期时间排序的双向链表**。取最近到期的定时器是 O(1)（取表头），但**插入是 O(n)
线性扫描**。这在只有几十个定时器时无所谓，STA 数量上去之后就不一样了——每个 STA 都有
不活动超时定时器，每次收到该 STA 的帧都要重新注册一次。这一点第四章展开。

单线程也决定了**任何阻塞调用都是全局停顿**。OpenWrt 的集成层在这里埋了两个坑，见 4.1。

### 2.2 接口启动状态机

一个射频从"配置读进来"到"Beacon 发出去"，中间可能要等好几件慢事：等内核更新监管域信道表、
等 ACS 扫描完、等 HT40 共存扫描、等 DFS 的 CAC（信道可用性检查，最长 10 分钟）。单线程
不能阻塞等，所以这里做成了**可暂停/可恢复的状态机**。

`enum hostapd_iface_state`（`src/ap/hostapd.h:565`）：

| 状态 | 在等什么 |
|------|----------|
| `HAPD_IFACE_UNINITIALIZED` | — |
| `HAPD_IFACE_DISABLED` | 停用或启动失败 |
| `HAPD_IFACE_COUNTRY_UPDATE` | 内核下发新监管域后的信道表 |
| `HAPD_IFACE_ACS` | 自动选信道的扫描/survey |
| `HAPD_IFACE_HT_SCAN` | 2.4G 的 20/40MHz 共存扫描 |
| `HAPD_IFACE_DFS` | 雷达检测 CAC |
| `HAPD_IFACE_NO_IR` | 6GHz / AFC 的 no-IR 限制 |
| `HAPD_IFACE_ENABLED` | 没在等，正常跑 |

主干调用链：

```
hostapd_setup_interface()                              [hostapd.c:2922]
  └─ setup_interface()                                 [2078]
       ├─ 把 bss[0] 的 drv_priv 复制给所有 BSS         [2104]
       ├─ 可能停在 COUNTRY_UPDATE ──→ return 0         [2120]
       └─ setup_interface2()                           [2215]
            ├─ hostapd_get_hw_features()
            ├─ hostapd_select_hw_mode()  → 可能进 ACS  [2250]
            ├─ hostapd_check_ht_capab()  → 可能进 HT_SCAN
            └─ hostapd_setup_interface_complete(iface, 0)

hostapd_setup_interface_complete()                     [2824]
  └─ hostapd_setup_interface_complete_sync()           [2545]
       ├─ hostapd_handle_dfs() → 返回 0 则挂起等 CAC   [2576]
       ├─ 逐个 hostapd_setup_bss()                     [2663]
       ├─ MBSSID：延迟到所有 BSS 建好再发 Beacon       [2683]
       └─ state = HAPD_IFACE_ENABLED                   [2755]
```

**所有异步分支的恢复点都是同一个函数** `hostapd_setup_interface_complete()`：

| 暂停原因 | 恢复回调 | 位置 |
|----------|----------|------|
| ACS 完成 | `hostapd_acs_completed()` | `hw_features.c:1197` |
| HT40 扫描完成 | `ieee80211n_check_scan()` | `hw_features.c:428` |
| DFS CAC 完成 | `hostapd_dfs_complete_cac()` | `dfs.c:1149` |
| 监管域超时（5 秒兜底） | `channel_list_update_timeout()` | `hostapd.c:2056` |

这个"单入口多恢复点"的设计很干净，但也意味着**调试启动失败时必须先确定卡在哪个 state**——
`hostapd_cli status` 里的 `state=` 字段就是这个枚举。

### 2.3 管理帧路径与 NEED_AP_MLME 分界

这是 1.1 里那个分界在代码里的具体形态：

```2564:2665:/tmp/hostap/src/ap/drv_callbacks.c
void wpa_supplicant_event(void *ctx, enum wpa_event_type event,
			  union wpa_event_data *data)
{
	struct hostapd_data *hapd = ctx;
	// ...
	case EVENT_RX_MGMT:
#ifdef NEED_AP_MLME
		hostapd_mgmt_rx(hapd, &data->rx_mgmt);
#else
		hostapd_action_rx(hapd, &data->rx_mgmt);
#endif
```

- **定义了 `NEED_AP_MLME`**（nl80211、hostap 驱动，以及只要开了 `CONFIG_SAE` 就强制定义）：
  走 `hostapd_mgmt_rx()` → `ieee802_11_mgmt()`，hostapd 亲自处理 Auth/Assoc/Deauth。
- **没定义**（BSD、Atheros 这类 fullMAC/驱动内 SME）：只走 `hostapd_action_rx()`，
  **只处理 Action 帧**，Auth/Assoc 由驱动做完，hostapd 通过 `EVENT_ASSOC` 事件事后得知。

`ieee802_11_mgmt()` 的分发（`ieee802_11.c:6748`）就是个 switch：

```6748:6783:/tmp/hostap/src/ap/ieee802_11.c
	switch (stype) {
	case WLAN_FC_STYPE_AUTH:        handle_auth(hapd, mgmt, len, ssi_signal, 0);      break;
	case WLAN_FC_STYPE_ASSOC_REQ:   handle_assoc(hapd, mgmt, len, 0, ssi_signal);     break;
	case WLAN_FC_STYPE_REASSOC_REQ: handle_assoc(hapd, mgmt, len, 1, ssi_signal);     break;
	case WLAN_FC_STYPE_DISASSOC:    handle_disassoc(hapd, mgmt, len);                 break;
	case WLAN_FC_STYPE_DEAUTH:      handle_deauth(hapd, mgmt, len);                   break;
	case WLAN_FC_STYPE_ACTION:      ret = handle_action(hapd, mgmt, len, freq);       break;
```

STA 的查找用一个**每 BSS 256 桶的哈希表**，哈希函数简单到有点粗暴：

```203:205:/tmp/hostap/src/ap/hostapd.h
#define STA_HASH_SIZE 256
#define STA_HASH(sta) (sta[5])
	struct sta_info *sta_hash[STA_HASH_SIZE];
```

直接拿 MAC 最后一个字节当桶号。对随机 MAC 和常见的厂商顺序分配都够均匀，不算问题。

### 2.4 RSN 认证器状态机

四次握手在 `src/ap/wpa_auth.c`（7698 行），用的是 IEEE 802.11 标准里那套 SM 伪代码的
**宏化直译**（`src/utils/state_machine.h`，138 行）：

```c
SM_STATE(WPA_PTK, PTKSTART) { ... }   // 展开成 static void sm_WPA_PTK_PTKSTART_Enter(...)
SM_STEP(WPA_PTK) { ... }              // 展开成 static void sm_WPA_PTK_Step(...)
```

这种写法的好处是**代码和标准文本能一行一行对上**，改的时候能直接查 spec；坏处是对不熟悉
这套约定的人来说完全不可读，grep `PTKSTART` 都找不到函数定义。

每 STA 的 PTK 状态机（`wpa_auth_i.h:27`，字段名 `wpa_ptk_state`）：

```
INITIALIZE → AUTHENTICATION → AUTHENTICATION2
           → INITPMK (802.1X) 或 INITPSK (PSK/SAE)
           → PTKSTART              (发 msg 1/4，含 ANonce)
           → PTKCALCNEGOTIATING    (收 msg 2/4，用 SNonce 算出 PTK，校验 MIC)
           → PTKCALCNEGOTIATING2
           → PTKINITNEGOTIATING    (发 msg 3/4，带 GTK)
           → PTKINITDONE           (收 msg 4/4，装 TK，置 AUTHORIZED)
```

另外还有两套：每 STA 的组密钥重协商机（`wpa_ptk_group_state`：IDLE / REKEYNEGOTIATING /
REKEYESTABLISHED / KEYERROR）和认证器全局的组密钥机（`wpa_group`：GTK_INIT / SETKEYS /
SETKEYSDONE / FATAL_FAILURE）。

`struct wpa_auth_callbacks`（`wpa_auth.h:360`）是认证器和 hostapd 主体之间的**唯一接口**——
`set_key`、`send_eapol`、`get_psk`、`get_msk`、`disconnect` 等。在 `wpa_auth_glue.c:1679`
的 `hostapd_setup_wpa()` 里被填成一个静态表。这个解耦做得不错：`wpa_auth.c` 理论上可以脱离
hostapd 复用（wpa_supplicant 的 AP 模式就是这么用的）。

### 2.5 SAE（WPA3）——单线程模型最痛的地方

SAE 用的是 Dragonfly（同时认证的对等体），核心运算在 `src/common/sae.c`（2495 行）。
它有两种把口令映射成椭圆曲线上一个点（PWE）的方法：

**hunt-and-peck（原始方法）**——循环猜，直到猜出合法点。为了防时序侧信道，即使早就猜中了
也**必须把循环跑满 k 次**：

```356:359:/tmp/hostap/src/common/sae.c
	for (counter = 1; counter <= k || !found; counter++) {
		u8 pwd_seed[SHA256_MAC_LEN];
		if (counter > 200) {
```

`k` 由 `dragonfly_min_pwe_loop_iter()` 给出，**对绝大多数 ECC 群返回 40**
（`src/common/dragonfly.c:34`）。也就是**每一次 SAE Commit 都要做 40 轮 SHA-256 + 点解压
尝试**。在 MT7621 这类 880MHz MIPS 上，这是几十毫秒量级的开销。

**H2E（hash-to-element，SSWU）**——一次算完，且可以把 PT（password token）预先算好缓存，
因为 PT 只和"口令 + SSID"有关，与 STA 无关。这是 WPA3 后续修订引入的，性能好一个数量级。
`sae_pwe` 配置项控制用哪种。

单线程 + 重运算的组合，上游给出的缓解手段**只有一个 eloop 队列，没有线程**：

```1929:1938:/tmp/hostap/src/ap/ieee802_11.c
	queue_len = dl_list_len(&hapd->sae_commit_queue);
	if (queue_len >= 15) {
		/* 队列满则丢弃 */
	}
	...
	eloop_register_timeout(0, queue_len * 10000, auth_sae_process_commit, ...)
```

新到的 SAE Commit 帧被排进 `sae_commit_queue`，**队列上限 15**，每个延后
`queue_len × 10ms` 处理。这只是**串行化 + 限流**，不是并行——真正的 `sae_process_commit()`
仍然在主循环线程上跑。所以"一群 WPA3 终端同时上线"是一个真实的拒绝服务面：第 16 个及以后
的 Commit 直接被丢，终端只能重试。

真正的解法是**驱动侧 SAE 卸载**（`WPA_DRIVER_FLAGS2_SAE_OFFLOAD_AP`，口令通过
`NL80211_ATTR_SAE_PASSWORD` 下发，`beacon.c:2508`），把整个 SAE 交换交给固件。支持的芯片
不多，但支持的话应该开。

### 2.6 802.1X / EAP / RADIUS 分层

企业级认证是四层转发，每层都是独立的状态机：

```
STA ──EAPOL──▶ ieee802_1x_receive()              [ieee802_1x.c:1120]
                  │
                  ├─ 是 EAPOL-Key(WPA/RSN)？ ──▶ wpa_receive()   [走 2.4 的四次握手]
                  │
                  └─ 是 EAP / EAPOL-Start？  ──▶ eapol_auth_sm   [AUTH_PAE + BE_AUTH 状态机]
                                                     │
                                                     ▼
                                              eap_server_sm_step()
                                                     │
                        eap_server=1 ────────────────┼──────────────── eap_server=0
                                │                                            │
                        本地 EAP 方法                                  cb.aaa_send
                        (src/eap_server/*)                                   │
                                                                             ▼
                                                              radius_client_send(RADIUS_AUTH)
                                                                             │
                                                                    ◀── Access-Accept + MSK
                                                                             │
                                                     ieee802_1x_receive_auth()  [:2015]
```

拿到 MSK 之后回到 `WPA_PTK_INITPMK`，接上四次握手。**这个分层的好处是 EAP 状态机对
"本地跑还是转发给 RADIUS"是无感的**——`eap_server` 配置项一改，同一套状态机换个出口。

OpenWrt 额外加了个东西：`770-radius_server.patch` + `src/hostapd/radius.c`（718 行），
让 `wpad` 能直接当一个独立的 RADIUS/EAP 服务器跑（`hostapd-radius` 符号链接 + `radius.init`），
用户库是 JSON 格式（`files/radius.users`）。这在小型部署里省掉一个 FreeRADIUS。

### 2.7 802.11r 快速漫游

`src/ap/wpa_auth_ft.c`（4984 行）。核心是密钥层次：MSK/PSK → PMK-R0（存在 R0KH）→
PMK-R1（分发给各个 R1KH，也就是各个 AP）。STA 漫游到新 AP 时不用重跑完整 EAP，
只做一次 FT Authentication。

两条路径：

- **Over-the-air**：STA 直接给目标 AP 发 FT Auth 帧 → `wpa_ft_process_auth()`
- **Over-the-DS**：STA 通过当前 AP 发 FT Action 帧 → `wpa_ft_action_rx()`（`:3724`），
  当前 AP 用 RRB（Remote Request Broker）协议通过**二层**转发给目标 AP

RRB 走的是原始以太网帧（`ETH_P_RRB`，以及较新的 `ETH_P_OUI`），绑在哪个接口上很关键：

```1836:1855:/tmp/hostap/src/ap/wpa_auth_glue.c
ft_iface = hapd->conf->bridge[0] ? hapd->conf->bridge : hapd->conf->iface;
hapd->l2 = l2_packet_init(ft_iface, NULL, ETH_P_RRB, hostapd_rrb_receive, ...);
```

默认用桥接口。OpenWrt 加了 `730-ft_iface.patch` 让这个可配置——因为在带 VLAN 过滤的桥上，
默认选择会把 RRB 帧发到错误的 VLAN 里，漫游直接失效。这是个典型的"上游默认值在 OpenWrt
的网络模型下不成立"的例子。

### 2.8 Beacon 构造

`src/ap/beacon.c`（3329 行）。整个文件遵循一个固定套路：

1. 一堆 `xxx_len()` 函数先估算需要多大缓冲区；
2. `os_zalloc()` 分配 head（固定 256 字节）和 tail（基础 1500 字节，按开了哪些特性增长）；
3. 一堆 `hostapd_eid_xxx(hapd, pos, end)` 函数往里写 IE，每个返回新的 `pos`；
4. 填进 `struct wpa_driver_ap_params`，调 `hostapd_drv_set_ap()` 下发。

`ieee802_11_set_beacon()`（`:3236`）是重建入口，全树有 11 处调用它。**它不只重建自己**：

```3257:3292:/tmp/hostap/src/ap/beacon.c
	/* 更新同址 6GHz / MLD 伙伴链路的 Beacon */
	for (i = 0; i < iface->interfaces->count; i++) { ... }
	for_each_mld_link(...) hostapd_gen_per_sta_profiles(...);
```

在开了 MBSSID + 6GHz 同址 + MLO 的场景下，**任何一个 BSS 的一个 IE 变化，会触发跨接口的
Beacon 重建风暴**。而触发 Beacon 重建的事件其实挺频繁——比如第一个 802.11b 终端上线导致
ERP IE 变化、`num_sta_non_erp` 变化、WPS 状态变化、邻居报告更新。这是 4.2 要说的。

### 2.9 ACS / DFS

**ACS**（`src/ap/acs.c`，1551 行）——被动扫描 + 拿 survey 数据，给每个信道算一个"干扰因子"：

```
factor = 10^(nf/5) + (busy/total) × 2^(10^(nf/10) − 10^(min_nf/10))
```

（`acs.c:371`，`nf` = 噪底，`busy` = 信道忙时间占比）。2.4G 上还会把相邻信道的干扰按
0.85（±5MHz）和 0.55（±10MHz）加权算进来，并对 1/6/11 给 0.8 的偏好系数。survey 数据
不够时退化成"数 BSS 个数"（`acs_study_bss_based`）。

如果驱动支持 `WPA_DRIVER_FLAGS_ACS_OFFLOAD`，整个过程交给固件。

**DFS**（`src/ap/dfs.c`，1676 行）——`hostapd_handle_dfs()`（`:836`）返回 0 表示"需要先做
CAC，你先挂起"，返回 1 表示"可以继续"。CAC 完成后 `hostapd_dfs_complete_cac()`（`:1149`）
回调进 `hostapd_setup_interface_complete()`。运行中检测到雷达则
`hostapd_dfs_radar_detected()`（`:1451`）触发 CSA 换台。

OpenWrt 在这里改了几个默认值：ACS 重试从 15 次提到 50 次（`360-acs_retry.patch`），
survey 数据缺字段时补默认值而不是拒绝（`470-survey_data_fallback.patch`）。都是为了对付
廉价芯片的驱动质量。

### 2.10 MBSSID 与 MLO——两个容易混淆的"多"

| | MBSSID | MLO (802.11be AP MLD) |
|---|---|---|
| 是什么 | 一个射频上多个 BSSID，共用一个 Beacon | 一个逻辑 AP 跨多个射频/频段 |
| 数据结构 | 一个 `hostapd_iface`，`bss[0..n]` | 多个 `hostapd_iface`，由 `struct hostapd_mld` 串起来 |
| TX BSS | 永远 `bss[0]` | `mld->fbss` |
| 上限 | `mbssid_max_interfaces`（硬件能力） | `MAX_NUM_MLD_LINKS` = 15（`src/common/defs.h:533`） |
| 代码位置 | `beacon.c` 的 `hostapd_eid_mbssid()` | `src/ap/ieee802_11_eht.c`（2766 行） |

`struct hostapd_mld`（`hostapd.h:532`）持有 `mld_addr`、`links`（一个 `dl_list`，挂着各链路
的 `hostapd_data`）、`fbss`。MLD 的第一个链路做 `hapd_init` 拿 `drv_priv`，后续链路复用——
和 1.3 里"第一个 BSS 拥有驱动"是同一个模式的推广。

顺带一提：补丁 `001~007` 全是 MLD 链路 ID 的越界检查。因为 `MAX_NUM_MLD_LINKS` 是 15，
而链路 ID 字段有 4 bit（0~15），**15 是保留值**，解析时不检查就会越界读写 `info->links[]`。
这类 bug 在 2025 年集中爆发了一批，说明 802.11be 的解析代码还很新。

### 2.11 OpenWrt 的控制平面：为什么要发明 ucode 层

这是本文最值得说清楚的一节。

**上游的模型**是：一个 hostapd 进程 = 一个配置文件 = 一组接口。改配置就重启进程，或者用
`hostapd_cli reload` 重读文件（但 reload 能改的东西非常有限）。

**OpenWrt 的需求**和这个模型冲突：

1. 用户在 LuCI 上改一个 SSID，不该导致同射频上其它 BSS 的终端掉线；
2. AP 和 STA 可能在同一个射频上（中继场景），需要 hostapd 和 wpa_supplicant 协商信道；
3. 配置源头是 UCI，不是 hostapd.conf；
4. 起停要受 procd 管理，不能每个射频 fork 一个进程。

于是 OpenWrt 做了这么一套东西：

```
                      procd（开机起 1 个全局进程）
                        │  /usr/sbin/hostapd -s -g /var/run/hostapd/global
                        ▼
      ┌──────────────────────────────────────────────┐
      │  wpad 进程                                    │
      │                                               │
      │   ┌────────────────────────────────────┐     │
      │   │  hostapd.uc（1428 行 ucode 脚本）   │     │  ← 601 补丁把 ucode VM
      │   │  发布 ubus 对象 "hostapd"           │     │     嵌进 hostapd 进程
      │   │  config_set / mld_set / apsta_state │     │
      │   └────────────┬───────────────────────┘     │
      │                │ hostapd.add_iface("data:…") │
      │                ▼                              │
      │   ┌────────────────────────────────────┐     │
      │   │  hostapd C 核心（上游代码）          │     │
      │   │  发布 ubus 对象 "hostapd.<ifname>"  │     │  ← 600 补丁
      │   └────────────────────────────────────┘     │
      └──────────────────────────────────────────────┘
                        ▲
                        │ ubus call hostapd config_set {phy, radio, config}
                        │
              netifd 的无线子系统
                （wifi-scripts 的 mac80211.sh / *.uc）
                        ▲
                        │
                   UCI /etc/config/wireless
```

关键点：

- **ucode VM 跑在 hostapd 进程内**，不是外部进程。所以 `hostapd.uc` 调 `hostapd.add_iface()`
  就是直接调 C 函数 `hostapd_add_iface()`（`src/src/ap/ucode.c:84`），零 IPC。
- **有两个层次的 ubus 对象**，职责完全不同：
  - `hostapd`（由 `hostapd.uc` 发布）——**配置面**。netifd 往这里推配置。
  - `hostapd.<ifname>`（由 `ubus.c` 发布）——**运行面**。LuCI / iwinfo / 漫游守护进程从这里
    拿客户端列表、发 BSS Transition 请求、踢人。名字在
    `hostapd_ubus_add_bss()` 里拼出来（`src/src/ap/ubus.c:1762`，
    `asprintf(&name, "hostapd.%s", hapd->conf->iface)`），BSS 起来之后才出现。
- **`hostapd.uc` 自己会做增量 diff**：`iface_set_config()` 比较新旧配置，能只 reload BSS 的
  就不重启整个射频。这就是"改一个 SSID 不掉全部终端"的实现。
- **AP/STA 共存**通过两个 ucode 脚本互相调 ubus 实现：wpa_supplicant 连上后调
  `hostapd.apsta_state` 告知信道，hostapd 再在这个信道上起 AP；反过来 hostapd 要换台时
  调 `wpa_supplicant.phy_set_state` 把 STA 先停掉。

这套设计的**代价**是：多了 2297 行 ucode + 1553 行 C 胶水，全部由 OpenWrt 独立维护；配置
的真实来源分散在 UCI → shell/ucode 脚本 → hostapd.conf 文本 → ucode 再解析 → C 结构体，
**中间转了四道**，排查配置问题时要一层层往下扒。第四章会具体说。

---

## 三、使用方法

### 3.1 从 UCI 到 Beacon 的完整链路

```
1. /etc/config/wireless （UCI）
        │
2. netifd 的 wireless 子系统读 UCI，按 wifi-device.type 找处理脚本
        │  package/network/config/wifi-scripts/files/lib/netifd/wireless.uc
        ▼
3. /lib/netifd/wireless/mac80211.sh  →  drv_mac80211_setup()
        │  （默认 WIFI_SCRIPTS_UCODE=y 时走 files-ucode/ 下的 ucode 版本）
        │  mac80211_hostapd_setup_base()  ← 射频段：channel/htmode/country
        │  hostapd_set_bss_options()      ← BSS 段：ssid/encryption/key
        ▼
4. 生成 /var/run/hostapd-phy0.conf
        │
5. flock /var/run/hostapd.lock \
     ubus call hostapd config_set '{"phy":"phy0","config":"/var/run/hostapd-phy0.conf",...}'
        ▼
6. hostapd.uc: iface_load_config() 解析 → iface_set_config() 决定 reload 还是重建
        │
7. hostapd.uc: hostapd.add_iface("phy0:data:<内联配置>")   ← 701 补丁的 data: 机制
        ▼
8. C 核心: hostapd_add_iface() → hostapd_setup_interface() → 状态机（2.2）
        ▼
9. Beacon 上天；hostapd_ubus_add_bss() 发布 ubus 对象 hostapd.phy0-ap0
        │
10. 返回 {"pid": N} 给 netifd，netifd 用 wireless_add_process 登记，进程死了就重拉 wifi
```

**注意第 5 步：netifd 不 fork hostapd**。hostapd 是开机由 procd 起的全局进程
（`files/wpad.init`），netifd 只是往它的 ubus 对象推配置。这和很多人的直觉不同。

常见 UCI → hostapd.conf 映射：

| UCI | hostapd.conf |
|-----|--------------|
| `option ssid 'X'` | `ssid=X` |
| `option encryption 'psk2'` | `wpa=2` `wpa_key_mgmt=WPA-PSK` `wpa_pairwise=CCMP` |
| `option encryption 'sae'` | `wpa=2` `wpa_key_mgmt=SAE` `ieee80211w=2` |
| `option encryption 'sae-mixed'` | `wpa_key_mgmt=WPA-PSK SAE` `ieee80211w=1` |
| `option encryption 'owe'` | `wpa=2` `wpa_key_mgmt=OWE` |
| `option key '...'` | `wpa_passphrase=` / `wpa_psk=` / `sae_password_file=` |
| `option channel 'auto'` | `channel=0` + `acs_survey`（触发 ACS） |
| `option htmode 'HE80'` | `ieee80211ax=1` `he_oper_chwidth=1` `he_oper_centr_freq_seg0_idx=` |
| `option country 'CN'` | `country_code=CN` `ieee80211d=1` |
| `option hidden '1'` | `ignore_broadcast_ssid=1` |
| `option isolate '1'` | `ap_isolate=1` |
| `option network 'lan'` | `bridge=br-lan` |

### 3.2 选包

39 个包看着吓人，实际决策就三步：

1. **要不要 STA 功能？**（中继、mesh、apcli）要 → `wpad-*`；纯 AP → `hostapd-*`；
   纯客户端 → `wpa-supplicant-*`。
2. **要什么档？** 只要 WPA2-PSK → `mini`（最省 flash）；要 WPA3/OWE/802.11r/11be →
   `basic`；要企业级 EAP/RADIUS/WPS/HS20 → `full`。
3. **哪个 TLS？** 系统里已经有谁就用谁。默认 `mbedtls`（最省）；有 OpenSSL 依赖的其它包
   就用 `openssl`；`internal` 只在极端裁剪时用，**注意它没有 SAE**。

默认是 `wpad-basic-mbedtls`。

配套的三个辅助包：`hostapd-common`（初始化脚本、公共文件，被所有变体依赖）、
`hostapd-utils`（`hostapd_cli`，调试必备，但会多占几十 KB）、`wpa-cli`（STA 侧的对应工具）。

### 3.3 运行时 ubus 接口

**配置面（对象 `hostapd`，由 `hostapd.uc` 发布）**——一般不手工调，netifd 用：

| 方法 | 用途 |
|------|------|
| `config_set` | 推一个射频的完整配置（reload 或重建） |
| `config_add` / `config_remove` | 单接口增删 |
| `reload` | 重读 `orig_file` |
| `switch_channel` | 主动 CSA |
| `apsta_state` | AP/STA 共存的信道协商（wpa_supplicant 调过来） |
| `mld_set` | MLO 配置 |
| `bss_info` | 查 BSS 列表 |
| `config_get_macaddr_list` | 查已分配 BSSID |

**运行面（对象 `hostapd.<ifname>`，由 `ubus.c` 发布）**——这是日常用得最多的：

```sh
ubus list | grep hostapd                        # 看有哪些 BSS
ubus call hostapd.phy0-ap0 get_clients          # 客户端列表（含 RSSI、速率、能力）
ubus call hostapd.phy0-ap0 get_status           # BSS 状态
ubus call hostapd.phy0-ap0 del_client \
    '{"addr":"aa:bb:cc:dd:ee:ff","reason":5,"deauth":true,"ban_time":30000}'
ubus call hostapd.phy0-ap0 list_bans            # 看封禁列表
ubus call hostapd.phy0-ap0 rrm_nr_list          # 802.11k 邻居报告
ubus call hostapd.phy0-ap0 bss_transition_request \
    '{"addr":"aa:bb:cc:dd:ee:ff","neighbors":["..."]}'   # 802.11v 引导漫游
ubus call hostapd.phy0-ap0 wps_start            # 开 WPS（full 档才有）
ubus call hostapd.phy0-ap0 switch_chan '{"freq":5220}'
ubus call hostapd.phy0-ap0 set_vendor_elements '{"vendor_elements":"dd..."}'
```

**订阅事件**（这是 usteer、dawn 这类漫游守护进程的工作方式）：

```sh
ubus subscribe hostapd.phy0-ap0
# 会收到 probe / auth / assoc / sta-authorized / beacon-report /
#        bss-transition-response / radar-detected / vlan_add ...
```

**订阅者可以否决关联**——`hostapd_ubus_handle_event()` 会等订阅者返回一个
802.11 status code，非 0 就拒绝这次 probe/auth/assoc。这是 OpenWrt 实现 band steering 的
机制。但要先调 `notify_response` 打开这个模式，而且**它是阻塞的**，见 4.1。

### 3.4 hostapd_cli 与全局控制接口

```sh
hostapd_cli -i phy0-ap0 status          # 看 state=（就是 2.2 那个枚举）
hostapd_cli -i phy0-ap0 all_sta         # 详细 STA 信息（full 档才有，mini 被 252 补丁裁了）
hostapd_cli -i phy0-ap0 get_config
hostapd_cli -i phy0-ap0 chan_switch 10 5220 sec_channel_offset=1 bandwidth=40 ht vht

# 全局控制接口（-g 起的那个）
hostapd_cli -p /var/run/hostapd -g /var/run/hostapd/global interfaces
```

`hostapd_cli` 走 UNIX DGRAM socket 发文本命令，服务端在
`hostapd_ctrl_iface_receive_process()`（`ctrl_iface.c:4092`）里用 **142 条 `os_strcmp` 链**
分发。

### 3.5 调试

```sh
# 1. 看 hostapd 在干什么（procd 管的，日志进 logread）
logread -f | grep -E "hostapd|wpa"

# 2. 前台跑一个实例，最高日志级别（先停掉 wifi）
wifi down
/usr/sbin/hostapd -dd /var/run/hostapd-phy0.conf

# 3. 看生成出来的配置到底长什么样（排查 UCI 映射问题的第一步）
cat /var/run/hostapd-phy0.conf

# 4. 看 netifd 推了什么给 hostapd
ubus monitor | grep -A5 hostapd

# 5. udebug 环形缓冲（OpenWrt 新增，能抓 netlink 收发，不刷屏）
ubus call udebug list
```

注意 `410-limit_debug_messages.patch`：`CONFIG_WPA_MSG_MIN_PRIORITY` 是**编译期**裁剪的，
低于该优先级的 `wpa_printf` 直接不编译进去。所以固件里 `-dd` 可能什么都看不到——
这时候要么换个编译配置，要么用 udebug。

### 3.6 脱离 OpenWrt 直接用

如果只想在普通 Linux 上跑上游 hostapd，最小配置：

```conf
interface=wlan0
driver=nl80211
ssid=TestAP
hw_mode=g
channel=6
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_passphrase=12345678
rsn_pairwise=CCMP
ctrl_interface=/var/run/hostapd
```

`hostapd -dd /etc/hostapd.conf`。配置项的权威文档是上游的 `hostapd/hostapd.conf`
（带注释的完整示例，几千行），比任何二手文档都准。

---

## 四、可以优化的点

先说清楚定位：**下面的条目都是【静态】结论**——从代码结构读出来的，没有 profile 数据。
我按"证据强度"排了序，前面的是代码里能直接看到问题形状的，后面的偏设计层面的讨论。

### 4.1 【高】单线程主循环里有两处同步阻塞等待（OpenWrt 引入）

这是我认为最值得先处理的一条，因为它是 OpenWrt 自己加的，不需要动上游。

**问题 1：`hostapd_ubus_handle_event()` 阻塞 100ms**

```1939:1946:package/network/services/hostapd/src/src/ap/ubus.c
	if (ubus_notify_async(ctx, &hapd->ubus.obj, type, b.head, &ureq.nreq))
		return WLAN_STATUS_SUCCESS;

	ureq.nreq.status_cb = ubus_event_cb;
	ubus_complete_request(ctx, &ureq.nreq.req, 100);

	if (ureq.resp)
		return ureq.resp;
```

`ubus_complete_request(..., 100)` 是**同步等待，最长 100 毫秒**。这个函数被插在
Probe Request / Auth / Assoc 三条路径上（`600-ubus_support.patch` 的 4 处调用点）。

单线程意味着这 100ms 里 hostapd **什么都做不了**：收不了其它管理帧、跑不了定时器、
处理不了 EAPOL。在人流密集的场所，Probe Request 每秒几百帧是常态。只要订阅者
（usteer / dawn / 自研 steering 守护进程）有一次响应慢，整个 AP 就卡一次。

缓解条件是有的：只在 `hapd->ubus.obj.has_subscribers` 且显式调过 `notify_response`
打开之后才走这条路（`ubus.c:1878` 和 `:1934`）。所以**没装漫游守护进程的用户不受影响**。
但装了的用户——恰恰是大部署场景——正好踩上。

**问题 2：`hostapd.uc` 的 `sta_auth` 阻塞 1000ms**

```1408:1417:package/network/services/hostapd/files/hostapd.uc
	sta_auth: function(iface, sta) {
		let msg = { iface, sta };
		let ret = {};
		let data_cb = (type, data) => { ret = { ...ret, ...data }; };
		if (hostapd.data.auth_obj)
			hostapd.data.auth_obj.notify("sta_auth", msg, data_cb, null, null, 1000);
		return ret;
	},
```

超时是 **1000 毫秒**，整整一秒。调用点在 `handle_auth()` 里，是同步的：

```
+	res = hostapd_ucode_sta_auth(hapd, sta);
+	if (res) { resp = res; goto fail; }
```
（`601-ucode_support.patch` 对 `src/ap/ieee802_11.c:3512` 的插桩）

同样只在有 `hostapd-auth` 订阅者时触发。但一秒的主循环停顿，足以让所有正在做四次握手的
终端超时重试。

**建议**：把这两条路径改成真正的异步——收到帧后挂起该 STA 的处理（hostapd 里已经有现成的
模式：SAE 的 `sae_commit_queue`、`ap_sta_wpa_psk_short` 的延迟回调），等订阅者回应或超时
后再走 `handle_auth` 的后半段。改造量不小，但这是唯一根治的办法。次优方案是把超时从
100ms/1000ms 降到 10ms/50ms 量级，并加一个"连续超时 N 次就自动降级为 fire-and-forget"
的熔断——代价小得多。

### 4.2 【中高】Beacon 重建的放大效应

`ieee802_11_set_beacon()` 在更新自己之后，会遍历 `interfaces->count` 更新同址 6GHz
和所有 MLD 伙伴链路（`beacon.c:3257-3292`）。每一次重建都是：重新估算长度 → 分配
head/tail → 逐个 IE 写入 → 打包 30~40 个 netlink 属性 → 下发内核。

触发源比想象中多：第一个 802.11b/g 终端上线（ERP IE 变化）、`num_sta_non_erp` 计数变化、
OLBC 检测、WPS 状态变化、邻居报告更新、BSS Load 更新。在一个开了 MBSSID（8 个 BSS）+
2.4G/5G/6G 三频 MLO 的设备上，一次 ERP 变化理论上会导致 3 个 iface × 8 个 BSS = 24 次
Beacon 重建 + 24 次 netlink 下发。

**建议**：加一个 per-iface 的"Beacon 脏标记 + 本轮 eloop 末尾统一 flush"的合并层。
上游其实已经有类似思路的雏形（MBSSID 场景下的 `start_beacon=false` 延迟），但没有做成
通用机制。这个改动比较适合推给上游。

### 4.3 【中】SAE 关联风暴

2.5 节已经铺垫过，这里归纳成可执行的建议：

1. **优先开 H2E**（`sae_pwe=1` 或 `2`）。hunt-and-peck 每次 Commit 固定 40 轮 EC 运算
   （`dragonfly.c:34` 返回 40），H2E 只要一次 SSWU，而且 PT 可以按"口令+SSID"缓存复用。
   OpenWrt 的 `wifi-scripts` 目前对 `sae_pwe` 的处理值得检查一下默认值是否已经是 H2E。
2. **驱动支持就开 SAE 卸载**（`WPA_DRIVER_FLAGS2_SAE_OFFLOAD_AP`）。
3. `sae_commit_queue` 上限 **15**（`ieee802_11.c:1929`）是个硬编码常量。在 100+ 终端的
   场景下这个值太小，超出直接丢帧。至少应该做成可配置，或者按 `max_num_sta` 缩放。
4. 更彻底的做法是把 SAE 的 EC 运算放到工作线程。但这会打破 hostapd "全程单线程"的
   基本假设，改动面极大，上游大概率不会接受。现实一点的中间方案是：把一次
   `sae_process_commit` 的 40 轮循环拆成分段执行，每轮之间让出 eloop——牺牲一点常量时间性
   （需要仔细论证不引入侧信道）换取不阻塞。

### 4.4 【中】eloop 定时器插入是 O(n)

```804:811:/tmp/hostap/src/utils/eloop.c
	/* Maintain timeouts in order of increasing time */
	dl_list_for_each(tmp, &eloop.timeout, struct eloop_timeout, list) {
```

排序链表，插入线性扫描。定时器主要来源是**每 STA 的不活动超时**（`ap_handle_timer`），
每收到一次该 STA 的帧就 cancel + re-register 一次。

设 N 个关联 STA，每个 STA 每秒有 f 次帧活动触发重注册，则每秒的比较次数约 **N × f × N/2**。
N=200、f=1 时是 2 万次比较/秒——还能接受；N=500、f=5 时是 62 万次/秒，在
MIPS 上就开始明显了。而且不活动超时都是同一个值，新插入的项**几乎总是排到链表尾部**，
也就是每次都要扫完整条链表——最坏情况。

**建议**：换成最小堆或时间轮。这是个自包含的改动（`eloop.c` 内部，接口不变），
适合推给上游。或者更取巧：既然大量定时器共享同一个超时值，可以从链表尾部反向扫描——
一行改动就能把最坏情况变成最好情况。

### 4.5 【中】配置解析是 547 条 strcmp 链

`hostapd_config_fill()`（`hostapd/config_file.c:2233-4959`，约 2700 行）是一条
**547 个 `os_strcmp(buf, "...")` 的 if/else-if 链**。解析一行配置平均要做 273 次字符串
比较，最坏 547 次。

单次启动无所谓（几百行配置 × 几百次比较 = 十几万次，毫秒级）。但在 OpenWrt 下**配置重载
是高频操作**：改一个 SSID、加一个 MAC 过滤、netifd 重跑 wifi，都会走一遍
`hostapd.uc` → `data:` 内联配置 → `hostapd_config_fill()`。多 BSS 时每个 BSS 一遍。

`ctrl_iface.c` 的命令分发同理，142 条链。

**建议**：改成 perfect hash（`gperf`）或者排序数组 + 二分。这是纯机械改造，
风险低、收益确定。同样适合推上游。不过说实话，这条的实际收益可能不如 4.1、4.2 大，
优先级排在后面。

### 4.6 【中】配置模型的可维护性负债

`struct hostapd_bss_config` 约 **400 个字段**，`struct hostapd_config` 109 个。
所有特性的配置项平铺在这两个结构体里，靠 `#ifdef CONFIG_XXX` 裁剪。

后果：

- **每个 BSS 都要分配这么大一个结构体**。16 BSS 的场景下，光配置结构就占可观内存
  （具体多少取决于编译时开了哪些 `#ifdef`，静态分析给不出准确数字，需要实测 `sizeof`）。
  其中大量字段是指针，指向额外分配的字符串/数组。
- **加一个新特性要改 5 个地方**：`ap_config.h` 加字段、`ap_config.c` 加默认值、
  `config_file.c` 加解析分支、`ap_config.c` 加释放逻辑、可能还要改 `ctrl_iface.c`。
  漏掉释放就是内存泄漏——补丁 `052-AP-add-missing-null-pointer-check-in-hostapd_free_ha.patch`
  就是这类问题。

这是个**设计层面的负债，短期内没法根治**（改动会波及整棵树）。提出来是为了让人在阅读
和改动时有心理准备，以及在评估"要不要给 hostapd 加一个新配置项"时把这个成本算进去。

### 4.7 【中】OpenWrt 配置链路转了四道

```
UCI 文本  →  shell/ucode 脚本拼字符串  →  hostapd.conf 文本
          →  hostapd.uc 再解析成 ucode 对象  →  重新拼成 data: 内联文本
          →  C 的 hostapd_config_fill() 解析成结构体
```

**同一份配置被序列化/反序列化了三次**。除了 CPU 开销（4.5），更麻烦的是**排障成本**：
一个选项没生效，你得依次检查 UCI 值对不对、脚本有没有映射、生成的 .conf 里有没有、
`hostapd.uc` 的 diff 逻辑有没有把它当成"无变化"跳过、C 侧有没有解析。

而且 `hostapd.uc` 需要**自己实现一个 hostapd.conf 的解析器**（`iface_load_config()`）来做
增量 diff——这个解析器和 C 侧的 `hostapd_config_fill()` 是两套独立实现，**语义必须保持一致
但没有任何机制保证**。这是一类隐蔽 bug 的温床。

**建议**：长期方向是让 netifd 直接产出结构化数据（blobmsg/JSON）推给 `hostapd.uc`，
中间不落 hostapd.conf 文本，由 ucode 一次性生成最终的 `data:` 配置。这样解析器只剩一个。
短期可行的改进是给 `iface_load_config()` 加一套单元测试，锁住它和 C 侧解析的行为一致性。

### 4.8 【中低】mbedTLS 移植的长期维护成本

`110-mbedtls-*.patch` 8058 行，新增两个大文件，上游从未合并。每次 bump
`PKG_SOURCE_VERSION` 都要 refresh 这个补丁；上游 crypto API 一变（比如新增一个
`crypto_ec_*` 函数），mbedTLS 后端就要跟着实现，否则编译断。

配套的 `120`/`130`/`135`/`150`/`160` 五个补丁全是在补这个移植的窟窿
（FIPS186-2 PRF、OWE 关联偏移、DPP PKEX 的 EC 点乘、各种 NULL 检查）。**这个 pattern
本身就是信号**：移植不完整，是靠 bug 报告一个个补出来的。

**建议**：没有便宜的解法。要么长期投入维护（并考虑把它推给上游，哪怕过程漫长），要么
评估一下 wolfSSL 是否能满足体积要求——wolfSSL 后端是上游支持的，维护成本为零。
这个决策应该基于实测的 flash 占用对比，而不是拍脑袋。

### 4.9 【低】其它零散点

- **`hostapd_ubus_handle_event()` 每次都重建 blob**（`ubus.c:1884` 起）。对 Probe Request
  这种高频事件，HT/VHT capabilities 的解包 + blobmsg 构造每帧都做一遍。可以对同一个 STA
  的连续 probe 做短期缓存。
- **`sta->sae_pt` 的缓存只覆盖 per-STA PSK 路径**（`601` 补丁对 `sae_get_password()` 的改动）。
  全局口令的 PT 走 `hapd->conf->ssid.pt`，是有缓存的；但每 STA 口令（ucode/RADIUS 下发）
  每次新建 STA 都要重算一次 `sae_derive_pt()`。当前实现已经把结果存在 `sta->sae_pt` 上了，
  合理；可以再考虑按口令哈希做进程级 LRU，避免同一口令的多个 STA 重复计算。
- **`STA_HASH(sta) = sta[5]`** 取 MAC 末字节。理论上如果部署环境的 STA MAC 末字节分布集中
  （比如某些虚拟化/测试场景批量分配），会退化成链表。实际场景（随机 MAC + 厂商顺序分配）
  分布够好，**不建议动**——列在这里只是为了说明我检查过。
- **`radius.users` 是 JSON 全量加载**（`770-radius_server.patch` + `src/hostapd/radius.c`）。
  用户量大时启动内存占用是线性的。不过这个特性定位就是小型部署，不算问题。

### 4.10 优化优先级建议

如果只能做三件事：

1. **4.1**——去掉主循环里的同步阻塞。收益最直接，且完全在 OpenWrt 自己的代码里，
   不用和上游打交道。
2. **4.3 的第 1、3 条**——确认 H2E 默认开启，把 SAE 队列上限做成可配置。改动量极小。
3. **4.4**——eloop 定时器从尾部反向扫描（一行）或换最小堆。自包含，可推上游。

4.2 收益也很大但改动面宽，建议先加 metrics 测一下真实的 Beacon 重建频率再决定。

---

## 五、几个容易踩的坑

这些不是"优化点"，是读代码/改代码时容易理解错的地方，单独列出来。

1. **`wpa_supplicant_event()` 在 hostapd 里也叫这个名字**。它是驱动层的统一事件回调，
   实现在 `src/ap/drv_callbacks.c:2564`。看到这个名字不要以为走错文件了。

2. **射频级操作必须用 `bss[0]`**。`hapd->drv_priv` 虽然每个 BSS 都有一份拷贝，但指向同一个
   对象；而扫描、survey、设频率这些操作在 nl80211 层是绑定在 `first_bss` 上的。
   在非首 BSS 的上下文里直接调会有微妙的错误。

3. **`hostapd_cli reload` 和 `ubus call hostapd reload` 完全不是一回事**。前者是上游的
   配置文件重读（能改的东西很有限）；后者是 `hostapd.uc` 的完整重配置流程（会做 diff、
   可能重建接口）。OpenWrt 下应该用后者。

4. **`-dd` 可能什么都不打印**。`CONFIG_WPA_MSG_MIN_PRIORITY` 是编译期裁剪
   （`410-limit_debug_messages.patch`），低优先级的日志根本没编进去。

5. **`internal` TLS 变体没有 SAE**。`CONFIG_SAE=y` 只在 openssl/wolfssl/mbedtls 三个分支里
   加（`Makefile:109/124/139`），选了 `wpad-full-internal` 就没有 WPA3。

6. **MLD 链路 ID 的 15 是保留值**。`MAX_NUM_MLD_LINKS` 就是 15，而字段有 4 bit。
   任何解析链路 ID 的新代码都必须检查 `link_id >= MAX_NUM_MLD_LINKS`——
   补丁 `001~007` 修的全是这个。

7. **改上游代码要发邮件列表，不是 GitHub PR**。hostap.git 不接受 PR。而且 OpenWrt 侧
   如果加了 patch，下次 bump 版本时要负责 refresh。

---

## 六、数据来源与复算方法

本文所有数字都可以复算。上游代码需要先拉下来：

```sh
mkdir -p /tmp/hostap && cd /tmp/hostap
git init -q
git remote add origin https://w1.fi/hostap.git
git fetch --depth 1 origin ca266cc24d8705eb1a2a0857ad326e48b1408b20
git checkout -q FETCH_HEAD
# 确认：git log --oneline -1
#   ca266cc nl80211: Fix crash by setting the drv->ctx properly
```

（这次 shallow fetch 花了约 9.5 分钟，w1.fi 比较慢，耐心等。）

各个数字的来源：

| 数字 | 命令 |
|------|------|
| 各目录行数 | `find src/ap -maxdepth 1 \( -name '*.c' -o -name '*.h' \) \| xargs cat \| wc -l` |
| `wpa_driver_ops` 187 个函数指针 | `awk '/^struct wpa_driver_ops \{/,/^\};/' src/drivers/driver.h \| grep -cE '\(\*[a-zA-Z_0-9]+\)\('` |
| `wpa_event_type` 74 个事件 | `awk '/^enum wpa_event_type \{/,/^\};/' src/drivers/driver.h \| grep -cE 'EVENT_[A-Z_0-9]+,'` |
| 配置关键字 547 个 | `grep -c 'os_strcmp(buf, "' hostapd/config_file.c` |
| ctrl 命令 142 个 | `awk '/hostapd_ctrl_iface_receive_process/,/^}$/' hostapd/ctrl_iface.c \| grep -cE 'os_strn?cmp\(buf, '` |
| `hostapd_bss_config` 约 400 字段 | `awk '/^struct hostapd_bss_config \{/,/^\};/' src/ap/ap_config.h \| grep -cE ';[[:space:]]*(/\*.*)?$'` |
| `hostapd_config` 109 字段 | 同上，换 struct 名 |
| SAE 队列上限 15 / 延迟 10ms | `grep -n 'queue_len' src/ap/ieee802_11.c` |
| hunt-and-peck 40 轮 | `sed -n '30,55p' src/common/dragonfly.c` |
| eloop 定时器 O(n) 插入 | `grep -n 'Maintain timeouts' -A 8 src/utils/eloop.c` |
| `STA_HASH` 定义 | `grep -n 'STA_HASH' src/ap/hostapd.h` |

OpenWrt 侧（本仓库）：

| 数字 | 命令 |
|------|------|
| 69 个补丁 / 16705 行 | `cd package/network/services/hostapd && ls patches/*.patch \| wc -l && cat patches/*.patch \| wc -l` |
| 集成层代码量 | `wc -l src/src/ap/ubus.c src/src/ap/ucode.c src/src/utils/ucode.c src/wpa_supplicant/ucode.c src/hostapd/radius.c files/hostapd.uc files/wpa_supplicant.uc` |
| 39 个包 | `grep -oE '^define Package/[a-z0-9-]+$' Makefile \| wc -l` |
| 39 处 `VARIANT:=` | `grep 'VARIANT:=' Makefile \| grep -vc DEFAULT_VARIANT` |
| ubus 阻塞 100ms | `grep -n 'ubus_complete_request' src/src/ap/ubus.c` |
| ucode sta_auth 阻塞 1000ms | `grep -n 'sta_auth' -A 8 files/hostapd.uc` |

**没有做的事**（以免高估本文的可信度）：

- 没有编译过 hostapd，没有跑过，没有 profile 数据。第四章的性能结论是从代码结构推导的。
- 没有验证 mac80211 侧 wifi-scripts 生成配置的完整细节（3.1 的映射表来自阅读
  `package/network/config/wifi-scripts/`，没有实际跑一遍对比输出）。
- 没有测量 `sizeof(struct hostapd_bss_config)` 的实际字节数（需要按具体的 `#ifdef` 组合
  编译才准）。
- 没有覆盖 wpa_supplicant 侧（12 万行）、WPS、P2P、DPP、MACsec 这几块。
