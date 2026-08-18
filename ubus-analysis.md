# ubus 代码分析

分析对象：`components/ubus`（约 6300 行 C 代码）
分析日期：2026-08-06

> **前提说明**
>
> 本仓库中的 `components/ubus` 就是 **upstream 本身**：`origin` 指向官方仓库
> `https://github.com/openwrt/ubus.git`，`master` 与 `origin/master` 完全一致，
> 无任何本地独有提交，工作区干净（共 404 个提交）。因此修复的正确路径是直接改
> 本仓库、按 OpenWrt 格式提交（`ubusd: fix ...` + `git commit -s`），再提 PR 或
> 发到 openwrt-devel。
>
> 上游近期有一轮密集加固，几乎都出自 Hauke Mehrtens（2026-04 ~ 2026-06）：
> `f29767f`/`5849870`（fd 泄漏）、`b099d05`（`ubus_shutdown` 幂等）、
> `bcc45ca`（FD_CLOEXEC）、`8188f5c`/`7a068ba`/`747013f`（libubus-io 三处）、
> `09d2df4`（OOM 空指针）、`239edcb`（ID 重试）、`24864e7`（`GRND_INSECURE`）。
> **报告缺陷前应先 `git fetch`**——本文撰写时最新提交为 2026-06-28。
>
> **本文第五章的每一条都做过验证。** 前三条不是静态推断，而是实际编译运行复现的：
> 用 sysroot 里的 json-c 头文件把 libubox + ubus 完整构建出来（含 CMake 自带的
> `ubusd-san` ASan 目标），写了三个 PoC 程序和一个 `close()` 的 LD_PRELOAD 追踪器，
> 在 user namespace 里跑通了整个场景。5.1 的 ASan 报告、5.2/5.3 的 backtrace
> 都是原文粘贴。5.4~5.9 是逐行核对的结论，未做运行时复现，文中会逐条注明。
>
> Lua 绑定（`lua/`）和 fuzz 语料未纳入分析。

---

## 目录

1. [代码架构](#一代码架构)
2. [设计原理](#二设计原理)
3. [使用方法](#三使用方法)
4. [可以优化的点](#四可以优化的点)
5. [已核验的潜在 bug](#五已核验的潜在-bug)
6. [复现环境](#六复现环境)

---

## 一、代码架构

### 1.1 构建产物划分

`CMakeLists.txt` 切出四个产物：

| 产物 | 源文件 | 依赖 |
|------|--------|------|
| `libubus.so` | `libubus.c` `libubus-io.c` `libubus-obj.c` `libubus-sub.c` `libubus-req.c` `libubus-acl.c` | 仅 `libubox` |
| `ubusd_library.a` | `ubusd.c` `ubusd_proto.c` `ubusd_id.c` `ubusd_obj.c` `ubusd_event.c` `ubusd_acl.c` `ubusd_monitor.c` | `libubox` + `blobmsg_json` + `json-c` |
| `ubusd` | `ubusd_main.c` + 上面的静态库 | 同上 |
| `ubus`（CLI，源码 `cli.c`） | `cli.c` | `libubus` + `blobmsg_json` + `json-c` |

**客户端库不依赖 json-c，只有守护进程和 CLI 依赖。** 守护进程需要它是因为
`ubusd_acl.c` 要读 `/usr/share/acl.d/*.json`；CLI 需要它是因为要把 blobmsg
打印成 JSON。这延续了 libubox「核心零 JSON 依赖」的分层。

`ubusd_library` 被单独拆成静态库（`c413be9`）纯粹是为了让 `UNIT_TESTING=ON`
时能构建 `ubusd-san` 这个 ASan/UBSan/LSan 版本——本文 5.1 就是靠它抓到的。

编译选项：`-Os -g3 -std=gnu99 -Wall -Werror`，gcc>6 时追加
`-Wextra -Werror=format-security -Werror=format-nonliteral`。

### 1.2 最关键的一条架构事实：守护进程与客户端库是两套独立实现

`ubusd*` 和 `libubus*` **没有任何共享的 .c 文件**，唯一共享的是三个头：

- `ubusmsg.h` —— 线协议（消息头、11 种消息类型、14 个属性、14 个状态码）
- `ubus_common.h` —— 一个 `ubus_strmatch_len()` 内联函数
- `libubus.h` —— 只有客户端用

也就是说，同一个协议被实现了两遍：收包、发包、blob 解析、fd 传递，两边各写一套。

这带来的好处是解耦（ubusd 可以完全不 link libubus），代价是**约定只能靠自觉维持**。
本文 5.2 和 5.3 正是这个代价的直接产物：libubus 在 `libubus.c:77-80` 用注释把
fd 所有权约定写得很清楚——

```c
/*
 * Note: any SCM_RIGHTS fd attached to the original message is not propagated
 * to the queued copy — the caller must close it (or hand it off) separately.
 */
static void
ubus_queue_msg(struct ubus_context *ctx, struct ubus_msghdr_buf *buf)
```

——而 ubusd 在 `ubusd.c:23-37` 做的恰好相反，排队副本**继承**了 fd：

```c
static struct ubus_msg_buf *ubus_msg_ref(struct ubus_msg_buf *ub)
{
	struct ubus_msg_buf *new_ub;
	if (ub->refcount == USES_EXTERNAL_BUFFER) {
		new_ub = ubus_msg_new(ub->data, ub->len, false);
		if (!new_ub)
			return NULL;
		memcpy(&new_ub->hdr, &ub->hdr, sizeof(struct ubus_msghdr));
		new_ub->fd = ub->fd;          /* 裸 fd 号被复制，没有 dup() */
		return new_ub;
	}

	ub->refcount++;
	return ub;
}
```

两个副本各自持有同一个 fd 号，各自都会在 `ubus_msg_free()` 里 `close()` 它。

### 1.3 分层

| 层 | 文件 | 职责 |
|----|------|------|
| 线协议 | `ubusmsg.h` | 消息头 / 类型 / 属性 / 状态码 |
| ubusd I/O | `ubusd_main.c` `ubusd.c` | accept、recvmsg/sendmsg、tx 队列、msgbuf 引用计数 |
| ubusd 路由 | `ubusd_proto.c` | `handlers[]` 消息分发表、客户端生命周期 |
| ubusd 对象模型 | `ubusd_obj.c` `ubusd_id.c` | 三棵 AVL 树、随机 ID 分配、订阅关系 |
| ubusd 策略 | `ubusd_acl.c` `ubusd_event.c` `ubusd_monitor.c` | 三个「系统对象」 |
| libubus I/O | `libubus-io.c` | `writev_retry` / `recv_retry`、msgbuf 伸缩、重连 |
| libubus 分发 | `libubus.c` | `ubus_process_msg` + pending 队列 + 连接管理 |
| libubus 请求 | `libubus-req.c` | 请求匹配、同步等待、notify 多播状态机 |
| libubus 对象 | `libubus-obj.c` `libubus-sub.c` | 本地对象注册、方法分发、订阅 |
| libubus ACL | `libubus-acl.c` | 客户端侧 ACL 缓存 |
| 工具 | `cli.c` | `ubus` 命令行（7 个子命令） |

### 1.4 三个系统对象

`ubusd_obj.c:227-235` 的 `__constructor` 在 `main()` 之前建好三棵树并注册：

| ID | 常量 | 文件 | 用途 |
|----|------|------|------|
| 1 | `UBUS_SYSTEM_OBJECT_EVENT` | `ubusd_event.c` | `register` / `send` |
| 2 | `UBUS_SYSTEM_OBJECT_ACL` | `ubusd_acl.c` | `query` |
| 3 | `UBUS_SYSTEM_OBJECT_MONITOR` | `ubusd_monitor.c` | `add` / `remove` |

它们的 `client == NULL`（不属于任何连接）且 `path.key == NULL`（不在 `path` 树里，
所以 `ubus list` 看不到它们），通过函数指针 `obj->recv_msg` 而不是转发来处理调用
（`ubusd_proto.c:347-348`）。普通对象的 ID 被强制 ≥ `UBUS_SYSTEM_OBJECT_MAX`(1024)，
给这个低号段留了 1024 个位置。

---

## 二、设计原理

### 2.1 ubusd 是纯转发器，不理解 payload

`ubusd_proto.c:26-35` 的解析策略只声明了 8 个属性：

```c
static const struct blob_attr_info ubus_policy[UBUS_ATTR_MAX] = {
	[UBUS_ATTR_SIGNATURE] = { .type = BLOB_ATTR_NESTED },
	[UBUS_ATTR_OBJTYPE]   = { .type = BLOB_ATTR_INT32 },
	[UBUS_ATTR_OBJPATH]   = { .type = BLOB_ATTR_STRING },
	[UBUS_ATTR_OBJID]     = { .type = BLOB_ATTR_INT32 },
	[UBUS_ATTR_STATUS]    = { .type = BLOB_ATTR_INT32 },
	[UBUS_ATTR_METHOD]    = { .type = BLOB_ATTR_STRING },
	[UBUS_ATTR_USER]      = { .type = BLOB_ATTR_STRING },
	[UBUS_ATTR_GROUP]     = { .type = BLOB_ATTR_STRING },
};
```

`UBUS_ATTR_DATA` **不在其中**——业务数据对 ubusd 是完全不透明的字节块，
`ubusd_forward_invoke()` 用 `blob_put(&b, UBUS_ATTR_DATA, blob_data(data), blob_len(data))`
原样搬运。

这个决定贯穿全局：ubusd 不需要知道任何业务 schema，新增服务不需要动 ubusd。
代价是 **ACL 的粒度天然只能到 `对象 + 方法`，做不了参数级授权**——想限制
"只能重启 wifi 接口、不能重启 wan" 就必须在服务端自己实现。

解析用的是 `blob_parse_untrusted()` 而不是 `blob_parse()`，这是对不可信输入的
正确选择（`blobmsg_check_attr` 的注释明说普通版本只能用于可信数据）。

### 2.2 三棵 AVL 树 + 随机 ID

`ubusd_obj.c:17-19`：

```c
struct avl_tree obj_types;   /* uint32 id  -> ubus_object_type */
struct avl_tree objects;     /* uint32 id  -> ubus_object     */
struct avl_tree path;        /* const char* -> ubus_object    */
```

`objects` 按 ID 查（invoke 用），`path` 按名字查（lookup 用），一个对象同时挂在两棵树上。
`obj_types` 做方法签名去重：同一个服务注册 N 个同类型对象时，签名只存一份，靠 `refcount` 管理。

ID 分配（`ubusd_id.c:75-91`）是随机而非递增的：

```c
	do {
		do {
			if (read_random(&id->id, sizeof(id->id)) != sizeof(id->id))
				return false;
		} while (id->id < UBUS_SYSTEM_OBJECT_MAX);
	} while (avl_insert(tree, &id->avl) != 0);
```

**为什么必须随机**：objid 本身就是一种 capability——只要知道 objid 就能对它
`UBUS_MSG_INVOKE`（ACL 之外没有别的检查）。递增 ID 可以被枚举和预测，随机 ID 不能。
内层循环保证不落进系统对象号段，外层循环处理碰撞（`239edcb` 修的就是这里
`continue` 用错导致跳过重试）。

`24864e7` 换成 `GRND_INSECURE` 是个务实的取舍：ID 只需要"难猜"不需要"密码学强度"，
而 `getrandom()` 默认会在熵池未初始化时阻塞——在熵源匮乏的路由器上足以卡死开机。

### 2.3 消息缓冲：常见路径零拷贝，慢路径才拷贝

`ubusd.h:32-38` 的 `refcount` 有一个特殊值：

```c
struct ubus_msg_buf {
	uint32_t refcount; /* ~0: uses external data buffer */
	struct ubus_msghdr hdr;
	struct blob_attr *data;
	int fd;
	int len;
};
```

ubusd 全程用**一个**全局 `struct blob_buf b` 组装出站消息，然后
`ubus_msg_new(b.head, len, /*shared=*/true)` 建一个只存指针、不拷贝数据的 msg_buf。
只要它在 `b` 被下一次 `blob_buf_init()` 覆盖之前发完，就是彻底零拷贝。

一旦对端 socket 满了需要排队，`ubus_msg_ref()` 就**必须**把它转成私有副本——因为
`b` 马上会被下一条消息覆盖。于是引用计数在这里承担了双重角色：真正的共享计数
（`refcount++`）和「外部缓冲，必须深拷贝」（`~0U`）。

同样的思路还体现在每个客户端的 `retmsg` 上（`ubusd_proto.c:555-570`）：

```c
static int ubusd_proto_init_retmsg(struct ubus_client *cl)
{
	struct blob_buf *b = &cl->b;

	if (blob_buf_init(&cl->b, 0))
		return -1;
	blob_put_int32(&cl->b, UBUS_ATTR_STATUS, 0);

	/* we make the 'retmsg' buffer shared with the blob_buf b, to reduce mem duplication */
	cl->retmsg = ubus_msg_new(b->head, blob_raw_len(b->head), true);
	...
}
```

每个连接预先构造好一条状态回复，此后所有 `UBUS_STATUS` 应答只需改 4 字节
status 和 8 字节头（`ubusd_proto.c:524-527`、`551`），零分配。ubus 的绝大多数
消息都是状态回复，这一条省下的是主路径上的分配。

### 2.4 客户端的重入模型：`stack_depth` + pending 队列

这是 libubus 里最需要理解的一点。ubus 的同步 API 并不阻塞在一个 `read()` 上：

```
ubus_invoke()
  └─ ubus_complete_request()          libubus-req.c:150
       └─ while (!req->status_msg)
            └─ ubus_poll_data()       libubus-io.c:346
                 └─ ubus_handle_data()   ← 继续处理所有到达的消息
```

也就是说，**等待应答期间，本进程的其他 ubus 对象方法照样会被调用**。这让
"在方法 handler 里再发一次 `ubus_invoke`" 成为合法用法（procd、netifd 大量依赖），
但也带来 handler 无限递归的风险。

解法是 `ctx->stack_depth`（`libubus.c:113-122`）：

```c
	case UBUS_MSG_INVOKE:
		if (ctx->stack_depth) {
			ubus_queue_msg(ctx, buf);
			break;
		}

		ctx->stack_depth++;
		ubus_process_obj_msg(ctx, buf, fd);
		ctx->stack_depth--;
		return;
```

只要已经在处理某个对象消息，新来的 INVOKE/NOTIFY/UNSUBSCRIBE 就被**拷贝**进
`ctx->pending`，由 `pending_timer`（1 ms）或事件循环顶部
（`libubus-io.c:318-319`、`337-338`）排空。深度永远不超过 1。

和 libubox 对照很有意思：libubox 用 free guard / 全局游标解决"回调里删对象"，
ubus 用队列解决"回调里收消息"。前者保护数据结构，后者保护调用栈。

### 2.5 msgbuf 的「借出—归还」

`libubus-obj.c:153-165`：

```c
	if (buf == &ctx->msgbuf) {
		prev_data = buf->data;
		buf->data = NULL;
	}

	cb(ctx, hdr, obj, attrbuf, fd);

	if (prev_data) {
		if (buf->data)
			free(prev_data);
		else
			buf->data = prev_data;
	}
```

`ctx->msgbuf` 是收包用的唯一缓冲。handler 里若递归收到新消息，
`alloc_msg_buf()` 看到 `data == NULL` 就会 `realloc(NULL, len)` 分配一块新的
（`libubus-io.c:246-247`），于是本层正在使用的数据不会被覆盖。回来时判断指针
有没有被换过：换过就释放旧的，没换过就物归原主。

这是一个零成本的写时分配：不递归时（绝大多数情况）一次分配都不多。

### 2.6 msgbuf 的迟滞收缩

`libubus-io.c:240-269`，`alloc_msg_buf()` 把长度按 64 KB（`UBUS_MSG_CHUNK_SIZE`）
向上取整，并且：

```c
	if (len < buf_len &&
	    ++ctx->msgbuf_reduction_counter > UBUS_MSGBUF_REDUCTION_INTERVAL) {
		ctx->msgbuf_reduction_counter = 0;
		buf_len = 0;
	}
```

连续 16 次用不满当前容量才收缩一次。这是标准的迟滞设计，防止大小消息交替时
反复 `realloc` 抖动——一次大 `ubus list` 之后不会立刻把缓冲还回去。

### 2.7 ACL：字典序前缀树 + 单调剪枝

`ubusd_acl.c:117-176`、`ubusd_acl.c:611-645`、`ubusd_event.c:164-186` 三处用了
同一个模式：

```c
	avl_for_each_element(&ubusd_acls, acl, avl) {
		const char *key = acl->avl.key;
		int cur_match_len;
		bool full_match;

		full_match = ubus_strmatch_len(obj, key, &cur_match_len);
		if (cur_match_len < match_len)
			break;              /* 公共前缀开始变短，后面不可能更长 */

		match_len = cur_match_len;
		...
	}
```

利用字符串 AVL 按 `strcmp` 有序这一点：遍历时公共前缀长度先增后减，一旦开始减
就可以提前退出。复杂度是 O(匹配区间) 而非 O(全表)。注释在 `ubusd_acl.c:111-116`
写得很清楚。

几条重要的策略事实：

- **root 完全绕过 ACL**：`ubusd_acl.c:108` 的 `if (!cl || !cl->uid || !obj) return 0;`。
  `cl == NULL` 是 ubusd 自己发的事件，`!cl->uid` 是 root，`!obj` 是无路径对象。
- 三个系统对象 `path.key == NULL`，因此 `ubusd_handle_invoke` 里那次
  `ubusd_acl_check(cl, obj->path.key, ...)` 对它们直接放行；它们各自实现检查
  （monitor 要求 `uid==0 && gid==0`，event 的 `register` 走 `UBUS_ACL_LISTEN`）。
- ACL 文件必须 root 拥有、且 group/other 不可写（`ubusd_acl.c:579-586`），否则跳过。
- `SIGHUP` 触发重载。`11ea1b3` 把信号处理器改成了 self-pipe + `uloop_fd`
  （`ubusd_main.c:251-272`），因为原来直接在处理器里调 `ubusd_acl_load()` 是
  异步信号不安全的。

### 2.8 LOOKUP 的增量续传

`ubus list` 的应答可能有几百 KB，远超 socket 缓冲。ubusd 的做法是边遍历边发，
发不动就把迭代器状态存下来（`ubusd_proto.c:204-240`）：

```c
		avl_for_element_range(obj, avl_last_element(&path, obj, path), obj, path) {
			/* Keep sending objects until buffering starts */
			if (list_empty(&cl->tx_queue)) {
				ubusd_send_obj(cl, ub, obj);
			} else {
				/* Queue command and continue on the next call */
				int ret;

				if (cmd == NULL) {
					ret = ubus_client_cmd_queue_add(cl, ub, obj);
				} else {
					cmd->obj = obj;
					ret = -2;
				}
				return ret;
			}
		}
```

返回值 `-2` 是一个特殊约定，表示"命令尚未完成、已入队"，
`ubusd_proto_receive_message()` 见到它就**不释放** `ub`（`ubusd_proto.c:542-544`），
所有权转移给了 `cl->cmd_queue`。等 socket 可写时
`ubus_client_cmd_queue_process()`（`ubusd_main.c:45-58`）再续上。

这是一个"把迭代器状态存到堆上、跨事件循环恢复"的设计——**也正是 5.1 的病根：
存下来的是一个裸 `struct ubus_object *`，而对象随时可能因为属主断开而消失。**

### 2.9 fd 传递（SCM_RIGHTS）

ubus 支持随消息传一个 fd，procd 的容器管理（`uxc`）和 rpcd 的文件传输依赖它。

发送侧 `ubus_msg_writev()`（`ubusd.c:83-134`）有一个关键判断：

```c
	*pfd = ub->fd;
	if (ub->fd < 0 || offset) {
		msghdr.msg_control = NULL;
		msghdr.msg_controllen = 0;
	}
```

`offset` 非 0 意味着这是断点续传，fd 已经随第一批字节传过去了，不能重复传。

接收侧在 `client_cb`（`ubusd_main.c:133-146`）用 `cl->pending_msg_fd` 暂存，
因为消息头和消息体可能分多次 `recvmsg` 到达，而 cmsg 只跟着第一批。

客户端侧的三个 API 组成一条所有权链：`ubus_invoke_fd()` 送出，
`ubus_request_get_caller_fd()` 在 handler 里取走（取走后置 -1），
`ubus_request_set_fd()` 随应答回传。**第五章有三条 bug 都出在这条链上。**

### 2.10 monitor：旁路镜像

`ubusd_monitor_message()` 在 `ubus_msg_send()` 入口和 `ubusd_proto_receive_message`
之前各调一次，把每条经过 ubusd 的消息重新打包成 `UBUS_MSG_MONITOR` 发给所有
监听者。靠 `if (ub->hdr.type != UBUS_MSG_MONITOR)`（`ubusd.c:170`）防止自我递归，
靠 `if (list_empty(&monitors)) return;`（`ubusd_monitor.c:82-83`）在没人监听时
提前退出。只有 root 能开启。

---

## 三、使用方法

### 3.1 服务端：注册对象

```c
enum { HELLO_ID, HELLO_MSG, __HELLO_MAX };

static const struct blobmsg_policy hello_policy[__HELLO_MAX] = {
	[HELLO_ID]  = { .name = "id",  .type = BLOBMSG_TYPE_INT32 },
	[HELLO_MSG] = { .name = "msg", .type = BLOBMSG_TYPE_STRING },
};

static int hello(struct ubus_context *ctx, struct ubus_object *obj,
                 struct ubus_request_data *req, const char *method,
                 struct blob_attr *msg)
{
	struct blob_attr *tb[__HELLO_MAX];
	static struct blob_buf b;

	blobmsg_parse(hello_policy, __HELLO_MAX, tb, blob_data(msg), blob_len(msg));

	blob_buf_init(&b, 0);
	blobmsg_add_string(&b, "message", "hello");
	ubus_send_reply(ctx, req, b.head);
	return 0;                          /* 返回值即 UBUS_STATUS_* */
}

static const struct ubus_method test_methods[] = {
	UBUS_METHOD("hello", hello, hello_policy),
	UBUS_METHOD_NOARG("ping", ping),
};

static struct ubus_object_type test_type = UBUS_OBJECT_TYPE("test", test_methods);

static struct ubus_object test_object = {
	.name      = "test",
	.type      = &test_type,
	.methods   = test_methods,
	.n_methods = ARRAY_SIZE(test_methods),
};

int main(void)
{
	struct ubus_context *ctx;

	uloop_init();
	ctx = ubus_connect(NULL);          /* NULL = UBUS_UNIX_SOCKET 编译期路径 */
	ubus_add_uloop(ctx);
	ubus_add_object(ctx, &test_object);
	uloop_run();
	ubus_free(ctx);
	uloop_done();
}
```

`ubus_add_object()` 必须在 `uloop_init()` 之后、`uloop_run()` 之前；它内部是同步的
（会跑一次嵌套的 `ubus_complete_request`）。

### 3.2 客户端：调用方法

```c
	uint32_t id;

	if (ubus_lookup_id(ctx, "test", &id))
		return -1;

	blob_buf_init(&b, 0);
	blobmsg_add_u32(&b, "id", 42);
	ubus_invoke(ctx, id, "hello", b.head, result_cb, NULL, 3000 /* ms */);
```

异步版本是 `ubus_invoke_async()` + `ubus_complete_request_async()`。
注意 `ubus_invoke()` 内部会重入事件处理（见 2.4），在事件回调里调用它是安全的。

### 3.3 事件（广播，无订阅关系）

```c
	/* 发送方 */
	blob_buf_init(&b, 0);
	blobmsg_add_string(&b, "iface", "wan");
	ubus_send_event(ctx, "network.interface", b.head);

	/* 接收方 —— 模式支持尾部 '*' 前缀匹配 */
	static struct ubus_event_handler ev = { .cb = event_cb };
	ubus_register_event_handler(ctx, &ev, "network.*");
```

`ubus.` 前缀被保留（`ubusd_event.c:234-235`），用户不能发 `ubus.object.add` 之类。

### 3.4 订阅（点对点，有生命周期）

```c
	static struct ubus_subscriber sub = {
		.cb        = notify_cb,
		.remove_cb = remove_cb,      /* 目标对象消失时回调 */
		.new_obj_cb = filter_cb,     /* 非 NULL 则自动订阅新出现的匹配对象 */
	};

	ubus_register_subscriber(ctx, &sub);
	ubus_lookup_id(ctx, "test", &id);
	ubus_subscribe(ctx, &sub, id);

	/* 发布方 */
	if (test_object.has_subscribers)
		ubus_notify(ctx, &test_object, "update", b.head, -1 /* 不等回复 */);
```

事件和订阅的区别：事件是无状态广播，发布方不知道谁在听；订阅是有状态的，
发布方能通过 `obj->has_subscribers` + `subscribe_cb` 知道有没有人在听，
从而跳过昂贵的数据采集。`ubus_notify` 的 `timeout < 0` 表示不等回复
（`UBUS_ATTR_NO_REPLY`），最多同时跟踪 `UBUS_MAX_NOTIFY_PEERS`(16) 个订阅者。

### 3.5 延迟回复

handler 里做不完的事（比如要等一个子进程）：

```c
static struct ubus_request_data saved_req;

static int slow_method(struct ubus_context *ctx, ...)
{
	ubus_defer_request(ctx, req, &saved_req);
	uloop_timeout_set(&work_timer, 1000);
	return 0;               /* 此时不会自动发应答 */
}

static void work_done(struct uloop_timeout *t)
{
	ubus_send_reply(ctx, &saved_req, b.head);
	ubus_complete_deferred_request(ctx, &saved_req, UBUS_STATUS_OK);
}
```

> **注意**：如果这个请求带了 caller fd，务必在 `ubus_defer_request()` **之前**
> 调用 `ubus_request_get_caller_fd()` 把 fd 取走。原因见 5.5。

### 3.6 自动重连

```c
static struct ubus_auto_conn conn = { .cb = ubus_connect_handler };
ubus_auto_connect(&conn);        /* 内部 1s 重试，重连后自动重新注册所有对象 */
```

重连后的对象重注册在 `ubus_refresh_state()`（`libubus-io.c:381-404`）：清空所有
type id 和 obj id，把对象从 AVL 树里摘干净再逐个重新 `ubus_add_object()`。

### 3.7 channel：socketpair 上的 ubus

```c
	int remote_fd;
	ubus_channel_create(&ctx, &remote_fd, request_handler);
	/* 把 remote_fd 通过 SCM_RIGHTS 传给子进程/容器 */
```

`ubus_channel_connect()` 把 `local_id` 设成 `UBUS_CLIENT_ID_CHANNEL`(1)，此后
`ubus_context_is_channel()` 为真，大部分需要 ubusd 参与的 API（lookup、notify、
subscribe、register_event_handler）都直接返回 `INVALID_ARGUMENT`——channel 上
只有点对点的 invoke/reply。procd 用它和容器通信。

### 3.8 CLI

```sh
ubus list                      # 列出所有对象
ubus -v list 'network.*'       # 带方法签名，支持尾部通配
ubus call test hello '{"id":1}'
ubus listen network.interface  # 监听事件
ubus send myevent '{"a":1}'
ubus subscribe test            # 订阅对象通知
ubus wait_for netifd           # 等对象出现（脚本里等服务就绪）
ubus monitor -m invoke -M t    # 抓包，需 root
ubus -S list                   # 脚本友好输出
```

### 3.9 ACL 文件

`/usr/share/acl.d/xxx.json`，必须 root:root 拥有、不能 group/other 可写：

```json
{
  "user": "nobody",
  "access": {
    "network.interface": { "methods": ["status", "dump"] },
    "system":            { "methods": ["*"] }
  },
  "publish":   [ "myservice" ],
  "subscribe": [ "network.*" ],
  "listen":    [ "network.interface.*" ],
  "send":      [ "myevent" ]
}
```

五种权限对应 `enum ubusd_acl_type`：`ACCESS`(调用方法)、`PUBLISH`(注册对象)、
`SUBSCRIBE`(订阅)、`LISTEN`(注册事件模式)、`SEND`(发事件)。改完发 `SIGHUP` 给 ubusd。

### 3.10 三个容易踩的坑

1. **`ubus_parse_msg()` 返回的是一个 `static` 数组**（`libubus-io.c:44-50`、
   `ubusd_proto.c:22`）。任何嵌套的 ubus 操作都会覆盖它，解析结果不能跨调用保存。
2. **`libubus` 内部共享一个全局 `struct blob_buf b`**。所以
   `ubus_register_event_handler()` 特意用了第二个缓冲（`libubus.c:270-271`
   的注释 `/* use a second buffer, ubus_invoke() overwrites the primary one */`）。
   自己的代码也要用自己的 `blob_buf`。
3. **同步 API 会重入事件处理**（2.4）。handler 里调 `ubus_invoke()` 是允许的，
   但要意识到期间对象状态可能被其他消息改变。

---

## 四、可以优化的点

### 4.1 把 msg_buf 的 fd 所有权写成一份显式契约

守护进程和客户端库对"排队副本要不要继承 fd"的答案是相反的（见 1.2），而这个
约定目前只存在于 libubus 的一段注释里。5.2 和 5.3 是同一个根因的两个症状。

建议：给 `struct ubus_msg_buf` 的 `fd` 字段加明确注释说明谁负责 `close`，
并且在 `ubus_msg_ref()` 里要么 `dup()` 要么把源的 `fd` 置 -1（转移而非复制）。
后者更符合零分配的整体风格。

### 4.2 `ubus_parse_msg()` 改成调用者传入数组

两个实现都用 `static struct blob_attr *attrbuf[UBUS_ATTR_MAX]`。在一个允许
重入的代码库里，共享的静态解析结果是长期隐患——目前靠"所有使用点都在嵌套调用
之前取完值"的巧合成立。改成 `ubus_parse_msg(msg, len, attrbuf)` 是一次机械改动，
可以彻底消除这类风险。

### 4.3 `cmd_queue` 应存 objid 而非对象指针

这是 5.1 的正确修法。`struct ubus_client_cmd` 存 `uint32_t obj_id`，恢复时
`ubus_find_id(&objects, obj_id)`；找不到就从头开始或直接结束。对象消失最多导致
"这次 list 少列几个"，而不是 UAF。

顺带能解决另一个语义问题：目前即使没有 UAF，`ubus list` 的结果在续传期间也不是
一致快照——中途新增的对象可能被重复列出或漏掉。

### 4.4 消息丢弃应该可观测

`ubus_msg_enqueue()`（`ubusd.c:143-163`）有三条静默返回路径：超过 txq 上限、
`calloc` 失败、`ubus_msg_ref` 失败。三条都是 `return;`，没有计数器、没有日志、
没有给调用者任何反馈。

线上排查"我的 notify 怎么丢了"时这是死路。至少应该有一个全局丢弃计数器，
最好能通过 `ubus.monitor` 或一个新的统计对象暴露出来。

同时 `UBUS_CLIENT_MAX_TXQ_LEN` 直接等于 `UBUS_MAX_MSGLEN`(1 MB)，意味着
**单个**最大消息就能占满整个队列——这个上限的语义（"队列"上限 vs "单消息"上限）
是混淆的，也直接导致了 5.6。

### 4.5 monitor 开启后的开销可以显著降低

`ubusd_monitor_message()` 对每条经过的消息都做一次完整重打包：
`blob_buf_init` + 5 次 `blob_put_int*` + 一次全量 `blob_put(DATA)` 拷贝，
然后给每个监听者再走一遍 `ubus_msg_send`（外部缓冲，排队时还要深拷贝）。

而过滤（`-m <type>` / `-M r|t`）完全发生在 CLI 侧（`cli.c:508-512`）。
把类型掩码在 `ubus_monitor_start` 时传给 ubusd，就能在 `ubusd_monitor.c:82`
那个早退判断里一起short-circuit 掉，省下绝大部分拷贝。

### 4.6 `mkdir_sockdir()` 应该用实际的 socket 路径

见 5.7。顺带：失败时应该打印错误而不是静默 `goto out`。

### 4.7 用 `SO_PEERGROUPS` 替代读 `/proc/<pid>/status`

见 5.8。Linux 4.13+ 提供 `SO_PEERGROUPS`，直接从内核拿连接建立时的补充组列表，
没有 TOCTOU，也不需要 `/proc` 可读。

### 4.8 `ubus_reconnect()` 不该在阻塞 socket 上同步读 HELLO

`libubus-io.c:430-448`：

```c
	ctx->sock.fd = usock(USOCK_UNIX, path, NULL);   /* 阻塞 socket */
	...
	if (read(ctx->sock.fd, &hdr, sizeof(hdr)) != sizeof(hdr))
		goto out_close;
```

`O_NONBLOCK` 要到 455 行才设上。如果 ubusd 活着但卡住（比如正在处理一个巨大的
lookup），所有正在启动的客户端都会**无限期阻塞在这里**，没有超时、没有中断点。
在 OpenWrt 的启动序列上这会表现为整机 hang。

另外这里假设 HELLO 一定能一次读全——unix stream socket 上短读是合法的，
虽然 12 字节的消息实际上不会被拆开。加超时 + 循环读是更稳妥的写法。

### 4.9 `ubus_refresh_state()` 的 `alloca` 与对象数量成正比

`libubus-io.c:394`：`objs = alloca(ctx->objects.count * sizeof(*objs));`
注册 10 万个对象的进程重连时会爆栈。改成 `calloc` + 失败回退即可，
这条路径不在性能热点上。

### 4.10 `libubus-acl.c` 的两个结构性问题

- `acl_cmp()`（`libubus-acl.c:26-43`）在 `user` 或 `group` 为 NULL 时**跳过**该字段
  比较而不是定义一个确定的序。这使比较器不满足传递性，AVL 树的行为在混合了
  "只有 user" 和 "只有 group" 的 ACL 条目时是未定义的。
- `acl_req`（`libubus-acl.c:23`）是全局单例。`acl_query()` 里的
  `ubus_invoke_async()` 会 `memset(req, 0, ...)`——如果上一次查询还挂在
  `ctx->requests` 链表上（ACL 序列事件密集时），这次 memset 会把它的
  `list` 指针清零，而链表邻居仍指向它，链表就此损坏。

### 4.11 空树上的 `avl_first_element` 靠巧合正确

`ubusd_proto.c:219-222`：

```c
		if (obj == NULL)
			obj = avl_first_element(&path, obj, path);

		avl_for_element_range(obj, avl_last_element(&path, obj, path), obj, path) {
```

`path` 树为空时 `avl_first_element` 返回的是由 `tree->list_head` 反推出来的
伪对象指针。这里之所以没崩，是因为 `avl_for_element_range` 的终止条件
`element->path.list.prev != &last->path.list` 恰好在这种情况下立即为假。

结论正确，但依赖的是两个宏的实现细节耦合。加一个显式的 `avl_is_empty(&path)`
判断，成本为零，可读性和健壮性都更好。

### 4.12 `ubus_find_notify_id()` 的下标无上界

`libubus-req.c:395-410`，`for (i = 0; pending; i++, pending >>= 1)` 访问
`n->id[i]`，而 `id[]` 只有 `UBUS_MAX_NOTIFY_PEERS + 1`(17) 个元素。目前安全，
因为 `pending` 的置位全部来自 `ubus_process_notify_status()` 且那里有
`if (idx == UBUS_MAX_NOTIFY_PEERS + 1) break;`。但这是一个只靠远端约束成立的
边界——加一句 `i < UBUS_MAX_NOTIFY_PEERS + 1` 是纯收益。

---

## 五、已核验的潜在 bug

> 5.1 / 5.2 / 5.3 是**实际编译运行复现**的，附原始输出。
> 5.4 ~ 5.9 是逐行核对的静态结论，未做运行时复现。

### 5.1 【高】ubusd：LOOKUP 续传游标悬垂，导致堆 use-after-free，可远程打崩 ubusd

**位置**：`ubusd.h:45-49`、`ubusd_proto.c:189-240`、`ubusd_main.c:45-58`

`struct ubus_client_cmd` 把一个裸对象指针当作跨事件循环的遍历游标：

```c
struct ubus_client_cmd {
	struct list_head list;
	struct ubus_msg_buf *msg;
	struct ubus_object *obj;     /* 续传起点，可能在两次事件之间被 free */
};
```

而 `ubusd_free_object()`（`ubusd_obj.c:202-225`）在对象销毁时**不会**去任何客户端的
`cmd_queue` 里把指向它的游标清掉——事实上 `cmd_queue` 只在 `ubusd_proto.c` 和
`ubusd_main.c` 各两处出现，二者之间没有任何联系。

于是：

1. 客户端 A 发起 `ubus list`（无 OBJPATH 的 LOOKUP），对象很多，A 的 socket 缓冲写满；
2. ubusd 在 `ubusd_proto.c:231` 把「命令 + 当前对象指针」存进 `A->cmd_queue` 并返回 `-2`；
3. 这期间对象的属主进程 B 退出 → `handle_client_disconnect` → `ubusd_proto_free_client`
   → `ubusd_free_object` → `free(obj->path.key)` + `free(obj)`；
4. A 开始读取，socket 可写 → `ubus_client_cmd_queue_process` → `ubusd_cmd_lookup`
   → `__ubusd_handle_lookup` 从**已释放的** `cmd->obj` 恢复遍历。

**实测结果**（`ubusd-san`，600 个对象，客户端把 `SO_RCVBUF` 调到 1 KB 后停止读取，
3 秒内 kill 掉对象属主）：

```
==1829041==ERROR: AddressSanitizer: heap-use-after-free on address 0x5020000023d0 ...
READ of size 2 at 0x5020000023d0 thread T0
    #1 0x405a4d in blob_put_string /tmp/inst/include/libubox/blob.h:209
    #2 0x405b4e in ubusd_send_obj        ubusd_proto.c:171
    #4 0x405cc3 in __ubusd_handle_lookup ubusd_proto.c:225
    #5 0x406306 in ubusd_cmd_lookup      ubusd_proto.c:295
    #6 0x404722 in ubus_client_cmd_queue_process ubusd_main.c:50
    #7 0x404722 in client_cb             ubusd_main.c:115

freed by thread T0 here:
    #1 0x4069ba in ubusd_free_object     ubusd_obj.c:217

previously allocated by thread T0 here:
    #1 0x406bbd in ubusd_create_object   ubusd_obj.c:150

SUMMARY: AddressSanitizer: heap-use-after-free in strlen.part.0
==1829041==ABORTING
```

**ubusd 进程当场死亡。** 在 OpenWrt 上 ubusd 是整机 IPC 的中枢，它挂掉等于
netifd、procd、rpcd、luci 全部失联。

值得强调的是：**触发它不需要任何特权，也不需要攻击者控制被释放的那个对象。**
攻击方只需要能连上 ubus 并发一个 LOOKUP 然后不读（这是 ACL 管不到的，
`ubusd_handle_lookup` 没有任何权限检查）。而"某个服务在这几百毫秒里重启"
在真实系统上是常态——也就是说这个崩溃**有可能自发触发**，不一定需要攻击者。

未开 ASan 的生产构建里，后果是读到被复用的堆内存：轻则发出错乱的对象列表，
重则 `blob_put_string` 对一个非字符串指针做 `strlen` 而越界。

**修法**：见 4.3，`cmd->obj` 改存 `uint32_t` objid，恢复时重新查树。

---

### 5.2 【高】ubusd：转发带 fd 的 INVOKE 时，同一个 fd 被 close 两次

**位置**：`ubusd_proto.c:77-92` 与 `ubusd_proto.c:546`

```c
void
ubus_proto_send_msg_from_blob(struct ubus_client *cl, struct ubus_msg_buf *ub,
			uint8_t type)
{
	/* keep the fd to be passed if it is UBUS_MSG_INVOKE */
	int fd = ub->fd;
	ub = ubus_reply_from_blob(ub, true);   /* ub 被重新赋值指向新消息 */
	if (!ub)
		return;

	ub->hdr.type = type;
	ub->fd = fd;                           /* 新消息也持有同一个 fd 号 */

	ubus_msg_send(cl, ub);
	ubus_msg_free(ub);                     /* close(fd)  ← 第一次 */
}
```

局部变量 `ub` 被重新赋值，但**调用者手里的原始 `ub` 的 `fd` 字段从未被清掉**。
回到 `ubusd_proto_receive_message()`：

```c
	if (ret == -2)
		return;

	ubus_msg_free(ub);                     /* close(fd)  ← 第二次 */
```

注意 `ubusd_proto.c:532-533` 特意把 `UBUS_MSG_INVOKE` 排除在
`ubus_msg_close_fd()` 之外，正是为了让 fd 能传下去——但传下去之后没有把
原消息的所有权标记清除。

**实测**（客户端用 `ubus_invoke_fd()` 传一个 fd，`close()` 用 LD_PRELOAD 追踪）：

```
[ubusd] close(13) = 0
[ubusd] close(13) = -1 EBADF  <<< DOUBLE CLOSE
    ubus_msg_free                ubusd.c:75
    ubusd_proto_receive_message  ubusd_proto.c:548
    client_cb                    ubusd_main.c:192
    main                         ubusd_main.c:365
```

**两个后果，严重程度不同：**

*上面这个（一次写完）* 两次 close 紧挨着，中间没有任何 fd 分配，所以第二次只是
拿到 `EBADF`，实际无害。

*真正危险的是排队路径*：如果对端 socket 满了，`ubus_msg_send` → `ubus_msg_enqueue`
→ `ubus_msg_ref()` 会造出**第三个**持有同一 fd 号的副本（`ubusd.c:31`，见 1.2）。
这个副本要等到 socket 可写时才被释放——中间隔了任意长的时间，期间
`get_next_connection()` 完全可能把这个 fd 号分配给一个新连接。届时
`ubus_msg_list_free` 的那次 `close()` 就会**悄无声息地掐断一个无辜客户端的连接**。

**修法**：`ubus_proto_send_msg_from_blob` 里把原消息的 fd 置 -1（所有权转移），
并让 `ubus_msg_ref()` 要么 `dup()` 要么不继承 fd。

---

### 5.3 【高】libubus：`ubus_request_set_fd()` 路径上同一个 fd 被 close 两次

**位置**：`libubus-io.c:152-153` 与 `libubus-req.c:205-206`

`ubus_send_msg()` 自 2014 年（`8f3c5a7`）起就会关掉传给它的 fd：

```c
	ret = writev_retry(ctx->sock.fd, iov, ARRAY_SIZE(iov), fd);
	if (ret < 0)
		ctx->sock.eof = true;

	if (fd >= 0)
		close(fd);
```

而 2025-01-02 的 `d996988`（"libubus: close file descriptor after sending it from
a request"）又在调用点加了一次：

```c
void ubus_complete_deferred_request(struct ubus_context *ctx, struct ubus_request_data *req, int ret)
{
	blob_buf_init(&b, 0);
	blob_put_int32(&b, UBUS_ATTR_STATUS, ret);
	blob_put_int32(&b, UBUS_ATTR_OBJID, req->object);
	ubus_send_msg(ctx, req->seq, b.head, UBUS_MSG_STATUS, req->peer, req->fd);
	if (req->fd >= 0)
		close(req->fd);                /* 重复 —— ubus_send_msg 已经关过 */
}
```

两次 close 之间没有任何条件，这是无歧义的重复释放。

**实测**（服务端 handler 里 `open("/dev/null")` 然后 `ubus_request_set_fd()`）：

```
[server] handler opened fd 10, handing it to libubus
[server] close(10) = 0
[server] close(10) = -1 EBADF  <<< DOUBLE CLOSE
    ubus_process_invoke            libubus-obj.c:120
    ubus_process_obj_msg           libubus-obj.c:160
    ubus_process_msg               libubus.c:121
    ubus_handle_data               libubus-io.c:324
```

这里两次 close 同样紧挨着，所以现状是"只报 EBADF"。但 `ubus_complete_deferred_request`
是公开 API，服务实现可以在任意上下文调它——尤其是**真正的延迟回复**场景：
handler 先 `ubus_defer_request()`，若干秒后在定时器/子进程回调里才
`ubus_complete_deferred_request()`。那时进程早已开关过大量 fd，第二次 close
命中一个已复用的 fd 号是完全现实的。多线程程序里更是直接的 fd 撕裂。

**修法**：删掉 `libubus-req.c:205-206` 这两行（fd 所有权已经在 `ubus_send_msg`
里转移了），或者反过来让 `ubus_send_msg` 不关而由调用者统一关——但后者要改
所有调用点，前者是一行修复。

---

### 5.4 【中】ubusd：客户端断开时整个 `cmd_queue` 泄漏

**位置**：`ubusd_main.c:23-36`

```c
static void handle_client_disconnect(struct ubus_client *cl)
{
	struct ubus_msg_buf_list *ubl, *ubl2;
	list_for_each_entry_safe(ubl, ubl2, &cl->tx_queue, list)
		ubus_msg_list_free(ubl);           /* tx_queue 清了 */

	ubusd_monitor_disconnect(cl);
	ubusd_proto_free_client(cl);
	if (cl->pending_msg_fd >= 0)
		close(cl->pending_msg_fd);
	uloop_fd_delete(&cl->sock);
	close(cl->sock.fd);
	free(cl);                                  /* cmd_queue 从头到尾没被碰过 */
}
```

`ubusd_proto_free_client()`（`ubusd_proto.c:613-626`）也只处理 `cl->objects`、
`retmsg`、`cl->b`、ACL 和 id，同样不管 `cmd_queue`。

只要一个客户端在有排队 LOOKUP 的状态下断开——这恰好是 5.1 的前半段场景，
也是 `ubus list` 被 Ctrl-C 掉的自然结果——就会泄漏一个 `struct ubus_client_cmd`
加上它持有的 `struct ubus_msg_buf`。后者如果带 fd，**fd 也一并泄漏**
（`ubus_msg_free` 才会 close 它）。

ubusd 是常驻进程，反复触发可以耗尽 fd 表。

**修法**：在 `handle_client_disconnect` 里加一个 `cmd_queue` 的清理循环，
复用已有的 `ubus_client_cmd_free()`。

---

### 5.5 【中】libubus：`ubus_defer_request()` 之后 caller fd 会被提前关闭

**位置**：`libubus.h:396-403` 与 `libubus-obj.c:111-116`

```c
static inline void ubus_defer_request(struct ubus_context *ctx,
				      struct ubus_request_data *req,
				      struct ubus_request_data *new_req)
{
    (void) ctx;
    memcpy(new_req, req, sizeof(*req));    /* req_fd 被复制到副本 */
    req->deferred = true;                  /* 但原件的 req_fd 没有清 */
}
```

`ubus_process_invoke()` 里关闭的时机在判断 `deferred` **之前**：

```c
	ret = handler(ctx, obj, &req, blob_data(attrbuf[UBUS_ATTR_METHOD]),
		      attrbuf[UBUS_ATTR_DATA]);
	if (req.req_fd >= 0)
		close(req.req_fd);             /* 副本里的同一个 fd 号被关掉了 */
	if (req.deferred || no_reply)
		return;
```

于是「handler 先 defer，稍后再从副本取 caller fd」这个看起来完全合理的顺序，
拿到的是一个已经关闭（甚至已被复用）的 fd。

对比 `ubus_request_get_caller_fd()`（`libubus.h:412-418`）——它取走 fd 后会把
`req->req_fd = -1`，所有权转移做得很干净。`ubus_defer_request` 缺的正是这一步。

**目前的正确用法**是在 `ubus_defer_request()` **之前**先
`ubus_request_get_caller_fd()`，但这个顺序要求在头文件里毫无提示。

**修法**：`ubus_defer_request` 在 memcpy 之后加 `req->req_fd = -1;`，
语义与 `get_caller_fd` 保持一致。

---

### 5.6 【中】ubusd：部分写之后消息被丢弃，残留 `txq_ofs` 导致字节流错位

**位置**：`ubusd.c:143-190`

```c
void ubus_msg_send(struct ubus_client *cl, struct ubus_msg_buf *ub)
{
	...
	if (list_empty(&cl->tx_queue)) {
		written = ubus_msg_writev(cl->sock.fd, ub, 0);
		if (written < 0)
			written = 0;
		if (written >= (ssize_t) (ub->len + sizeof(ub->hdr)))
			return;

		cl->txq_ofs = written;          /* 已经承诺"后面还有 T-W 字节" */
		cl->txq_len = -written;
		uloop_fd_add(&cl->sock, ULOOP_READ | ULOOP_WRITE | ULOOP_EDGE_TRIGGER);
	}

	ubus_msg_enqueue(cl, ub);           /* 但这里可能静默失败 */
}
```

`ubus_msg_enqueue()` 有三条静默返回路径（见 4.4）。一旦在**部分写之后**丢弃：
`tx_queue` 是空的，`cl->txq_ofs` 却停在 W。此时

- 对端已经收到半条消息，永远等不到剩下的；
- 下一次 `ubus_msg_send` 看到 `list_empty(&cl->tx_queue)` 为真，会从 offset 0
  写一条**全新的**消息，直接拼在半条消息后面 → 对端解析出的消息边界彻底错乱；
- 若后续有消息成功入队，`client_cb` 会用残留的 `cl->txq_ofs` 去发它
  （`ubusd_main.c:90`），凭空跳过开头 W 个字节。

`calloc` 失败是 OOM 才有的路径，但队列上限那条不需要 OOM：
`UBUS_CLIENT_MAX_TXQ_LEN == UBUS_MAX_MSGLEN`，检查式是
`txq_len + 8 + len > UBUS_MAX_MSGLEN`，此刻 `txq_len == -W`，即
`T - W > UBUS_MAX_MSGLEN`。对一条最大尺寸的消息（`T = UBUS_MAX_MSGLEN + 8`），
只要 `W < 8` 就会命中。窗口很窄，但它是纯粹的内核缓冲时序问题，不需要任何异常条件。

**修法**：`ubus_msg_send` 里把"已部分写出"的消息强制入队（这条不该受上限约束，
因为它的一部分已经在线上了）；入队真的失败时应该断开该客户端而不是让字节流错位。

---

### 5.7 【低】ubusd：`-s` 选项在编译期目录不可创建时失效，且静默退出

**位置**：`ubusd_main.c:274-300`、`ubusd_main.c:350-352`

```c
static int mkdir_sockdir()
{
	ubus_sock_dir = strdup(UBUS_UNIX_SOCKET);   /* 编译期常量，不是 ubus_socket */
	...
}

	ret = mkdir_sockdir();
	if (ret)
		goto out;                            /* 直接退出，一个字都不打印 */
	unlink(ubus_socket);
	server_fd.fd = usock(..., ubus_socket, NULL);
```

`getopt` 已经把 `-s` 的值放进了 `ubus_socket`，但 `mkdir_sockdir()` 用的是宏
`UBUS_UNIX_SOCKET`（默认 `/var/run/ubus/ubus.sock`）。只要编译期那个目录建不出来，
ubusd 就退出——**哪怕用户明确用 `-s` 指定了一个完全可写的路径**。

**实测**：

```
$ ls -ld /var/run/ubus
ls: cannot access '/var/run/ubus': No such file or directory
$ ubusd -s /tmp/mysock.sock
$ echo $?
255
$ ls -l /tmp/mysock.sock
ls: cannot access '/tmp/mysock.sock': No such file or directory
```

没有任何错误输出，socket 也没建出来。本文的复现环境正是被这一条卡住的，
最后是靠 user namespace + tmpfs 把 `/run/ubus` 造出来才绕过。

影响面：非 root 测试、多实例、容器内运行、CI——全部受影响。这也是为什么
`tests/cram` 只能整套跑在特定布局下。

**修法**：`mkdir_sockdir(const char *sock)` 接受实际路径，失败时
`fprintf(stderr, ...)` 或 `ULOG_ERR`。

---

### 5.8 【低】ubusd_acl：补充组通过 `/proc/<pid>/status` 读取，存在 TOCTOU / PID 复用

**位置**：`ubusd_acl.c:181-224`

```c
static int
ubusd_acl_load_extra_gids(struct ubus_client *cl, pid_t pid)
{
	snprintf(path, sizeof(path), "/proc/%d/status", (int)pid);
	f = fopen(path, "r");
	...
	while (fgets(line, sizeof(line), f)) {
		if (strncmp(line, "Groups:", 7) != 0)
			continue;
		...
	}
```

`uid` / `gid` 来自 `SO_PEERCRED`，那是内核在 `connect()` 那一刻快照的、不可伪造的；
但补充组是事后另开一次 `/proc` 读的。这两者之间存在时间窗：连接方进程可能已经退出，
PID 被回收给另一个进程，读到的就是**别人的**组列表。

组列表直接参与授权判断（`ubusd_acl_match_cred`，`ubusd_acl.c:89-96`），
所以这不是纯理论问题：本地攻击者可以尝试用 fork 风暴去命中一个持有特权 gid 的 PID。
利用难度不低（需要赢得竞态且目标 gid 恰好在 ACL 里），但这是一个真实的授权决策
依赖了不可信、可变输入的例子。

顺带两个类型问题：`cl->extra_gid` 声明为 `int *`，却按 `gid_t`（`unsigned int`）
分配和填充（`ubusd_acl.c:215-218`）。Linux 上两者同宽所以能工作，但这是靠巧合。

**修法**：改用 `SO_PEERGROUPS`（Linux 4.13+），由内核在连接建立时一并提供，
没有竞态。

---

### 5.9 【低】ubusd_acl：ACL 文件里的未知 user/group 被静默当成 uid/gid 0

**位置**：`ubusd_acl.c:453-473`

```c
	if (tb[ACL_USER]) {
		file->user = blobmsg_get_string(tb[ACL_USER]);
		pwd = getpwnam(file->user);
		if (pwd)
			file->uid = pwd->pw_uid;
		else
			file->uid = 0;            /* 用户不存在 → 当成 root */
	} else if (tb[ACL_GROUP]) {
		...
		else
			file->gid = 0;            /* 组不存在 → 当成 root 组 */
	}
```

一个 ACL 文件写了 `"user": "typo_here"`（拼错、或者对应的包还没装），
这条规则不会被拒绝，而是被安装成 uid 0 的规则。

**实际危害有限**，因为 root 本来就在 `ubusd_acl_check` 开头被无条件放行
（`ubusd_acl.c:108`），所以这条规则等于死代码。但后果是**本该生效的授权
静默失效**：管理员以为给 `myservice` 用户开了权限，实际上它一条也没拿到，
而 syslog 里什么都没有。`gid = 0` 那一支略微差一点——补充组里含 gid 0 的
非 root 用户会意外命中。

**修法**：查不到就 `syslog(LOG_ERR, ...)` 并跳过整个文件，而不是退化到 0。

---

## 六、复现环境

本文 5.1 / 5.2 / 5.3 的实测在以下环境完成，脚本和 PoC 都在 `/tmp/poc`：

```
内核      linux 6.9.12
编译器    gcc 14.2.0
libubox   components/libubox @ master（本仓库）
ubus      components/ubus   @ 24864e7（本仓库，未打任何补丁）
json-c    0.15（借用 sysroot 的头文件 + 系统 libjson-c.so.5）
```

构建方式（ubus 的 `UNIT_TESTING=ON` 会额外产出 ASan/UBSan/LSan 版的
`ubusd-san` 和 `ubus-san`）：

```sh
cmake -S components/libubox -B /tmp/ubx -DCMAKE_INSTALL_PREFIX=/tmp/inst \
      -DBUILD_LUA=OFF -Djson=/tmp/jc/lib/libjson-c.so
cmake --build /tmp/ubx && cmake --install /tmp/ubx

cmake -S components/ubus -B /tmp/ubus -DBUILD_LUA=OFF -DUNIT_TESTING=ON \
      -Dubox_library=/tmp/inst/lib/libubox.so \
      -Dblob_library=/tmp/inst/lib/libblobmsg_json.so \
      -Djson=/tmp/jc/lib/libjson-c.so \
      -Dubox_include_dir=/tmp/inst/include
cmake --build /tmp/ubus --target ubusd ubusd-san cli
```

因为 5.7，整个测试必须跑在能创建 `/var/run/ubus` 的环境里：

```sh
unshare -r -m ./run.sh     # 内部 mount -t tmpfs tmpfs /run && mkdir /run/ubus
```

用到的 PoC：

| 文件 | 作用 |
|------|------|
| `closetrace.c` | `LD_PRELOAD` 包住 `close()`，`EBADF` 时打印 `backtrace` |
| `fdserver.c` | handler 里 `ubus_request_set_fd()` → 触发 5.3 |
| `fdclient.c` | `ubus_invoke_fd()` 传 fd → 触发 5.2 |
| `objserver.c` | 注册 600 个对象，让 list 应答撑爆 socket 缓冲 |
| `stallclient.c` | 裸 socket：`SO_RCVBUF=1024` → 发 LOOKUP → 停读 3 秒 → 恢复读 |

5.1 的场景编排：`objserver` 起来后 `stallclient` 发 LOOKUP 并停读，
1 秒后 `kill -9 objserver`，再让 `stallclient` 恢复读取，`ubusd-san` 即刻报
use-after-free 并 abort。
