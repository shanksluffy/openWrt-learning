# libubox 代码分析

分析对象：`components/libubox`（约 11500 行 C 代码）
分析日期：2026-08-06

> **前提说明**
>
> 本仓库中的 `components/libubox` 就是 **upstream 本身**：`origin` 指向官方仓库
> `https://github.com/openwrt/libubox.git`，`master` 与 `origin/master` 一致，
> 无任何本地独有提交，历史从首个 `Initial import` 完整保留（共 612 个提交）。
>
> 上游正被维护者主动加固。2026-07-04 有一批集中修复出自 Felix Fietkau
> （libubox 作者兼维护者），针对 uloop / ustream 的回调 use-after-free
> （`a9ab90b`、`c08a4ab`、`17f527f`）以及 udebug 的对端数据校验
> （`9d7fb82`、`2e0e7f0`）。**因此在报告任何缺陷前应先 `git fetch`**——
> 本文撰写时该 clone 的最新提交为 2026-07-21，且此后未再 fetch。
>
> 由于不存在本地补丁，修复的正确路径是直接改本仓库、按 OpenWrt 格式提交
> （`libubox: fix ...` + `Signed-off-by`，即 `git commit -s`），再向
> openwrt/libubox 提 PR 或发送至 openwrt-devel 邮件列表。
>
> 第五章列出的问题均为**逐行核对过的**结论。ustream 缓冲与重入、avl 旋转逻辑、
> usock/base64/md5、json_script/runqueue 四块的完整审计尚未纳入本文。

---

## 目录

1. [代码架构](#一代码架构)
2. [设计原理](#二设计原理)
3. [使用方法](#三使用方法)
4. [可以优化的点](#四可以优化的点)
5. [已核验的潜在 bug](#五已核验的潜在-bug)

---

## 一、代码架构

### 1.1 构建产物划分

`CMakeLists.txt` 中核心库仅包含以下文件：

```
avl.c avl-cmp.c blob.c blobmsg.c uloop.c usock.c ustream.c ustream-fd.c
vlist.c utils.c safe_list.c runqueue.c md5.c kvlist.c ulog.c base64.c
udebug.c udebug-remote.c
```

而 `blobmsg_json.c`、`json_script.c`、`jshn` 被单独编译成三个产物，且整段包在 `find_library(json NAMES json-c)` 条件内。

这是一个明确的架构决策：**核心 libubox 零外部依赖**，凡涉及 JSON 的都切分出去。不需要 JSON 的守护进程（ubusd、netifd 的大部分路径）不会被拖上 json-c 依赖。

头文件安装时使用 `LIST(FILTER headers EXCLUDE REGEX "-priv.h$")` 剔除私有头，公私边界是强制的。

编译选项为 `-Os -std=gnu99 -Wall -Werror`（gcc > 6 时追加 `-Wextra`）。优化目标是**体积而非速度**，这一点贯穿影响后文的各项取舍判断。

### 1.2 依赖层次

四层结构，无循环依赖：

| 层级 | 模块 | 依赖 |
|------|------|------|
| L0 基础设施 | `utils.h`（字节序、`calloc_a`、`BUILD_BUG_ON`）、`list.h`、`avl`、`safe_list`、`md5`、`base64`、`usock`、`ulog` | 无 |
| L1 事件循环 | `uloop` + `uloop-epoll.c` / `uloop-kqueue.c` | `utils.h`、`list.h` |
| L1' 序列化 | `blob`、`blobmsg` | 仅 `utils.h` |
| L2 组合层 | `ustream` / `ustream-fd`、`runqueue`、`udebug`、`vlist`、`kvlist` | L0 + L1 |
| L3 JSON | `blobmsg_json`、`json_script`、`jshn` | + json-c |

**关键点：`blob` / `blobmsg` 完全不依赖 `uloop`。** 序列化与事件循环是两条独立的轴。这使得 ubus 能把 blobmsg 当作纯编解码库使用，也让不需要事件循环的工具（`jshn`）直接复用它。许多同类库会把两者耦合成单一"框架"，libubox 没有。

### 1.3 L2 层的自我复用

- `runqueue` 建立在 `safe_list` + `uloop_timeout` + `uloop_process` 之上
- `ustream` 建立在 `uloop_fd` + `uloop_timeout` 之上。其 `state_change` 用一个 0 ms 定时器把状态变更推迟到循环顶层，避免在回调栈内做状态转换
- `udebug` 建立在 `uloop_fd` + `blobmsg` + `avl` 之上
- `vlist` / `kvlist` 都是 `avl` 的薄封装

### 1.4 对外扩展面

Lua 绑定只有 `lua/uloop.c` 一个文件——暴露给脚本的仅事件循环，序列化部分刻意未做绑定。

---

## 二、设计原理

### 2.1 侵入式容器 + 零分配

这是最核心的一条。`uloop_fd`、`uloop_timeout`、`avl_node`、`safe_list`、`vlist_node`、`runqueue_task` 全部由调用者分配并嵌入自身结构，库通过 `container_of` 反推。整个 `uloop.c` 中没有一次 `malloc`。

带来的连锁收益：

- 没有分配失败路径，绝大多数函数的错误处理可以极简
- 生命周期完全由调用者掌握，库无需引用计数
- 对象与业务数据同处一块内存，cache 局部性好

代价是所有权契约变成隐式的：调用者必须保证对象在 `_delete` 之前一直存活。这也直接导致了后文 2.4 节那一整套重入保护机制。

`avl_node` 的设计尤能说明取舍（`avl.h:56-95`）：

```c
struct avl_node {
  struct list_head list;      /* 必须是第一个成员 */
  struct avl_node *parent;
  struct avl_node *left;
  struct avl_node *right;
  const void *key;
  signed char balance;
  bool leader;                /* 标记一串同键节点的头 */
};
```

树节点里冗余挂了一条链表，原因是这样**有序遍历退化为 O(1) 步进的链表 walk**，无需中序遍历栈；重复键也能直接串在一起。代价是每节点 56 字节（64 位平台）。对 `kvlist` 那种短字符串键的场景，元数据开销超过数据本身——这是拿内存换遍历简洁性，在"配置项数量有限"的前提下成立。

### 2.2 编译期多态 vs 运行期虚表

**uloop 用编译期多态**，后端通过直接 `#include` 源文件选择（`uloop.c:85-91`）：

```c
#ifdef USE_KQUEUE
#include "uloop-kqueue.c"
#endif

#ifdef USE_EPOLL
#include "uloop-epoll.c"
#endif
```

后端接口是一组**隐式函数契约**，没有任何头文件声明：`uloop_init_pollfd`、`register_poll`、`__uloop_fd_delete`、`uloop_fetch_events`、`timer_register`、`timer_remove`、`timer_next`。后端反过来还会使用主文件的全局变量（`poll_fd`、`cur_fds`、`ULOOP_MAX_EVENTS`）。

收益是所有代码在同一编译单元内，编译器能内联并跨函数优化，无间接跳转。代价是接口无文档、无类型检查，且一次构建只能有一个后端（无法运行时降级）。

**ustream 用运行期虚表**（`ustream.h:88-109`）：

```c
int (*write)(struct ustream *s, const char *buf, int len, bool more);
void (*free)(struct ustream *s);
void (*set_read_blocked)(struct ustream *s);
bool (*poll)(struct ustream *s);
```

两者选择不同的原因是维度不同：uloop 后端是**构建期二选一**（Linux 还是 BSD），而 ustream 的实现需要**运行期多实例并存**（同一进程内既有 fd stream 又有 SSL stream）。

### 2.3 blob 的 TLV 格式设计

`blob.h:46-55`：

```c
#define BLOB_ATTR_ID_MASK  0x7f000000
#define BLOB_ATTR_ID_SHIFT 24
#define BLOB_ATTR_LEN_MASK 0x00ffffff
#define BLOB_ATTR_ALIGN    4
#define BLOB_ATTR_EXTENDED 0x80000000

struct blob_attr {
	uint32_t id_len;
	char data[];
} __packed;
```

一个 32 位头承担三件事：7 位类型、24 位长度、1 位"扩展"标志。扩展位区分裸 blob 与带名字的 blobmsg——这是 blobmsg 能**架在 blob 之上而不改变格式**的原因，同一段字节流两层都能解析。4 字节对齐让嵌套容器可以原地指针遍历，无需拷贝。

采用网络字节序（`cpu_to_be32`）在小端机器上每次访问都需 byteswap。对纯本机 IPC 而言这是额外开销，但换来跨架构的格式稳定性——该格式是 ubus 的线上协议，一旦冻结便不可改动。`utils.h:139-142` 那套 `__builtin_choose_expr(__is_constant(x), ...)` 技巧就是为了让常量在编译期完成 swap、仅变量走运行期，把成本降至最低。

### 2.4 回调重入安全：三代机制并存

这是 libubox 中最有历史层次的部分。所有 API 都允许"回调内删除任意对象、甚至递归回到事件循环"，而库演化出**三套不同解法**。

**第一代：`safe_list`（2013）** — 通用机制，链表节点内挂迭代器指针（`safe_list.h:35-38`）：

```c
struct safe_list {
	struct list_head list;
	struct safe_list_iterator *i;
};
```

`runqueue` 是唯一使用者。

**第二代：uloop 的专用游标** — `signal_next` / `process_next` 全局游标（`uloop.c:63-69`），加上 `fd_stack` 那条住在 C 栈帧里的链表。

```c
/*
 * Entries the signal and process dispatch loops are about to visit. A callback
 * may delete (and free) an arbitrary sibling entry; the matching *_delete()
 * advances these so the loops never dereference a freed entry.
 */
static struct uloop_signal *signal_next;
static struct uloop_process *process_next;
```

核心难点在于：回调可以删除并 free **任意兄弟节点**，而 `list_for_each_entry_safe` 只能保护"当前节点被删"这一种情况。解法是把迭代器状态从栈上挪到全局，让 mutation 方修正它。这两个游标是 `a9ab90b`、`c08a4ab` 两个提交才加入的。

**第三代：`ustream_free_guard`（`17f527f`，最新）** — `ustream.c:97-114`：

```c
void ustream_free_guard_set(struct ustream *s, struct ustream_free_guard *g)
{
	g->freed = false;
	g->prev = s->free_flag;
	s->free_flag = &g->freed;
}

bool ustream_free_guard_check(struct ustream *s, struct ustream_free_guard *g)
{
	if (g->freed) {
		if (g->prev)
			*g->prev = true;
		return true;
	}

	s->free_flag = g->prev;
	return false;
}
```

结构与 uloop 的 `fd_stack` 同源：guard 是局部变量，通过 `prev` 串成一条住在栈上的链表；`ustream_free()` 只置最内层标志，`check` 负责把信号向外层转发，于是每层嵌套调用点都能各自安全退出。配合 `pending_cb` 位（`ustream.c:353-362`）防止同一回调被重入。

三套机制解决同一类问题，是"按需长出来的"而非设计出来的。这也是最值得收敛的地方，详见 4.6。

### 2.5 uloop 的其它要点

**"一切皆 fd"**：每种异步事件都被归一化为 fd 上的可读事件，使主循环只保留**一个阻塞点**（`epoll_wait`），从根本上消除"等 A 时 B 来了"的竞态。

| 事件源 | 归一化手段 |
|--------|-----------|
| 信号 | self-pipe，`waker_fd` 注册进 epoll |
| 子进程退出 | SIGCHLD → 同一条 self-pipe |
| 周期定时器 | `timerfd_create`，每 interval 一个 fd |
| 一次性定时器 | **例外**，用 `epoll_wait` 的 timeout 参数 |

一次性定时器故意不用 timerfd：这类定时器往往成百上千个（每个 ubus 请求一个超时），每个占一个 fd 太贵，而它们只需要"最早那个还有多久"这一个数字。

**信号处理是标记而非计数**：信号编号压成 1 字节写入管道，读出后归并进 `uint64_t` 位图。这承认了一个事实——同一信号来 10 次与来 1 次语义相同，因此管道满时丢字节是可接受的降级。这也正是写端必须非阻塞的原因（否则会在信号处理器内死锁，主循环永无机会腾空管道）。

**每次只处理一个 fd 事件就返回**（`uloop.c:260`）：这样每处理一个 fd 都会回到主循环顶部，重新检查定时器与 `uloop_cancelled`。代价是每事件多一次定时器检查，换来定时器精度不被大批 fd 事件拖累、`uloop_end()` 能立即生效。

**边沿触发的重入缓冲**（`uloop.c:250-261`）：

```c
		stack_cur.next = fd_stack;
		stack_cur.fd = fd;
		fd_stack = &stack_cur;
		do {
			stack_cur.events = 0;
			fd->cb(fd, events);
			events = stack_cur.events & ULOOP_EVENT_MASK;
		} while (stack_cur.fd && events);
		fd_stack = stack_cur.next;

		return;
```

`stack_cur` 是局部变量，`fd_stack` 把这些局部变量串成链表——"当前正在执行的 fd 回调栈"这个数据结构的节点住在 C 调用栈的栈帧里，零分配、自动回收、深度天然等于递归深度。

它解决边沿触发下的重入：若 fd X 的回调内又跑了一层 loop、期间 X 再次就绪，既不能重入 X 的回调，又不能丢事件（EPOLLET 不会重复通知）。于是事件被攒在栈节点上，回调返回后由外层 `do/while` 重放。若 X 在回调中被删除，节点的 `fd` 被置 NULL，循环条件立即发现。

水平触发的 fd 被显式排除在这套机制外（`uloop.c:202-203`），注释写明：反正内核会一直报告，让内核当队列即可。

### 2.6 库的礼貌与显式的信任边界

**不偷全局状态**（`uloop.c:545-555`）：只在当前 handler 为 `SIG_DFL` 时才安装自己的（不覆盖用户已有的），只在当前仍是自己安装的那个时才恢复（不踩别人后来的修改）。`SIGPIPE` 的 ignore 同样双向检查。

**信任级别编码进函数名**：`blob_parse` vs `blob_parse_untrusted`、`blobmsg_check_attr` vs `blobmsg_check_attr_len`。头文件明确写有 *"This method may be used with trusted data only. Providing malformed blobs will cause out of bounds memory access."* 把安全契约写进 API 名字，比写在文档里可靠得多。

---

## 三、使用方法

### 3.1 最小事件循环

```c
#include <libubox/uloop.h>

static struct uloop_timeout t;

static void on_timeout(struct uloop_timeout *t)
{
    /* 周期任务：在回调内重新 arm */
    uloop_timeout_set(t, 1000);
}

int main(void)
{
    uloop_init();
    t.cb = on_timeout;
    uloop_timeout_set(&t, 1000);
    uloop_run();          /* 阻塞直到 uloop_end() 或 SIGINT/SIGTERM */
    uloop_done();
    return 0;
}
```

关键约定：

- 对象生命周期由调用者负责，`uloop_timeout_set` 之后结构体不可被释放或移动
- 一次性定时器在回调**之前**已被摘除，因此回调内可安全 free 自己或重新 arm

### 3.2 监听文件描述符

```c
static void on_readable(struct uloop_fd *u, unsigned int events)
{
    if (events & ULOOP_READ) {
        /* 必须循环读到 EAGAIN */
    }
    if (u->eof)
        uloop_fd_delete(u);
}

static struct uloop_fd ufd = { .fd = sock, .cb = on_readable };
uloop_fd_add(&ufd, ULOOP_READ);
```

注意 `uloop_fd_add` 会自动把 fd 设为非阻塞，除非传入 `ULOOP_BLOCKING`。同一函数兼作 add / modify / delete：事件位全空即转为 delete。

### 3.3 带缓冲的流

```c
static struct ustream_fd sfd;

static void read_cb(struct ustream *s, int bytes)
{
    char *data;
    int len;

    /* string_data = true 时 data 保证 NUL 结尾 */
    while ((data = ustream_get_read_buf(s, &len)) && len) {
        int used = process(data, len);
        if (!used)
            break;
        ustream_consume(s, used);
    }
}

sfd.stream.notify_read = read_cb;
sfd.stream.string_data = true;
ustream_fd_init(&sfd, fd);
ustream_printf(&sfd.stream, "hello %d\n", 42);
```

头文件中的重要承诺：`notify_read` / `notify_write` 内**允许**调用 `ustream_free(s)`（由 2.4 节的 guard 机制保障）；但 `notify_state` 没有这条承诺。

### 3.4 序列化

```c
static struct blob_buf b = {};   /* 必须先零初始化 */
void *tbl;

blob_buf_init(&b, 0);
blobmsg_add_string(&b, "name", "eth0");
blobmsg_add_u32(&b, "mtu", 1500);

tbl = blobmsg_open_table(&b, "stats");
blobmsg_add_u64(&b, "rx", rx);
blobmsg_close_table(&b, tbl);

/* b.head 即完整消息，blob_raw_len(b.head) 为其长度 */
blob_buf_free(&b);
```

### 3.5 解析

必须使用 policy；对不可信数据要用带 `_len` 的变体。

```c
enum { CFG_NAME, CFG_MTU, __CFG_MAX };

static const struct blobmsg_policy pol[__CFG_MAX] = {
    [CFG_NAME] = { .name = "name", .type = BLOBMSG_TYPE_STRING },
    [CFG_MTU]  = { .name = "mtu",  .type = BLOBMSG_TYPE_INT32 },
};

struct blob_attr *tb[__CFG_MAX];

if (blobmsg_parse(pol, __CFG_MAX, tb, blobmsg_data(msg), blobmsg_len(msg)))
    return;

if (tb[CFG_NAME])
    printf("%s\n", blobmsg_get_string(tb[CFG_NAME]));
```

`blobmsg_parse` 只填充 policy 命中的项，未出现的保持 NULL，因此**每个 `tb[i]` 都必须判空**。这是 OpenWrt 代码中最常见的漏检点。

### 3.6 并发受限的任务队列

```c
static RUNQUEUE(q, 2);   /* 最多 2 个并发 */

struct runqueue_process p = {};
p.task.type = &my_type;  /* 需提供 run / cancel / kill 三个回调 */
runqueue_process_add(&q, &p, pid);
```

### 3.7 shell 中操作 JSON

`sh/jshn.sh` 提供函数库：

```sh
. /usr/share/libubox/jshn.sh

json_init
json_add_string "name" "eth0"
json_add_int "mtu" 1500
json_dump                      # 输出 JSON

json_load "$(cat cfg.json)"    # 反向：JSON → shell 变量
json_get_var mtu mtu
```

---

## 四、可以优化的点

### 4.1 `blob_buffer_grow` 线性增长导致构建大 blob 时 O(n²)

`blob.c:32-36`：

```c
	delta = ((minlen / 256) + 1) * 256;
	if (buf->buflen < 0 || delta > INT_MAX - buf->buflen)
		return false;

	new = realloc(buf->buf, buf->buflen + delta);
```

`minlen` 是"还差多少"，逐个添加小属性时它总是很小，因此每次只增长 256 字节。构建 1 MB 消息需约 4096 次 `realloc`，最坏情况每次都拷贝整个缓冲区。

改为按 `buflen` 比例增长（翻倍，或 `max(256, buflen/2)`）即可降至 O(n)。realloc 常能原地扩展所以实测未必这么糟，但在 musl + 碎片化堆的路由器上是真实风险。

**这是收益最明确、改动最小的一处优化。**

### 4.2 每个事件两次 `clock_gettime`

`uloop_run_timeout` 主循环内 `uloop_process_timeouts()`（`uloop.c:690`）与 `uloop_get_next_timeout()`（`uloop.c:670`）各自独立调用 `uloop_gettime()`。由于 `uloop_run_events` 每次只处理一个 fd 就返回，这两次调用是**每事件**发生的。

在有 vDSO 的平台上很便宜，但部分 MIPS 目标上 `clock_gettime` 是真实系统调用。把时间戳求一次传下去即可减半。

### 4.3 `pipe`+4×`fcntl` → `pipe2`；`epoll_create` → `epoll_create1`

`waker_init` 用 `pipe()` 后对两端各做两次 `fcntl`（共 5 个系统调用），`pipe2(fds, O_CLOEXEC|O_NONBLOCK)` 一个即可。`uloop-epoll.c:31,35` 的 `epoll_create(32)` + `fcntl` 同理可换为 `epoll_create1(EPOLL_CLOEXEC)`（`epoll_create` 的 size 参数自 2.6.8 起已被忽略）。

仅影响启动路径，收益小但代码更干净。历史原因是 `pipe2` 需要 `_GNU_SOURCE` 且早年要兼容不支持它的平台（uloop 同时支持 kqueue）。

### 4.4 `ULOOP_MAX_EVENTS 10` 偏保守

每次 `epoll_wait` 最多取 10 个事件，繁忙的 ubusd 会产生更多系统调用。提升到 32 或 64 的代价是 `events[]` 与 `cur_fds[]` 两个静态数组变大（10 时约 280 字节，64 时约 1.8 KB）。

在 `-Os` 目标下这是有意识的选择，但对 x86 上运行的 ubusd 值得做成可配置。

### 4.5 `blobmsg_parse` 每次调用都重算 policy 名字长度

`blobmsg.c:191-197`：

```c
	pslen = alloca(policy_len * sizeof(*pslen));
	for (i = 0; i < policy_len; i++) {
		if (!policy[i].name)
			continue;

		pslen[i] = strlen(policy[i].name);
	}
```

policy 表实际全是 `static const`，这些 `strlen` 结果编译期即确定，却在每次解析时重算。ubusd 的分发路径上这是纯浪费。

彻底的修法是在 `struct blobmsg_policy` 内加长度字段，但会破坏 ABI；折中方案是提供接受预计算长度数组的新入口。

### 4.6 三套重入保护机制值得收敛

`safe_list`、uloop 的专用游标、`ustream_free_guard` 解决的是同一类问题。`ustream_free_guard` 的"栈上链式转发"形式最通用，若抽成公共设施（如 `struct ub_guard`），uloop 的两个全局游标与 `fd_stack` 都能统一进去。

从提交历史看，这类 use-after-free 已反复出现三次，统一后可减少每次新增回调点都要重新发明一遍的风险。

### 4.7 `blobmsg_namelen` 在头文件中缺 `inline`

`blobmsg.h:73-76`：

```c
static uint16_t blobmsg_namelen(const struct blobmsg_hdr *hdr)
{
	return be16_to_cpu(hdr->namelen);
}
```

同一头文件内其他函数均为 `static inline`，仅此处是裸 `static`。任何包含 `blobmsg.h` 却未用到它的下游 TU 会触发 `-Wunused-function`；libubox 自带 `-Werror`，下游项目若也开启即构建失败。加 `inline` 即可。

---

## 五、已核验的潜在 bug

按严重度排列。均为逐行核对确认。

### 5.1 内存安全相关

#### `__calloc_a` 失败时不清空出参（最高优先级）

`utils.c:66-69` 在 `calloc` 失败时**提前返回 NULL**，而给出参赋值的循环在 `utils.c:72-75`：

```c
	ptr = calloc(1, alloc_len);
	if (!ptr) {
		va_end(ap);
		return NULL;         /* 出参此时仍未初始化 */
	}

	alloc_len = 0;
	foreach_arg(ap, cur_addr, cur_len, &ret, len) {
		*cur_addr = &ptr[alloc_len];
		alloc_len += (cur_len + C_PTR_ALIGN - 1) & C_PTR_MASK;
	}
```

调用者若不检查返回值，就会拿到未初始化的指针。`udebug-remote.c:241` 正是如此，紧接着 `memcpy(data_buf, ...)` 即为**野指针写**。

进一步搜索发现该模式在整棵树上有 8 处漏检，多数紧跟 `strcpy` / `memcpy`：

| 位置 | 后续操作 |
|------|---------|
| `procd/service/service.c:100` | `strcpy(new_name, name)` |
| `procd/uxc.c:335` | `strcpy(new_name, container_name)` |
| `procd/uxc.c:436` | `strcpy(new_name, container_name)` |
| `procd/service/watch.c:72` | `strcpy(name, _name)` |
| `netifd/interface.c:940` | `strcpy(iface_name, name)` |
| `netifd/config.c:252` | `memcpy(attrbuf, data, blob_pad_len(data))` |
| `procd/service/trigger.c:215` | `t->type = _t` |
| `procd/service/validate.c:147` | `vr->avl.key = vr->option = option` |

libubox 自身的两处（`json_script.c:47`、`kvlist.c:80`）都正确检查了。

**推荐修法**：在 `__calloc_a` 失败分支内把所有出参置 NULL。一处改动即可让十余个漏检调用点从"野指针写"退化为"NULL 解引用"——一个干脆的崩溃，而非静默的内存破坏。

#### `udebug_ring_ptr` 的 TOCTOU

`udebug-proto.h:60-65`：

```c
static inline struct udebug_ptr *
udebug_ring_ptr(struct udebug_hdr *hdr, uint32_t idx)
{
	struct udebug_ptr *ring = (struct udebug_ptr *)&hdr[1];
	return &ring[idx & (hdr->ring_size - 1)];
}
```

掩码用的是共享内存中的 `hdr->ring_size`，而非校验过的私有副本 `buf->ring_size`。对比紧邻的姊妹函数即可看出意图不一致：

```c
static inline void *udebug_buf_ptr(struct udebug_buf *buf, uint32_t ofs)
{
	return buf->data + (ofs & (buf->data_size - 1));
}
```

恶意对端在映射通过校验后修改该值即可越界。`udebug-remote.c:250-258` 的 `memcpy` 长度混用了两个来源（目标大小来自私有副本，指针差值来自共享内存），可导致**堆溢出**。

最新的两个 udebug 提交（`9d7fb82`、`2e0e7f0`）只加固了映射时的一次性校验，恰好留下这个口子。

#### `blob_len()` 的无符号下溢

`blob.h:99-103`：

```c
static inline size_t
blob_len(const struct blob_attr *attr)
{
	return (be32_to_cpu(attr->id_len) & BLOB_ATTR_LEN_MASK) - sizeof(struct blob_attr);
}
```

长度字段是 24 位、完全由报文提供方控制。若其值为 1、2 或 3，该减法在 `size_t` 中下溢，返回约 `SIZE_MAX`；而 `blob_raw_len()` 加回 4 又绕回 1~3，`blob_pad_len()` 对齐成 4——**恰好通过迭代宏的两个下界检查**（`blob.h:246-247`）。

迭代本身是安全的（每次前进 4 字节、`rem` 同步递减），危险在于循环会把这样的属性交给循环体，而循环体一旦调用 `blob_len()` 或 `blobmsg_data_len()` 就拿到垃圾长度。

libubox 内部的硬化路径均正确地先验长度再解析（`blobmsg.c:69-71`、`blob.c:251`），但这把风险外推给了每一个调用者。可顺着 `json_script.c:130,145` 核查——那里把 `blobmsg_data_len(cur)` 直接喂给 `blobmsg_parse_array`。

#### `blobmsg_parse_array` 缺少入参校验

`blobmsg.c:142-166` 与紧邻的 `blobmsg_parse`（`blobmsg.c:186-191` 有完整守卫）不对称，完全没有检查：

```c
	memset(tb, 0, policy_len * sizeof(*tb));
	__blob_for_each_attr(attr, data, len) {
		if (policy[i].type != BLOBMSG_TYPE_UNSPEC &&
		    blob_id(attr) != policy[i].type)
			continue;
		...
		tb[i++] = attr;
		if (i == policy_len)
			break;
	}
```

`policy_len == 0` 时：`policy[0]` 越界读，`tb[0]` 越界写，且 `i == policy_len` 永不成立，于是每来一个属性就往 `tb[1]`、`tb[2]`… **无界越界写**。`policy_len` 为负时 `memset` 的第三参数提升为巨大 `size_t` 直接崩溃。

树内所有调用者都传非零常量，故为潜伏缺陷。注意 `tests/fuzz/test-fuzz.c` 的 harness 写死了 `__FOO_MAX`，结构上碰不到这个参数空间。

### 5.2 正确性相关

| 位置 | 问题 |
|------|------|
| `udebug-remote.c:102-106` | `udebug_remote_buf_set_poll` 拿 `udebug_remote_get_handle` 的返回码（恒为 0/-1）当位移量，导致句柄非 0 的客户端轮询永久失效 |
| `udebug-remote.c:154,164` | 把 `u32_sub` 的 `int32_t` 返回值存进 `uint32_t diff`，符号丢失使二分查找失控。同文件 119、126、143 行都直接比较 `int32_t`，仅此处不同 |
| `udebug-remote.c:35` | `ctx->poll_handle = msg->id` 直接采信对端的 32 位值，随后用作移位量，`id >= 32` 时是 UB |
| `uloop.c:369` | `if (time->tv_usec > 1000000)` 应为 `>=`，留下非规范 `timeval`。因 `time` 是公开成员，下游若传给 `select`/`setitimer` 会得到偶发 `EINVAL` |
| `blob.c:186-194` | `blob_nest_start` 在 `blob_new` 失败时已把 `buf->head` 写成 NULL，损坏整个 buffer。`blobmsg_open_nested`（`blobmsg.c:290-294`）对同一场景处理正确，两者应一致 |
| `blobmsg.c:251-259` | `blobmsg_new` 把 `int namelen` 截断进 `uint16_t`，名字 ≥ 64 KiB 时静默产出畸形 blob（不越界，但会被校验函数拒绝）。JSON key 可达该长度 |
| `blob.c:213-226` | `blob_check_type` 只检查上界未检查 `type < 0`。`struct blob_attr_info.type` 声明为 `unsigned int` 却在 `blob.c:259` 被取进 `int type` |

### 5.3 jshn 专项

- `MAX_VARLEN`（`jshn.c:32`）是**死代码**，全库仅此一处出现。它显然本应给下面两处设上限
- `get_keys`（`jshn.c:200`）/ `get_var`（`jshn.c:210`）的 `alloca` 大小完全由环境变量内容决定、无上界。足够长的名字直接冲掉栈
- env → JSON 方向（`jshn.c:226-276`）存在**无界递归**：`jshn_add_objects` 与 `jshn_add_object_var` 互相递归，深度由环境变量决定。构造自引用（`T_J_V_x=object` 配 `J_V_x=J_V`）即可栈溢出。反方向（JSON → shell）靠 json-c 的 32 层深度上限是安全的——两个方向防护不对等
- `main`（`jshn.c:410-416`）在 environ 为空时 `calloc(0, ...)` 可能合法返回 NULL，导致打印 `%m` 后退出

### 5.4 确认**不是** bug 的几处

为了明确"负空间"，以下几处经核对是正确的：

- **jshn 的 shell 转义是正确的**：key 被 `write_key_string` 洗成仅 `[A-Za-z0-9_]`（`jshn.c:94-100`），value 走 `add_json_string` 的标准 `'\''` 转义（`jshn.c:81-88`）。单引号内除 `'` 本身外无任何字节有特殊含义，不存在命令注入
- `blob_check_type` 中的 `data[len-1]`（`blob.c:229`）有 `minlen = 1` 保证，不会越界
- `blobmsg_parse` 传给 `blobmsg_check_attr_len` 的 `len` 确实是被迭代宏递减后的剩余长度，不是全长——迭代宏直接修改第三个实参
- `blobmsg_check_attr_len` 的检查顺序正确：先验 `blob_raw_len >= sizeof(struct blob_attr)` 再调 `blobmsg_check_name`，避免了 5.1 节的下溢
- `udebug` 的数据区双重映射使得从任意掩码偏移 `memcpy` 至多 `data_size` 字节仍在 2×`data_size` 区域内

### 5.5 测试覆盖现状

`tests/` 下有 avl、b64、blob-buflen、blob-parse、blobmsg、blobmsg_check_array、blobmsg-parse、blobmsg-types、json-script、list、runqueue 的单元测试，`tests/fuzz/` 有 blobmsg 的模糊测试，且 CMake 提供 `ADD_UNIT_TEST_SAN` 宏（ASan + UBSan + LSan，`-fno-sanitize-recover=all`）。

由此可推断缺陷的分布：

- **blob 属性内容校验**已被充分 fuzz，易发缺陷基本清除
- **`policy_len == 0` 等参数空间未被 fuzz**（harness 写死常量）
- **整个写入侧**（`blob_buf` / `blobmsg_new` / `blobmsg_add_*`）**完全没有 fuzz 覆盖**
- ustream、udebug、avl 旋转逻辑均不在 fuzz 范围内

后两类是剩余缺陷最可能藏身的地方。
