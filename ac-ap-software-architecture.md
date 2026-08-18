# AC/AP 软件系统架构：OpenWrt · 驱动 · Linux 内核

> 从软件栈的角度剖析一台 OpenWrt AP 的层次关系与各层职责，映射回 AC/AP 组网中的功能分工，并给出与 AI 结合的技术路径与商业化建议。
>
> 配套文档：[`ac-ap-networking-guide.md`](./ac-ap-networking-guide.md)（组网方案与转发模式）
>
> 代码位置基于本仓库（OpenWrt 25.12 分支）。该版本无线部分已全面 ucode 化，与旧版的 shell 脚本 + 写 conf 文件方式差异很大。

## 目录

- [一、整体分层](#一整体分层)
- [二、L0/L1：硬件、驱动与内核无线栈](#二l0l1硬件驱动与内核无线栈)
- [三、L2：hostapd / wpa_supplicant](#三l2hostapd--wpa_supplicant)
- [四、L3/L4：OpenWrt 配置编排层](#四l3l4openwrt-配置编排层)
- [五、一次 wifi up 的完整调用链](#五一次-wifi-up-的完整调用链)
- [六、数据平面：一个包实际怎么走](#六数据平面一个包实际怎么走)
- [七、映射回 AC/AP 架构](#七映射回-acap-架构)
- [八、AI 结合：技术路径与市场竞争力](#八ai-结合技术路径与市场竞争力)
- [附录：关键代码位置速查](#附录关键代码位置速查)

---

## 一、整体分层

```
┌───────────────────────────────────────────────────────────┐
│ L5 管理面 / AC 侧                                          │
│   LuCI · rpcd · uhttpd  |  CAPWAP/OpenWISP/TR-069 · usteer │
├───────────────────────────────────────────────────────────┤
│ L4 配置编排层（OpenWrt 特有）                               │
│   UCI(/etc/config/wireless) → netifd → wireless.uc         │
│                              → mac80211.uc → hostapd(ubus) │
├───────────────────────────────────────────────────────────┤
│ L3 系统基础设施                                             │
│   procd(init/进程监管) · ubusd(IPC) · libubox/uloop · ucode │
├───────────────────────────────────────────────────────────┤
│ L2 用户态 802.11 协议栈                                     │
│   hostapd / wpa_supplicant (打包为 wpad)                    │
│   AP MLME · 4次握手 · EAPOL/RADIUS · 11r/k/v · ACS · WPS    │
├──────────────────── nl80211 (netlink) ────────────────────┤
│ L1 内核                                                     │
│   cfg80211 (配置API+管制域) · mac80211 (SoftMAC栈)          │
│   厂商驱动 ath9k/ath11k/ath12k/mt76/...                     │
│   通用网络栈: netdev · bridge · VLAN · netfilter · qdisc    │
├───────────────────────────────────────────────────────────┤
│ L0 硬件与固件                                               │
│   Wi-Fi SoC · 射频前端 · firmware blob · 硬件卸载引擎        │
└───────────────────────────────────────────────────────────┘
```

**一句话概括各层职责**

| 层 | 管什么 |
|---|---|
| 内核 | 帧的收发和加解密 |
| hostapd | 该不该让你上、密钥怎么协商 |
| netifd | 配置怎么变成运行状态 |
| procd / ubus | 进程怎么活着、进程间怎么说话 |
| AC | 全网这些配置从哪来 |

---

## 二、L0/L1：硬件、驱动与内核无线栈

### 2.1 SoftMAC 与 FullMAC —— 最关键的架构分水岭

**SoftMAC**（ath9k、ath10k/11k/12k、mt76 等，OpenWrt 上的绝对主流）

芯片只做 PHY 和时间敏感的 lower MAC（ACK、退避、聚合的硬件部分），**MLME 状态机、加解密、速率控制、分片聚合的软件部分都在 Linux 的 `mac80211` 子系统里**。

- 协议行为可控、可打补丁
- 各厂商芯片行为一致 —— 这是 OpenWrt 生态能统一管理各种芯片的根本原因

**FullMAC**（brcmfmac、部分 Marvell/高通移动芯片）

整个 MAC 层跑在芯片固件里，驱动直接对接 `cfg80211`，绕过 `mac80211`。

- 优点：省 CPU
- 缺点：能控制的东西全看固件给不给接口，很多高级特性（细粒度 11v 引导、精确的 station 统计）拿不到

这个区分在代码里是显式的，`hostapd.uc` 一上来就引入了判断函数：

```3:3:package/network/services/hostapd/files/hostapd.uc
import { wdev_remove, is_equal, vlist_new, phy_is_fullmac, phy_open, wdev_set_radio_mask, wdev_set_up } from "common";
```

> **选型结论**：做 AP 产品应优先选 SoftMAC 芯片，否则 11k/v/r、精细化引导、AC 侧射频调优基本都无从实现。

### 2.2 内核里的三个层次

**厂商驱动**
负责总线（PCIe/SDIO/USB/AHB）、固件加载、DMA 收发队列、寄存器操作，向上实现 `mac80211` 的 `ieee80211_ops` 回调。

**`mac80211`（SoftMAC 框架）**

- TX 路径：分片、加密（CCMP/GCMP）、聚合（A-MPDU/A-MSDU）、速率选择（minstrel_ht）、功率保存队列
- RX 路径：解密、重排序、去聚合、上送 netdev
- AP 模式下把管理帧（auth/assoc/probe）通过 nl80211 上送给 hostapd 决策

**`cfg80211`**
统一配置 API 和**管制域（regulatory domain）**管理，向用户态暴露 `nl80211` netlink 接口。国家码、信道可用性、DFS 状态、最大发射功率都由它裁决。

> **排障提示**：工程上"改功率改不动"往往不是驱动问题，而是管制域限制。

### 2.3 OpenWrt 的特殊之处：backports

`mac80211` 和绝大部分驱动**不是用内核自带的**，而是从 **Linux backports** 单独编译的内核模块：

```11:22:package/kernel/mac80211/Makefile
PKG_NAME:=mac80211
PKG_VERSION:=6.18.39
PKG_SOURCE_URL:=https://github.com/openwrt/backports/releases/download/backports-v$(PKG_VERSION)
```

意思是内核可能是 6.6/6.12，但无线栈是 6.18 的。

- **好处**：解耦无线特性演进和内核版本，老内核平台也能拿到新的 Wi-Fi 6E/7、MLO 支持
- **代价**：`package/kernel/mac80211/patches/` 下要维护大量补丁，升级时容易冲突

`mt76`（联发科）则更进一步，是完全独立的 git 包（`package/kernel/mt76`），不在 backports 里。

---

## 三、L2：hostapd / wpa_supplicant

内核不做的那部分 802.11 逻辑全在这里。OpenWrt 把 hostapd 和 wpa_supplicant 编译成一个 multicall 二进制 **`wpad`**（`package/network/services/hostapd/files/multicall.c`），以节省 flash。

### 3.1 hostapd 的职责边界

很多人以为 AP 的一切都在内核，实际上：

- **Beacon 和 Probe Response 的内容**由 hostapd 生成，通过 nl80211 `START_AP` 灌进驱动，之后由硬件定时发出（Probe Response 有时也卸载到固件）
- **接入控制**：收到 Auth/Assoc 后判断黑白名单、最大接入数、能力匹配，决定接不接
- **密钥协商**：WPA2/WPA3 四次握手、SAE、PMK 缓存；算出的 PTK/GTK 通过 nl80211 下发给 `mac80211` 装进硬件加密引擎
- **802.1X/EAPOL** 和 RADIUS 客户端（认证 + 计费）
- **漫游相关全在这里**：11r 的 FT 密钥体系和 R0KH/R1KH 分发、11k 邻居报告、11v BSS Transition Management 帧
- **ACS 自动选信道**
- **DFS 雷达处理的用户态部分**：内核检测到雷达后上报，hostapd 决定切到哪个信道
- **WPS**

### 3.2 OpenWrt 的 hostapd 补丁：ubus 支持

OpenWrt 对 hostapd 打了约 69 个补丁（`package/network/services/hostapd/patches/`），其中最重要的一类是**加入 ubus 支持**。

每个 BSS 会注册一个 ubus 对象（如 `hostapd.wlan0`），提供：

| 方法 | 用途 |
|---|---|
| `get_clients` | 读取关联终端列表及 RSSI、速率、能力 |
| `del_client` | 踢除终端（可带 ban 时长） |
| `rrm_nr_get_own` / `rrm_nr_set` | 11k 邻居报告的读取与下发 |
| `wnm_disassoc_imminent` | 发送 11v 引导/去关联通告 |
| `bss_mgmt_enable` | 开启管理帧上报事件 |

> **这就是 AC / 引导类软件（usteer、自研控制器）能够干预漫游的接口** —— 不用改 hostapd 源码，通过 ubus 就能踢客户端、发 11v 引导帧、读取每个 STA 的 RSSI。

### 3.3 25.12 的变化：配置走 ubus 而非 conf 文件

hostapd 自带 ucode 解释器，配置**不再是写 `/var/run/hostapd-phy0.conf` 然后 SIGHUP**，而是 netifd 直接通过 ubus 把配置对象传给 hostapd，由 `hostapd.uc` 决定哪些字段变化需要重启 BSS、哪些可以热更新。

`hostapd.uc` 里的 `file_fields` / `bss_info_fields` 两张表就是干这个的。这大幅减少了改配置导致全射频重启的情况——对 AC 下发配置的场景意义很大。

---

## 四、L3/L4：OpenWrt 配置编排层

这是 OpenWrt 相对于「裸 Linux + hostapd」的核心增值部分。

### 4.1 各组件职责

**`procd`** —— PID 1
负责 init、服务监管（wpad 挂了自动拉起）、hotplug 事件分发、`/etc/init.d/` 的 service 定义。无线相关入口：

- `package/network/services/hostapd/files/wpad.init`
- `package/network/config/wifi-scripts/files/etc/hotplug.d/ieee80211/10-wifi-detect`（内核探测到新 phy 时自动生成默认配置）

**`ubusd` + `libubox/uloop`**
全系统的 IPC 总线和事件循环。netifd、hostapd、rpcd、LuCI 全部通过它通信。

**`UCI`**
`/etc/config/wireless` 中的两类节：

- `wifi-device`：射频层面 —— 信道、带宽、功率、国家码
- `wifi-iface`：BSS 层面 —— SSID、加密、模式、桥到哪个 network

**`netifd`**
核心网络守护进程，管理 interface、protocol、device、bridge、VLAN。无线是它的一个子系统：netifd 读 UCI，把每个 `wifi-device` 交给对应的 **wireless handler**（`/lib/netifd/wireless/mac80211.sh`），handler 负责创建 wdev、配置射频、拉起 hostapd/wpa_supplicant，再通过 notify 把结果（接口名、进程 PID）回报给 netifd。

**`ucode`**
OpenWrt 自研的类 JS 脚本语言，带 ubus/uloop/fs/uci 模块。无线这条链路已从 shell 迁移到 ucode，性能和可维护性都好很多（不用反复 fork `iw`、`jshn` 解析 JSON）。

### 4.2 代码落点

`wireless-device.uc` 是 netifd notify 协议的 ucode 实现，文件开头的常量正是协议命令字：

```7:10:package/network/config/wifi-scripts/files/lib/netifd/wireless-device.uc
const NOTIFY_CMD_UP = 0;
const NOTIFY_CMD_SET_DATA = 1;
const NOTIFY_CMD_PROCESS_ADD = 2;
const NOTIFY_CMD_SET_RETRY = 4;
```

而 `handle_link()` 正是组网优化项在代码里的落点 —— 组播转单播、ARP 代理、客户端隔离，都在这里把无线接口属性告诉 netifd 的设备层：

```63:69:package/network/config/wifi-scripts/files/lib/netifd/wireless-device.uc
	if (ap && config.multicast_to_unicast != null)
		dev_data.multicast_to_unicast = config.multicast_to_unicast;

	if (data.type == "vif" && config.mode == "ap") {
		dev_data.wireless_proxyarp = !!config.proxy_arp;
		dev_data.wireless_isolate = !!config.isolate;
	}
```

---

## 五、一次 wifi up 的完整调用链

把上面的层串起来看，这是排障时最有用的一张图：

```
1. wifi 脚本 (/sbin/wifi)
      └─ ubus call network reload
2. netifd 读 UCI /etc/config/wireless
      └─ 按 wifi-device 分组，校验 (validate.uc + JSON schema)
3. netifd 拉起 wireless handler: /lib/netifd/wireless/mac80211.sh
      └─ ucode: mac80211.uc / wireless.uc
4. 配置射频: nl80211 → cfg80211
      └─ 设置国家码/信道/带宽/HT-VHT-HE 能力, 创建 wdev (iw dev add)
5. 启动/配置 BSS: ubus call hostapd config_set   (25.12 新方式)
      └─ hostapd.uc 比对差异, 决定重启还是热更新
6. hostapd → nl80211 START_AP
      └─ 下发 beacon 模板、密钥策略, mac80211 开始发 Beacon
7. netifd notify 回报接口就绪
      └─ handle_link() 把 wlanX 加进 br-lan, 应用 isolate/proxyarp/m2u
8. 通知完成
      └─ ubus event netifd.wireless.done   (见 wireless_config_done())
```

### 故障定位对应表

| 现象 | 对应步骤 | 常见原因 |
|---|---|---|
| 射频参数不生效（信道/功率） | 第 4 步 | 管制域（国家码）限制 |
| SSID 起不来 | 第 5、6 步 | 看 `logread` 里 hostapd 的报错 |
| 无线起来了但不通网 | 第 7 步 | 桥接、VLAN 配置问题 |
| 配置改了没反应 | 第 2 步 | UCI 校验失败，看 schema |

---

## 六、数据平面：一个包实际怎么走

控制面之外，转发路径是另一条独立链路，也是性能瓶颈所在：

```
空口 → PHY/固件 → 驱动 DMA/NAPI → mac80211 (解密/重排序/去聚合)
     → netdev wlan0 → bridge br-lan (FDB 查表)
     → [netfilter / flow offload]
     → eth0 → 交换芯片 → 上行
```

### 加速机制

做 AP 产品性能调优基本绕不开这几个：

| 机制 | 说明 | 位置 |
|---|---|---|
| 软件 flow offload / 硬件 NAT 卸载 | 已建连接的后续包早期短路，跳过完整 netfilter | 内核 |
| **`bridger`** | 用 eBPF 加速桥转发，避开 bridge 慢路径。对 AP 这类大量二层转发的设备提升明显 | `package/network/services/bridger` |
| **WED**（Wireless Ethernet Dispatch） | 联发科方案，硬件直接在 Wi-Fi 和以太之间搬包，CPU 不参与 | mt76 驱动 |
| **NSS** | 高通同类思路的硬件加速 | `package/kernel/qca-nss-dp` |
| 组播转单播 / ARP 代理 | 直接影响空口效率 | mac80211 + bridge |

> **重要坑点**：开了硬件卸载后，很多统计和策略（限速、审计、部分 ACL）会失效或被绕过。AC 集中管控场景中，"AC 上看到的流量统计和实际不符"往往就是这个原因。

---

## 七、映射回 AC/AP 架构

### 7.1 功能分工

| 功能 | 运行位置 | OpenWrt 组件 |
|---|---|---|
| 射频收发、加解密 | AP 内核 | `mac80211` + 驱动 |
| 接入控制、密钥协商 | AP 用户态 | `hostapd` |
| 11r 密钥分发（R0KH/R1KH） | AP 间 / AC 协调 | `hostapd` + 外部密钥分发 |
| 11k/v 引导决策 | AP 本地或 AC | `usteer`（本地）/ 控制器经 hostapd ubus |
| 信道功率调优（RRM） | AC | 控制器 → UCI → netifd → `wireless-device.uc` |
| 配置下发、固件升级 | AC | OpenWISP / TR-069 / 自研，最终落到 UCI + `sysupgrade` |
| 认证（RADIUS/Portal） | AC 或外部服务器 | `hostapd`（802.1X）、`uspot` / `opennds` |
| 直接转发 | AP 内核 | bridge + `bridge-vlan` |
| 隧道转发 | AP 内核 | VXLAN / GRE / L2TPv3（OpenWrt 无原生 CAPWAP 数据面） |

### 7.2 自研 AC 的接口选择

按侵入性从低到高：

1. **读写 UCI + `ubus call network reload`**
   最稳定，跟随上游升级无痛。粒度粗，会重启 BSS。适合配置下发、批量变更。

2. **直接调 hostapd 的 ubus 对象**
   粒度细、不重启，适合漫游引导、实时统计、终端踢除。

3. **改 netifd / wifi-scripts 源码**
   能力最强，但要长期跟上游合并冲突搏斗。

> **建议**：绝大多数 AC 功能靠前两者就够了，不建议动第三层。

---

## 八、AI 结合：技术路径与市场竞争力

> 本章按"哪些真能落地、按 ROI 排序"组织，而非罗列 AI 概念。行业路径已被 Juniper Mist/Marvis、Aruba Central、华为 CampusInsight 验证过，可直接参照其成败经验。

### 8.1 前提：竞争力不在模型，在数据管道

这是最容易被跳过、却决定成败的一步。绝大多数"AI Wi-Fi"项目失败不是模型不行，而是数据**采不全、采不准、采不起**。

**需要采集的四类遥测数据**

| 类别 | 具体指标 | OpenWrt 采集点 |
|---|---|---|
| 每终端时序质量 | RSSI、SNR、协商 MCS、重传率、PER、空口占用时间 | `hostapd` ubus `get_clients`、station dump |
| 信道环境 | channel busy/tx/rx time、噪声底、邻居 AP 列表及强度 | nl80211 survey dump、`usteer` 测量汇总 |
| 事件流 | 关联/认证/漫游/去关联及 **reason code** | `hostapd` ubus 事件、`bss_mgmt_enable` |
| 上下文 | DHCP/DNS/RADIUS 的响应时延与失败 | 网关侧日志、`hostapd` RADIUS 统计 |

> 事件流里的 `reason code` 是根因分析的金矿，务必完整保留。

**关键工程约束：采样频率与开销的平衡。** 1 秒一次全量 station dump 在几百终端的 AP 上会明显吃 CPU。正确做法是 **AP 本地做聚合和变化检测**（只上报超阈值的变化 + 周期性摘要），上云数据量能压到原始的百分之一。

> 没有这一步，后面所有 AI 场景都是空的。反过来，光把这一步做扎实、配上可视化，不上任何模型，客户感知的价值就已经很高。

### 8.2 落地场景（按 ROI 排序）

#### 第一梯队：运维智能化（AIOps）—— 最值钱，最该先做

**用户体验量化评分**
把"用户说卡"变成可测量的数字：给每个终端、每个 AP、每个楼层算体验分，拆解为连接成功率、连接耗时、空口质量、吞吐达成率、漫游成功率。技术上不难（统计 + 加权），但它是所有后续价值的载体——**没有量化，AI 优化的效果就无法证明，客户就不会付钱**。

**故障预测与根因定位**
真正省钱的地方。可预测的典型问题：

- AP 因 PoE 电压抖动或内存泄漏即将掉线
- DHCP 池即将耗尽
- RADIUS 响应时延持续上升，预示服务器过载
- 某段上行链路误码率爬升
- 某台 AP 射频性能相比**同型号同环境**的 AP 显著劣化（横向对比法特别有效，不需要复杂模型）

根因定位的核心价值是**把故障归位**：自动判断问题在空口、回传、DHCP/DNS、认证服务器还是终端自身。运维中 60% 以上的"Wi-Fi 问题"根本不在 Wi-Fi 上，能自动证明这一点就极有价值。

**商业论证最直接**：一线工单自动关闭率、MTTR 下降百分比——可直接换算成客户人力成本节省。

> 这一层用**统计基线 + 规则**就能拿到 80% 的价值，不需要深度学习。建议先做这个。

#### 第二梯队：终端感知与自动化策略

**终端指纹与画像**
识别设备类型、OS、IoT 品类，自动分组、自动分配 VLAN 和策略，把组网文档里那些手工配置（IoT 单独 SSID、band steering 例外、漫游策略差异化）自动化掉。

技术手段：DHCP 指纹（Option 55 参数请求列表顺序）、802.11 能力位组合、MAC OUI、行为特征（流量模式、活跃时段）综合判断。

> **最大挑战是 MAC 随机化**。iOS/Android 默认随机 MAC 使 OUI 失效，必须靠行为指纹和能力位组合。这反而是技术门槛，做好了就是差异化。

**差异化漫游策略**
最能被终端用户直接感知的改进。不同 OS 漫游行为差异极大（iOS 相对激进、部分 Android 和 IoT 极度粘滞、Windows 看网卡驱动），统一的 RSSI 踢除门限必然顾此失彼。识别终端类型后按类别调参，甚至预测移动轨迹提前发 11v 引导，能显著降低漫游丢包。

**实现路径**：`usteer` 已有决策框架，把其固定阈值换成按终端画像自适应的参数，是改动小、收益明显的切入点。

#### 第三梯队：射频智能优化（RRM）

信道、带宽、功率的自动调优。**这是被过度宣传的一块，实际 ROI 低于前两类，风险还更高**——调错一次导致业务瞬断，客户就再也不敢开自动模式。

务实原则：

1. 优化目标必须多维（不只干扰最小，还要覆盖无洞、不在高峰期变更）
2. 严格的变更窗口和护栏
3. 每次变更可解释、可回滚
4. 先跑"建议模式"人工确认，积累信任后再开自动

技术选择上，**组合优化（图着色思路）+ 贝叶斯优化通常比强化学习更实用**，因为在线 RL 在生产网络上没法安全试错。

**干扰源分类**是本梯队性价比最高的子项：靠芯片 spectral scan 能力（ath9k/ath10k/mt76 都支持）采频谱，用轻量分类器识别微波炉、蓝牙、无线摄像头、雷达。能给出"你这里卡是因为隔壁的微波炉"这种具体结论，客户感知很强。

#### 第四梯队：LLM 运维助手 —— 差异化最快

当前最容易做出体感差异、技术成本相对低的方向。

**自然语言诊断**
运维问"为什么三楼东侧昨天下午两点到三点卡"，LLM 通过工具调用查询时序库、事件流、拓扑，返回带证据的诊断报告。本质是给 8.1 的遥测数据做自然语言前端，**复用已有投入，边际成本很低**。

**配置生成与审计**
自然语言转 UCI/AC 配置，带 schema 校验和 dry-run；或反向做配置审计（"扫描全网找出所有没关低速率、SSID 超过 4 个、功率设满的 AP"）。

> **OpenWrt 的天然优势**：UCI 配置是结构化的，且 `wifi-scripts` 已带 JSON schema（`files-ucode/usr/share/schema/wireless.*.json`），LLM 生成的配置可直接严格校验，幻觉风险可控。

**关键约束**：LLM 只做只读诊断和"生成建议 + 人工确认"，不直接下发变更。这既是安全需要，也是客户能接受的心理边界。

#### 第五梯队：Wi-Fi Sensing —— 硬件差异化的长线方向

利用 CSI（信道状态信息）做无源感知：人员存在检测、人数统计、跌倒检测、呼吸监测、入侵告警。802.11bf 标准正在推进，AP 从"网络设备"变成"传感器网络节点"，是**跳出 Wi-Fi 价格战的少数路径之一**。

现实约束：

- 需要芯片开放 CSI 上报（ath9k、部分 mt76 和 Intel 网卡可以，很多商用芯片不给）
- 算力要求不低
- 跨环境泛化差，换个房间就要重新标定
- 隐私争议严重

适合作为特定垂直场景（养老监护、安防、商业客流）的定制方案，不适合作为通用功能。

### 8.3 三层 AI 架构

按决策时延分配算力：

```
┌──────────────────────────────────────────────────────────┐
│ 云端（分钟~天级）                                          │
│   模型训练 · 跨站点联合学习 · LLM 服务 · 长周期趋势        │
│   → 数据飞轮在此形成                                       │
├──────────────────────────────────────────────────────────┤
│ AC/控制器（秒~分钟级）                                     │
│   集群级负载均衡 · RRM 决策 · 跨 AP 关联分析               │
│   算力充裕，可跑正经模型                                   │
├──────────────────────────────────────────────────────────┤
│ AP 边缘（毫秒~秒级）                                       │
│   终端指纹 · 异常快速检测 · 漫游引导本地闭环               │
│   典型 4核ARM/512MB，还要跑转发，AI 可用算力仅百分之几     │
└──────────────────────────────────────────────────────────┘
```

**边缘侧的硬约束**：模型必须是决策树、轻量 GBDT 或极小的量化神经网络，用 TFLite Micro / ONNX Runtime，体积在几十 KB 到几百 KB 量级。**不要试图在 AP 上跑大模型。**

**模型下发**要有版本管理、灰度发布和 A/B 对照，能证明"开了 AI 比不开好多少"。这个能力本身就是销售材料。

### 8.4 市场与商业模式

**从卖硬件转向卖订阅**是已验证的路径。Mist 被 Juniper 以约 4 亿美元收购、Marvis 成为主打卖点，本质卖的不是 AP，是"网络体验保障服务"。国内客户对订阅制接受度较低，更容易接受的包装是"运维平台一次性授权 + 年度服务费"。

**基于 OpenWrt 方案的差异化优势**（大厂方案贵、封闭、强绑定自家硬件）：

| 优势 | 说明 |
|---|---|
| 纳管异构硬件 | 客户存量设备不用换 |
| 私有化部署 | 教育、政企、医疗对数据不出园区要求很硬 |
| 成本低一个数量级 | 中小型园区、连锁、教育、工业等大厂覆盖不好的市场是切入点 |

**垂直行业定制比通用功能更好卖**：医院的医疗设备漫游保障、工厂的 AGV 不断线、教育的考试防作弊与终端管控、零售的客流分析。同样的底层遥测能力，包装成行业场景就能溢价。

> **销售提醒**：客户对"AI"一词已经疲劳。用可量化指标说话——"漫游丢包率从 3% 降到 0.5%"、"一线工单量下降 40%"、"平均故障定位时间从 2 小时到 8 分钟"。说"我们用了 AI"没有说服力，说这三个数字才有。

### 8.5 实施路线图

| 阶段 | 内容 | 说明 |
|---|---|---|
| **一（3~6 个月）打地基** | 遥测 agent：采 hostapd ubus + survey + usteer 数据，本地聚合上报，云端时序库落盘，做体验评分和可视化 | 不含任何 AI，但是后面一切的前提，且单独就能卖 |
| **二 统计式自动化** | 基线异常检测、同型号 AP 横向对比、故障根因规则引擎 | 低风险高回报，拿到 AIOps 大部分价值 |
| **三 引入模型** | 终端指纹分类、漫游预测、故障预测 | 此时数据已积累够 |
| **四 差异化** | LLM 运维助手 + 垂直场景（Sensing 或行业定制） | 拉开与竞品差距 |

### 8.6 要避开的坑

- **不要第一步就做强化学习调信道** —— ROI 最低、风险最高、最难证明效果
- **不要在 AP 上追求大模型** —— 算力现实不允许
- **不要做黑盒决策** —— 不可解释的 AI 在网络运维领域客户不敢开自动闭环
- **不要低估数据质量问题** —— 全网 NTP 不同步会让所有时序分析失效（这也是组网文档把 NTP 统一列为运维必做项的原因）
- **警惕 AI washing** —— 客户已疲劳，只认可量化指标

---

## 附录：关键代码位置速查

| 组件 | 路径 |
|---|---|
| netifd 无线编排（ucode） | `package/network/config/wifi-scripts/files/lib/netifd/wireless-device.uc` |
| | `package/network/config/wifi-scripts/files/lib/netifd/wireless.uc` |
| mac80211 handler | `package/network/config/wifi-scripts/files/lib/netifd/wireless/mac80211.sh` |
| | `package/network/config/wifi-scripts/files/lib/wifi/mac80211.uc` |
| UCI schema 校验 | `package/network/config/wifi-scripts/files-ucode/usr/share/schema/wireless.*.json` |
| | `package/network/config/wifi-scripts/files-ucode/usr/share/ucode/wifi/validate.uc` |
| hostapd 配置逻辑（ucode） | `package/network/services/hostapd/files/hostapd.uc` |
| hostapd 补丁（含 ubus 支持） | `package/network/services/hostapd/patches/` |
| wpad 服务定义 | `package/network/services/hostapd/files/wpad.init` |
| phy 热插拔检测 | `package/network/config/wifi-scripts/files/etc/hotplug.d/ieee80211/10-wifi-detect` |
| 内核无线栈（backports） | `package/kernel/mac80211/` |
| 联发科驱动 | `package/kernel/mt76/` |
| netifd 本体（外部 git） | `package/network/config/netifd/Makefile` |
| 桥转发加速 | `package/network/services/bridger/` |
| VXLAN（隧道转发用） | `package/network/config/vxlan/` |
