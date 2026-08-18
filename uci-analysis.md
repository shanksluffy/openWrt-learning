# UCI 代码分析

分析对象：`components/uci`（约 6100 行 C 代码，其中库本体约 2900 行）
分析日期：2026-08-07

> **前提说明**
>
> 本仓库中的 `components/uci` 就是 **upstream 本身**：`origin` 指向官方仓库
> `https://github.com/openwrt/uci.git`，`master` 与 `origin/master` 完全一致，
> 无本地独有提交，工作区干净。最新提交为 `74f6277`（2026-03-12），也就是说
> 这份代码已经 5 个月没有变动了。**报告缺陷前应先 `git fetch`。**
>
> 注意一个容易踩的坑：`package/system/uci/Makefile` 里 pin 的是
> `PKG_SOURCE_VERSION:=66127cd`（2025-12-02），比 `components/uci` 的 HEAD
> 落后两个提交（`ccc1719` errorstr 修复、`74f6277` batch 模式修复）。
> 也就是**实际编译进固件的代码和这里读到的代码不完全一样**。
>
> **本文第五章的 9 条全部实际复现过。** 用系统 gcc 14.2 把
> `libuci.c file.c util.c delta.c parse.c` 编成带 ASan/UBSan 的 `libuci.so`
> 和 CLI（跳过 `blob.c`，它只是 libubox 桥接层，与这些问题无关），
> 另外编了一份无插桩版本给 valgrind 用。文中贴的 ASan 报告、valgrind 输出、
> CLI 输出都是原文粘贴，没有推断成分。
>
> `lua/` 绑定和 fuzz 语料未纳入分析；`ucimap.c` 只做了扫读（见 1.5 说明原因）。

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
| `libuci.so` / `libuci.a` | `libuci.c` `file.c` `util.c` `delta.c` `parse.c` `blob.c` | `libubox` |
| `uci`（CLI） | `cli.c` | `libuci` |
| `libucimap.a` | `ucimap.c` | 无（且无人使用） |
| `lua/uci.so` | `lua/uci.c` | `libuci` + `liblua` |

有一个值得注意的事实：**整个 libuci 里只有 `blob.c` 需要 libubox**，
它是 uci ↔ blobmsg 的转换层（`uci_to_blob()` / `uci_blob_diff()`，netifd 靠它
把配置段变成 blob 再做差分）。我在验证 bug 时把 `blob.c` 摘掉后，
其余五个文件不加任何 libubox 头文件就能编译链接成完整可用的 `libuci.so`。
换句话说，配置解析这个核心功能对 libubox 是零依赖的，
libubox 只是为了给 netifd 提供一个便利的桥接 API 才被拖进来。

编译选项：`-Os -g3 -std=gnu99 -Wall -Werror`，gcc>6 时追加
`-Wextra -Werror=format-security -Werror=format-nonliteral`。
`UNIT_TESTING=ON` 且用 clang 时会额外产出 `uci-san` / `libuci-san.so`
（ASan+UBSan+LSan），cram 测试里的 `test-san_uci_import.t` 就是跑它。

### 1.2 数据模型：四层同构链表

UCI 的整个对象模型建立在一个「所有节点共享同一个头」的设计上：

```386:391:components/uci/uci.h
struct uci_element
{
	struct uci_list list;
	enum uci_type type;
	char *name;
};
```

`uci_package`、`uci_section`、`uci_option`、`uci_delta`、`uci_backend` 全都把
`struct uci_element e` 放在结构体的**第一个字段**，于是：

- 任何节点都能挂进任何 `uci_list`；
- `container_of` 双向转换（`uci_to_section()` 等宏）零开销；
- `uci_lookup_list()` 一个函数就能查 package / section / option 三层。

层级关系是纯粹的树 + 双向循环链表：

```
uci_context.root ──┬── uci_package "network"
                   │      ├── sections ──┬── uci_section "lan"  (type="interface")
                   │      │              │      └── options ──┬── uci_option "proto" (STRING)
                   │      │              │                    └── uci_option "dns"   (LIST)
                   │      │              │                             └── v.list ── uci_element ── ...
                   │      │              └── uci_section "cfg0142ab" (anonymous)
                   │      ├── delta        ← 本次进程内还没落盘的变更
                   │      └── saved_delta  ← 从 delta 文件读回来的历史变更
                   └── uci_package "firewall"
```

`uci_option` 用一个 union 同时表示标量和列表：

```461:470:components/uci/uci.h
struct uci_option
{
	struct uci_element e;
	struct uci_section *section;
	enum uci_option_type type;
	union {
		struct uci_list list;
		char *string;
	} v;
};
```

列表的元素本身也是 `uci_element`，但分配时按 `sizeof(struct uci_option)` 大小申请
（`list.c:612`），这样 `uci_free_option()` 可以不加区分地处理它们——
calloc 清零让 `o->type` 恰好等于 `UCI_TYPE_STRING`(0)、`o->v.string` 恰好等于 NULL，
于是释放分支自动走空。这是个「靠零初始化碰巧成立」的技巧，不是显式设计。

### 1.3 分层与文件职责

| 层 | 文件 | 职责 |
|----|------|------|
| 公共 API / 上下文 | `libuci.c` | `uci_alloc_context` / `uci_load` / `uci_commit` / 错误字符串 / backend 注册 |
| 对象模型 | `list.c` | 增删改查全部核心逻辑：`uci_set` / `uci_delete` / `uci_add_list` / `uci_rename` / `uci_lookup_ptr` |
| 文件后端 | `file.c` | 词法/语法解析、导入导出、原子提交、`list_configs` |
| 变更日志 | `delta.c` | delta 的记录、序列化、回放、过滤（`uci_revert`）、`uci_save` |
| 工具层 | `util.c` | 抛异常版的 malloc/strdup、名字合法性校验、带 flock 的开流 |
| 只读辅助 | `parse.c` | `uci_parse_section()` 批量取 option、MurmurHash2 |
| blobmsg 桥 | `blob.c` | uci section ↔ blob_attr |
| CLI | `cli.c` | 命令分发、batch 模式、`@type[n]` 显示 |
| 遗留 | `ucimap.c` | UCI ↔ C struct 自动映射 |

### 1.4 一个反直觉的事实：`list.c` 不是独立编译单元

```40:41:components/uci/libuci.c
#include "uci_internal.h"
#include "list.c"
```

`list.c` 有 757 行、包含 `uci_set` / `uci_delete` / `uci_lookup_ptr` 这些最核心的
函数，但它**不在 `CMakeLists.txt` 的 `LIB_SOURCES` 里**，而是被 `libuci.c`
文本包含进来。后果：

- 改 `list.c` 一行，构建系统看到的是 `libuci.c` 需要重编（靠 CMake 的头依赖扫描
  才没漏，因为 `.c` 也会被当依赖记录）；
- `list.c` 里所有 `static` 函数（`uci_alloc_section`、`uci_free_option`……）
  的作用域实际上是 `libuci.c` 这个 TU；
- 单测无法单独 link `list.c`。

`file.c` 和 `delta.c` 反而是正常编译单元，通过 `uci_internal.h` 里的
`__private`（`visibility("hidden")`）符号互相调用。所以这个 `#include "list.c"`
是历史遗留，不是某种统一约定。

### 1.5 `ucimap.c`：930 行的死代码

`ucimap.c`（930 行）+ `ucimap.h`（323 行）实现了一套 UCI 段到 C 结构体的
声明式映射框架。它被编译成 `libucimap.a`，并且 `package/system/uci/Makefile`
的 `Build/InstallDev` 还在往 sysroot 里装 `ucimap.h` 和 `libucimap.a`。

但在整个 OpenWrt 树里 grep `ucimap`，除了 uci 自己没有任何使用者。
它也没有任何测试覆盖，不在 `libuci.so` 里（是独立静态库），
CI 的 sanitizer 目标也不包含它。扫读中发现的问题（如
`ucimap_resize_list()` 的 `memset((char *)a->ptr + offset, 0, size - offset)`
起始偏移用的是「增量大小」而非「旧数据末尾」，看起来会清掉已有元素）
因此没有实际影响，本文不展开。**如果要动 uci，这 1250 行应该优先考虑删掉。**

---

## 二、设计原理

### 2.1 setjmp/longjmp 异常模型

UCI 最有特色的设计是用 `setjmp`/`longjmp` 在 C 里模拟异常。所有内部工具函数
（`uci_malloc`、`uci_strdup`、`uci_realloc`）失败时直接抛，不返回错误码：

```35:44:components/uci/util.c
__private void *uci_malloc(struct uci_context *ctx, size_t size)
{
	void *ptr;

	ptr = calloc(1, size);
	if (!ptr)
		UCI_THROW(ctx, UCI_ERR_MEM);

	return ptr;
}
```

每个对外 API 的第一行都是 `UCI_HANDLE_ERR(ctx)`，它负责 `setjmp` 并在捕获时
把错误码 return 出去。这让业务代码可以一路直写，不用层层判 NULL——
代价是**所有中间状态的清理必须靠 `UCI_TRAP_SAVE`/`UCI_TRAP_RESTORE` 手工做**，
而这对宏本身设计得相当危险。

#### 宏展开的真相

```213:225:components/uci/uci_internal.h
#define UCI_TRAP_SAVE(ctx, handler) do {   \
	jmp_buf	__old_trap;		\
	int __val;			\
	memcpy(__old_trap, ctx->trap, sizeof(ctx->trap)); \
	__val = setjmp(ctx->trap);	\
	if (__val) {			\
		ctx->err = __val;	\
		memcpy(ctx->trap, __old_trap, sizeof(ctx->trap)); \
		goto handler;		\
	} while(0)
#define UCI_TRAP_RESTORE(ctx)		\
	memcpy(ctx->trap, __old_trap, sizeof(ctx->trap)); \
} while(0)
```

这是一对**大括号不平衡**的宏：`SAVE` 只开 `do {` 不闭合，`RESTORE` 只闭合不开。
更微妙的是末尾那个 `while(0)`。我用预处理器把它展开出来确认过：

```c
/* 源码写法 */
UCI_TRAP_SAVE(ctx, done);
work();
UCI_TRAP_RESTORE(ctx);

/* 实际展开（gcc -E -P） */
do { jmp_buf __old_trap; int __val;
     memcpy(__old_trap, ctx->trap, sizeof(ctx->trap));
     __val = _setjmp(ctx->trap);
     if (__val) { ctx->err = __val; memcpy(...); goto done; }
     while(0);            /* ←←← 这个 ';' 是调用方写的！ */
 work();
 memcpy(ctx->trap, __old_trap, sizeof(ctx->trap)); } while(0);
```

也就是说：**调用方写在 `UCI_TRAP_SAVE(...)` 后面的那个分号，是 `while(0)` 的
循环体**。它是一个退化的空循环，纯属让语法合法。这带来两个后果：

1. **漏写分号会静默吞掉整个保护块。** 如果写成 `UCI_TRAP_SAVE(ctx, done)`
   （无分号），展开成 `while(0) work();`——`work()` 一次都不执行。
   我实测过：`gcc -Wall -Wextra -O2` 编译**零警告**，程序直接跳过整段逻辑。

2. **保护块内不能用 `break`/`continue`。** 它们会绑定到隐藏的
   `do{...}while(0)`，而不是外层循环。`file.c` 和 `delta.c` 的解析循环里
   `continue` 都写在 `UCI_TRAP_RESTORE` **之后**，这不是巧合而是必须的。

另外还有一个静默陷阱：**从保护块内部直接 `goto handler`（而不是靠 longjmp）
不会恢复 `ctx->trap`。** `file.c:788` 就是这么写的：

```787:789:components/uci/file.c
		/* flush delta */
		if (!uci_load_delta(ctx, p, true))
			goto done;
```

跳到 `done:` 之后，`ctx->trap` 仍然指向本函数内 `UCI_TRAP_SAVE` 的 setjmp 点。
`done:` 结尾恰好有一句 `if (ctx->err) UCI_THROW(ctx, ctx->err);`——如果此时
`ctx->err` 非零就会 longjmp 回本函数中部，然后第二次执行 `done:` 里的
`free(name); free(path);` 造成 double free。目前不成立，因为
`uci_load_delta()` 返回前无条件执行了 `ctx->err = 0`（`delta.c:351`）。
**这是一个只靠远端函数的一行赋值撑住的安全性**，属于结构性隐患而非现存缺陷。

#### `internal` / `nested` 两个旁路

```233:236:components/uci/uci_internal.h
#define UCI_INTERNAL(func, ctx, ...) do { \
	ctx->internal = true;		\
	func(ctx, __VA_ARGS__);		\
} while (0)
```

公共 API 之间互相调用时，`UCI_INTERNAL` 让被调方跳过自己的 `setjmp`
（异常直接穿透到外层 handler）。这个标志还**兼任语义开关**：
`uci_set` / `uci_delete` / `uci_add_list` 里都有
`bool internal = ctx && ctx->internal;`，`internal` 为真时**不记录 delta**。
解析器和 delta 回放正是靠这个避免「读配置文件也被记成一次变更」。

一个 flag 同时表达「异常穿透」和「不记账」两件事，是这套代码里最容易读错的地方。

### 2.2 内嵌 payload 的分配策略

`uci_alloc_generic(ctx, type, name, size)` 一次 calloc 出
`sizeof(struct uci_xxx) + payload`，payload 通过 `uci_dataptr(ptr)` 访问
（就是 `ptr + sizeof(*ptr)`）。于是一个 option 只需要 **2 次 malloc**
（结构体+值 一次，`e->name` 一次），section 同理（type 内嵌）。

释放时靠指针比较判断值是不是内嵌的：

```94:99:components/uci/list.c
	switch(o->type) {
	case UCI_TYPE_STRING:
		if ((o->v.string != uci_dataptr(o)) &&
			(o->v.string != NULL))
			free(o->v.string);
		break;
```

这个设计对路由器的内存碎片很友好，代价是**值不能原地扩容**。
所以 `uci_set` 更新 option 时有两条路径：

```715:723:components/uci/list.c
		if (ptr->o->type == UCI_TYPE_STRING && strlen(ptr->o->v.string) == strlen(ptr->value)) {
			strcpy(ptr->o->v.string, ptr->value);
		} else {
			struct uci_option *old = ptr->o;
			ptr->o = uci_alloc_option(ptr->s, ptr->option, ptr->value, &old->e.list);
			if (ptr->option == old->e.name)
				ptr->option = ptr->o->e.name;
			uci_free_option(old);
		}
```

**「长度相同就原地改，否则重建节点」**——这个分支正是 5.4 那个 bug 的根源：
重建 section 时丢掉了 `anonymous` 标志，于是同一个操作的可见行为取决于
新旧字符串长度是否相等。

### 2.3 delta：UCI 的「事务日志」

这是 UCI 区别于「直接改文件」的核心机制。

**写路径**：`uci set` 不碰 `/etc/config`，只往内存里的 `p->delta` 追加一条记录；
`uci_save()` 把这些记录**追加**到 `/tmp/.uci/<package>`；
`uci commit` 才真正重写 `/etc/config/<package>` 并清空 delta 文件。

**读路径**：`uci_file_load()` 先解析 `/etc/config/<package>` 得到基线，
再调 `uci_load_delta()` 遍历 `ctx->delta_path` 里每个目录下的同名文件，
**逐行回放**到内存对象上。所以 `uci show` 看到的永远是「基线 + 未提交变更」。

delta 文件是一行一条的文本，格式为
`[<cmd-char>]<pkg>.<section>[.<option>]='<value>'`，命令字符：

```149:157:components/uci/delta.c
char const uci_command_char[] = {
	[UCI_CMD_ADD] = '+',
	[UCI_CMD_REMOVE] = '-',
	[UCI_CMD_CHANGE] = 0,
	[UCI_CMD_RENAME] = '@',
	[UCI_CMD_REORDER] = '^',
	[UCI_CMD_LIST_ADD] = '|',
	[UCI_CMD_LIST_DEL] = '~'
};
```

回放时 `uci_parse_delta_line()` 把每条命令翻译成对应的
`UCI_INTERNAL(uci_set / uci_delete / uci_add_list / ...)` 调用——
**注意是 `UCI_INTERNAL`，所以回放本身不会再生成新的 delta**，
同时它会往 `p->saved_delta` 里存一份副本供 `uci changes` 展示。

`uci revert` 走的是完全不同的路径（`delta.c:420`）：
先 flush、记住名字、**释放整个 package**、用
`uci_filter_delta()` 逐行重写 delta 文件（丢掉匹配的行）、再重新 load。
「回滚」实际上是「重放一个删掉了某些行的日志」。

这个设计的关键取舍是：**delta 是命令日志而不是状态快照**，所以
`p->delta` 里的记录是顺序相关的。`uci delete <list>=<index>` 记的是**下标**，
回放时下标含义会随前面的命令变化——这就是 5.3 那个 bug 能把
`-net.lan.dns=' '` 这种垃圾持久化进日志的原因。

### 2.4 匿名段的命名哈希

`config interface`（无名字）在内存里必须有个名字才能被引用。
`uci_fixup_section()` 生成它：

```162:176:components/uci/list.c
	hash = djbhash(hash, s->type);
	uci_foreach_element(&s->options, e) {
		struct uci_option *o;
		hash = djbhash(hash, e->name);
		o = uci_to_option(e);
		switch(o->type) {
		case UCI_TYPE_STRING:
			hash = djbhash(hash, o->v.string);
			break;
		default:
			break;
		}
	}
	sprintf(buf, "cfg%02x%04x", s->package->n_section, hash % (1 << 16));
	s->e.name = uci_strdup(ctx, buf);
```

名字 = `cfg` + 段序号(hex) + 「类型和所有 option 名值对」的 djb2 哈希低 16 位。
上面那段注释说明了意图：**这是一种乐观并发控制**。如果 A 进程拿到
`cfg016d96` 并写进 delta，而 B 进程在这期间改了这个段的内容，
A 的 delta 回放时就找不到这个名字，变更被拒绝而不是错误地打到别的段上。

顺便一提，`char buf[16]` 对这个 `sprintf` 是**恰好够用、零余量**的：
`n_section` 是 `int`，`%02x` 最多输出 8 位十六进制，
`3 + 8 + 4 + 1 = 16`。不越界，但没有任何裕度。

### 2.5 三层配置目录

| 目录 | 常量 | 作用 |
|------|------|------|
| `/var/run/uci` | `UCI_CONF2DIR` | **override 目录**，优先于 `/etc/config` |
| `/etc/config` | `UCI_CONFDIR` | 持久配置 |
| `/tmp/.uci` | `UCI_SAVEDIR` | 未提交变更（delta） |

`uci_file_load()` 先 stat conf2dir 下的同名文件，存在就用它并置
`p->uses_conf2 = true`（commit 时写回 conf2dir）：

```932:942:components/uci/file.c
	default:
		/* config in /etc/config */
		conf2 = true;
		filename = uci_config_path(ctx, name, conf2);
		if (!filename || stat(filename, &st) != 0) {
			conf2 = false;
			free(filename);
			filename = uci_config_path(ctx, name, conf2);
		}
		confdir = true;
		break;
```

delta 则是一条**有序搜索路径** `ctx->delta_path`，而且有个不变式：
**savedir 永远是链表最后一个元素**。`uci_add_delta_path()` 插在倒数第二位，
`uci_set_savedir()` 把目标移到末尾。`uci_load_delta()` 按顺序回放每一层。

这正是 `/var/state` 运行时状态机制的基础。`uci_toggle_state`（见 3.3）用
`uci -P /var/state`：savedir 变成 `/var/state`，而默认的 `/tmp/.uci` 仍留在
delta_path 里排在前面。于是**读的时候是 `/etc/config` → `/tmp/.uci` → `/var/state`
三层叠加，写的时候只写 `/var/state`**，并且 CLI 会自动带上 `CLI_FLAG_NOCOMMIT`
禁止 commit——运行时状态永远不会污染持久配置。

还有一个细节：`p->has_delta` 只有从 confdir/conf2dir 加载时才为 true。
用绝对路径 `uci_load(ctx, "/tmp/foo", &p)` 加载的包**完全不走 delta 机制**，
`uci_save()` 会直接退化成 `uci_commit()` 原地改文件（`delta.c:480`）。
这条「旁路」正是 5.1 那个 UAF 的必要条件。

### 2.6 文件锁与原子提交

`uci_open_stream()` 对每个文件加 `flock`：读用 `LOCK_SH`，写用 `LOCK_EX`，
并且容忍 `ENOSYS`（某些 fs 不支持）。

`uci_file_commit()` 的顺序很讲究：

```758:777:components/uci/file.c
	/* open the config file for writing now, so that it is locked */
	f1 = uci_open_stream(ctx, p->path, NULL, SEEK_SET, true, true);

	/* flush unsaved changes and reload from delta file */
	UCI_TRAP_SAVE(ctx, done);
	if (p->has_delta) {
		if (!overwrite) {
			name = uci_strdup(ctx, p->e.name);
			path = uci_strdup(ctx, p->path);
			/* dump our own changes to the delta file */
			if (!uci_list_empty(&p->delta))
				UCI_INTERNAL(uci_save, ctx, p);

			/*
			 * other processes might have modified the config
			 * as well. dump and reload
			 */
			uci_free_package(&p);
			uci_cleanup(ctx);
			UCI_INTERNAL(uci_import, ctx, f1, name, &p, true);
```

1. **先排他锁住目标文件**，锁住之后才做后面的事；
2. 把自己的内存变更 flush 进 delta 文件；
3. **丢弃整个内存 package，从锁住的 fd 重新解析**——因为别的进程可能在这期间
   改过 `/etc/config`；
4. 回放合并后的全部 delta（`uci_load_delta(ctx, p, true)`，`flush=true` 会
   `ftruncate` 掉 delta 文件）；
5. 导出到 `mkstemp` 建的临时文件，`fflush` + `fsync`；
6. `chmod` 成原文件权限后 `rename()` 原子替换。

这是一套设计得相当完整的「读-改-写」并发方案。第 3 步那个「先释放再重载」
就是 5.2 那个 UAF 的现场：`uci_free_package(&p)` 之后如果 `uci_import` 抛异常，
调用方持有的 `*package` 就成了悬垂指针。

### 2.7 解析器

手写的字符级递归下降解析器，全部在 `file.c` 前 550 行。核心是一个
`struct uci_parse_context`，用 `pos` 游标在一个可增长的行缓冲 `buf` 上移动。

最巧妙也最容易出错的地方是：**它不申请输出缓冲，而是在输入缓冲上原地压缩**。

```116:123:components/uci/file.c
static inline void addc(struct uci_context *ctx, size_t *pos_dest, size_t *pos_src)
{
	struct uci_parse_context *pctx = ctx->pctx;

	pctx_char(pctx, *pos_dest) = pctx_char(pctx, *pos_src);
	*pos_dest += 1;
	*pos_src += 1;
}
```

`*pos_dest` 永远 ≤ `*pos_src`（去掉引号只会让结果更短），所以可以安全地往回写。
解析完一个 token 后用 `pctx_char(pctx, *target) = 0` 就地打 NUL 终止符。
零拷贝、零额外分配，但也意味着**写 NUL 的位置有可能踩到还没解析的字符**——
5.9 那个「分号被吃掉」的 bug 就是这么来的。

支持的语法（`uci_parse_line`）比文档里写的多：

- `package <name>` / `config <type> [<name>]` / `option <k> <v>` / `list <k> <v>`
- **单字母缩写**：`p` / `c` / `o` / `l` 完全等价于全拼
  （`word[1] == 0` 分支，`file.c:525-543`）。我实测 `p net` + `c interface lan`
  + `o proto static` 可以正常导入。
- `#` 行内注释、`;` 命令分隔符、`\` 行尾续行
- 单引号/双引号，且**引号内可以跨行**（`parse_double_quote` 的 `case 0:`
  会调 `uci_getln` 继续读下一行）

值的合法性检查分三档：名字（`uci_validate_name`，只允许 `[A-Za-z0-9_]`）、
类型（`uci_validate_type`，允许可打印 ASCII 33~126）、包名
（额外允许 `-`）、值（`uci_validate_text`，只拒绝 `\t\n\r` 之外的控制字符）。

---

## 三、使用方法

### 3.1 CLI

```bash
# 读
uci show network                    # 全部，带 @type[n] 扩展语法
uci show network.lan
uci get  network.lan.proto          # 只输出值，不带引号
uci get  network.@interface[-1].proto   # 最后一个 interface 段

# 写（只进 /tmp/.uci，不动 /etc/config）
uci set      network.lan.proto=static
uci add_list network.lan.dns=1.1.1.1
uci del_list network.lan.dns=1.1.1.1
uci delete   network.lan.dns        # 删整个 option
uci delete   network.lan.dns=0      # 删列表第 0 项
uci rename   network.cfg0142ab=lan  # 给匿名段命名
uci reorder  network.lan=0          # 移到最前
uci add      network interface      # 加匿名段，stdout 打印生成的名字

# 事务
uci changes [network]               # 看未提交变更
uci revert  network.lan.proto       # 撤销
uci commit  network                 # 落盘

# 批量 / 导入导出
uci export network                  # 规范化文本
uci import network < file           # 注意：会 overwrite 提交，直接覆盖 /etc/config
uci batch <<EOF
set network.lan.proto=static
set network.lan.ipaddr=10.0.0.1
commit network
EOF
```

几个容易忽略的开关：

| 选项 | 含义 | 注意 |
|------|------|------|
| `-c <dir>` | 换 confdir | 测试必备 |
| `-C <dir>` | 换 override 目录（conf2dir） | 见 5.6，会泄漏内存 |
| `-t <dir>` | 换 savedir（delta 落盘位置） | |
| `-p <dir>` | **追加**一个只读 delta 搜索路径 | 叠加层，永不提交 |
| `-P <dir>` | 同 `-p` 且设为 savedir | **自动打开 NOCOMMIT** |
| `-q` | 静默 | 脚本里配合 `$?` 用 |
| `-S` | 关闭 strict，容忍解析错误 | **触发 5.1 的必要条件** |
| `-X` | 不用 `@type[n]` 扩展语法显示 | |
| `-n` / `-N` | 导出时是否给匿名段写出名字 | `-n` 是默认 |

`uci import` 是最危险的一个：它读 stdin 然后对**所有**解析出的 package 执行
`uci_commit(overwrite=true)`，直接覆盖 `/etc/config`，不经过 delta。

### 3.2 C API

典型的读流程：

```c
struct uci_context *ctx = uci_alloc_context();
struct uci_package *p = NULL;

if (uci_load(ctx, "network", &p) != UCI_OK) {
    uci_perror(ctx, "load");
    uci_free_context(ctx);
    return -1;
}

struct uci_element *e;
uci_foreach_element(&p->sections, e) {
    struct uci_section *s = uci_to_section(e);
    if (strcmp(s->type, "interface") != 0)
        continue;
    const char *proto = uci_lookup_option_string(ctx, s, "proto");
    printf("%s: proto=%s\n", s->e.name, proto ? proto : "(unset)");
}
uci_free_context(ctx);   /* 会连带释放所有 package */
```

写流程用 `uci_ptr`。**注意 `uci_lookup_ptr()` 会就地修改传入的字符串
并让 `ptr` 里的各个 `const char *` 指向它**，所以那块内存必须活得比 `ptr` 长，
且不能是字符串字面量：

```c
struct uci_ptr ptr;
char tuple[] = "network.lan.proto=static";   /* 数组，不是 char* */

uci_lookup_ptr(ctx, &ptr, tuple, true);
uci_set(ctx, &ptr);
uci_save(ctx, ptr.p);        /* 写 /tmp/.uci */
uci_commit(ctx, &ptr.p, false);  /* 落盘，注意传的是 &ptr.p，指针会被换掉 */
```

两个必须记住的返回值语义：

- `uci_lookup_ptr()` **只有 package 找不到才返回 `UCI_ERR_NOTFOUND`**。
  section/option 找不到照样返回 `UCI_OK`，得自己查
  `ptr.flags & UCI_LOOKUP_COMPLETE`。头文件里对此有明确注释（`uci.h:160-165`）。
- `uci_commit()` 可能**换掉** `*package`（内部会先 free 再重新 load）。
  而且失败时不会把它置 NULL —— 见 5.2。

批量取 option 用 `parse.c` 提供的辅助函数，比逐个 `uci_lookup_option` 快：

```c
enum { OPT_PROTO, OPT_IPADDR, OPT_DNS, __OPT_MAX };
static const struct uci_parse_option opts[__OPT_MAX] = {
	[OPT_PROTO]  = { .name = "proto",  .type = UCI_TYPE_STRING },
	[OPT_IPADDR] = { .name = "ipaddr", .type = UCI_TYPE_STRING },
	[OPT_DNS]    = { .name = "dns",    .type = UCI_TYPE_LIST   },
};
struct uci_option *tb[__OPT_MAX];

uci_parse_section(s, opts, __OPT_MAX, tb);
if (tb[OPT_PROTO])
	printf("%s\n", tb[OPT_PROTO]->v.string);

/* 还能算个哈希，用来判断这个段有没有变过 */
uint32_t h = uci_hash_options(tb, __OPT_MAX);
```

线程安全性：**没有**。context 之间独立，但 `uci_get_errorstr()` 用了
一个函数内 `static char error_info[128]`（`libuci.c:160`），多线程同时报错会互相
踩。文档里也没写这一点。

### 3.3 shell 绑定

有两层，容易混淆：

**第一层** `components/uci/sh/uci.sh`：只定义 `config` / `option` / `list` /
`config_get` / `config_foreach` 这些**回调和访问器**，不调用 `/sbin/uci`。

**第二层** `package/system/uci/files/lib/config/uci.sh`：`uci_load` /
`uci_set` / `uci_get` / `uci_commit` 等，是 `/sbin/uci` 的 shell 包装。

两层的接合点是这一句：

```39:41:package/system/uci/files/lib/config/uci.sh
	DATA="$(/sbin/uci ${UCI_CONFIG_DIR:+-c $UCI_CONFIG_DIR} ${LOAD_STATE:+-P /var/state} -S -n export "$PACKAGE" 2>/dev/null)"
	RET="$?"
	[ "$RET" != 0 -o -z "$DATA" ] || eval "$DATA"
```

**`uci export` 的输出被直接 `eval`**，而 `config` / `option` / `list` 是第一层
定义的 shell 函数，于是配置文件就变成了一段可执行脚本，加载后所有值都躺在
`CONFIG_<section>_<option>` 环境变量里。这是 OpenWrt 所有 init 脚本
`config_load network` 的实现原理。

这个设计的安全性**完全建立在 `uci_escape()` 上**（`file.c:563`）：
它把值里的 `'` 转义成 `'\''`，其余内容原样放进单引号。
值本身只被 `uci_validate_text()` 检查过（只拒控制字符），所以转义是唯一防线。
见 5.9 关于 OOM 时该防线会失效的说明。

`uci_toggle_state` 那组函数演示了 2.5 的运行时状态叠加：

```73:76:package/system/uci/files/lib/config/uci.sh
uci_toggle_state() {
	uci_revert_state "$1" "$2" "$3"
	uci_set_state "$1" "$2" "$3" "$4"
}
```

写的是 `/var/state`，`uci_load` 加载时用 `-P /var/state` 把它叠在
`/etc/config` 之上，重启后 `/var`（tmpfs）自动清空。

### 3.4 blob 桥接（netifd 用法）

```c
static const struct blobmsg_policy iface_attrs[] = {
	[IFACE_PROTO]  = { .name = "proto",  .type = BLOBMSG_TYPE_STRING },
	[IFACE_METRIC] = { .name = "metric", .type = BLOBMSG_TYPE_INT32  },
	[IFACE_DNS]    = { .name = "dns",    .type = BLOBMSG_TYPE_ARRAY  },
};
static const struct uci_blob_param_list iface_attr_list = {
	.n_params = ARRAY_SIZE(iface_attrs),
	.params = iface_attrs,
};

struct blob_buf b = {};
blob_buf_init(&b, 0);
uci_to_blob(&b, s, &iface_attr_list);
```

`uci_to_blob()` 按 policy 做类型转换（字符串 → u32/u64/bool），
`BLOBMSG_TYPE_ARRAY` 的 option 如果在 uci 里是 `option`（不是 `list`），
会**按空格/制表符切分**成数组（`blob.c:81`）——这就是为什么
`option dns '1.1.1.1 8.8.8.8'` 和两条 `list dns` 在 netifd 眼里等价。

`uci_blob_diff()` 配合 `uci_bitfield_set()` 输出一个变更位图，
netifd 用它判断「配置变了但不需要重启接口」这类增量场景。

---

## 四、可以优化的点

### 4.1 所有查找都是线性扫描，加载是 O(n²)（有实测数据）

`uci_lookup_list()` 是整个库唯一的查找原语，实现就是遍历链表：

```286:296:components/uci/list.c
__private struct uci_element *
uci_lookup_list(struct uci_list *list, const char *name)
{
	struct uci_element *e;

	uci_foreach_element(list, e) {
		if (!strcmp(e->name, name))
			return e;
	}
	return NULL;
}
```

问题在于**解析器每读到一个具名段就要调一次它**（`file.c:449`，为了检测重名），
每读到一个 option 也要调一次（`file.c:492`）。于是「加载一个配置文件」这个
最基础的操作是 O(n²)。

我用生成的配置文件实测了一下（`uci export`，x86-64，纯解析+输出）：

| 段数 | 具名段 | 匿名段（不查名字） |
|------|--------|--------------------|
| 2000 | 0.01s | 0.00s |
| 4000 | 0.03s | 0.00s |
| 8000 | 0.20s | 0.01s |
| 16000 | 0.83s | 0.01s |
| 32000 | **4.38s** | 0.03s |

具名段每翻一倍耗时约 ×4（教科书式的 O(n²)），匿名段是线性的。
32000 段时两者差 **146 倍**。单个段内的 option 数量同样是 O(n²)：

| 单段 option 数 | 耗时 |
|------|------|
| 5000 | 0.09s |
| 10000 | 0.37s |
| 20000 | 1.58s |

这不是纯理论问题。带大量静态租约的 `/etc/config/dhcp`、
带几千条规则的 `/etc/config/firewall` 都能到几千段的量级，
而路由器的 MIPS/ARM 核比这里的 x86 慢一个数量级以上。

**改法**：libuci 已经链接 libubox 了（为了 `blob.c`），
`libubox/avl.h` 就在手边。给 `uci_package` 加一棵 section 的 AVL 树、
给 `uci_section` 加一棵 option 的 AVL 树，保留现有链表维持顺序语义
（UCI 的段是有序的，导出和 `reorder` 依赖顺序），查找走树。
这正是 ubusd 对它的三类对象所做的事。改动可以完全封在
`uci_lookup_list()` / `uci_alloc_section()` / `uci_free_section()` 内部，
不影响 ABI。

### 4.2 每次 CLI 调用都全量重解析 + 全量回放

`uci set a.b.c=1` 这一条命令要做：解析整个 `/etc/config/a` → 回放
`/tmp/.uci/a` 里已有的**全部** delta → 追加一行 → 退出。
所以一个 shell 脚本里连续 20 条 `uci set`，总代价是
O(20 × 配置大小) + O(20²/2 × 单条 delta 代价)。

实测：在一个 4000 段的配置上跑 20 次 `uci set`，总共 0.74s（每次约 37ms）。
init 脚本里几十上百条 `uci set` 是常态。

**改法**：CLI 已经有 `batch` 模式（一次进程、多条命令），但
`uci_batch()` 在每条命令后都把所有 package 卸载了：

```608:611:components/uci/cli.c
		/* clean up */
		uci_foreach_element_safe(&ctx->root, tmp, e) {
			uci_unload(ctx, uci_to_package(e));
		}
```

这一句让 batch 模式退化成「省了 fork，没省解析」。把它改成只在
`commit`/`revert` 后卸载，batch 模式就能真正线性化。
更进一步，`package/base-files` 里那些 `uci_set` shell 包装应该
支持攒批后一次性喂给 `uci batch`。

### 4.3 异常宏应该重写

2.1 已经详述。具体建议：

1. 把 `UCI_TRAP_SAVE`/`UCI_TRAP_RESTORE` 改成大括号平衡的形式，
   例如 `UCI_TRAP_BEGIN(ctx, handler) { ... } UCI_TRAP_END(ctx)`，
   干掉那个靠调用方分号撑着的 `while(0)`；
2. 加一个编译期检查（或者至少在 `uci_internal.h` 顶部写清楚
   「后面的分号是语法必需的」）；
3. 保护块内的 `goto handler` 统一改成先 `UCI_TRAP_RESTORE` 再跳，
   消掉 2.1 末尾那个隐患；
4. `ctx->internal` 承担的两个语义拆成两个字段
   （`internal`=异常穿透，`no_delta`=不记账）。

### 4.4 其它

- **`uci_get_errorstr()` 的 `static char error_info[128]`**（`libuci.c:160`）
  应该改成栈上缓冲。目前它让整个错误报告路径非线程安全、非可重入。
- **`uci_escape()` 的 `ctx->buf` 只增不减**。导出过一个 1MB 的值之后，
  这块缓冲会一直挂在 context 上直到 `uci_cleanup()`。
  同理 `pctx->buf`（行缓冲）会涨到最长逻辑行的 2 倍（按 2 的幂增长）。
  我实测导入一个含 64MiB 单值的配置，峰值 RSS 135MB。
  对 64MB 内存的路由器，这意味着一个畸形（但合法）的配置文件就能 OOM。
  建议在 `uci_import` 结束时把缓冲缩回初始大小。
- **`uci_export_package()` 每个 option 发两次 `fprintf`**，因为
  option 名和值都要用同一个 `ctx->buf` 做转义。用两个独立缓冲就能合并成一次
  格式化，减少 syscall（导出大配置时可观）。
- **`uci_list_config_files()` 的 `uci_malloc(ctx, size - skipped)`**
  （`file.c:887`）量纲是错的：`size` 里每个被跳过的条目占了
  `sizeof(char *)` 字节，却只减掉 1 字节。结果是**多分配**（安全），
  但表达的意图和实际行为不一致，应该写成 `size - skipped * sizeof(char *)`。
- **`get_filename()` 里 `p = strrchr(path, '/'); p++;`**（`file.c:846`）
  没判 NULL。目前所有调用点的路径都必然含 `/`（来自 `glob("<confdir>/*")`），
  但这是靠调用点保证的，不是函数自己保证的。
- **`uci_open_stream()` 里 `basename((char *) origfilename)`**（`util.c:198`）
  把 const 强转掉了。POSIX 版 `basename()` 允许修改入参，**musl 的实现确实会写**
  （去掉末尾斜杠），而 OpenWrt 用的就是 musl。当前所有调用点
  `origfilename` 都传 NULL（这个分支是死代码），实际写入的是
  `filename`（堆上的 `p->path`），所以不会崩。但这是个应该清理掉的 const 违规。
- **`ucimap.c` + `ucimap.h` 共 1250 行是死代码**（见 1.5），建议删除，
  同时从 `Build/InstallDev` 里摘掉 `libucimap.a` 和 `ucimap.h`。

---

## 五、已核验的潜在 bug

以下 9 条**全部实际复现**。5.1 / 5.2 / 5.6 是 ASan 报告原文，
5.3 是 valgrind 输出原文，其余是 CLI 实际行为。

### 5.1 `uci_file_load()`：package 已经接管的路径又被 free 一次（use-after-free + double free）

**位置**：`components/uci/file.c:945-964`

```945:964:components/uci/file.c
	UCI_TRAP_SAVE(ctx, done);
	file = uci_open_stream(ctx, filename, NULL, SEEK_SET, false, false);
	ctx->err = 0;
	UCI_INTERNAL(uci_import, ctx, file, name, &package, true);
	UCI_TRAP_RESTORE(ctx);

	if (package) {
		package->uses_conf2 = conf2;
		package->path = filename;      /* ← package 接管了 filename 的所有权 */
		package->has_delta = confdir;
		uci_load_delta(ctx, package, false);
	}

done:
	uci_close_stream(file);
	if (ctx->err) {
		free(filename);                /* ← 但这里又释放了一次 */
		UCI_THROW(ctx, ctx->err);
	}
	return package;
```

`done:` 是**正常流程也会落到的位置**（不只是异常跳转），所以只要执行到这里时
`ctx->err` 非零，`package->path` 就变成悬垂指针，而 package 已经挂在
`ctx->root` 上了。

**为什么 `ctx->err` 会非零**：非 strict 模式下 `uci_import()` 容忍单行解析错误，
但**不清 `ctx->err`**：

```697:703:components/uci/file.c
error:
		if (ctx->flags & UCI_FLAG_PERROR)
			uci_perror(ctx, NULL);
		if ((ctx->err != UCI_ERR_PARSE) ||
			(ctx->flags & UCI_FLAG_STRICT))
			UCI_THROW(ctx, ctx->err);
```

它 return 0 但把 `ctx->err = UCI_ERR_PARSE` 留在那儿。

正常情况下这被 `uci_load_delta()` 结尾的 `ctx->err = 0`（`delta.c:351`）兜住了。
但 `uci_load_delta()` 第一行就是 `if (!p->has_delta) return 0;`——
**从绝对路径/相对路径加载的包 `has_delta == false`，直接返回，兜不住。**

**触发条件**：`UCI_FLAG_STRICT` 关闭 + 用路径（而非包名）加载 + 文件有语法错误。
CLI 有两条现成的路径：`uci -S -m import <path>` 和 `uci -S add <path> <type>`。

**复现**（ASan）：

```
$ printf "config interface 'lan'\n\toption proto 'static'\nbogus_command here\n" > /tmp/badcfg
$ echo "" | ./uci-gsan -S -m import /tmp/badcfg

Parse error (invalid command) at line 3, byte 0
==1881935==ERROR: AddressSanitizer: heap-use-after-free on address 0x503000000070
READ of size 2 at 0x503000000070 thread T0
    #3 uci_open_stream  src/util.c:202
    #4 uci_file_commit  src/file.c:759
    #5 uci_commit       src/libuci.c:212
    #6 uci_do_import    src/cli.c:401
freed by thread T0 here:
    #1 uci_file_load    src/file.c:961
    #2 uci_load         src/libuci.c:222
    #3 uci_do_import    src/cli.c:387
previously allocated by thread T0 here:
    #1 uci_strdup       src/util.c:59
```

无插桩版本（模拟真实固件）：

```
$ echo "" | ./uci-plain -S -m import /tmp/badcfg
Parse error (invalid command) at line 3, byte 0
./uci-plain: I/O error
free(): double free detected in tcache 2
Aborted            (rc=134)
```

`uci -S add` 路径同样触发：

```
==1882126==ERROR: AddressSanitizer: attempting double-free on 0x503000000070
    #1 uci_file_load  src/file.c:961
    #3 uci_do_add     src/cli.c:450
```

**影响范围**：确定的部分是两件事——`uci_file_commit()` 会拿这块已释放的内存
当路径去 `open()`（ASan 抓到的就是这次读），以及 `uci_free_context()` 时的
double free（无插桩版直接 abort）。至于「被释放和被读取之间那块堆内存会不会
被复用成攻击者可控的内容」，我**没有验证**，只是指出中间确实存在若干次分配、
存在这个可能性。触发需要 `-S`，而 `-S` 在树内基本只出现在
`uci_load()` shell 包装里，那里传的是包名不是路径——所以这**不是一条
直接可从网络利用的路径，但对任何用文件路径加载配置的程序是实打实的内存破坏**。

树内使用者检查：`components/netifd/config.c:590` 确实执行了
`ctx->flags &= ~UCI_FLAG_STRICT;`，但它随后用 `uci_set_confdir()` + 包名加载
（`has_delta == true`），**不受影响**。

**修法**：`package->path = filename;` 之后把局部变量置空，
或者在 `done:` 里改成 `if (ctx->err) { if (!package) free(filename); ... }`。
更彻底的修法是让 `uci_import()` 在容忍错误后清掉 `ctx->err`。

### 5.2 `uci_file_commit()` 失败时让调用方的 `*package` 悬空（use-after-free）

**位置**：`components/uci/file.c:774-781`

```774:781:components/uci/file.c
			uci_free_package(&p);
			uci_cleanup(ctx);
			UCI_INTERNAL(uci_import, ctx, f1, name, &p, true);

			p->path = path;
			p->has_delta = true;
			*package = p;
```

`uci_free_package(&p)` 把局部 `p` 置 NULL 了，但**调用方的 `*package` 还指向
那块已释放的内存**。只有 `uci_import()` 成功返回后第 780 行才会更新它。
如果 `uci_import()` 抛异常（longjmp 到 `done:`），`*package` 保持旧值不变——
一个悬垂指针。

而 `cli.c` 的 `package_cmd()` 恰好在 commit 失败后就用它：

```364:367:components/uci/cli.c
out:
	if (ptr.p)
		uci_unload(ctx, ptr.p);
	return ret;
```

**触发条件**：package 已加载 + 有未提交变更 + commit 时磁盘上的
`/etc/config/<pkg>` 已经变得无法解析。对单次 CLI 调用来说 load 和 commit
只隔几微秒，很难撞上；但对**长驻的库使用者**（netifd、rpcd 会把 package
在内存里持有很久）这就是一个正常的并发窗口。

我写了个 PoC 模拟长驻进程（`poc_commit.c`：load → set → save →
第三方改坏配置文件 → commit → 按 `cli.c` 的写法 unload）：

```
==1883454==ERROR: AddressSanitizer: heap-use-after-free on address 0x50c000000080
READ of size 8 at 0x50c000000080 thread T0
    #0 uci_free_package  src/list.c:256
    #1 uci_unload        src/list.c:754
    #2 main              poc_commit.c:55
freed by thread T0 here:
    #1 uci_free_element  src/list.c:69
    #2 uci_free_package  src/list.c:266
    #3 uci_file_commit   src/file.c:775
    #4 uci_commit        src/libuci.c:212
previously allocated by thread T0 here:
    #4 uci_switch_config src/file.c:385
    #8 uci_file_load     src/file.c:948
```

**修法**：`uci_free_package(&p)` 之后立刻 `*package = NULL;`。
一行的事，而且这也让 `uci.h` 里 "the supplied pointer is updated accordingly"
的承诺在失败路径上也成立。

### 5.3 `uci_delete()` 读取未初始化的 `index`（sscanf 返回 EOF 没被判掉）

**位置**：`components/uci/list.c:565-580`

```565:570:components/uci/list.c
	if (ptr->o && ptr->o->type == UCI_TYPE_LIST && ptr->value && *ptr->value) {
		if (!sscanf(ptr->value, "%d", &index))
			return 1;

		uci_foreach_element_safe(&ptr->o->v.list, tmp, e2) {
			if (index == 0) {
```

`sscanf` 有三种返回值：成功转换数返回 ≥1，格式不匹配返回 0，
**在任何转换发生前遇到输入结束返回 `EOF`（-1）**。
`!sscanf(...)` 只挡住了 0，挡不住 -1。

对 `%d` 而言，前导空白会先被跳过，然后如果字符串结束了就是「输入失败」→ 返回 EOF。
所以 `ptr->value = " "`（单个空格）就能走进去，`index` 从未被赋值。
而空格是合法的 uci 值（`uci_validate_text()` 只拒绝 `\t\n\r` 以外的控制字符）。

`cli.c:530` 那道前置检查犯的是**完全一样的错误**，所以拦不住：

```529:531:components/uci/cli.c
	case CMD_DEL:
		if (ptr.value && !sscanf(ptr.value, "%d", &dummy))
			return 1;
```

**复现**（valgrind，配置为 `list dns` 三条：1.1.1.1 / 8.8.8.8 / 9.9.9.9）：

```
$ valgrind -q ./uci-plain -c ./etc/config -t ./tmpsave delete 'net.lan.dns= '
==1881778== Conditional jump or move depends on uninitialised value(s)
==1881778==    at 0x485EE89: uci_delete (list.c:570)
==1881778==    by 0x4032ED: uci_do_section_cmd (cli.c:532)
==1881778==    by 0x403843: uci_cmd (cli.c:668)
==1881778==    by 0x403BBC: main (cli.c:771)

$ cat tmpsave/net
-net.lan.dns=' '
$ ./uci-plain ... show net.lan.dns
net.lan.dns='8.8.8.8' '9.9.9.9'
```

这次栈上的垃圾恰好是 0，所以删掉了第 0 项。换个调用栈就会删掉别的项或者不删——
**行为不确定，而且被删的是哪条 DNS/防火墙规则取决于未初始化内存**。

更麻烦的是第二行：那条垃圾 delta `-net.lan.dns=' '` **被持久化进了变更日志**，
之后每次加载都会重新回放一遍这个未初始化读。

**修法**：`if (sscanf(ptr->value, "%d", &index) != 1) return UCI_ERR_INVAL;`
（`cli.c:530` 同改）。顺带修掉 5.8。

### 5.4 改匿名段的类型会让它静默变成具名段（取决于新旧类型长度是否相等）

**位置**：`components/uci/list.c:724-738`

```724:738:components/uci/list.c
	} else if (ptr->s && ptr->section) { /* update section */
		if (!strcmp(ptr->s->type, ptr->value))
			return 0;

		if (strlen(ptr->s->type) == strlen(ptr->value)) {
			strcpy(ptr->s->type, ptr->value);
		} else {
			struct uci_section *old = ptr->s;
			ptr->s = uci_alloc_section(ptr->p, ptr->value, old->e.name, &old->e.list);
			uci_section_transfer_options(ptr->s, old);
			if (ptr->section == old->e.name)
				ptr->section = ptr->s->e.name;
			uci_free_section(old);
			ptr->s->package->n_section--;
		}
```

长度相同走原地 `strcpy`，长度不同就重建 section。重建时把 `old->e.name`
（对匿名段来说是自动生成的 `cfgXXXXXXXX`）当成名字传给
`uci_alloc_section()`，而后者只在 `name == NULL` 时才置 `anonymous = true`：

```211:212:components/uci/list.c
	if (name == NULL)
		s->anonymous = true;
```

于是**匿名段静默变成具名段**，那个本来只存在于内存里的哈希名会被写进
`/etc/config`。

**复现**（同一条命令，只是新类型的长度不同）：

```
$ cat etc/config/net
config interface
	option proto 'dhcp'

# 长度相同：interface(9) -> ifaceabcd(9)
$ uci set net.@interface[0]=ifaceabcd && uci commit net && cat etc/config/net
config ifaceabcd
	option proto 'dhcp'                    ← 仍然匿名，正确

# 长度不同：interface(9) -> iface(5)
$ uci set net.@interface[0]=iface && uci commit net && cat etc/config/net
config iface 'cfg016d96'                   ← 匿名性丢失，哈希名被写死
	option proto 'dhcp'
```

**为什么这是 bug 而不是特性**：`cfg016d96` 这个名字的设计意图（见 2.4）
是「内容的哈希」，一旦被写死成真名字，它就不会再随内容变化了，
2.4 描述的乐观并发保护随之失效。而且 `uci show` 的输出从
`net.@interface[0]` 变成 `net.cfg016d96`，所有按 `@type[n]` 语法写的脚本
在这个段上的行为都会改变。

**这个 bug 的形状是已知的**——`delta.c` 里对 `UCI_CMD_ADD` 打了完全对应的补丁：

```245:252:components/uci/delta.c
	case UCI_CMD_ADD:
		UCI_INTERNAL(uci_set, ctx, &ptr);
		if (!ptr.option && ptr.s)
			ptr.s->anonymous = true;
		break;
	case UCI_CMD_CHANGE:
		UCI_INTERNAL(uci_set, ctx, &ptr);
		break;
```

`UCI_CMD_ADD` 分支手工把 `anonymous` 补回来了，`UCI_CMD_CHANGE` 没有。

**修法**：在 `list.c` 的重建分支里保留 `old->anonymous`
（`ptr->s->anonymous = old->anonymous;`，放在 `uci_free_section(old)` 之前），
这样 `delta.c` 里那个补丁也可以一起删掉。

顺带一提同一段代码里的计数问题：`uci_alloc_section()` 做了 `p->n_section++`，
这里手工 `ptr->s->package->n_section--` 抵消掉。但 `uci_free_section()`
（其它所有删除路径）从来不减。全库 `n_section` 只有这一处减法
（`list.c:737`）。所以删段之后再加匿名段，名字里的序号前缀会跳号。
不影响正确性（哈希名只要唯一即可），但说明 `n_section` 的语义其实是
「历史累计分配数减去这一处特判」，不是「当前段数」。

### 5.5 `while (!feof())` 不查 `ferror()`：不可读的流会导致 100% CPU 死循环

**位置**：`components/uci/file.c:689`、`components/uci/delta.c:274`、
`components/uci/cli.c:599`

```689:694:components/uci/file.c
	while (!feof(pctx->file)) {
		pctx->pos = 0;
		uci_getln(ctx, 0);
		UCI_TRAP_SAVE(ctx, error);
		if (pctx->buf[0])
			uci_parse_line(ctx, single);
```

`uci_getln()` 里 `fgets` 返回 NULL 时直接 return，不区分 EOF 和错误：

```62:64:components/uci/file.c
		p = fgets(p, pctx->bufsz - ofs, pctx->file);
		if (!p || !*p)
			return;
```

于是任何「读失败但不置 EOF」的流都会让循环空转。`uci_open_stream()` 对读
有 `S_IFREG` 检查，但 **CLI 的 `-f` 选项直接 `fopen()` 后交给
`uci_import()`，绕过了这个检查**（`cli.c:718`）。

**复现**（目录可以 `fopen` 成功，但 `read()` 返回 `EISDIR`）：

```
$ mkdir /tmp/adir
$ timeout 5 ./uci-plain -f /tmp/adir import foo
rc=124                      ← 超时，即死循环

$ timeout 5 ./uci-plain -f /tmp/adir batch
rc=124
```

两条路径都挂死（纯 CPU 空转，内存不增长）。`delta.c:274` 的 delta 解析循环
是同样的写法，只是那里的流经过了 `uci_open_stream()` 的 `S_IFREG` 检查，
需要真实 I/O 错误（比如坏块）才能触发。

**修法**：`while (!feof(f) && !ferror(f))`，三处都改。
另外 `cli.c` 的 `-f` 应该拒绝非普通文件。

### 5.6 `uci_free_context()` 不释放 `conf2dir`（内存泄漏）

**位置**：`components/uci/libuci.c:72-95`

```72:80:components/uci/libuci.c
void uci_free_context(struct uci_context *ctx)
{
	struct uci_element *e, *tmp;

	if (ctx->confdir != uci_confdir)
		free(ctx->confdir);
	if (ctx->savedir != uci_savedir)
		free(ctx->savedir);
```

`confdir` 和 `savedir` 都处理了，`conf2dir` 漏了。而
`uci_set_conf2dir()` 是会 `strdup` 的（`libuci.c:103`）。
这个字段是较新加进来的（`fb3c234 add support for an override config directory`），
显然是漏改了对应的释放路径。

**复现**（LSan）：

```
$ cat poc_leak.c
struct uci_context *ctx = uci_alloc_context();
uci_set_conf2dir(ctx, "/var/run/uci-override");
uci_free_context(ctx);

==1883863==ERROR: LeakSanitizer: detected memory leaks
Direct leak of 22 byte(s) in 1 object(s) allocated from:
    #1 uci_set_conf2dir  src/libuci.c:103
SUMMARY: AddressSanitizer: 22 byte(s) leaked in 1 allocation(s).
```

**影响**：CLI 是一次性进程，无所谓。真正受影响的是 lua 绑定
（`lua/uci.c:925` 在 `uci.cursor(confdir, savedir, conf2dir)` 里调它），
每创建销毁一个 cursor 泄漏一次；以及任何反复建销 context 的长驻进程。

**修法**：加上 `if (ctx->conf2dir != uci_conf2dir) free(ctx->conf2dir);`。
注意 `uci_conf2dir` 目前只在 `libuci.c` 里定义、没在 `uci_internal.h` 里声明
（`uci_confdir` 和 `uci_savedir` 都声明了），顺手补上。

### 5.7 `uci_free_context(NULL)` 段错误

`uci_free_context()` 是全库唯一不做 NULL 检查的公共 API——
其它函数都靠 `UCI_HANDLE_ERR(ctx)` 里的 `if (!ctx) return UCI_ERR_INVAL;` 挡住了，
但 `uci_free_context()` 返回 void，没用那个宏，第一行就解引用。

```
$ ./poc_misc
calling uci_free_context(NULL)...
Segmentation fault      (rc=139)
```

`free(NULL)` 合法、`uci_free_package(NULL)` 也做了检查，
所以「释放函数容忍 NULL」在这个库里本来是一致的约定，这里是个例外。
`cli.c:695` 的写法（alloc 失败就 return，不调 free）说明作者是知道的，
但对库使用者来说这是个意外。

**修法**：函数开头加 `if (!ctx) return;`。

### 5.8 `uci_delete()` 用 `return 1` 表示参数错误，被报成 "Out of memory"

**位置**：`components/uci/list.c:567`

```566:567:components/uci/list.c
		if (!sscanf(ptr->value, "%d", &index))
			return 1;
```

`1` 就是 `UCI_ERR_MEM`：

```45:49:components/uci/uci.h
enum
{
	UCI_OK = 0,
	UCI_ERR_MEM,
	UCI_ERR_INVAL,
```

**复现**：

```
$ ./poc_err ./etc/config ./tmpsave      # uci_delete(net.lan.dns=abc)
uci_delete() = 1  -> reported as: Out of memory
```

一个「列表下标不是数字」的用户输入错误，被报成内存耗尽。
在嵌入式环境里这会把排障方向带偏得很远。应为 `UCI_ERR_INVAL`（2）。

### 5.9 未加引号的值紧跟 `;` 会静默吞掉该行剩余的命令

**位置**：`components/uci/file.c:230-242`

```230:242:components/uci/file.c
done:

	/*
	 * if the string was unquoted and we've stopped at a whitespace
	 * character, skip to the next one, because the whitespace will
	 * be overwritten by a null byte here
	 */
	if (pctx_cur_char(pctx) && next)
		pctx->pos += 1;

	/* terminate the parsed string */
	pctx_char(pctx, *target) = 0;
```

2.7 说过，解析器在输入缓冲上原地压缩，`*target ≤ pctx->pos`。
对**未加引号**的值来说，`addc()` 让两者同步前进，走到 `;` 时
`*target == pctx->pos`，于是最后那句 `pctx_char(pctx, *target) = 0`
**把分隔符 `;` 本身覆盖成了 NUL**。`uci_parse_line()` 随后
`strtok` 到一个空字符串，直接 return，该行剩余部分被丢弃——
**strict 模式下也不报错**。

加引号时 `*target` 停在闭合引号的位置（严格小于 `pctx->pos`），
`;` 得以保留；`;` 前有空格时终止符打在空格上，也没事。

**复现**（每行前半段是输入，后半段是 `uci show` 的结果）：

```
option x 1;option y 2        => net.a=s net.a.x='1'                     ← y 丢了
option x '1';option y '2'    => net.a=s net.a.x='1' net.a.y='2'         ← 正常
option x "1";option y "2"    => net.a=s net.a.x='1' net.a.y='2'         ← 正常
option x 1 ;option y 2       => net.a=s net.a.x='1' net.a.y='2'         ← 正常
list l 1;list l 2            => net.a=s net.a.l='1'                     ← 第二条丢了
```

**影响**：`;` 和不加引号的值都是解析器明确支持的语法（`uci export` 输出的
是带引号的规范形式，但手写的配置和第三方生成的配置未必）。
`uci import` 遇到这种输入会静默丢数据，然后 `uci import` 又是
overwrite 提交——**数据就这么没了，退出码 0，什么都不提示**。

**修法**：`parse_str()` 在 `case ';': next = false;` 这条路径上，
把终止符写在 `*target` 之前先确认 `*target != pctx->pos`；
或者更简单，先把 `;` 读走再打终止符。

### 5.10 静态审查发现、未复现的条目

这几条只在 OOM 路径上成立，我没有构造 malloc 失败注入去验证，列在这里备查：

1. **`uci_escape()` 在 malloc 失败时返回未转义的原串**（`file.c:568-573`）。
   ```c
   if (!ctx->buf) {
       ctx->bufsz = LINEBUF;
       ctx->buf = malloc(LINEBUF);
       if (!ctx->buf)
           return str;      /* ← 原样返回，单引号没被转义 */
   }
   ```
   而 3.3 说过，`uci export` 的输出会被 shell `eval`。一个含 `'` 的配置值
   在这条路径上就能逃出单引号 → **OOM 时的 shell 注入**。
   即使不考虑注入，静默产生语法错误的导出结果也不可接受。
   应该改成抛 `UCI_ERR_MEM`（这个函数所在的调用链是有 trap 的）。

2. **`cli.c:117-125` realloc 失败后留下悬垂的全局 `typestr`**：
   ```c
   void *p = realloc(typestr, maxlen);
   if (!p) {
       free(typestr);       /* 旧块释放了 */
       return NULL;         /* 但全局 typestr 还指着它 */
   }
   ```
   下一次 `uci_lookup_section_ref()` 会 `realloc(悬垂指针)`，
   `uci_reset_typelist()` 会再 `free()` 一次。应在 `free` 后置 NULL。

3. **`uci_file_commit()` 的 `realpath()` 结果在错误路径上泄漏**
   （`file.c:820-827`）：`stat`/`chmod`/`rename` 任一失败就
   `unlink` + throw，没有 `free(path)`。

4. **`uci_load_delta()` 在 `uci_file_load()` 里是在 trap 之外调用的**
   （`file.c:955`）。它内部可以抛 `UCI_ERR_MEM`（asprintf）或
   `UCI_ERR_IO`（ftruncate），一旦抛出就绕过 `done:`，
   已打开的 `FILE *file` 不会被关闭 → fd 泄漏。

5. **`uci_batch_cmd()` 在 `uci_parse_argument()` 失败时泄漏已 strdup 的参数**
   （`cli.c:565-568`）：它把 `i` 置 0 再 break，于是末尾的
   `for (j = 0; j < i; j++) free(argv[j]);` 一个都不释放。
   `exit` 命令那条 `return 254` 路径同理。

6. **`uci_file_commit()` 用 `alloca(sz + 1)` 构造临时文件名**（`file.c:755`），
   长度来自 `confdir + 包名`。虽然两者都有实际上界，但 `alloca` 的失败模式是
   栈溢出而不是返回 NULL，这种地方用 `malloc` 更稳妥。

---

## 六、复现环境

```
OS        : linux 6.9.12-1.NSN.el8.x86_64
gcc       : 14.2.0        （ASan/UBSan/LSan 用它，clang 19 的 sanitizer
                            运行时缺 libunwind.so.1，跑不起来）
valgrind  : 3.26.0
被测代码  : components/uci @ 74f6277（== origin/master，工作区干净）
```

构建方式（跳过 `blob.c`，避开 libubox 依赖；本文所有问题都不在 `blob.c` 里）：

```bash
cp -r components/uci /tmp/uci-analysis/src && cd /tmp/uci-analysis/src
printf '#define UCI_PREFIX "/usr"\n' > uci_config.h

# ASan / UBSan / LSan 版
gcc -g -O0 -fno-omit-frame-pointer -fsanitize=address,undefined -std=gnu99 -I. \
    -shared -fPIC -o libuci-gsan.so libuci.c file.c util.c delta.c parse.c
gcc -g -O0 -fno-omit-frame-pointer -fsanitize=address,undefined -std=gnu99 -I. \
    -o uci-gsan cli.c -L. -luci-gsan -Wl,-rpath,$PWD

# 无插桩版（给 valgrind 和「真实固件行为」用）
gcc -g -O0 -std=gnu99 -I. -shared -fPIC -o libuci-plain.so \
    libuci.c file.c util.c delta.c parse.c
gcc -g -O0 -std=gnu99 -I. -o uci-plain cli.c -L. -luci-plain -Wl,-rpath,$PWD
```

各条的验证手段：

| 条目 | 手段 |
|------|------|
| 5.1 | ASan（`uci -S -m import <path>` / `uci -S add <path>`）+ 无插桩版 glibc double-free abort |
| 5.2 | ASan + 自写 PoC `poc_commit.c`（模拟长驻库使用者） |
| 5.3 | valgrind memcheck |
| 5.4 | CLI 行为对比（同一命令，新类型长度相同 vs 不同） |
| 5.5 | `timeout 5` 观察退出码 124 |
| 5.6 | LeakSanitizer |
| 5.7 | 直接运行，SIGSEGV |
| 5.8 | 自写 PoC 打印返回码 + `uci_perror` |
| 5.9 | CLI 行为矩阵（引号 × 空格 × `;`） |
| 4.1 性能数据 | 生成 2000~32000 段的配置，`/usr/bin/time` 测 `uci export` |

`UCI_TRAP_SAVE`/`UCI_TRAP_RESTORE` 的展开（2.1）用 `gcc -E -P` 验证，
「漏写分号静默吞掉整块代码」用一个独立的最小样例在 `-Wall -Wextra -O2`
下编译运行确认（零警告，代码不执行）。
