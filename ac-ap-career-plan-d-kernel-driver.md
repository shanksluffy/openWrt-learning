# 出口 D 执行计划：无线驱动与内核数据面

> [`ac-ap-career-plan-12m.md`](./ac-ap-career-plan-12m.md) 中「出口 D」的展开。周期 2026-09 ~ 2027-08，业余 10 小时/周。
>
> 本文所有代码位置、版本号、补丁名均取自本仓库实际状态，可直接执行核对。
>
> 配套文档：[`ac-ap-software-architecture.md`](./ac-ap-software-architecture.md)（L0/L1 层的概念底座）· [`openwrt-build-engineering.md`](./openwrt-build-engineering.md)（M1 的构建前置）· [`openwrt-reading-guide.md`](./openwrt-reading-guide.md)（读法方法论）

## 目录

- [零、这条路的真实代价与三个硬约束](#零这条路的真实代价与三个硬约束)
- [一、与主计划的差异：作品形态要换](#一与主计划的差异作品形态要换)
- [二、本仓库的实际抓手](#二本仓库的实际抓手)
- [三、硬件选型](#三硬件选型)
- [四、M1–M3：能改、能量、拿出第一个可公开产出](#四m1m3能改能量拿出第一个可公开产出)
- [五、M4–M6：收发路径打通与卸载理论](#五m4m6收发路径打通与卸载理论)
- [六、M7–M9：卸载实测、驱动实战、上游冲刺](#六m7m9卸载实测驱动实战上游冲刺)
- [七、M10–M12：性能工程收口与求职](#七m10m12性能工程收口与求职)
- [八、每月产出物一览](#八每月产出物一览)
- [附录 A：go/no-go 检查点](#附录-agono-go-检查点)
- [附录 B：调试手法速查](#附录-b调试手法速查)
- [附录 C：D 方向面试题库](#附录-cd-方向面试题库)
- [附录 D：上游贡献路径与礼节](#附录-d上游贡献路径与礼节)

---

## 零、这条路的真实代价与三个硬约束

先把不好听的说完，再谈怎么走。

**代价一：前六个月对外几乎无可见产出。** 协议栈方向两个月就能有个能演示的东西，驱动方向前半年都在补基础和读路径。这会带来真实的心理消耗，也意味着如果你在 2027 年 3 月被迫紧急跳槽，这条路的成果还不足以支撑。计划里用 M3 的产出提前对冲这个风险（见下文）。

**代价二：必须自购硬件。** 这是与主计划最大的出入。主计划说业余可以靠 `mac80211_hwsim` 顶，那对协议栈和上层完全成立，但对 D 方向不成立——`hwsim` 没有 DMA 环、没有中断、没有固件接口、没有硬件卸载、没有真实速率控制，而这些恰好是 D 方向的全部内容。公司真机也顶不了：你需要反复刷带 `lockdep`、`KASAN`、大量 tracepoint 的调试内核，还需要故意把设备刷坏来练恢复，这些在公司设备上做不了。

**代价三：工作杠杆可能失效。** 主计划的核心假设是"用公司真机产出量化数据"。如果你公司的产品线是基于芯片原厂 SDK 做上层集成，团队根本不碰 mac80211 和驱动，那 D 方向的真机数据只能在自购设备上做，进度会明显慢。**这是本计划最大的风险点**，也是[附录 A](#附录-agono-go-检查点) 里 M6 检查点要判断的核心问题。

对应的三个硬约束：

| 约束 | 内容 | 不满足的后果 |
|---|---|---|
| 硬件 | 至少两台可自由刷写的 MT7981/MT7986 设备 + 串口 | 无法做任何有意义的驱动实验 |
| 基础 | sk_buff / NAPI / RCU / 锁上下文 / DMA 映射要过关 | 读收发路径会变成看天书，且改出来的代码没人敢收 |
| 工作面 | 每季度至少争取到一个与性能或转发相关的任务 | 只剩业余时间，12 个月压缩成 24 个月 |

代价说完，说为什么还值得走：这条路上能改 mac80211、能定位卸载路径丢包、能读懂芯片厂 SDK 的人，在市场上是**按个数的**。它的能力不随公司、业务模式、协议流行度变化而贬值，五年后依然值钱。前面四条出口里，只有它同时满足"稀缺"和"抗周期"。

---

## 一、与主计划的差异：作品形态要换

主计划 M4–M5 安排的是做一个开源的轻量 AC 控制面。**走 D 就不要做它了。**

原因是在这个领域，作品的可信度排序和上层方向完全不同：

| 产出形态 | 上层/协议方向的说服力 | D 方向的说服力 |
|---|---|---|
| 被合并进 `linux-wireless` 的 mac80211 patch | 中 | **最高**，无法伪造，且面试官大概率认识 review 你的人 |
| 被合并进 `openwrt/mt76` 的 patch | 中 | 很高 |
| 一份带真实 trace 数据的路径分析报告 | 中 | 高，直接证明你读得动、量得准 |
| 自建 GitHub 项目（协议栈/控制面） | **最高** | 中，容易被看成"这跟驱动没关系" |

所以 D 方向的作品重心从"造一个项目"转为**"上游 patch + 可复现的实验数据"**。原本 M4–M5 造轮子的 80 小时，改成：路径逐行打通、把实验床工具化、提交第一个 patch。

唯一保留的自建项目是一个小而实用的工具集（下文的"无线性能实验床"），它的定位是支撑实验、顺带可开源，而不是简历主角。

主计划里其余条目全部保留不变：合规红线、季度复盘、量化指标模板、时间纪律、M11–M12 的求职执行。

---

## 二、本仓库的实际抓手

动手前先搞清这棵树里 D 方向的东西都在哪。以下全部已核对。

### 版本与来源

| 组件 | 版本/来源 | 位置 |
|---|---|---|
| 内核 | 6.12.94 | `target/linux/generic/kernel-6.12` |
| mac80211 / cfg80211 | **backports 6.18.39**（非内核自带） | `package/kernel/mac80211/Makefile` |
| mt76 驱动 | git `39c960c`，2026-03-19 | `package/kernel/mt76/Makefile`（源码需 prepare 后才出现） |
| hostapd | 69 个补丁 | `package/network/services/hostapd/patches/` |

### 一个必须先知道的陷阱

内核配置里这两行是关键：

```
# target/linux/generic/config-6.12
# CONFIG_CFG80211 is not set      (879 行)
# CONFIG_MAC80211 is not set      (3313 行)
```

**内核自带的 mac80211/cfg80211 是关掉的，实际运行的是 backports 包编出来的 6.18.39 版本。** 两个直接推论：

1. 读代码要去 `build_dir/target-*/linux-*/backports-6.18.39/net/mac80211/`，**不是** `linux-6.12.94/net/mac80211/`。后者的代码根本没编进固件。这是个能白白浪费一周的坑。
2. 改 mac80211 的补丁放 `package/kernel/mac80211/patches/subsys/`，**不是** `target/linux/*/patches-6.12/`。

另一个推论很有意思：这棵树跑的是 6.12 的内核骨架 + 6.18 的无线栈。所以 backports 里有大量兼容层垫片（`backport-include/`），读代码时要能分辨哪些是真实逻辑、哪些是版本适配噪音。

### 关键代码与补丁位置

| 内容 | 位置 |
|---|---|
| OpenWrt 对 mac80211 的私有改动（18 个） | `package/kernel/mac80211/patches/subsys/` |
| ath9k/ath10k/ath11k/ath12k 补丁 | `package/kernel/mac80211/patches/ath*/` |
| WED（Wireless Ethernet Dispatch） | `target/linux/mediatek/patches-6.12/940-944-*`、`target/linux/generic/backport-6.12/731-733-*` |
| 软件流表卸载 | `target/linux/generic/hack-6.12/650-netfilter-add-xt_FLOWOFFLOAD-target.patch` |
| mtk 以太网驱动改动 | `target/linux/generic/pending-6.12/736-*`、`737-*`；`hack-6.12/730-*`（强制复位时的硬件 dump） |
| Filogic 设备定义 | `target/linux/mediatek/image/filogic.mk` |
| 无线配置编排（ucode 化） | `package/network/config/wifi-scripts/files-ucode/` |

`subsys/` 里那 18 个补丁值得单独点出来，它们是 M3 的主要素材：

| 补丁 | 内容 | 为什么重要 |
|---|---|---|
| `301`~`304` | minstrel_ht 的 BA 令牌随机化、宏修正、速率波动抑制、降速逻辑重写 | 速率控制的实际行为与上游不同 |
| `305` | airtime 调度器 quantum 改为 `weight << 3` | 多站点并发下的调度开销 |
| `320` | 给广播报文加 AQL 限额（`AQL_TXQ_LIMIT_BC = 50000`） | 广播洪泛压塌单播吞吐的修复 |
| `350` | 允许在 radar 信道上扫描 | DFS 行为差异 |
| `360`/`361` | 期望吞吐的估算 | 直接影响 mesh 选路和上报指标 |
| `370`/`371` | MLO 探测、eMLSR/eMLMR 动作帧解析 | Wi-Fi 7 相关，是当下最热的面试话题 |

---

## 三、硬件选型

### 首选：OpenWrt One ×1 + 廉价 MT7981 整机 ×1

**OpenWrt One** 在本树里就是 `openwrt_one`（`filogic.mk:2795`），配置是 MT7981B + MT7976（走 `mt7915e` 驱动），包列表里带 `mt7981-wo-firmware`——**WO 就是 WED 用的那颗卸载协处理器的固件**，也就是说这块板子开箱即可做卸载实验。

选它的决定性理由是**刷不死**：镜像定义里 NOR 和 SPI-NAND 两套 preloader/FIP 都在（`nor-preloader.bin`、`snand-preloader.bin`），且 UBI 里单独有一个 `recovery` 卷装 initramfs。NOR 启动 + NAND 启动互为后备，加上板载 USB-C 串口，你可以放开手脚刷任何实验内核。对一条"每周要刷十几次调试内核"的路线，这个属性比性能重要得多。

第二台建议买便宜的量产整机做对照和 STA 端，本树支持的 MT7981 机型很多，例如 `cudy_wr3000-v1`、`xiaomi_mi-router-ax3000t`、`cmcc_rax3000m`、`qihoo_360t7`，二手或新机都在百元级。要点是**同 SoC 系列**，这样两台的驱动行为可比。

预算量级：一台 One 加一台整机，加 USB-TTL 串口线和网线，千元内。价格以实际渠道为准，但相对这条路的收益，这是全年最值的一笔投入。

### 为什么必须两台

- 一台跑 AP、一台跑 STA，才能做真实空口打流；`hwsim` 测不出速率控制和聚合的真实行为
- 一台刷实验内核、一台保持稳定版做对照，避免"到底是我改坏了还是本来就这样"
- WED 实验需要真实的有线↔无线转发路径，单机测不出来

### 可选补充

- 如果后期想碰 Wi-Fi 7 与 MLO：MT7988（`mediatek_mt7988a-rfb`）或 BPi-R4，价格高一档，建议 M6 决策点之后再买
- 想对比另一个生态：`qualcommax`/`qualcommbe` 目标 + ath11k/ath12k，但高通线的固件封闭度更高，不建议作为主战场

---

## 四、M1–M3：能改、能量、拿出第一个可公开产出

### M1（2026-09）可反复刷、可调试、可救砖

前置：先把 [`openwrt-build-engineering.md`](./openwrt-build-engineering.md) 的容器化与缓存方案落地，否则一次全量重编会劝退你。

1. **编出固件并刷进去**：target 选 `mediatek/filogic`，device 选 `openwrt_one`。第二台整机对应机型也编一份。
2. **串口与救砖**：接上串口拿到 U-Boot 与内核 console；**主动把 NAND 刷坏一次**，用 NOR 启动 + recovery 卷恢复回来。这条不是折磨自己，是让你之后敢做危险实验的心理和技能基础。
3. **打开调试能力**（`make kernel_menuconfig`）：`DEBUG_INFO`、`KALLSYMS_ALL`、`DYNAMIC_DEBUG`、`FTRACE` + `FUNCTION_TRACER` + `FUNCTION_GRAPH_TRACER`、`PROVE_LOCKING`、`DEBUG_ATOMIC_SLEEP`、`DEBUG_LIST`。包侧打开 `PACKAGE_MAC80211_DEBUGFS`、`PACKAGE_MAC80211_TRACING`、`PACKAGE_CFG80211_TESTMODE`（都在 `package/kernel/mac80211/Makefile` 的 `PKG_CONFIG_DEPENDS` 里）。
   > 注意 `lockdep` 和 tracing 会明显拉低吞吐。**做性能测量时必须换成干净内核**，否则数据全是废的。这一条听起来是常识，实际上是新手最常犯的错。
4. **走通三条改动流程**（这是 D 方向的日常动作，务必肌肉记忆）：
   - 改内核：`make target/linux/{clean,prepare} V=s QUILT=1` → `quilt new` / `add` / `refresh` → `make target/linux/update`
   - 改 mac80211：同样流程作用于 `package/kernel/mac80211`，补丁落到 `patches/subsys/`
   - 改 mt76：`make package/kernel/mt76/{clean,prepare} V=s QUILT=1`（源码是 git 拉取的，prepare 之后才在 `build_dir` 出现）
5. **第一个内核改动**：给 mt76 的收包路径加一个 debugfs 计数器，或加一个自己的 tracepoint。

验收：真机 `/sys/kernel/debug/` 下能读到你自己加的计数器；刷坏过一次并成功恢复；三条 quilt 流程各走通一次并把补丁提交进 git。

### M2（2026-10）内核基础与观测武装

基础补课必须**贴着真实代码**做，不要抱着书从头读。每一项都要求你能在 backports 或 mt76 源码里找出对应实例：

| 主题 | 要能回答的问题 | 找实例的地方 |
|---|---|---|
| `sk_buff` | headroom/tailroom 怎么被 802.11 头和加密开销消耗？何时会触发 `skb_cow`/线性化？clone 与 share 的区别在无线路径上意味着什么？ | `ieee80211_skb_resize` |
| NAPI / softirq | 收包为什么要在 NAPI 里做？budget 用完会怎样？`ieee80211_rx_napi` 的调用上下文是什么？ | mt76 `dma.c` 的 poll |
| RCU | 站点表为什么用 RCU？`rcu_read_lock` 区间内不能做什么？ | `sta_info` 查找路径 |
| 锁与上下文 | `spin_lock_bh` 与 `spin_lock_irqsave` 何时用哪个？为什么 tx 路径上不能睡眠？ | `local->active_txq_lock` |
| DMA 映射 | 一致性映射与流式映射的区别？`dma_map_single` 之后为什么不能碰缓冲区？ | mt76 DMA 环 |

观测工具，逐个跑通并留下笔记：

- `trace-cmd` / `ftrace`：抓 mac80211 的 `drv_*` 与 `api_*` tracepoint（`MAC80211_TRACING` 打开后可用），以及 mt76 自己的 trace 点
- **debugfs 是这条路的主战场**：`/sys/kernel/debug/ieee80211/phy0/` 下的 `aqm`、`aql_pending`、`aql_txq_limit`、`airtime`、`stations/<mac>/*`；mt76 目录下的队列、DMA、固件状态
  > 顺带说，`320` 号补丁给 `aql_pending` 加了 `BC/MC` 一行、给 `aql_txq_limit` 加了 `mcast` 写入接口——这正是 M3 做 A/B 实验的观测入口
- `perf record` 直接在设备上采样，看 softirq 里的热点
- `dynamic_debug`：按文件/函数粒度开关内核日志，比重编快得多

**建"无线性能实验床 v0"**：脚本化完成"起 AP → STA 关联 → 打流（单流/多流/双向/广播）→ 采集吞吐、pps、CPU 各核 softirq 占比、AQL pending、队列深度、重传率、MCS 分布 → 出报告"。

验收：同条件连跑三次，关键指标偏差 < 5%。**达不到这个稳定性就别往下走**，后面所有 A/B 结论都会是噪音。

### M3（2026-11）逆向解剖 OpenWrt 的 mac80211 分支

这是整条路线里性价比最高的一个月，因为它同时给你四样东西：真实的代码理解、可量化的数据、一个完全不涉及公司信息因此可以公开发表的产出、以及一批面试硬题的亲身答案。

任务：

1. **把 `patches/subsys/` 的 18 个补丁逐条读懂**，按"功能扩展 / 性能优化 / 兼容性 hack"分类，每条写清它改了什么、为什么上游没有、去掉会发生什么。
2. **选三个做 A/B 实测**（把补丁 revert 掉重编，与原版对比）：

| 实验 | 构造的负载 | 预期观测 |
|---|---|---|
| `305` airtime quantum（`weight << 3` vs `weight`） | 多站点并发饱和打流 | 调度遍历开销、CPU 占用、站点间公平性 |
| `320` 广播 AQL 限额 | 大量广播/组播 + 少量单播竞争 | 去掉后单播吞吐塌陷、FQ 失效；`aql_pending` 的 `BC/MC` 变化 |
| `303`/`304` minstrel 速率波动与降级 | 信号由强变弱（拉远或加衰减） | 速率震荡幅度、恢复速度、有效吞吐 |

3. **写成报告**：每个实验含负载构造方法、原始数据、图、以及"为什么维护者要这么改"的解释。

验收：一份可公开发表的技术报告，含三组 A/B 数据。**这是你的第一个 STAR 案例，也是 M7–M8 试水面试时手上唯一的硬货**，务必做扎实。

> 提醒：`320` 那个补丁的 commit message 已经把动机写清楚了（广播洪泛塞满硬件队列、破坏广播数据的 FQ）。你的任务不是重复它的结论，而是**把它量出来**——能给出数字的人和能背出结论的人，在面试里是两个物种。

---

## 五、M4–M6：收发路径打通与卸载理论

### M4（2026-12）发送路径逐跳实测

目标不是"读过一遍"，而是能画出带**实测数字**的路径图。

主链路：

```
ieee80211_subif_start_xmit
  → ieee80211_xmit_fast（快路径）/ 慢路径 tx handlers
  → ieee80211_txq_enqueue（FQ-CoDel，fq_impl.h）
  → ieee80211_next_txq / ieee80211_txq_may_transmit（airtime 调度 + AQL 门限）
  → ieee80211_tx_dequeue
  → 驱动 wake_tx_queue → mt76_tx → mt76_dma_tx_queue_skb → DMA ring → 硬件
```

要搞明白的机制：

- **TXQ 与 FQ-CoDel**：每站点每 TID 一个队列，凭什么能避免 bufferbloat？codel 的 target/interval 在无线上为什么要改？
- **AQL**：它限制的是"排在硬件里的空口时间"而不是包数，为什么必须这样？`aql_txq_limit_low/high` 与 `AQL_THRESHOLD` 的切换逻辑
- **airtime fairness**：`deficit` 怎么累加（就是 `305` 补丁改的地方）、`airtime_weight` 从哪来
- **A-MSDU / A-MPDU**：组包发生在哪一层，BA session 怎么建立、怎么超时
- **`ieee80211_calc_expected_tx_airtime`**：AQL 的输入，估不准会怎样（`360`/`361` 补丁相关）

方法：每一跳加 tracepoint，量出耗时、队列深度、批量大小。跑单流/多流/多站点三种负载各测一遍。

产出：《发送路径解剖（带实测数据）》。

### M5（2027-01）接收路径、速率控制、首个上游 patch

接收链路：

```
硬件中断 → mt76 NAPI poll → mt76_dma_rx_poll → mt76_rx
  → ieee80211_rx_napi → ieee80211_rx_handlers
      去重 → 解密（CCMP/GCMP）→ A-MSDU 解聚合
      → BA reorder buffer（ieee80211_sta_reorder_release，乱序与超时释放）
  → netif_receive_skb / GRO
```

要搞明白的：reorder buffer 满和超时各自的表现是什么；解密失败计数在哪里；`rx_path_lock` 保护什么；GRO 在无线上的收益与副作用。

同时把 **minstrel_ht** 吃透：采样表怎么建、probe 帧怎么插、什么条件下降速、`301`~`304` 补丁分别修了什么病。这块是"AP 明明信号满格但速率很低"这类经典现场问题的答案来源。

**本月必须提交第一个上游 patch。** 不要挑难的，从 M1–M4 过程中真实遇到的小问题入手：一个错误的 debugfs 输出、一处缺失的错误处理、一段过时的注释、一个能明确复现的小 bug。目的是跑通流程、挨一次 review，而不是一战成名。路径见[附录 D](#附录-d上游贡献路径与礼节)。

产出：《接收路径与速率控制解剖》+ 1 个已提交的 patch。

### M6（2027-02）硬件卸载理论与 go/no-go 复核

含春节，强度下调，本月只做读和判断，不做实验。

**WED 原理**。按顺序读：

1. `target/linux/generic/backport-6.12/731-v6.18-net-mediatek-wed-Introduce-MT7992-WED-support-to-MT7*.patch`
2. `732-v6.18-wifi-mt76-wed-use-proper-wed-reference-*.patch`
3. `733-v6.18-net-mtk-wed-add-dma-mask-limitation-and-GFP_DMA32-*.patch`
4. `target/linux/mediatek/patches-6.12/940`~`944-*`（内存区域与 DTS 节点的重构：ILM、DLM、cpuboot）

要能回答的核心问题：

- WED 到底卸载了什么？（无线与以太之间的转发路径绕过主 CPU）
- WO 那颗协处理器在做什么？固件（`mt7981-wo-firmware`）承担哪部分？
- token 机制怎么在主 CPU 与 WED 之间管理 buffer 所有权？
- **哪些帧不能卸载、会回落到 CPU？** 这是面试必问，M7 要实测验证
- 那三个 backport 补丁分别解决了什么工程问题（尤其 `733` 的 DMA mask 与 `GFP_DMA32`，背后是地址空间约束）

**软件流表卸载**：读 `hack-6.12/650-netfilter-add-xt_FLOWOFFLOAD-target.patch`，理解 netfilter flowtable 与 OpenWrt 这个私有 target 的关系，以及它与硬件 PPE/HNAT 卸载的分层。

**go/no-go 复核**：对照[附录 A](#附录-agono-go-检查点) 逐条打分。这是唯一一次成本可控的转向机会——M6 转向出口 A 或 C，前五个月的路径分析和实验床全部仍然有用；M9 之后再转就真的浪费了。

产出：《卸载路径原理笔记》+ 决策结论。

---

## 六、M7–M9：卸载实测、驱动实战、上游冲刺

### M7（2027-03）WED 卸载边界实测

**这是整条路线上最值钱的一个月。** 硬件卸载是每个 WLAN 厂商都在用、但极少有人能讲清边界的东西，能拿数据说话的候选人凤毛麟角。

实验设计：

| 对比项 | 测什么 | 关键观测 |
|---|---|---|
| WED 开 / 关 | 有线↔无线双向转发吞吐、pps、各核 softirq 占比 | CPU 省了多少、吞吐涨了多少 |
| 卸载回落判据 | 逐类构造流量：组播/广播、非 4-addr、VLAN tag、加密例外、管理帧、分片 | 哪些命中卸载、哪些回落，如何从计数器确认 |
| 流表卸载开 / 关 | NAT 转发吞吐 | PPE 流表命中率、老化行为 |
| 卸载下的异常 | 硬件复位、token 泄漏、队列满 | 恢复路径是否干净（`hack-6.12/730` 的 dump 派上用场） |

**"如何确认某个包真的走了卸载路径"是本月的核心技能**，也是这个话题里最容易含糊过去的地方。要落到具体证据：卸载引擎的计数器、CPU 侧收包计数的缺口、PPE 流表 dump、tracepoint。含糊的回答在面试里一追就穿。

产出：《WED 卸载边界实测报告》——含卸载/回落分类表和量化数据。**这份报告的稀缺度高于前面所有产出。**

同时：金三银四窗口，投 5~8 家试水，目的是拿反馈而非跳槽。手上此时有 M3 报告和本月报告，足以支撑一轮有质量的技术面。把被问倒的题直接记下来。

### M8（2027-04）驱动内部实战

从"读驱动"进到"改驱动、修驱动"。

- **DMA 环管理**（mt76 `dma.c`）：环大小、生产消费指针、`GFP_DMA32` 约束、buffer 回收
- **token 机制**：tx token 分配与释放，泄漏时的表象（发送逐渐停止但没有报错）
- **固件接口**（`mcu.c`）：MCU 命令与事件、超时处理、固件加载
- **复位与恢复**：`mt76_reset` 路径，触发条件、状态重建、恢复后为什么有时会丢站点

选一个真实问题做到根因：DMA 队列满、token 泄漏、固件超时、复位后异常、特定 MCS 下的丢包。可以从 mt76 的 GitHub issue 里挑一个能复现的。

同时把 M7 试水面试暴露的短板补掉。

产出：一个驱动层 RCA + 尽量转化成一个 patch。

### M9（2027-05）上游冲刺与影响力

- **patch 目标累计 5 个**（已合并或在评审）：OpenWrt 仓库 2 个、mt76 2 个、mac80211 1 个。mac80211 那个最难也最值钱，素材来自 M3–M5 的路径分析
- **专利 1~2 个**：走公司流程。这个方向可专利的点很多——调度与队列策略、卸载路径优化、速率控制改进、异常检测与自愈
- **公开输出**：M3 报告和 M7 报告脱敏后发成技术文章。这两篇都基于开源代码和自购硬件，合规风险最低
- 补 **Wi-Fi 7 / MLO**：读 `370`/`371` 补丁，搞清 MLO 的链路管理与 eMLSR 的实际收益和代价。这是 2027 年面试的高频话题，值得单独花时间

---

## 七、M10–M12：性能工程收口与求职

### M10（2027-06）端到端性能案例与架构表达

**一个完整的性能工程案例**，串起前面所有能力：从"这台 AP 的吞吐达不到标称"出发，定位到软件的具体某一层（空口调度 / 队列 / 驱动 / 卸载 / 有线侧），优化，量化。这个案例的价值在于展示**你的定位方法是系统性的而不是撞运气的**。

**一份架构文档**：《AP 转发数据面架构与瓶颈分析》。结构按主计划第七章的五段式，选型论证的核心议题是"SoftMAC + 软件转发 / 硬件卸载 / FullMAC 三条路的成本与可控性权衡"，并给出演进路线。这份文档是架构岗面试的直接材料。

### M11（2027-07）材料与面试准备

按主计划执行，D 方向额外做两件事：

1. 过一遍[附录 C](#附录-cd-方向面试题库)，答不上的立即补
2. 把上游 patch 列成一份清单附在简历里，每条注明解决了什么问题。**这是简历上最有分量的一段**，比任何自述都可信

### M12（2027-08）冲刺

按主计划执行。D 方向的目标公司优先级：芯片原厂（联发科、高通）的软件团队 > 头部厂商的预研/平台部门 > 一般厂商的驱动岗。注意芯片原厂的面试更偏硬件细节和固件接口，头部厂商预研更偏规模化和架构。

---

## 八、每月产出物一览

| 月 | 时间 | 硬产出 | 可否公开 |
|---|---|---|---|
| M1 | 2026-09 | 可调试固件 + 一次救砖 + 三条 quilt 流程 + 自加的 debugfs 计数器 | 是 |
| M2 | 2026-10 | 无线性能实验床 v0 + 基线报告（偏差 < 5%） | 是 |
| M3 | 2026-11 | **18 个 mac80211 补丁解析 + 3 组 A/B 实测报告** | 是，建议发表 |
| M4 | 2026-12 | 发送路径解剖（带实测数据） | 是 |
| M5 | 2027-01 | 接收路径与速率控制解剖 + 首个 patch 已提交 | 是 |
| M6 | 2027-02 | 卸载原理笔记 + go/no-go 决策结论 | 是 |
| M7 | 2027-03 | **WED 卸载边界实测报告** + 试水面试反馈 | 是，建议发表 |
| M8 | 2027-04 | 驱动层 RCA + patch | 部分 |
| M9 | 2027-05 | 累计 5 个 patch + 1~2 专利 + 2 篇文章 | patch/文章公开 |
| M10 | 2027-06 | 端到端性能案例 + 架构文档 | 脱敏后可 |
| M11 | 2027-07 | 简历 + 5 个 STAR + patch 清单 | — |
| M12 | 2027-08 | offer | — |

这条路线的一个额外好处：**产出几乎全部基于开源代码和自购硬件，可公开发表的比例远高于其他三条出口**，不用在合规红线上反复纠结。

---

## 附录 A：go/no-go 检查点

M6（2027-02）逐条打分，**任意两条不满足就应当转向出口 A 或 C**。

| # | 检查项 | 判据 |
|---|---|---|
| 1 | 硬件与调试链路 | 能在 30 分钟内完成"改内核 → 编译 → 刷机 → 观测"一整轮 |
| 2 | 数据可信度 | 实验床同条件三次跑偏差 < 5% |
| 3 | 代码理解深度 | 能不看笔记讲出发送路径每一跳，并说出对应的观测入口 |
| 4 | M3 产出质量 | A/B 报告拿给同行看，对方认为有价值而非"读书笔记" |
| 5 | 工作杠杆 | 半年内至少争取到一个性能或转发相关的任务 |
| 6 | 兴趣与耐受度 | 面对一个查了三天没结果的内核问题，仍然想继续 |

第 5 条要特别说明：如果公司完全不碰驱动层，也不是必须立刻放弃——但要清醒认识到进度会慢一倍，并考虑把目标从"一年后跳到驱动岗"改成"一年后跳到愿意让你做驱动的岗"，把这条路当成两年计划的第一年。

第 6 条不是煽情。这条路上"查了很久没结果"是常态而不是例外，如果这种状态让你持续痛苦，那它只是不适合你，与能力无关，早转向是理性选择。

---

## 附录 B：调试手法速查

**改三类代码**

```bash
# 内核
make target/linux/{clean,prepare} V=s QUILT=1
cd build_dir/target-*/linux-*/linux-6.12.94
quilt new 999-my-change.patch && quilt add net/xxx.c && vim ... && quilt refresh
cd - && make target/linux/update

# mac80211（backports 6.18.39，补丁落到 patches/subsys/）
make package/kernel/mac80211/{clean,prepare} V=s QUILT=1

# mt76（git 源，prepare 后才有源码）
make package/kernel/mt76/{clean,prepare} V=s QUILT=1
```

**快速迭代**：只重编模块并 scp 到设备，比整包刷机快得多

```bash
make package/kernel/mt76/compile V=s
# 产物在 build_dir 下，找 .ko 直接 scp，设备上 rmmod/insmod
```

**观测入口**

```bash
# mac80211 状态（打开 MAC80211_DEBUGFS 后）
/sys/kernel/debug/ieee80211/phy0/aqm
/sys/kernel/debug/ieee80211/phy0/aql_pending      # 320 补丁加了 BC/MC 一行
/sys/kernel/debug/ieee80211/phy0/aql_txq_limit    # 可写，支持 "mcast <n>"
/sys/kernel/debug/ieee80211/phy0/airtime
/sys/kernel/debug/ieee80211/phy0/netdev:*/stations/*/

# tracepoint（打开 MAC80211_TRACING 后）
trace-cmd record -e mac80211 -e mt76 ...

# 按需开日志，免重编
echo 'file net/mac80211/tx.c +p' > /sys/kernel/debug/dynamic_debug/control

# 站点与射频
iw dev wlan0 station dump ; iw phy0 info ; iw dev wlan0 survey dump
```

**性能测量纪律**：`lockdep`、tracing、`KASAN` 都会显著改变性能特征。测性能必须用干净内核，测逻辑才用调试内核，两套固件分开保管并在报告里标注用的是哪套。

---

## 附录 C：D 方向面试题库

分三层。第一层答不上说明还没入门，第三层能答说明可以要高职级。

### 第一层：概念与机制

1. SoftMAC 与 FullMAC 的分界在哪？mac80211 承担了哪些 MLME 职责，哪些留给 hostapd？
2. `cfg80211` 与 `mac80211` 的分工？nl80211 在中间是什么角色？
3. A-MPDU 与 A-MSDU 的区别，各自的收益和风险？
4. BA session 是怎么建立的？reorder buffer 超时会造成什么现象？
5. 为什么无线不能直接套用有线的队列管理？bufferbloat 在无线上为什么更严重？

### 第二层：路径与调优

6. 一个下行包从 `ieee80211_subif_start_xmit` 到进入硬件队列，经过哪些队列和调度点？
7. AQL 限制的是什么量？为什么不用包数？不开 AQL 会出现什么现象？
8. airtime fairness 的 deficit 怎么累加？quantum 取太小会有什么后果？
   > 这道题的答案就在 `305` 号补丁里：quantum 太小导致要频繁遍历所有队列才能找到一个可发的，纯属 CPU 浪费。
9. 大量广播流量为什么会压塌单播吞吐？怎么修？
   > 对应 `320` 号补丁：广播塞满硬件队列并破坏了 FQ，解法是给广播也套 AQL 限额。
10. `minstrel_ht` 在什么情况下会误判速率？表象是什么？怎么观测它的决策？
11. 信号满格但速率上不去，你的排查顺序是什么？

### 第三层：卸载与架构

12. WED 卸载了哪条路径？WO 协处理器承担什么？token 机制解决什么问题？
13. **哪些帧不能被卸载、会回落到 CPU？你怎么验证某个包真的走了卸载路径？**
14. 硬件卸载与 netfilter flowtable 是什么分层关系？两者都开会怎样？
15. 卸载路径上出问题（丢包、卡死、复位）时怎么定位？卸载引擎是个黑盒，你的抓手在哪？
16. 为什么这棵树的内核 `CONFIG_MAC80211` 是关掉的？这对你调试有什么影响？
    > 这道题是"你是否真的动手编译过 OpenWrt"的试金石，只读过代码的人答不上来。
17. MLO 相比传统双频并发，真实收益和代价分别是什么？对驱动和固件提出了什么新要求？
18. 如果让你评估"自研驱动 / 用原厂 SDK / 走上游 mac80211+mt76"三条路，你的判据是什么？

### 自我认知

19. 你改过的最难的一个内核问题是什么？怎么定位的？
20. 你提的 patch 被 reject 或要求大改过吗？对方的理由是什么，你怎么看？

---

## 附录 D：上游贡献路径与礼节

**三条渠道，难度递增**

| 渠道 | 方式 | 特点 |
|---|---|---|
| OpenWrt | GitHub PR（`openwrt/openwrt`） | 门槛最低，适合第一个 patch，review 相对宽松 |
| mt76 | GitHub PR（`openwrt/mt76`） | 维护者是 Felix Fietkau（nbd），要求高但反馈直接 |
| mac80211 / cfg80211 | 邮件列表 `linux-wireless@vger.kernel.org` | 最难也最值钱，`git send-email` 纯文本流程 |

注意一个链条关系：mac80211 的改动最终要经由 backports 才能进 OpenWrt。所以你在 `patches/subsys/` 里做的改动，正确的归宿是先推 upstream，再等它随 backports 版本流回来。**面试时能讲清这个链条，本身就说明你在这个生态里真正待过。**

**必须做对的几件事**

- 一个 patch 只做一件事。把"顺手改的格式"混进功能修复是被打回的头号原因
- commit message 写清"为什么"，而不是"改了什么"。看 `320` 那个补丁的写法：先说现象（广播洪泛塞满队列）、再说后果（吞吐问题、FQ 失效）、最后说方案。三句话，是标准范式
- 严格遵守 `checkpatch.pl` 和 `Documentation/process/submitting-patches.rst`
- 被要求改三四轮是正常的，尤其是第一次。**不要把 review 意见当成否定**，这是这个圈子的正常沟通密度
- 不确定要不要发时就发。方向错了会有人直说，比自己憋着强

**从哪找第一个 patch 的素材**：M1–M4 过程中你必然会遇到"这里的注释是错的""这个 debugfs 输出不对""这个错误分支没释放资源"。**随手记下来，不要放过。** 这些小东西恰好是最容易被合并的第一个 patch。
