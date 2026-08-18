# CAPWAP / TR-069 行业解决方案与协议栈工程实践

> 内容涵盖：两个协议在产业中的真实定位与选型、协议原理（状态机 / 报文 / 数据模型）、协议栈自研的工程架构、控制面与数据面的优化手段、双栈共存的配置权威问题，以及在 OpenWrt 上的落地路径。
>
> 配套文档：
> [`ac-ap-networking-guide.md`](./ac-ap-networking-guide.md)（组网方案与转发模式）·
> [`ac-ap-software-architecture.md`](./ac-ap-software-architecture.md)（软件系统架构：OpenWrt · 驱动 · Linux 内核）·
> [`ac-ap-aiops-statistical.md`](./ac-ap-aiops-statistical.md)（统计式 AIOps）
>
> 规范基线：CAPWAP = RFC 5415/5416/5417；TR-069 = Broadband Forum TR-069 Amendment 6（CWMP 1.4）+ TR-106 / TR-181i2。

## 目录

- [一、产业定位与选型](#一产业定位与选型)
- [二、CAPWAP 协议原理](#二capwap-协议原理)
- [三、CAPWAP 协议栈开发实现](#三capwap-协议栈开发实现)
- [四、CAPWAP 优化](#四capwap-优化)
- [五、TR-069 / CWMP 协议原理](#五tr-069--cwmp-协议原理)
- [六、TR-069 协议栈开发实现](#六tr-069-协议栈开发实现)
- [七、TR-069 优化](#七tr-069-优化)
- [八、双栈共存与配置权威](#八双栈共存与配置权威)
- [九、演进：TR-369/USP 与替代方案](#九演进tr-369usp-与替代方案)
- [十、在 OpenWrt 上的落地](#十在-openwrt-上的落地)
- [十一、测试、压测与交付](#十一测试压测与交付)
- [附录 A：CAPWAP 报文与默认参数速查](#附录-acapwap-报文与默认参数速查)
- [附录 B：TR-069 事件码与故障码速查](#附录-btr-069-事件码与故障码速查)
- [附录 C：排障手法](#附录-c排障手法)

---

## 一、产业定位与选型

两个协议经常被放在一起讨论，但它们解决的是**完全不同层次**的问题，不是替代关系。

| | CAPWAP | TR-069 / CWMP |
|---|---|---|
| 定义方 | IETF（RFC 5415，2009） | Broadband Forum（2004 起，Amendment 6 至 2018） |
| 管理对象 | 瘦 AP（WTP） | 家庭网关、ONT、STB、VoIP ATA、企业 CPE |
| 关系 | AC ↔ AP，紧耦合、有状态、实时 | ACS ↔ CPE，松耦合、无常连、准实时 |
| 承载 | UDP 5246/5247 + DTLS | SOAP/HTTP(S)，CPE 主动发起 |
| 是否管数据面 | **是**（5247 是业务隧道） | 否，纯管理面 |
| 典型时间粒度 | 秒级（Echo 30s、Add Station 即时） | 分钟到小时级（PeriodicInform 常为 1~24h） |
| 规模单位 | 单 AC 数百~数千 AP | 单 ACS 十万~千万 CPE |
| 谁在用 | 企业网/园区网 WLAN 厂商 | 电信运营商装维体系 |

### 落地形态

**运营商宽带（TR-069 主场）**：装维流程强绑定 ACS——开户下发 PPPoE 账号、WiFi SSID/密码、远程升级固件、TR-143 测速、故障预诊断。这套体系在中国移动/电信的家宽里是硬性入网要求（配合各省 ITMS 规范，通常在 TR-069 之上叠加运营商私有数据模型分支和 X_ 厂商扩展）。

**企业/园区 WLAN（CAPWAP 主场）**：AC 集中管理 AP 的射频、SSID、认证、漫游，见 [`ac-ap-networking-guide.md`](./ac-ap-networking-guide.md)。注意一个现实：**主流厂商的"CAPWAP"多数只是 RFC 5415 的框架 + 大量 Vendor Specific Payload**，跨厂商 AC/AP 互通基本不成立。所以"支持标准 CAPWAP"在招标文件里的实际价值有限，真正的价值在于自研 AC 时不用从零设计协议。

**运营商的企业 WLAN / FTTR（两者叠加）**：AP 本体通过 TR-069 向 ACS 注册（穿 NAT、跨公网、做固件和账号管理），AP 的射频与用户接入通过 CAPWAP 或私有云管协议对接 AC/云平台。这是最容易出问题的形态，见[第八章](#八双栈共存与配置权威)。

**云管平台（正在吞掉两者）**：新做的产品线越来越多选择"HTTPS/MQTT + 私有 JSON/protobuf"，因为 CAPWAP 穿 NAT 难、TR-069 的 SOAP/XML 在小内存设备上代价高。但存量太大，所以工程上真实的选择往往是：**新平台自研轻量协议，同时保留 TR-069 兼容层以满足运营商入网**。

### 选型判据

1. **要不要管数据面**：需要隧道转发、大二层、集中审计 → CAPWAP（或 VXLAN/L2TPv3 自研隧道）。只管配置和升级 → TR-069/USP 足够。
2. **设备在不在 NAT 后面 / 跨不跨公网**：跨公网 → TR-069（CPE 主动出局）或 USP over MQTT。CAPWAP 跨 NAT 需要额外保活和端口映射，很脆。
3. **规模与实时性**：需要秒级站点表同步（漫游、踢除、Add/Delete Station）→ 必须常连协议，TR-069 做不了。
4. **有没有运营商入网要求**：有 → TR-069 是入场券，没有讨论空间。

---

## 二、CAPWAP 协议原理

### 2.1 两条隧道

```
AP(WTP)                                                   AC
  |                                                        |
  |== UDP 5246  控制隧道（DTLS 加密）====================>  |  状态机、配置、统计、站点表
  |                                                        |
  |== UDP 5247  数据隧道（可选 DTLS）====================>  |  用户业务报文（仅隧道转发模式）
```

控制面是**请求/响应式、有序号、有重传**的可靠层，跑在 UDP 之上自己实现可靠性（因为要复用同一套 DTLS 记录层，且不希望 TCP 的队头阻塞影响心跳）。数据面是纯封装转发，无确认。

本地转发模式下 5247 隧道仍可能建立（用于把部分特殊报文如 802.1X EAPOL、DHCP 上送 AC），但用户数据不走它。

### 2.2 CAPWAP 头

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|CAPWAP Preamble|  HLEN   |   RID   | WBID    |T|F|L|W|M|K|Flags|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Fragment ID          |     Frag Offset       |Rsvd   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              (optional) Radio MAC Address                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          (optional) Wireless Specific Information              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Payload ....                            |
```

关键字段：

- **Preamble**：高 4 位版本（0），低 4 位 Payload Type。`0` = 明文 CAPWAP，`1` = DTLS 封装。这一字节让收包路径可以在解析前就判断该送 DTLS 还是直接解析。
- **HLEN**：头长度，单位 4 字节。**这是自研栈最容易出安全漏洞的地方**——必须校验 `HLEN*4` 落在合法区间且不超过实际报文长度。
- **RID / WBID**：Radio ID（区分 2.4G/5G/6G 射频）、Wireless Binding ID（`1` = IEEE 802.11）。
- **T 位**：Payload 是否为原生 802.11 帧（Split MAC 场景），否则为 802.3 帧。
- **F/L + Fragment ID/Offset**：CAPWAP 自己的分片机制，独立于 IP 分片。
- **W 位**：是否携带 Wireless Specific Information（802.11 绑定里放 RSSI、SNR、Data Rate，这是 AC 侧做射频算法的重要输入）。
- **M 位**：是否携带 Radio MAC。
- **K 位**：Keep-alive，用于数据隧道保活。

基础头 8 字节，带 Radio MAC + WSI 后常见 16~24 字节；加上外层 IPv4(20) + UDP(8)，**隧道开销约 32~48 字节**，这就是 [MTU 问题](./ac-ap-networking-guide.md#mtu--分片)的根源。

### 2.3 控制报文与 TLV

控制报文头之后是消息体：

```
Message Type (4B: 3B IANA Enterprise Number + 1B Type) | Seq Num (1B) | Msg Element Length (2B) | Flags (1B)
然后是若干 Message Element：
  Type (2B) | Length (2B) | Value (...)
```

标准消息类型的 Enterprise Number 为 0，厂商自定义消息用自己的企业号——这是厂商扩展的第一条路径。第二条是 Type 37 `Vendor Specific Payload`（内含 Vendor ID + Element ID + Data）。**实际产品中厂商私有 TLV 的数量通常超过标准 TLV**，射频调优、Portal 联动、日志上报都在这里。

请求/响应用 8 位 Seq Num 配对。8 位空间在高频配置下发时会绕回，实现上必须以"未响应窗口"而非单纯比较大小来判断，否则重传报文会错配到新请求的响应。

### 2.4 状态机

```
                  ┌──────┐
                  │ Start│
                  └───┬──┘
                      v
                  ┌──────┐   收到 AC 地址(静态/DHCP/DNS/组播)
                  │ Idle │──────────────┐
                  └──────┘              v
                                  ┌───────────┐  Discovery Req/Resp
                    ┌────────────>│ Discovery │  选定最优 AC
                    │             └─────┬─────┘
              SilentInterval            v
              后重试                ┌───────────┐
                    │             │ DTLS Setup│  证书/PSK 握手
                    │             └─────┬─────┘
                    │                   v
                    │             ┌───────────┐  Join Req/Resp
                    └─────────────│   Join    │  能力协商、版本比对
                                  └─────┬─────┘
                                        │ 版本不匹配
                                        ├──────────> ┌────────────┐
                                        │            │ Image Data │ 固件下载
                                        │            └──────┬─────┘
                                        v                   │ Reset
                                  ┌───────────┐             v
                                  │ Configure │<────────────┘
                                  │  Config Status Req/Resp │
                                  └─────┬─────┘
                                        v
                                  ┌───────────┐  Change State Event
                                  │ Data Check│  数据隧道 Keep-alive
                                  └─────┬─────┘
                                        v
                                  ┌───────────┐  Echo / WTP Event /
                                  │    Run    │  Config Update /
                                  └─────┬─────┘  Station Config
                                        │ Echo 超时 / Reset / 授权失效
                                        v
                                  ┌───────────┐
                                  │   Reset   │──> Idle
                                  └───────────┘
```

现场排障的关键是**定位卡在哪一态**：

| 卡住的状态 | 现场症状 | 高频原因 |
|---|---|---|
| Idle → Discovery | AP 一直闪灯找不到 AC | Option 43/138 未下发、组播被交换机丢弃、UDP 5246 被 ACL 挡 |
| DTLS Setup | 抓到 Discovery 成功但无 Join | 证书过期/时间不同步（**NTP 未同步导致证书校验失败是经典坑**）、PSK 不一致、MTU 导致握手报文分片被丢 |
| Join | Join Response 带 Failure | AP 未在白名单、AC 授权 License 用尽、AP 型号不被该 AC 版本支持 |
| Image Data | 反复升级重启 | 固件版本策略配错形成升级死循环；镜像分片速率过高打爆 AP flash 写入 |
| Configure | 反复回 Idle | 下发了 AP 不支持的射频参数（国家码/信道/带宽组合非法），AP 拒绝后复位 |
| Run 抖动 | AP 频繁上下线 | Echo 丢失（网络抖动 / AC CPU 打满）、PoE 供电不足、AP 内存泄漏触发看门狗 |

### 2.5 发现机制

四种方式，实现上应按此优先级并行/串行尝试：

| 方式 | 细节 | 适用 |
|---|---|---|
| 静态配置 | AP 本地保存 AC IP 列表 | 现场预配、分支机构固定 AC |
| DHCP 选项 | RFC 5417：DHCPv4 **Option 138**（AC IPv4 地址列表）、DHCPv6 Option 52。厂商实践普遍用 **Option 43** 私有子选项（华为常用 Option 43 sub-option 78，Cisco 用 241） | 跨三层最主流 |
| DNS | 约定名如 `CAPWAP-AC-address.<domain>` | 云 AC、地址会变的场景 |
| 广播/组播 | 二层广播；IPv4 组播 `224.0.1.140`，IPv6 `FF0X::18C` | 同二层域，最省配置 |

**Primary Discovery Request/Response（19/20）** 是常被忽略的机制：AP 在 Run 态下仍会周期探测优先级更高的主 AC，用于主备回切。回切会导致业务瞬断，所以生产环境通常要**关闭自动回切或限定回切时间窗**。

### 2.6 Split MAC 与 Local MAC

RFC 5416 定义了 802.11 功能在 WTP 与 AC 之间的划分：

| 功能 | Split MAC | Local MAC |
|---|---|---|
| Beacon / Probe Response | WTP | WTP |
| 802.11 控制帧（RTS/CTS/ACK） | WTP | WTP |
| Auth / Assoc 管理帧 | **AC** | WTP（结果通知 AC） |
| 加解密（CCMP/GCMP） | AC 或 WTP | **WTP** |
| 分片/重组、聚合 | WTP | WTP |
| 上送 AC 的帧格式 | 802.11（T=1） | 802.3（T=0） |
| RSN/802.1X 状态机 | AC | WTP（AC 只做 RADIUS 代理） |

**工程结论：Split MAC 已基本淘汰。** Wi-Fi 6/7 下把原生 802.11 帧隧道到 AC 做加解密，AC 的算力和时延都撑不住（还会破坏 A-MPDU/BlockAck 的时序要求）。现在几乎全是 Local MAC + 数据隧道封装 802.3 帧。对应到 OpenWrt，`hostapd` 就在 AP 本地跑完整的 auth/assoc/4-way handshake，AC 只通过 ubus 拿结果，见 [`ac-ap-software-architecture.md`](./ac-ap-software-architecture.md#三l2hostapd--wpa_supplicant)。

### 2.7 Run 态的四类交互

- **Echo Request/Response（13/14）**：心跳，默认 30s。丢失 `MaxRetransmit` 次判死。
- **Configuration Update Request/Response（7/8）**：AC 主动下发配置变更（射频参数、SSID、ACL）。
- **WTP Event Request/Response（9/10）**：AP 上报事件与统计（射频统计、站点统计、告警、Delete Station 通知）。**这是 AIOps 的数据入口**。
- **Station Configuration Request/Response（25/26）**：站点表同步——Add Station（含 Session Key、QoS、VLAN）、Delete Station。漫游性能强依赖这条路径的时延。

---

## 三、CAPWAP 协议栈开发实现

### 3.1 进程与线程模型

AC 侧承载数千 AP，模型选择直接决定上限。

**推荐：控制面单线程事件循环 + 分片（sharding）**

```
                 ┌─ 监听 socket (UDP 5246, SO_REUSEPORT) ─┐
                 │                                        │
       ┌─────────v────────┐                    ┌──────────v───────┐
       │  worker 0        │       ...          │  worker N-1      │
       │  epoll 事件循环   │                    │  epoll 事件循环   │
       │  WTP 会话表分片 0 │                    │  WTP 会话表分片 N-1│
       │  时间轮定时器     │                    │  时间轮定时器      │
       └─────────┬────────┘                    └──────────┬───────┘
                 └──────────── 无锁：每个 WTP 只属于一个 worker ─┘
                                        │
                              ┌─────────v─────────┐
                              │ 配置/状态存储（DB） │  异步、批量提交
                              └───────────────────┘
```

要点：

- **`SO_REUSEPORT` + 内核按四元组哈希**，保证同一 AP 的报文始终落到同一 worker，会话表无需加锁。这比"一个线程收包再派发"少一次跨核队列，且天然避开惊群。
- **会话表分片**后，AP 上下线不影响其他分片，锁竞争消失。跨分片的操作（如漫游时 AP-A 的站点要通知 AP-B）走无锁消息队列。
- 数据库/持久化操作**永不在事件循环里同步做**。一次同步 SQL 就能让整个分片的 Echo 超时，进而引发批量掉线。

避免的做法：每 AP 一线程（数千线程调度开销 + 内存）、全局大锁保护会话表（上千 AP 时锁成为瓶颈）。

### 3.2 状态机实现：表驱动

不要用 `switch` 嵌套 `if`。RFC 5415 有 13 个状态、24 种消息类型，手写分支会在两个版本后不可维护。

```c
typedef enum { ST_START, ST_IDLE, ST_DISCOVERY, ST_DTLS_SETUP, ST_JOIN,
               ST_IMAGE_DATA, ST_CONFIGURE, ST_DATA_CHECK, ST_RUN,
               ST_RESET, ST_DEAD, ST_MAX } capwap_state_t;

typedef struct {
    capwap_state_t next;
    int (*action)(struct wtp_session *s, struct capwap_msg *m);
} transition_t;

/* [当前状态][消息类型] -> 迁移 */
static const transition_t fsm[ST_MAX][MSG_TYPE_MAX] = {
    [ST_JOIN][MSG_CONFIG_STATUS_REQ] = { ST_CONFIGURE, on_config_status_req },
    [ST_RUN][MSG_ECHO_REQ]           = { ST_RUN,       on_echo_req },
    [ST_RUN][MSG_WTP_EVENT_REQ]      = { ST_RUN,       on_wtp_event },
    /* 未填充项 = 非法迁移，统一计数并丢弃 */
};
```

好处：非法迁移自动被拒绝（防止乱序/伪造报文推错状态）、可以自动生成状态迁移图和统计计数器、新增厂商消息只是加一行。

### 3.3 定时器：时间轮

每个 WTP 至少 4 个定时器（Echo、重传、状态等待、统计），5000 AP 就是 2 万个定时器，且以"重置"操作最频繁（每收一个 Echo 就重置）。

- **红黑树/最小堆**：插入删除 O(log n)，重置需要先删再插。
- **分层时间轮**：O(1) 插入/删除/重置，是这个场景的正确选择。粒度取 100ms，两级轮（秒级 + 分钟级）足够覆盖 Echo(30s) 到 Idle Timeout(300s)。

### 3.4 TLV 解析：安全第一

协议栈漏洞几乎全部出在这里。**每一个 TLV 解析函数都必须先做三项校验**：

```c
static int tlv_iter_next(struct tlv_iter *it, uint16_t *type,
                         uint16_t *len, const uint8_t **val)
{
    if (it->pos + 4 > it->end)              return -1;  /* 头不完整 */
    *type = ntohs(*(uint16_t *)(it->buf + it->pos));
    *len  = ntohs(*(uint16_t *)(it->buf + it->pos + 2));
    if (it->pos + 4 + *len > it->end)       return -1;  /* 长度越界 */
    if (*len > TLV_MAX_LEN(*type))          return -1;  /* 超过该类型上限 */
    *val = it->buf + it->pos + 4;
    it->pos += 4 + *len;
    return 0;
}
```

配套约束：

- **定长 TLV 必须校验精确长度**，而不是 `>=`。历史上多个厂商 AC 的远程代码执行漏洞源于对 `WTP Board Data` 里的字符串 TLV 直接 `strcpy`。
- 字符串类 TLV 一律按 `len` 拷贝并手动补 `\0`，永不信任报文内的终止符。
- 用**声明式 TLV 表**（type → 最小/最大长度、是否必选、解析回调）生成解析器，避免逐个手写。这同时让"必选 TLV 缺失"的检查变成表遍历。
- 对未知 TLV：控制报文中未知的**必选**元素应回 Failure，未知的可选元素静默忽略并计数——不要因为对端多带一个私有 TLV 就断开会话，这是跨版本兼容的关键。

### 3.5 DTLS 集成

DTLS 是 CAPWAP 落地时最大的性能与稳定性来源。

- **不要让 DTLS 库自己管 socket**。使用 OpenSSL/mbedTLS 的 memory BIO（`BIO_s_mem`）模式，把收发交给自己的事件循环，否则无法与 epoll 模型统一。
- **握手是 CPU 密集的**（RSA-2048 签名/验签约几 ms，ECDSA P-256 好一些）。5000 AP 同时握手，单核只能做几百个/秒 → 必须限速（见 [4.1](#41-注册风暴)）。有条件用**硬件加速**（QAT / ARMv8 Crypto Extensions）。
- **Session Resumption / Session Ticket** 能把重连握手从完整握手降到 1-RTT 且省掉非对称运算，对"AC 重启后全网重连"场景收益极大。RFC 5415 允许 DTLS session resumption，但**很多厂商实现没做**，这是自研时低成本的差异化点。
- **PMTU**：DTLS 握手报文（尤其带证书链）容易超 MTU。必须启用 DTLS 层分片（`SSL_set_mtu` / `DTLS_set_link_mtu`），并把 MTU 设保守值（如 1400），否则在有隧道的中间网络上握手会神秘失败。证书链要精简，别把整条 CA 链塞进去。
- **DoS 防护**：DTLS 的 `HelloVerifyRequest`（cookie）机制必须启用，否则伪造源 IP 就能让 AC 为每个假请求分配握手状态。OpenSSL 里对应 `SSL_CTX_set_cookie_generate_cb`。
- **要不要开 DTLS**：控制面建议开（配置和密钥明文传输不可接受）。数据面（5247）在企业内网通常**关闭**——对每个用户报文做 DTLS 的成本无法接受，且业务本身已有 WPA2/3 和上层 TLS。

### 3.6 数据面：三条实现路径

隧道转发的封装/解封装在哪做，决定了 AC 的吞吐天花板。

| 路径 | 吞吐量级（单机参考） | 复杂度 | 适用 |
|---|---|---|---|
| 用户态 socket 收发 | 百 Mbps~1Gbps | 低 | 只做控制面 + 少量特殊报文上送 |
| **内核模块 / 虚拟网卡** | 5~20Gbps | 中 | 主流商用 AC 的做法 |
| DPDK / XDP 旁路 | 40~100Gbps+ | 高 | 大容量集中式 AC、云 AC 网关 |

**推荐的内核态实现形态**：注册一个虚拟网卡类型（参照内核 `drivers/net/vxlan.c` 的结构），封装时在 `ndo_start_xmit` 里加 CAPWAP 头 + UDP/IP 头，解封装走 `udp_tunnel` 框架的 `encap_rcv` 回调。这样自动获得：

- **GRO/GSO 支持**：内核的 UDP tunnel 框架支持 `SKB_GSO_UDP_TUNNEL`，网卡可以做隧道内层的 TSO。这是隧道性能最大的单项来源，能差 3~5 倍。
- **`skb` 头部空间预留**：`needed_headroom` 设够，避免每包 `skb_realloc_headroom`（一次重分配就吃掉大半性能）。
- 与 conntrack / tc / nftables 的天然协作。

**用 eBPF 也是可行路径**：TC egress 挂 `bpf_skb_adjust_room` + `bpf_redirect` 做封装，好处是不用维护内核模块和 out-of-tree 编译，坏处是复杂 TLV/分片逻辑在 eBPF 里写起来受限。适合"封装格式固定、逻辑简单"的场景。

### 3.7 AP（WTP）侧实现要点

- **必须与业务解耦**：CAPWAP 客户端进程崩了不能带崩转发。OpenWrt 上让它由 procd 托管、`respawn` 自动重启，配置落盘到 UCI 后由 netifd/hostapd 生效，见 [`procd-analysis.md`](./procd-analysis.md)。
- **逃生（Fail-open）是硬需求**：与 AC 断连后保持最后一次配置继续本地转发服务，而不是关射频。实现上要求配置**持久化落盘**且有版本号，而不是只存在内存里。
- **内存足迹**：AP 上常见 128MB DDR，CAPWAP 客户端应控制在 5MB RSS 以内。避免为每个统计项分配小对象，用固定大小的环形缓冲聚合上报。
- **配置应用的原子性**：Configuration Update 涉及多个子系统（射频、SSID、VLAN、ACL）。应先全量校验再统一 apply，失败整体回滚并回 Failure，绝不允许"改了一半"——半套配置的 AP 是现场最难排查的故障。

---

## 四、CAPWAP 优化

### 4.1 注册风暴

**这是 AC 的头号可用性风险。** AC 重启或上行链路闪断后，N 台 AP 在同一时刻重连：DTLS 握手打满 CPU → Join 超时 → AP 重试 → 更多并发 → 正反馈，系统永远收敛不了。

三层防护，缺一不可：

1. **AP 侧退避随机化**：`DiscoveryInterval`、`SilentInterval` 加 ±50% 抖动，重试用指数退避（上限如 120s）。仅这一项就能把峰值削掉一半以上。
2. **AC 侧准入限速**：对 DTLS 握手和 Join 分别设令牌桶（例如握手 200/s、Join 500/s），超出的**直接静默丢弃**而不是回 Failure——回 Failure 反而促使 AP 立即重试。
3. **准入优先级**：已有会话的 Echo/Configuration 报文优先于新 Join。实现上把控制报文分两个队列，Run 态会话的报文永远先处理。**保住在线的比接新的重要**，否则会出现"老 AP 被挤掉、新 AP 又进不来"的雪崩。

容量估算：

```
DTLS 握手能力 ≈ 核数 × 单核每秒非对称运算次数 / 每次握手运算次数
全网恢复时间 ≈ AP 总数 / min(握手限速, 握手能力) + 配置下发时间
```

对 3000 AP、限速 200/s，恢复时间约 15s + 配置下发；若无限速，实测可能 10 分钟都收敛不了。**验收时必须做"AC 冷启动全网恢复"测试并给出时间承诺。**

### 4.2 心跳与假死检测

- Echo 间隔（默认 30s）+ MaxRetransmit（5）× RetransmitInterval（3s）意味着**最坏 45s 才发现 AP 掉线**。对语音业务太慢，但调太激进会在网络抖动时误判。
- 推荐：**Echo 间隔 10~30s 可配，且与"数据面 Keep-alive"结合判断**。控制面丢包但数据面正常 → 优先怀疑 AC 自身处理慢，不要立刻踢会话。
- **Echo 处理必须走快速路径**：在解析完 CAPWAP 头和消息类型后立即回应，不要经过完整的 TLV 解析和状态机。AC 高负载时，Echo 处理慢会导致自我加速的掉线雪崩。
- AC 侧对 Echo 做**批量时间轮到期处理**，而不是每个会话一个定时器回调里做重活。

### 4.3 数据面性能

| 手段 | 收益 | 注意 |
|---|---|---|
| GRO/GSO + 隧道 offload | 3~5× | 需要网卡支持 `tx-udp_tnl-segmentation`，`ethtool -k` 确认 |
| RSS + CPU 亲和 + 中断绑核 | 2~3×（多核扩展性） | 隧道流量的外层四元组单一（AP IP ↔ AC IP），**RSS 会哈希到同一队列**。需要网卡支持内层哈希，或按 AP 分配不同源端口打散 |
| 批量收发（`recvmmsg`/`sendmmsg`、XDP batch） | 1.5~2×（用户态路径） | 减少系统调用 |
| 预留 headroom、避免重分配 | 1.3~2× | `needed_headroom` |
| 硬件流表 offload（MTK PPE / QCA NSS） | 大幅提升 | **绝大多数硬件 offload 不认识自定义隧道封装**，一旦走 CAPWAP 封装就退回软件转发。这是很多国产平台上"跑隧道性能掉一个数量级"的真实原因 |

**"外层四元组单一导致 RSS 失效"是最常被忽略的一条。** 表现为：10 个核只有 1 个核 100%，总吞吐卡在 1~2Gbps。解决办法是让每个 AP 的隧道使用不同的 UDP 源端口（AC 侧按会话分配），使外层四元组分散。

### 4.4 MTU 与分片

优先级从高到低：

1. **中间链路开巨帧**（1600+）——根治，代价是要能改交换机。
2. **TCP MSS clamping**（钳到 1360~1400）——覆盖 90% 的实际业务（HTTP/HTTPS），成本最低。
3. **CAPWAP 层分片**（F/L 位）——协议自带，但分片重组在 AC 上是 CPU 和内存开销，且丢一片全丢。**只应作为兜底，不应是常态**。
4. **依赖 IP 分片**——最差，中间设备常直接丢 IP 分片。DF 位 + ICMP 需要网络放通 ICMP，而现实中经常被防火墙挡掉，形成典型的 PMTU 黑洞。

自研栈应主动使用 **MTU Discovery Padding（TLV type 32）** 在 Join 阶段探测路径 MTU，并把结果作为该会话的封装上限，而不是全网写死一个值。

### 4.5 大规模配置下发

- **增量而非全量**：Configuration Update 只带变化的 TLV。全量下发 1000 个 AP 会同时引发 1000 次射频重启（业务瞬断）。
- **配置版本号 + 摘要比对**：AP 在 Join 时上报当前配置摘要，AC 比对后决定是否下发。这能让 AP 重连时跳过配置阶段，显著缩短恢复时间。
- **分批 + 灰度**：按 AP Group 分批，每批不超过 10~20%，批间留观察窗。射频类变更（信道/功率/带宽）尤其要错峰，因为会导致关联终端重连。
- **区分"需重启射频"与"不需重启"的参数**：ACL、限速、VLAN 映射通常可热更新；国家码、带宽、信道必须重启射频。把它们混在一个下发事务里，等于每次改 ACL 都断一次业务。

### 4.6 高可用

- **AC 双机**：VRRP 提供 IP 漂移，但**会话状态不同步则切换后全网重新 Join**（等于一次注册风暴）。真正的热备需要同步 WTP 会话、DTLS 上下文（难，通常放弃）、站点表。工程折中：控制面会话不同步（接受切换后重连），但**配置和站点表同步**，并配合 [4.1](#41-注册风暴) 的限速保证收敛时间可控。
- **AP 双 AC 注册**：AP 同时知道主备 AC 地址，主 AC 判死后立即切备，不重新走 Discovery，能省掉几十秒。
- **关闭或限制主 AC 自动回切**（见 [2.5](#25-发现机制)）。
- **AP 逃生 + 配置持久化**（见 [3.7](#37-apwtp-侧实现要点)）。

---

## 五、TR-069 / CWMP 协议原理

### 5.1 会话模型：CPE 永远是客户端

这是理解 TR-069 一切设计的钥匙。**CPE 在 NAT 后面，ACS 无法主动连它**，所以协议规定：所有会话由 CPE 发起，ACS 想干什么都得排队等 CPE 上来。

```
CPE                                                    ACS
 |  HTTP POST + SOAP: Inform (含 EventCode, 设备标识)    |
 |----------------------------------------------------->|
 |  200 OK + InformResponse (MaxEnvelopes=1)            |
 |<-----------------------------------------------------|
 |  HTTP POST, body 为空  ← "我说完了，你有事吗？"        |
 |----------------------------------------------------->|
 |  200 OK + GetParameterValues 请求                    |
 |<-----------------------------------------------------|
 |  HTTP POST + GetParameterValuesResponse              |
 |----------------------------------------------------->|
 |  200 OK + SetParameterValues 请求                    |
 |<-----------------------------------------------------|
 |  HTTP POST + SetParameterValuesResponse (Status=0/1) |
 |----------------------------------------------------->|
 |  **204 No Content**  ← 会话结束                       |
 |<-----------------------------------------------------|
```

要点：

- 一次 HTTP 连接里多次 POST/响应，靠 **HTTP Keep-Alive + Cookie** 维持会话上下文。**Cookie 处理是互通性问题的高发区**——不少 ACS 依赖 Cookie 做会话绑定，CPE 侧 HTTP 客户端若不保存 Cookie，就会表现为"每个 RPC 都被要求重新 Inform"。
- **空 POST 是必需的**，不是可选优化。它是 CPE 交出话语权的信号。
- `MaxEnvelopes` 恒为 1：一个 HTTP 消息里只能有一个 SOAP 请求，不能流水线。这直接决定了 TR-069 的吞吐特性——**RPC 数量 × RTT = 会话时长**，取参数要尽量批量。
- 会话结束标志是 HTTP **204**（或 200 空 body）。

**ACS 主动触发**：`Connection Request` —— ACS 对 CPE 的 `ManagementServer.ConnectionRequestURL` 发 HTTP GET（Digest 认证），CPE 收到后立刻发起一次带 `6 CONNECTION REQUEST` 事件的会话。在 NAT 后失效，补救方案：

| 方案 | 规范 | 现实评价 |
|---|---|---|
| STUN + UDP Connection Request | TR-069 Annex G | 复杂、对 NAT 类型敏感，逐渐弃用 |
| XMPP Connection Request | TR-069 Amendment 5 Annex K | 需要 XMPP 服务器，运营商部署过，较重 |
| 缩短 PeriodicInform | —— | 最常见的"土办法"，代价是 ACS 侧负载线性上升 |
| 长连接（私有/MQTT/WebSocket） | 非 TR-069 | 新平台的实际选择，也是 USP 的方向 |

### 5.2 RPC 方法集

| 方向 | 方法 | 说明 |
|---|---|---|
| ACS → CPE | `GetParameterNames` | 遍历参数树（`NextLevel` 控制是否递归）。全树遍历在大数据模型上很慢 |
| | `GetParameterValues` | 批量取值，**必须支持一次取多个** |
| | `SetParameterValues` | **必须原子**：全成功或全失败。返回 Status=0（已生效）或 1（需重启生效） |
| | `GetParameterAttributes` / `SetParameterAttributes` | 通知属性与访问控制 |
| | `AddObject` / `DeleteObject` | 多实例对象增删，返回分配的 InstanceNumber |
| | `Download` / `Upload` | 固件、配置文件、日志。FileType 定义了 `1 Firmware Upgrade Image`、`3 Vendor Configuration File` 等 |
| | `Reboot` / `FactoryReset` | |
| | `ScheduleInform` | 让 CPE 在 N 秒后再来一次 |
| | `ScheduleDownload` / `CancelTransfer` | Amendment 3+ |
| | `ChangeDUState` | TR-157 软件模块管理 |
| CPE → ACS | `Inform` | 每次会话第一个 RPC，带事件码和 DeviceId、ParameterList |
| | `TransferComplete` | 下载/上传完成（**升级重启后仍必须上报**，见 [6.5](#65-download-与固件升级)） |
| | `AutonomousTransferComplete` | 非 ACS 触发的传输完成 |
| | `RequestDownload` | CPE 请求 ACS 给文件 |
| | `Kicked` | 与 Portal/门户联动（少用） |

### 5.3 数据模型

```
TR-106  数据模型模板与命名/寻址规则（Object.{i}. 实例寻址、Alias）
  ├── TR-098  InternetGatewayDevice:1   —— 老根模型，存量巨大
  └── TR-181 Issue 2  Device:2          —— 现行根模型
        ├── TR-104  VoIP
        ├── TR-135  STB
        ├── TR-140  存储
        ├── TR-143  诊断（吞吐测试、Ping、Traceroute、UDPEcho）
        └── TR-157  组件对象（软件模块、Bulk Data Collection、Alias）
```

参数命名如 `Device.WiFi.SSID.1.SSID`、`InternetGatewayDevice.LANDevice.1.WLANConfiguration.1.PreSharedKey.1.KeyPassphrase`。

工程上几个关键规则：

- **实例号必须跨重启持久**。`Device.WiFi.SSID.3.` 重启后不能变成 `.1.`——ACS 侧存的是完整路径，实例号漂移会导致 ACS 下发到错误对象。这是自研栈最容易踩的坑：用数组下标当实例号，删一个中间元素后全部错位。
- **实例号不可复用**（至少在合理时间内）。删掉 `.3.` 后新增应给 `.4.`。
- **Alias-Based Addressing**（Amendment 5 / TR-157）用 `[Alias]` 稳定别名代替实例号寻址，能根治漂移问题，但要求 ACS 也支持。
- 参数类型严格：`string`、`int`、`unsignedInt`、`boolean`、`dateTime`、`base64`。类型不匹配要回 `9006`。
- 运营商模型分支：中国的 ITMS 规范普遍在标准模型上加 `X_CT-COM_`、`X_CMCC_` 等厂商/运营商扩展节点。**入网测试考的主要是这些扩展节点的完整性**，而不是标准部分。

### 5.4 通知机制

通过 `SetParameterAttributes` 为参数设置 `Notification`：

| 值 | 含义 |
|---|---|
| 0 | 关闭 |
| 1 | Passive：值变了，**等下次会话**顺便在 Inform 的 ParameterList 里报 |
| 2 | Active：值变了，**立即发起会话**（带 `4 VALUE CHANGE` 事件） |
| 3~6 | 加上 Lightweight Notification（UDP 单向轻量上报，Amendment 5） |

**Active Notification 是 ACS 被打爆的常见原因**：给 `Device.WiFi.AccessPoint.1.AssociatedDeviceNumberOfEntries` 这类高频变化的参数设了 Active，每个终端上下线都触发一次全量 CWMP 会话。设计时应把高频参数明确列入"禁止 Active"清单，或在设备侧做变更聚合与最小间隔限制。

### 5.5 状态与安全

- **ParameterKey**：ACS 在 `SetParameterValues`/`Download` 里带一个不透明字符串，CPE 持久保存并在每次 Inform 中回报。ACS 靠它判断"我下发的配置是否真的生效了、设备有没有被恢复出厂"。**必须持久化，且只在配置真正成功应用后才更新**。
- **BOOTSTRAP vs BOOT**：`0 BOOTSTRAP` 表示"我是第一次连这个 ACS（或被恢复出厂/ACS URL 变了）"，ACS 会下发全量初始配置；`1 BOOT` 只是普通重启。**把两者搞混（每次重启都报 BOOTSTRAP）会导致 ACS 反复全量下发**，在十万级规模下能把 ACS 打瘫。判定依据是持久化的一个标志位 + 上次成功连接的 ACS URL 哈希。
- **认证**：CPE→ACS 用 HTTP Basic/Digest（`ManagementServer.Username/Password`）或 TLS 客户端证书；ACS→CPE 的 Connection Request 用 Digest（`ConnectionRequestUsername/Password`）。
- **TLS**：现在应强制 TLS 1.2+ 并校验证书。历史设备普遍存在"不校验证书"或"证书过期仍连"的问题，是真实的中间人风险。**与 CAPWAP 一样，NTP 未同步导致证书校验失败是首次上线失败的经典原因**——实现上应允许"首次连接时若时间明显不可信，先 NTP 同步再校验"。

---

## 六、TR-069 协议栈开发实现

### 6.0 分层与模块划分

```
┌──────────────────────────────────────────────────────────┐
│ 会话调度器  事件队列(持久化) · PeriodicInform · 重试退避     │
├──────────────────────────────────────────────────────────┤
│ HTTP 客户端  Keep-Alive · Cookie · Digest · TLS           │
├──────────────────────────────────────────────────────────┤
│ SOAP 编解码  流式 XML 解析 · 信封构造 · Fault 生成         │
├──────────────────────────────────────────────────────────┤
│ RPC 分发     每个方法一个 handler · 参数校验               │
├──────────────────────────────────────────────────────────┤
│ 数据模型引擎  参数树 · 实例管理 · 类型与权限 · 事务         │
├──────────────────────────────────────────────────────────┤
│ 后端适配层    UCI / ubus / netlink / 厂商 SDK              │
└──────────────────────────────────────────────────────────┘
```

**后端适配层必须存在且薄。** 数据模型引擎不应直接调 `uci_set()`，而应通过统一的 get/set 回调，这样才能让"数据模型定义"变成数据（表或 JSON），而不是代码。业界的成熟做法（如 icwmp + libbbfdm）正是把整棵树描述成静态表 + 回调。

### 6.1 参数树的数据结构

需求：数万个参数、支持通配/前缀查询、支持动态实例、内存受限。

**不要用哈希表存全路径。** 路径字符串本身就占几十字节，数万参数光键就是 MB 级。正确做法是**树 + 静态描述表**：

```c
struct dm_param {
    const char *name;          /* 叶子名，如 "SSID" */
    uint8_t     type;          /* DM_STRING / DM_UINT / DM_BOOL ... */
    uint8_t     flags;         /* WRITABLE | NOTIF_CAPABLE | FORCED_INFORM */
    int (*get)(struct dm_ctx *, char **out);
    int (*set)(struct dm_ctx *, const char *val);
};

struct dm_object {
    const char *name;                    /* 如 "SSID" */
    uint8_t     flags;                   /* MULTI_INSTANCE | ADDDEL */
    const struct dm_object *children;
    const struct dm_param  *params;
    int (*browse)(struct dm_ctx *, dm_inst_cb cb);   /* 枚举实例 */
    int (*add)(struct dm_ctx *, uint32_t *new_inst);
    int (*del)(struct dm_ctx *, uint32_t inst);
};
```

- 静态部分（`const` 表）放 `.rodata`，在 flash 上而非 RAM 里，对 64/128MB 设备至关重要。
- 动态实例通过 `browse` 回调**按需枚举**（从 UCI/ubus 现场读），不做常驻缓存。缓存只在单次会话内有效。
- 路径解析用**逐段匹配**下降，天然支持前缀查询和 `GetParameterNames` 的 `NextLevel`。

### 6.2 实例号持久化

核心问题：数据模型的实例号是 TR-069 的稳定标识，而后端（UCI section、内核 ifindex）没有这个概念。

做法：维护一张持久映射表（单独的 UCI 文件或 `/etc/cwmp/instances`）：

```
Device.WiFi.SSID.  ->  { 1: wireless.@wifi-iface[0]/@name=ap0,
                         2: wireless.@wifi-iface[1]/@name=ap1,
                         5: wireless.@wifi-iface[2]/@name=guest }
next_instance = 6
```

规则：

- 后端出现新对象（如有人用 LuCI 加了个 SSID）→ 分配 `next_instance++` 并落盘。
- 后端对象消失 → 映射项标记删除，但**不回收该实例号**。
- 映射表和 `next_instance` 必须与配置一起备份/恢复；恢复出厂时清空并触发 `0 BOOTSTRAP`。

### 6.3 SetParameterValues 的原子性

规范要求全成功或全失败，且要么立即生效（Status=0）要么重启后生效（Status=1）。实现分三阶段：

```
阶段 1  校验：逐个参数检查存在性(9005)、类型(9006)、值域(9007)、可写(9008)
        任一失败 → 返回 SetParameterValuesFault，列出所有失败项，不做任何改动
阶段 2  暂存：写入事务缓冲区（UCI 的 delta 未 commit 正好适配这个模型）
阶段 3  提交：uci_commit + 通知各子系统 reload；失败则回滚整个事务
```

坑点：

- **Fault 必须列出所有失败参数**，不能遇到第一个就返回。ACS 依赖完整列表做诊断。
- 阶段 3 不可能真正原子（多子系统），所以要**尽可能把不确定的检查前移到阶段 1**。例如设 SSID 密码：长度、字符集、加密方式兼容性都在阶段 1 校验完。
- 跨参数依赖（如同时改 `Enable=false` 和 `Channel=165`）必须在阶段 1 做组合校验，不能逐参数独立判断。

### 6.4 XML/SOAP 解析：流式而非 DOM

在 64MB 内存的 CPE 上，一个 `GetParameterValues` 响应可能有上万个参数、几 MB 的 XML。用 DOM 解析（libxml2 全树）会直接 OOM。

- **解析**：用流式/SAX 或极简解析器（`libmicroxml`、`expat`）。CWMP 的 SOAP 结构简单固定，一个 500 行的手写状态机解析器就够，且内存恒定。
- **生成**：**边生成边发送**，不要先在内存里拼完整个 XML。HTTP 用 chunked encoding，参数树遍历时逐个 `write` 到 socket。这把内存占用从 O(参数数) 降到 O(1)。
- **响应分片**：对超大结果，规范允许通过多次会话或让 ACS 用更精确的路径查询来缓解。设备侧应对单次响应设上限，超限返回 `9004 Resources exceeded` 而不是 OOM 重启——**OOM 重启会触发新的 BOOT 事件和新会话，形成死循环**。

### 6.5 Download 与固件升级

```
ACS: Download(FileType=1 Firmware Upgrade Image, URL, Username, Password, FileSize, TargetFileName, DelaySeconds)
CPE: DownloadResponse(Status=1 表示"稍后完成")
     ↓ 下载（HTTP/HTTPS/FTP）
     ↓ 校验（大小、签名、型号匹配、版本回退策略）
     ↓ 写 flash → 重启
     ↓ 重启后：Inform 携带 "1 BOOT" + "7 TRANSFER COMPLETE" + "M Download"
CPE: TransferComplete(CommandKey, FaultStruct, StartTime, CompleteTime)
ACS: TransferCompleteResponse
```

工程要点：

- **`TransferComplete` 必须在重启后仍能上报**，所以传输状态（CommandKey、开始时间、结果）**必须在下载前就持久化**。这是入网测试的必考项，也是最常见的失败点。
- **未确认的 TransferComplete 要一直重试**，直到收到 `TransferCompleteResponse` 才能删除记录。事件队列必须持久化。
- 校验顺序：先校验再写 flash。签名校验、机型匹配、`FileSize` 比对缺一不可——远程刷错固件等于批量变砖。
- **双 Bank + 回滚**：写备份分区，重启后由 bootloader 判断新分区是否成功启动（看门狗计数），失败自动回切。没有这个机制的远程升级在十万级规模下必然产生批量砖机。
- `DelaySeconds` 与分批：ACS 侧要错峰，否则十万台同时下载能把文件服务器和出口带宽打满。设备侧应对下载失败做指数退避。

### 6.6 事件队列与重试

事件队列是 TR-069 客户端的核心状态，必须**持久化**（掉电不丢）：

- 待上报事件（BOOT / VALUE CHANGE / TRANSFER COMPLETE / M Reboot 等）
- 每个事件的 CommandKey
- 重试计数

**重试退避规范**（TR-069 表 3-2）：会话失败后按区间随机重试，第 1 次 5~10s，之后每次乘 2，最多到第 10 次（约 2560~5120s 区间）。**必须实现随机化**，否则全网设备的重试会同步（见 [7.1](#71-inform-风暴)）。

---

## 七、TR-069 优化

### 7.1 Inform 风暴

场景：某地区停电恢复，10 万台 CPE 同时上电 → 同时 `1 BOOT` Inform → ACS 崩 → 全部按同一退避曲线重试 → 再崩。

| 措施 | 位置 | 效果 |
|---|---|---|
| PeriodicInform 随机化 | CPE | 上报时刻在周期内均匀分散（对周期性负载有效，对停电恢复无效） |
| 首次 Inform 随机延迟（0~N 分钟） | CPE | **对停电恢复最有效**，直接把峰值摊平 |
| 重试区间随机化 | CPE | 防止重试同步（规范已要求，但不少实现写死了固定值） |
| ACS 前端限流 + 503 + Retry-After | ACS | 保护后端；CPE 侧必须正确处理 503 而不是当成失败疯狂重试 |
| ScheduleInform 主动调度 | ACS | 把设备的下次上报错峰安排开 |
| 会话准入队列 | ACS | 优先处理有待办任务的设备，纯心跳 Inform 快速 204 打发走 |

**ACS 容量估算**：

```
稳态会话速率 = 设备数 / PeriodicInformInterval
```

10 万设备 + 1 小时周期 ≈ 28 会话/秒稳态。但峰值（停电恢复、批量升级）可达稳态的 50~100 倍，**容量必须按峰值设计或按限流兜底**。这就是运营商把 `PeriodicInformInterval` 设成 24 小时甚至更长的原因——它是被 ACS 容量倒逼出来的，不是技术偏好。

### 7.2 会话时长与 RPC 数量

`MaxEnvelopes=1` 意味着每个 RPC 一个 RTT。ACS 若用 30 个 `GetParameterValues` 逐个取参数，在 100ms RTT 下就是 3 秒纯等待，乘以设备数就是 ACS 的并发压力。

- **ACS 侧**：一次 `GetParameterValues` 带多个路径；用对象前缀（`Device.WiFi.SSID.`）而不是全树 `GetParameterNames`。
- **设备侧**：对同一会话内的多次查询做树缓存（会话结束即失效）。
- **避免全树 `GetParameterNames`**：TR-181 完整树可达数万节点，一次遍历在低端 CPU 上要几秒且产生几 MB XML。设备侧可对全树遍历做限速或返回 `9004`；ACS 侧应缓存树结构，只在固件版本变化时重新发现。

### 7.3 连接层优化

- **HTTP Keep-Alive** 必开。一次会话 10 个 RPC，不复用连接就是 10 次 TCP + TLS 握手，在低端 CPU 上 TLS 握手就要几百 ms。
- **TLS Session Resumption / Ticket**：把周期性 Inform 的握手成本降一个数量级。ACS 侧的 TLS 卸载层（Nginx/HAProxy）要开 session cache，且多实例间要共享 ticket key。
- **证书链精简 + OCSP 关闭或 stapling**：CPE 做 OCSP 在线校验会引入额外 RTT 和失败点。
- **DNS 缓存与 ACS URL 容错**：ACS URL 解析失败要重试而不是放弃；域名多 A 记录时要轮转。

### 7.4 设备侧资源占用

| 项 | 目标 | 手段 |
|---|---|---|
| RSS | < 8MB | 流式 XML、按需枚举实例、静态树放 rodata |
| 常驻 CPU | ~0% | 纯事件驱动，无轮询 |
| Value Change 检测 | 不轮询 | 让后端主动通知（ubus 事件 / netlink / inotify），而不是定时扫全树。定时扫描是低端设备上 CWMP 客户端吃 CPU 的头号原因 |
| flash 写 | 尽量少 | 事件队列与实例映射用小文件 + 合并写；避免每次 Inform 都写盘（会磨穿 flash） |

### 7.5 可运维性

- **本地日志留存**：会话失败原因（DNS/TCP/TLS/HTTP 状态码/SOAP Fault）必须能在设备上查到。运营商现场排障时拿不到 ACS 日志。
- **诊断入口**：提供一个"立即发起 CWMP 会话并打印全过程"的命令，别让工程师靠改 PeriodicInform 等一小时。
- **TR-143 诊断**要真实实现（吞吐测试、Ping、Traceroute），这是运营商定位"用户报慢"的主要工具，不是可选项。

---

## 八、双栈共存与配置权威

同时接 AC（CAPWAP）和 ACS（TR-069）的设备，是现场最难缠的一类故障来源。根本原因是**两个管理面都认为自己是配置权威**。

典型事故：

1. ACS 下发 SSID 密码 → 写入 UCI → AP 生效。
2. AC 的 Configuration Update 携带全量 SSID 配置（含旧密码）→ 覆盖。
3. AP 的 Value Change 检测到 SSID 密码变了 → 上报 ACS。
4. ACS 认为设备配置漂移 → 重新下发。
5. **回到步骤 2，形成配置震荡**：SSID 每分钟重启一次，用户表现为"WiFi 一直断"。

解法（按推荐度）：

| 方案 | 说明 |
|---|---|
| **参数分域（推荐）** | 明确划分：TR-069 只管设备级（固件、ACS 参数、WAN、管理账号、日志），CAPWAP 只管无线业务（SSID、射频、认证、VLAN）。在代码里对参数打 owner 标记，非 owner 的写入直接拒绝并返回 Fault/Failure |
| 单向只读 | 一侧降为纯监控：ACS 只 Get 不 Set 无线参数 |
| 权威切换标志 | 设备有一个"当前配置权威"状态（如与 AC 在线时 AC 优先，AC 断连时允许 ACS 接管），切换点明确且有日志 |
| 全量下发改增量 | 减少覆盖面，缓解但不根治 |

**同时必须做的**：给 Value Change 通知加"来源过滤"——由 CAPWAP 触发的配置变更不产生 TR-069 Active Notification，否则任何配置同步都会互相激发。

另一个易忽略的点是**两套栈争抢同一后端**：都在改 UCI，都在 `uci_commit`，并发提交会互相丢改动。UCI 层面需要一把跨进程的配置锁（或统一走一个配置代理进程），这在 OpenWrt 上通常靠自建的配置服务或 ubus 串行化来解决。

---

## 九、演进：TR-369/USP 与替代方案

TR-069 的三个结构性缺陷：CPE 单向发起（穿 NAT 靠打补丁）、SOAP/XML 冗长、只能一对一（一个设备一个 ACS）。TR-369（USP，User Services Platform）针对这三点重做：

| 维度 | TR-069 | TR-369 / USP |
|---|---|---|
| 角色 | ACS ↔ CPE | Controller ↔ Agent（**多控制器**，带细粒度权限） |
| 承载（MTP） | HTTP(S) | MQTT / WebSocket / STOMP / CoAP / UDS |
| 编码 | SOAP + XML | **Protobuf** |
| 通信方向 | CPE 发起 | 双向，天然支持主动下发 |
| 数据模型 | TR-181i2 | **同一个 TR-181i2**（这是迁移的关键） |
| 操作 | RPC 方法集 | Get/Set/Add/Delete/Operate/Notify + 路径表达式（支持通配和搜索表达式） |

**迁移策略**：因为数据模型层共用 TR-181，正确的架构是把[第六章](#六tr-069-协议栈开发实现)里的**数据模型引擎与协议层彻底解耦**——同一棵树同时挂 CWMP 前端和 USP 前端。这样支持 USP 只是新增一个前端 + MTP，而不是重写。**如果你现在正在自研 TR-069 栈，这一条是最重要的架构决策。**

其他现实中的替代/并存方案：

| 方案 | 特点 | 适用 |
|---|---|---|
| USP over MQTT | 双向、轻量、云原生友好 | 新一代运营商/云管平台 |
| 私有 HTTPS/MQTT + JSON | 开发最快、完全可控、无互通性 | 自有云管平台、企业产品 |
| NETCONF/YANG | 强模型、事务性好、生态偏运营商城域网 | 企业级/城域网设备 |
| OpenWISP / openwisp-config | 开源、配置即模板、HTTP 拉取 | 中小规模 OpenWrt 集群、社区网络 |
| gNMI/gRPC | 高性能遥测 + 配置 | 数据中心、白盒设备 |

选型现实：**运营商渠道 → TR-069 必须有，USP 逐步加；企业/海外自有品牌 → 私有云管协议 + 少量 TR-069 兼容**。

---

## 十、在 OpenWrt 上的落地

### 10.1 现状

| 能力 | OpenWrt 生态现状 |
|---|---|
| CAPWAP | **无原生实现**。主线内核和 OpenWrt 都不带 CAPWAP 数据面。有 OpenCAPWAP 等开源项目但久未维护 |
| CAPWAP 数据隧道替代 | VXLAN（`kmod-vxlan`）、GRETAP、L2TPv3（`kmod-l2tp-eth`）—— 功能等价，都需处理 MTU/MSS |
| TR-069 客户端 | `easycwmp`（packages feed，`net/easycwmp`，libmicroxml + JSON 脚本映射 UCI）；`icwmp`（iopsys，配套 `libbbfdm` 提供完整 TR-181 树，工程完备度更高）；`freecwmp`（已过时） |
| 集中管理替代 | OpenWISP（`openwisp-config` 代理）、`usteer`（射频引导/负载均衡） |
| 数据模型后端 | UCI（配置）+ ubus（运行态）+ netlink/nl80211（无线），见 [`ac-ap-software-architecture.md`](./ac-ap-software-architecture.md#附录关键代码位置速查) |

### 10.2 自研 CAPWAP AP 客户端的落点

```
capwapc (用户态守护进程, procd 托管)
   │
   ├─ 控制面：UDP 5246 + DTLS(mbedTLS/OpenSSL) + 状态机
   │
   ├─ 配置下发 → UCI (wireless/network/firewall) → 提交后触发
   │      ubus call network reload / wifi reload
   │      （25.12 起无线配置走 ucode + ubus，见配套文档第 3.3 节）
   │
   ├─ 状态采集 → ubus call hostapd.<iface> get_clients
   │             ubus call iwinfo assoclist
   │             nl80211 station dump（更细粒度：RSSI/速率/重传）
   │
   ├─ 站点事件 → 订阅 ubus hostapd 事件（关联/去关联/认证失败）
   │             → 转成 WTP Event / Station Config 上报
   │
   └─ 数据面（如需隧道转发）：
          方案 A：把 wifi-iface 桥进 vxlan0（最省事，非标准 CAPWAP 封装）
          方案 B：自研 kmod，走 udp_tunnel 框架 + GSO（标准封装、性能好）
          方案 C：TC eBPF 做封装（无需 out-of-tree kmod）
```

关键接口就是 `hostapd` 的 ubus 补丁（OpenWrt 特有，见配套文档 3.2 节）——它让"AC 侧要的站点信息和控制能力"不用改 hostapd 代码就能拿到。

### 10.3 自研 TR-069 客户端的落点

- **进程管理**：procd + `respawn`，配置变更用 procd 的 `reload` 触发而非重启进程。
- **数据模型后端**：优先 ubus（有结构化 schema、能订阅事件），配置类落 UCI。**不要直接读写 `/etc/config` 文件**，绕过 UCI 会破坏事务和通知。
- **Value Change**：订阅 ubus 事件（`network.interface`、`hostapd.*`、`wireless`）替代轮询。
- **持久化**：实例映射表和事件队列放 `/etc/cwmp/`（overlay，掉电保留），注意 flash 写入次数。
- **升级**：`sysupgrade` + `-c`（保留配置）；双 Bank 需 bootloader 配合。`TransferComplete` 状态必须在 `sysupgrade` 前落盘到会被保留的路径。
- **不要自己实现 HTTP/TLS**：用 `libcurl`（`ustream-ssl` 太薄，Digest 和 Cookie 都得自己写）。

---

## 十一、测试、压测与交付

### 11.1 一致性与互通

- **TR-069**：Broadband Forum 认证（BBF.069）+ 运营商入网测试（各省 ITMS 规范）。实测中最常挂的项：`0 BOOTSTRAP` 判定、实例号持久性、`TransferComplete` 跨重启、`SetParameterValues` 原子性、扩展节点完整性。
- **CAPWAP**：无实质互通认证。工程上做**版本矩阵回归**（AP 固件 × AC 版本），因为私有 TLV 的兼容性只能靠测试保证。要求协议栈对未知可选 TLV 宽容（见 [3.4](#34-tlv-解析安全第一)）。

### 11.2 压测方法

| 目标 | 方法 |
|---|---|
| AC 承载 AP 数 | 写**模拟 AP 客户端**（单进程模拟数百 WTP 会话），逐步加压到 Echo 超时率 > 0.1% |
| 注册风暴恢复 | 全量 AP 断连后同时重连，测"全网恢复到 Run 态"的时间与曲线 |
| 隧道吞吐 | `iperf3` 多流 + 观察 AC 单核是否打满（验证 RSS 是否失效，见 [4.3](#43-数据面性能)） |
| ACS 承载 | 模拟 CPE 客户端并发 Inform；分别测稳态速率和"全量 BOOT"峰值 |
| 升级并发 | 批量 Download，观察文件服务器带宽、设备失败率、回滚触发率 |
| 长稳 | 7×24 运行观察 RSS 曲线（内存泄漏）、fd 数、flash 写入量 |

**故障注入必测项**：中间链路丢包 5%/时延 200ms、DNS 失败、ACS/AC 返回 5xx、证书过期、时间跳变、断电重启、flash 写满。

### 11.3 交付指标

| 指标 | 参考目标 |
|---|---|
| AP 从上电到 Run 态 | < 60s（含 DHCP + Discovery + DTLS + 配置） |
| AC 单机稳定承载 AP | 按型号标称的 80% 以内，Echo 超时率 < 0.1% |
| 全网注册风暴恢复（1000 AP） | < 5 min，且过程中在线 AP 不掉线 |
| 站点表同步时延（Add Station） | < 200ms |
| 隧道转发时延增量 | < 1ms（同机房） |
| CPE 首次 BOOTSTRAP 成功率 | > 99.5% |
| CWMP 单次周期会话时长 | < 5s |
| 远程升级成功率 | > 99.9%，失败可自动回滚 |
| CWMP 客户端常驻内存 | < 8MB RSS |

---

## 附录 A：CAPWAP 报文与默认参数速查

**消息类型**（Enterprise Number = 0）

| 值 | 消息 | 值 | 消息 |
|---|---|---|---|
| 1/2 | Discovery Request/Response | 15/16 | Image Data Request/Response |
| 3/4 | Join Request/Response | 17/18 | Reset Request/Response |
| 5/6 | Configuration Status Request/Response | 19/20 | Primary Discovery Request/Response |
| 7/8 | Configuration Update Request/Response | 21/22 | Data Transfer Request/Response |
| 9/10 | WTP Event Request/Response | 23/24 | Clear Configuration Request/Response |
| 11/12 | Change State Event Request/Response | 25/26 | Station Configuration Request/Response |
| 13/14 | Echo Request/Response | | |

**RFC 5415 默认定时器**

| 参数 | 默认 | 说明 |
|---|---|---|
| DiscoveryInterval | 5s | 发现重试间隔 |
| MaxDiscoveries | 10 | 最大发现次数 |
| SilentInterval | 30s | 发现失败后静默 |
| WaitJoin | 60s | AC 等待 Join |
| RetransmitInterval | 3s | 控制报文重传 |
| MaxRetransmit | 5 | 最大重传次数 |
| EchoInterval | 30s | 心跳间隔 |
| ChangeStatePendingTimer | 25s | |
| DataCheckTimer | 30s | |
| IdleTimeout | 300s | 数据隧道空闲 |
| StatisticsTimer | 120s | 统计上报 |
| MaxFailedDTLSSessionRetry | 3 | |

**端口与地址**

| 用途 | 值 |
|---|---|
| 控制隧道 | UDP 5246 |
| 数据隧道 | UDP 5247 |
| IPv4 发现组播 | 224.0.1.140 |
| IPv6 发现组播 | FF0X::18C |
| DHCPv4 AC 地址选项 | Option 138（RFC 5417）；厂商实践常用 Option 43 私有子选项 |
| DHCPv6 | Option 52 |

---

## 附录 B：TR-069 事件码与故障码速查

**事件码**

| 码 | 含义 | 码 | 含义 |
|---|---|---|---|
| 0 BOOTSTRAP | 首次接入该 ACS / 恢复出厂 / ACS URL 变更 | 8 DIAGNOSTICS COMPLETE | 诊断完成 |
| 1 BOOT | 普通重启 | 9 REQUEST DOWNLOAD | |
| 2 PERIODIC | 周期上报 | 10 AUTONOMOUS TRANSFER COMPLETE | |
| 3 SCHEDULED | ScheduleInform 触发 | 11/12 DU STATE CHANGE COMPLETE | TR-157 |
| 4 VALUE CHANGE | Active Notification 触发 | 13 WAKEUP | |
| 5 KICKED | | M Reboot / M ScheduleInform | 方法触发（带 CommandKey） |
| 6 CONNECTION REQUEST | ACS 主动唤醒 | M Download / M Upload | 方法触发 |
| 7 TRANSFER COMPLETE | 传输完成 | | |

**故障码**

| 码 | 含义 | 码 | 含义 |
|---|---|---|---|
| 9000 | Method not supported | 9007 | Invalid parameter value |
| 9001 | Request denied | 9008 | Attempt to set a non-writable parameter |
| 9002 | Internal error | 9009 | Notification request rejected |
| 9003 | Invalid arguments | 9010 | Download failure |
| 9004 | Resources exceeded | 9011 | Upload failure |
| 9005 | Invalid parameter name | 9012 | File transfer server authentication failure |
| 9006 | Invalid parameter type | 9013 | Unsupported protocol for file transfer |
| | | 9800+ | 厂商自定义 |

---

## 附录 C：排障手法

**CAPWAP**

```bash
# AP 侧确认拿到管理 IP 与 AC 地址（Option 43/138）
udhcpc -i br-lan -n -q -f ; logread | grep -i dhcp

# 抓控制面（注意 DTLS 加密后只能看到握手和报文长度模式）
tcpdump -i any -nn -s0 'udp port 5246' -w capwap-ctrl.pcap
# 抓数据面
tcpdump -i any -nn 'udp port 5247' -c 100

# 只看是否有 Discovery 往返：观察包方向与长度即可判断卡在哪一态
tcpdump -i any -nn 'udp port 5246' -c 20 -v

# MTU 黑洞验证（1472 = 1500 - 20 - 8）
ping -M do -s 1472 <AC_IP>      # 通过 → 路径 1500 OK
ping -M do -s 1400 <AC_IP>      # 逐步下探找到真实 PMTU

# 隧道相关的 offload 与队列
ethtool -k eth0 | grep -i tnl
ethtool -S eth0 | grep -i -E 'rx_queue|drop'
cat /proc/interrupts | grep eth   # 观察是否只有一个核在收包（RSS 失效）
```

**TR-069**

```bash
# 当前 ACS 参数
uci show easycwmp 2>/dev/null || uci show cwmp

# 立即发起一次会话并看全过程
/etc/init.d/easycwmp restart ; logread -f | grep -i cwmp

# 分层排除：DNS → TCP → TLS → HTTP → SOAP
nslookup <acs-host>
nc -z <acs-host> 443 && echo tcp-ok
echo | openssl s_client -connect <acs-host>:443 -servername <acs-host> 2>&1 | \
    grep -E 'Verify return code|subject=|notAfter'
date                              # 证书校验失败先看时间是否同步

# 抓 CWMP（TLS 下只能看连接行为；调试期可临时用 http:// 的 ACS URL 抓明文）
tcpdump -i any -nn -s0 'tcp port 80 or tcp port 443' -w cwmp.pcap

# 事件队列/实例映射是否正常持久化
ls -l /etc/cwmp/ ; cat /etc/cwmp/instances 2>/dev/null
```

**判定口诀**

- AP 上不了线：先看**有没有 IP**，再看**有没有 AC 地址**，再看**5246 有没有往返**，最后看**时间对不对**（证书）。
- AP 反复上下线：先看**PoE 供电**和**AP 内存曲线**，再看 **Echo 丢包**，最后看 **AC CPU**。
- 能上网但部分网站打不开：**MTU**，没有例外。
- CPE 连不上 ACS：**DNS → 时间 → 证书 → 认证 → Cookie**，按这个顺序查，90% 在前三项。
- ACS 反复全量下发配置：查 **BOOTSTRAP 判定逻辑**和 **ParameterKey 持久化**。
- WiFi 每分钟断一次且配置来回变：**双管理面配置震荡**，见[第八章](#八双栈共存与配置权威)。
