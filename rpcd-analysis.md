# rpcd 代码分析

分析对象：`components/rpcd`（约 9000 行 C 代码）
分析日期：2026-08-19

> **前提说明**
>
> 本仓库中的 `components/rpcd` 就是 **upstream 本身**：`origin` 指向官方仓库
> `https://github.com/openwrt/rpcd.git`，`HEAD` 与 `origin/master` 完全一致
> （`e37ed9d`，2026-07-19，共 273 个提交），工作区干净，无本地独有提交。
> `package/system/rpcd/Makefile` 里的 `PKG_SOURCE_VERSION` 也正好是 `e37ed9d`，
> 说明构建系统拉的就是这份代码。因此修 bug 的正确路径是直接改本仓库、按
> OpenWrt 格式提交（`file: fix ...` + `git commit -s`）后提 PR 或发
> openwrt-devel。**报告缺陷前请先 `git fetch`。**
>
> 最新一条提交 `e37ed9d`（file: re-authorize ACL against resolved path to close
> symlink bypass）刚刚堵掉了 file 插件的符号链接绕过，本文 2.6 节会详细讲这个
> 修复的形状，第五章会说明它留下的残余 TOCTOU 窗口。
>
> **第四、五章的结论分两类，文中逐条标注：**
>
> - **【实测】**——我在本机把 rpcd 完整编译并跑起来复现过。做法是用之前分析
>   ubus 时留下的 `/tmp/inst`（libubox + blobmsg_json）和 `/tmp/jc`（json-c），
>   补编译 `components/uci` 得到 `libuci.so`，再用 `-DIWINFO_SUPPORT=OFF
>   -DUCODE_SUPPORT=OFF` 编出 `rpcd` + `file.so` + `rpcsys.so`，然后在
>   `unshare -Urm` 的 user/mount namespace 里 bind-mount 出 `/etc/config`、
>   tmpfs 挂 `/var/run`，跑 `ubusd` + `rpcd`，用 `ubus` CLI 打请求。完整脚本见
>   第六章，文中贴的输出都是原文粘贴。
> - **【静态】**——逐行核对源码得出的结论，未做运行时复现。
>
> iwinfo 插件因缺少 `libiwinfo` 未纳入实测，只做了静态阅读。

---

## 目录

1. [代码架构](#一代码架构)
2. [设计原理](#二设计原理)
3. [使用方法](#三使用方法)
4. [可以优化的点](#四可以优化的点)
5. [已核验的问题](#五已核验的问题)
6. [复现环境](#六复现环境)

---

## 一、代码架构

### 1.1 rpcd 在 OpenWrt 里的位置

rpcd 的名字容易误导：它**不是** RPC 协议的实现，也不监听任何网络端口。它是一个
普通的 ubus 客户端进程，作用是把「不适合放进 ubusd、也不该让每个守护进程各自实现
一遍」的一堆系统能力挂到 ubus 总线上，并额外提供一套**会话（session）+ ACL** 机制
供 HTTP 前端使用。

一次 LuCI 页面请求的完整链路是这样的：

```
浏览器
  │  POST /ubus   {"jsonrpc":"2.0","method":"call",
  │                "params":["<sid>","uci","get",{...}]}
  ▼
uhttpd + uhttpd-mod-ubus
  │  ① 用 <sid> 调 session.access 问 rpcd：这个会话能不能调 uci/get？
  │  ② 通过后把 "ubus_rpc_session":"<sid>" 塞进消息体
  │  ③ 转发成一次普通 ubus 调用
  ▼
ubusd（消息路由 + 连接级 ACL，/usr/share/acl.d/*.json）
  ▼
rpcd（会话级 ACL，/usr/share/rpcd/acl.d/*.json）
  │  uci / file / rc / rpc-sys / iwinfo / session / ucode 脚本对象
  ▼
libuci、fork+exec、/etc/init.d、libiwinfo、ucode VM ...
```

**这里有两套完全独立、极易混淆的 ACL：**

| | ubusd ACL | rpcd 会话 ACL |
|---|---|---|
| 目录 | `/usr/share/acl.d/*.json` | `/usr/share/rpcd/acl.d/*.json` |
| 读取者 | `ubusd`（`ubusd_acl.c:75`） | `rpcd`（`session.h:37`） |
| 主体 | unix socket 对端的 **user/group** | **会话 ID**（32 位十六进制串） |
| 粒度 | 对象名 / 方法名 / 事件 / 是否可 publish | scope / object / function 三元组 |
| 谁在用 | 非 root 进程连 ubusd 时 | HTTP 前端（LuCI、其它 REST 网关） |

`unauthenticated.json`（rpcd 自带的唯一一个 ACL 文件）只放开两个方法：

```1:13:components/rpcd/unauthenticated.json
{
	"unauthenticated": {
		"description": "Access controls for unauthenticated requests",
		"read": {
			"ubus": {
				"session": [
					"access",
					"login"
				]
			}
		}
	}
}
```

也就是「未登录用户只能问『我能不能干这件事』和『登录』」，这是整个 LuCI 权限模型
的地基。

### 1.2 构建产物

`CMakeLists.txt` 切出 1 个可执行文件 + 4 个可选插件：

| 产物 | 源文件 | 链接 | 开关 |
|------|--------|------|------|
| `rpcd`（`/sbin`） | `main.c` `exec.c` `session.c` `uci.c` `rc.c` `plugin.c` | ubox ubus uci blobmsg_json json-c crypt dl | 必选 |
| `file.so` | `file.c` | ubox ubus | `FILE_SUPPORT` |
| `rpcsys.so` | `sys.c` | ubox ubus | `RPCSYS_SUPPORT` |
| `iwinfo.so` | `iwinfo.c` | ubox ubus iwinfo | `IWINFO_SUPPORT` |
| `ucode.so` | `ucode.c` | ucode | `UCODE_SUPPORT` |

插件装到 `${prefix}/lib/rpcd/`（OpenWrt 下即 `/usr/lib/rpcd`）。编译选项
`-Os -Wall -Werror --std=gnu99 -g3 -Wmissing-declarations`，并把
`-DINSTALL_PREFIX` 编进去——所有运行时路径都由它拼出来，这也是我能在
`/tmp/rpcd-root` 下做沙箱测试的原因。

注意 `ucode.so` **只链接 `libucode`**，不链接 libubus——它靠 rpcd 主程序已经
加载的符号；而且它在 `rpc_ucode_api_init()` 里会把自己用 `RTLD_GLOBAL` 再
`dlopen()` 一遍，为的是让后续 ucode 脚本 `require()` 的动态扩展能解析到
libucode 的符号：

```1074:1079:components/rpcd/ucode.c
	/* reopen ucode.so with RTLD_GLOBAL in order to export libucode runtime
	 * symbols for ucode extensions loaded later at runtime */
	if (!dlopen(RPC_LIBRARY_DIRECTORY "/ucode.so", RTLD_LAZY|RTLD_GLOBAL)) {
		fprintf(stderr, "Failed to dlopen() ucode.so: %s, dynamic ucode plugins may fail\n",
		        dlerror());
	}
```

### 1.3 进程内分层

| 层 | 文件 | 行数 | 职责 |
|----|------|------|------|
| 启动 / 信号 | `main.c` | 137 | 参数解析、ubus 连接、四个 `*_api_init()`、SIGHUP 自我 exec |
| 会话与 ACL | `session.c` | 1473 | 会话 AVL 树、通配符 ACL、login、freeze/thaw |
| 配置读写 | `uci.c` | 1808 | uci 对象、按会话隔离的 delta、apply/confirm/rollback |
| 服务管理 | `rc.c` | 417 | `/etc/init.d` 扫描与调用 |
| 插件装载 | `plugin.c` | 552 | 扫 `libexec/rpcd`（exec 插件）与 `lib/rpcd`（.so 插件） |
| 子进程执行 | `exec.c` | 413 | fork/exec + ustream + 延迟应答的通用封装 |
| file 插件 | `file.c` | 1254 | read/write/list/stat/md5/remove/exec |
| sys 插件 | `sys.c` | 502 | 改密码、sysupgrade、恢复出厂、重启、包列表 |
| iwinfo 插件 | `iwinfo.c` | 955 | libiwinfo 的 blobmsg 包装 |
| ucode 插件 | `ucode.c` | 1109 | 用 ucode 脚本定义 ubus 对象 |

`main()` 的初始化顺序是有讲究的：

```117:127:components/rpcd/main.c
	rpc_session_api_init(ctx);
	rpc_uci_api_init(ctx);
	rpc_rc_api_init(ctx);
	rpc_plugin_api_init(ctx);

	hangup = getenv("RPC_HANGUP");

	if (!hangup || strcmp(hangup, "1"))
		rpc_uci_purge_savedirs();
	else
		rpc_session_thaw();
```

session 必须最先初始化，因为 `rpc_uci_api_init()` 会通过
`rpc_session_destroy_cb()` 注册一个「会话销毁时删掉它的 uci delta 目录」的回调；
插件最后加载，因为它们要通过 `rpc_daemon_ops` 拿到 session 和 exec 的函数指针。

### 1.4 四种「插件」形态

这是 rpcd 架构上最值得注意的一点：它同时支持**四种**扩展方式，代价和能力差别很大。

| 形态 | 位置 | 注册时机 | 每次调用的开销 | 典型用户 |
|------|------|----------|----------------|----------|
| 内建对象 | 编译进 `rpcd` | `main()` | 函数调用 | `session` `uci` `rc` |
| `.so` 插件 | `/usr/lib/rpcd/*.so` | 启动时 `dlopen` + `rpc_plugin` 符号 | 函数调用 | `file` `rpc-sys` `iwinfo` |
| exec 插件 | `/usr/libexec/rpcd/*`（可执行） | 启动时执行 `<prog> list` 拿方法签名 | **fork + exec + JSON 编解码** | shell 脚本类扩展 |
| ucode 脚本 | `/usr/share/rpcd/ucode/*` | 启动时编译 + 执行一次拿签名 | VM 内函数调用（**无 fork**） | 新代码首选 |

exec 插件的调用路径是这样的（`plugin.c:201-255`）：把整个请求 blobmsg
`blobmsg_format_json()` 成 JSON 写进子进程 stdin，执行 `<prog> call <method>`，
再把 stdout 的 JSON 解析回 blobmsg。ucode 插件则是常驻 VM，签名对象在启动时求值
一次，之后每次调用只是往 VM 栈上压函数和请求对象再 `uc_vm_call()`。

**这就是 OpenWrt 近年把 rpcd 扩展从 shell 脚本迁到 ucode 的根本原因**：前者每次
请求一次 fork+exec+两次 JSON 转换，后者是一次函数调用。

### 1.5 插件 ABI：`rpc_daemon_ops`

.so 插件与主程序之间只有一个结构体，没有别的耦合：

```47:63:components/rpcd/include/rpcd/plugin.h
struct rpc_daemon_ops {
    bool (*session_access)(const char *sid, const char *scope,
                           const char *object, const char *function);
    void (*session_create_cb)(struct rpc_session_cb *cb);
    void (*session_destroy_cb)(struct rpc_session_cb *cb);
    int (*exec)(const char **args,
                rpc_exec_write_cb_t in, rpc_exec_read_cb_t out,
                rpc_exec_read_cb_t err, rpc_exec_done_cb_t end,
                void *priv, struct ubus_context *ctx,
                struct ubus_request_data *req);
    int *exec_timeout;
};

struct rpc_plugin {
    struct list_head list;
    int (*init)(const struct rpc_daemon_ops *ops, struct ubus_context *ctx);
};
```

插件只需导出一个名为 `rpc_plugin` 的全局符号。这个 ABI 没有版本号、没有大小字段
——在结构体中间插字段就会静默破坏所有外部编译的插件（`rpcd-mod-*` 之外还有
`rpcd-mod-luci`、`rpcd-mod-lxc` 等第三方包）。只能在尾部追加。

---

## 二、设计原理

### 2.1 会话与 ACL：两级 AVL + 通配符前缀键

会话数据结构（`session.h:41-67`）是三层 AVL：

```
sessions (avl, key=sid)
  └── rpc_session
        ├── data  (avl, key=名字)        →  任意 blobmsg 键值对
        └── acls  (avl, key=scope, dup)  →  rpc_session_acl_scope
                                              └── acls (avl, key=通配符前缀, dup)
                                                    →  rpc_session_acl {object, function}
```

最巧妙的是 ACL 树的 key 设计。ACL 的 object 可以带通配符（`luci-rpc.*`），
如果直接拿完整 pattern 做 key，查找时只能全表扫描。rpcd 的做法是**用第一个通配符
之前的部分做 key**：

```425:429:components/rpcd/session.c
static int
uh_id_len(const char *str)
{
	return strcspn(str, "*?[");
}
```

查找时从「小于等于目标对象名的最后一个元素」开始**倒着走**，只要 key 还是目标名
的前缀就继续，再用 `fnmatch()` 做精确判定：

```137:147:components/rpcd/session.c
#define uh_foreach_matching_acl_prefix(_acl, _avl, _obj, _func)		\
	for (_acl = avl_find_le_element(_avl, _obj, _acl, avl);			\
	     _acl;														\
	     _acl = avl_is_first(_avl, &(_acl)->avl) ? NULL :			\
		    avl_prev_element((_acl), avl))

#define uh_foreach_matching_acl(_acl, _avl, _obj, _func)			\
	uh_foreach_matching_acl_prefix(_acl, _avl, _obj, _func)			\
		if (!strncmp((_acl)->object, _obj, (_acl)->sort_len) &&		\
		    !fnmatch((_acl)->object, (_obj), FNM_NOESCAPE) &&		\
		    !fnmatch((_acl)->function, (_func), FNM_NOESCAPE))
```

这是一个「用有序树近似前缀树」的经典技巧，`O(log n)` 定位 + 少量回溯。
不过外层循环**没有提前终止条件**：一旦 key 不再是前缀，它仍会一路倒着走到树头。
`rpc_session_grant()` 的去重检查用的正是不带 `fnmatch` 过滤的 prefix 版本
（`session.c:446`），所以每次 grant 实际是 `O(n)` 回溯。登录时要 grant 几百条
ACL，这就是 `O(n²)`（见 4.2）。

顺带一提，`sort_len` 这个字段在 `struct rpc_session_acl` 里声明了、在
`uh_foreach_matching_acl` 里用了，但 `rpc_session_grant()` **从未给它赋值**
（`calloc_a` 清零，恒为 0），于是那句 `strncmp(..., 0)` 恒真，成了纯粹的空操作。
不影响正确性（后面的 `fnmatch` 才是真判定），但那行代码是死的。

### 2.2 ACL 文件如何展开成会话授权

`/usr/share/rpcd/acl.d/*.json` 的四层结构与 `/etc/config/rpcd` 里 login 段的
`list read` / `list write` 是这样咬合的：

```
ACL 文件：      <access-group>: { <permission>: { <scope>: { <object>: [<function>...] } } }
                     ↑                 ↑
             login 段的 list read/     只能是 "read" 或 "write"
             list write 用 fnmatch
             匹配这个组名
```

`rpc_login_setup_acl_file()`（`session.c:1041-1110`）遍历每个 ACL 文件的每个组，
对组里的 `read`/`write` 块调用 `rpc_login_test_permission()` 判断当前用户是否拥有
该组的该权限；通过了才把里面的 scope/object/function 逐条 `rpc_session_grant()`。

四个容易踩坑的细节。

**其一，`list write` 蕴含 `list read`，但只在「组」这一级**：

```977:979:components/rpcd/session.c
	/* make sure that write permission implies read permission */
	if (!strcmp(perm, "read"))
		return rpc_login_test_permission(s, "write", group);
```

意思是「用户有该组的 write 权限 ⇒ 该组的 read 块也生效」。但如果 ACL 文件里某个组
**只写了 `write` 块、没写 `read` 块**，那么用户拿到的 function 就只有 `write`，
`read` 一律拒绝。我第一次做实测时就被这个坑了：只授了 write 的会话调 `uci.changes`
返回 Permission denied。LuCI 的 ACL 文件永远两块都写，所以平时看不出来。

其余三条：

- **negative pattern 优先**：`list read '!luci-app-*'` 这类以 `!` 开头的表达式
  先被扫一遍，命中即拒（`session.c:951-966`）。
- **数组记法 vs 表记法**：scope 的值是数组时，权限名本身（`read`/`write`）被当作
  function 名；是表时才逐个列 function（`session.c:984-1039`）。所以
  `"uci": ["testcfg"]` 展开成 `uci/testcfg/write`，而不是「所有 uci 方法」。
- **`access-group` 元 scope**：每匹配一个组，rpcd 额外授一条
  `access-group/<组名>/<权限>`，让前端可以一次性问「我属于哪些组」而不必逐个
  三元组试探（`session.c:1101-1103`）。实测输出见 5.7。

### 2.3 freeze / thaw：靠 SIGHUP 原地 re-exec 保住会话

rpcd 没有守护进程热重载机制——插件是 `dlopen` 进来的，装了新 `rpcd-mod-*` 只能
重启。但重启会把所有登录会话打掉，LuCI 用户就被踢下线了。rpcd 的解法是
**把自己 exec 掉，中间把会话序列化到 tmpfs**：

```39:45:components/rpcd/main.c
static void
handle_signal(int sig)
{
	rpc_session_freeze();
	uloop_cancelled = true;
	respawn = (sig == SIGHUP);
}
```

- `rpc_session_freeze()`：把每个非默认会话 `blob` 化后写成
  `/var/run/rpcd/sessions/<sid>`（`0600`，且**不含 ACL**）。
- `uloop_run()` 返回后，`exec_self()` 设置 `RPC_HANGUP=1` 再 `execv(argv[0])`
  ——PID 不变，进程映像换新。
- 新映像看到 `RPC_HANGUP=1` 就走 `rpc_session_thaw()`，逐个文件恢复会话、
  `unlink`，并**根据会话数据里的 username 重新从 `/etc/config/rpcd` 推导 ACL**
  （`session.c:1337-1341`），而不是恢复旧 ACL。

最后这点是有意为之：既然重启的目的通常就是加载新插件、新 ACL 文件，那么 ACL 就该
按新配置重算。副作用是**如果 login 段被删了，会话会以零 ACL 复活**——它还在，但
什么都干不了。

`rpcd.init` 的 `reload_service()` 只做 `procd_send_signal rpcd`（默认 SIGHUP），
`rpcd-mod-*` 的 postinst 也调它，整条链路就是为这个机制服务的。

【实测】SIGHUP 前后 PID 不变，新映像的环境里出现了 `RPC_HANGUP=1`，会话依旧在：

```
### T7b: SIGHUP really re-execs rpcd (same pid, new image)
RPC_HANGUP before: ''
pid: 3652635 -> 3652635
RPC_HANGUP after : 'RPC_HANGUP=1'
session still known: 1
```

### 2.4 uci：按会话隔离的 delta + 三段式 apply

这是 rpcd 里设计最完整的一块。核心思想是**利用 libuci 的 savedir（delta 目录）
机制，给每个会话一个独立的暂存区**：

```288:303:components/rpcd/uci.c
static void
rpc_uci_set_savedir(struct blob_attr *sid)
{
	char path[PATH_MAX];

	if (!sid)
	{
		rpc_uci_replace_savedir("/tmp/.uci");
		return;
	}

	snprintf(path, sizeof(path) - 1,
	         RPC_UCI_SAVEDIR_PREFIX "%s", blobmsg_get_string(sid));

	rpc_uci_replace_savedir(path);
}
```

于是：A 用户改的 network 与 B 用户改的 network 互不可见，各自 `uci.changes`
只看到自己的；**没有会话 ID 的本地调用**（比如 shell 里直接 `ubus call uci set`）
落到 `/tmp/.uci`，也就是和命令行 `uci` 工具共用暂存区。会话销毁时回调
`rpc_uci_purge_savedir_cb()` 把目录删掉；进程冷启动时 `rpc_uci_purge_savedirs()`
把所有残留目录清掉（re-exec 时故意不清，见 1.3 的代码）。

`apply` 的状态机是为「远程改网络配置把自己关在门外」这个经典问题设计的：

```
uci.set/add/delete ...        → 只写会话 delta 目录
uci.apply {rollback:true}     → ① 把 /etc/config/<c> 快照到 /var/run/rpcd/snapshot-files/
                                ② 把 delta 快照到 snapshot-delta/
                                ③ 真正 uci_commit + 通知 service 对象 config.change
                                ④ 记下 apply_sid，装 60s（可配）定时器
uci.confirm                   → 定时器取消、快照删除、变更坐实
（超时或 uci.rollback）       → 用快照恢复 /etc/config，重新 commit + 通知，
                                若原会话还在则连 delta 一并还原
```

回滚时有一个细节很讲究：

```1524:1525:components/rpcd/uci.c
	/* avoid merging unrelated uci changes when restoring old configs */
	rpc_uci_replace_savedir("/dev/null");
```

把 savedir 指到 `/dev/null`，确保恢复快照文件时不会把任何 delta 又合并进去。

**代价是这套状态机是全局单例**：`apply_sid`、`apply_timer`、`apply_ctx` 都是文件
级静态变量，同一时刻只允许一个待确认的 apply。第二个会话再 apply 会怎样？见 5.2
——它拿到成功返回，但什么都没发生。

### 2.5 exec 子系统：ustream + 延迟应答

`exec.c` 把「跑一个子进程并把输出变成 ubus 应答」这件事封装成 `rpc_exec()`。
关键点：

- 三条管道分别接子进程 stdin/stdout/stderr，父进程侧用 `ustream_fd` 挂到 uloop。
- **调用 `ubus_defer_request()` 立刻放行事件循环**，rpcd 不会因为子进程而阻塞。
- stdout/stderr 各限 `RPC_EXEC_MAX_SIZE`（4096×64 = 256 KB），写满即以
  `UBUS_STATUS_NOT_SUPPORTED` 结束。
- 两条管道都 EOF 才应答（`rpc_exec_*_state_cb`），而不是子进程退出即应答
  ——避免丢掉缓冲区里的尾巴。
- 超时（`-t`，默认 120 s，OpenWrt init 传 30 s）到点直接 `SIGKILL`。
- 应答不是在回调里直接发，而是 `uloop_timeout_set(&c->timeout, 0)` 排到下一轮，
  这样可以安全地在 ustream 回调里释放 ustream 自身。

`rpc_exec_process_cb()` 里有一段值得单独看的注释，它是一个已修复 double-close 的
现场：

```196:203:components/rpcd/exec.c
	close(c->opipe.fd.fd);
	close(c->epipe.fd.fd);

	/* ustream_free() does not reset the fd, and rpc_exec_reply() closes it
	 * again later.  Mark the descriptors as consumed so that the second
	 * close() cannot accidentally close an unrelated, meanwhile reused fd. */
	c->opipe.fd.fd = -1;
	c->epipe.fd.fd = -1;
```

【实测】异步性确实成立——`file.exec` 跑 `sleep 3` 期间，另一个客户端的
`session.list` 在 307 ms 内就返回了：

```
### T12: file.exec is async - rpcd stays responsive while a child runs
session.list answered after 307ms while sleep 3 was running
```

需要强调的是：**只有 exec 路径是异步的**。`uci.*`、`iwinfo.*`、ucode 脚本都在
主循环里同步跑完（见 4.1）。

### 2.6 file 插件：路径规范化与刚补上的符号链接重授权

`file` 插件是 rpcd 里攻击面最大的部分（能读写任意路径、能执行任意命令），它的
访问控制分三步：

1. `rpc_canonicalize_path()`——**纯文本**折叠 `//`、`/./`、`/x/../`，不碰
   符号链接（`file.c:189-248`）。
2. 用折叠后的路径查 ACL：`ops->session_access(sid, "file", path, perm)`。
3. `rpc_check_symlink_access()`——`realpath()` 解析真实路径，**如果解析结果与
   输入不同，就拿解析结果再查一次 ACL**，并用解析结果替换后续 `open()` 的路径。

第 3 步是 `e37ed9d` 刚加的。在它之前，只要在 ACL 覆盖的目录里放一个指向
`/etc/shadow` 的符号链接，`file.read` 就会跟着链接读出去。现在的处理还额外区分了
「悬空符号链接」和「路径确实不存在」——后者是 `file.write` 创建新文件的合法场景，
前者必须拒绝，否则可以用符号链接把新文件创建到 ACL 之外：

```293:301:components/rpcd/file.c
	if (errno == ENOENT)
	{
		/* realpath() also fails with ENOENT for a dangling symlink whose
		 * final target component is missing. Distinguish that case (an
		 * *existing* symlink we must not silently create-through, e.g.
		 * via open(O_CREAT) on file.write) from a genuinely nonexistent
		 * path by lstat()'ing the requested path itself. */
		errno = (lstat(*path, &lst) == 0 && S_ISLNK(lst.st_mode)) ? EACCES : 0;
	}
```

【实测】修复有效：ACL 覆盖 `/tmp/rpcd-test/*`，在里面放一个指向 `/etc/passwd`
的符号链接，读被拒绝；指向同样在 ACL 内的文件则允许：

```
### T6: file ACL vs symlink
-- read through symlink inside the ACL-covered dir:
{ "data": "secret\n" }
-- symlink pointing OUTSIDE the ACL scope:
Command failed: ... file read {...\"path\":\"/tmp/rpcd-test/sandbox/passwd\"} (Permission denied)
```

`file.exec` 的 ACL 则是另一套逻辑（`file.c:1060-1083`）：**先拿裸可执行文件路径
查一次**，命中就放行（此时参数不受任何限制）；不命中才把
`"<可执行文件> <参数1> <参数2> ..."` 拼成一个字符串（上限 1024 字节）再查一次。
这个设计的后果见 5.3。

### 2.7 ucode 插件：常驻 VM

每个 `/usr/share/rpcd/ucode/*.uc` 脚本：

1. 启动时 `uc_compile()` + `uc_vm_execute()` 各一次，**脚本的返回值**就是对象签名
   （见 `examples/ucode/example-plugin.uc`）。
2. `rpc_ucode_script_validate()` 严格校验签名形状：顶层键=对象名，二层=方法名，
   三层必须有可调用的 `call`，可选 `args` 声明参数类型。
3. `rpc_ucode_method_register()` 把 `args` 的 ucode 类型翻译成 `blobmsg_policy`
   ——所以 `ubus -v list` 能看到 ucode 插件的参数签名。
4. 调用时 `rpc_ucode_validate_call_args()` 逐个参数比对策略（比 blobmsg_parse
   更严：**多余的参数直接 `INVALID_ARGUMENT`**，只有 `ubus_rpc_session` 例外）。

回复有三种写法，覆盖了同步/异步两种场景：`call` 直接 `return {...}`；
`request.reply({...})`；或者 `return ubus.defer(...)` 把嵌套 ubus 调用的 deferred
对象返回给 rpcd，rpcd 会装一个 guard timer 并把 request 对象塞进
`pending_replies` 数组防止被 GC 掉（`ucode.c:462-477`）。

VM 是常驻的，意味着**脚本的全局变量在请求之间保持**——这既是特性（可以缓存
连接、状态）也是陷阱（一个脚本泄漏内存会一直累积，而 rpcd 没有插件卸载机制，
源码注释里直言 "we don't have a teardown mechanism in rpcd plugins"）。

---

## 三、使用方法

### 3.1 启动与配置

```1:17:package/system/rpcd/files/rpcd.init
#!/bin/sh /etc/rc.common

START=12

USE_PROCD=1
NAME=rpcd
PROG=/sbin/rpcd

start_service() {
	local socket=$(uci -q get rpcd.@rpcd[0].socket)
	local timeout=$(uci -q get rpcd.@rpcd[0].timeout)

	procd_open_instance
	procd_set_param command "$PROG" ${socket:+-s "$socket"} ${timeout:+-t "$timeout"}
	procd_set_param respawn
	procd_close_instance
}
```

`/etc/config/rpcd`：

```1:9:package/system/rpcd/files/rpcd.config
config rpcd
	option socket /var/run/ubus/ubus.sock
	option timeout 30

config login
	option username 'root'
	option password '$p$root'
	list read '*'
	list write '*'
```

- `option timeout`：**子进程执行超时**（秒，1~600），不是会话超时。会话超时是
  `session.create`/`login` 的 `timeout` 参数，默认 300 s。
- `option password '$p$root'`：`$p$<user>` 是 rpcd 特有记法，表示「去
  `/etc/shadow`（或 `/etc/passwd`）里查 `<user>` 的真实哈希」
  （`session.c:831-848`）。**空密码等于放行任何密码**（`session.c:824-827`）。
- 命令行只有两个参数：`-s <ubus socket>`、`-t <exec 超时秒数>`。

### 3.2 ubus 对象与方法总表

| 对象 | 来源 | 方法 |
|------|------|------|
| `session` | 内建 | `create` `list` `grant` `revoke` `access` `set` `get` `unset` `destroy` `login` |
| `uci` | 内建 | `configs` `get` `state` `add` `set` `delete` `rename` `order` `changes` `revert` `commit` `apply` `confirm` `rollback` `reload_config` |
| `rc` | 内建 | `list` `init` |
| `file` | `file.so` | `read` `write` `list` `stat` `lstat` `md5` `remove` `exec` |
| `rpc-sys` | `rpcsys.so` | `packagelist` `password_set` `upgrade_test` `upgrade_start` `upgrade_clean` `factory` `reboot` |
| `iwinfo` | `iwinfo.so` | `devices` `info` `scan` `assoclist` `freqlist` `txpowerlist` `countrylist` `survey` `phyname` |

几个容易忽略的语义：

- `uci.state` 与 `uci.get` 共用 handler，区别只是把 savedir 指到 `/var/state`。
- `uci.apply` 不带 `rollback` 时就是一次「提交所有本会话的暂存变更」；带
  `rollback:true` 才进入 2.4 的确认流程。
- `rc.list` 的 `running` 字段只对 `USE_PROCD=1` 的脚本给出，实现方式是**真的去
  跑一次 `/etc/init.d/<name> running`**。
- `rpc-sys.upgrade_start` 固定升级 `/tmp/firmware.bin`，固件得先用 `file.write`
  或 uhttpd 的 cgi-upload 传上去。

### 3.3 会话工作流

```bash
# 1. 登录，拿 sid
SID=$(ubus call session login '{"username":"root","password":"secret"}' \
      | jsonfilter -e '@.ubus_rpc_session')

# 2. 之后每次调用都带上 ubus_rpc_session
ubus call uci get "{\"ubus_rpc_session\":\"$SID\",\"config\":\"network\"}"

# 3. 改配置（只进会话暂存区）
ubus call uci set "{\"ubus_rpc_session\":\"$SID\",\"config\":\"network\",
                    \"section\":\"lan\",\"values\":{\"ipaddr\":\"192.168.9.1\"}}"
ubus call uci changes "{\"ubus_rpc_session\":\"$SID\",\"config\":\"network\"}"

# 4. 带回滚保护地生效：60 秒内不 confirm 就自动还原
ubus call uci apply "{\"ubus_rpc_session\":\"$SID\",\"rollback\":true,\"timeout\":60}"
#    改完能连上 → 确认
ubus call uci confirm "{\"ubus_rpc_session\":\"$SID\"}"
#    连不上 → 什么都不做，等它自己回滚

# 5. 查权限 / 登出
ubus call session access "{\"ubus_rpc_session\":\"$SID\",\"scope\":\"uci\",
                           \"object\":\"network\",\"function\":\"write\"}"
ubus call session destroy "{\"ubus_rpc_session\":\"$SID\"}"
```

不带 `ubus_rpc_session` 直接调用时，所有 ACL 检查被跳过（`file.c:179-187`、
`uci.c:310-337` 都是 `if (!sid) return true;`）。这是有意的：能连上 ubus socket
的本地进程本来就等价于 root。

### 3.4 写一个 ACL 文件

放到 `/usr/share/rpcd/acl.d/my-app.json`：

```json
{
  "my-app": {
    "description": "My application",
    "read": {
      "ubus": { "my-object": [ "status", "list" ] },
      "uci":  [ "my-config" ],
      "file": { "/var/log/my-app.log": [ "read" ] }
    },
    "write": {
      "ubus": { "my-object": [ "set" ] },
      "uci":  [ "my-config" ],
      "file": { "/etc/my-app/*": [ "read", "write" ],
                "/usr/bin/my-tool reload": [ "exec" ] }
    }
  }
}
```

然后在 `/etc/config/rpcd` 的 login 段里把这个组授给用户：

```
config login
	option username 'operator'
	option password '$p$operator'
	list read 'my-app'
	list write 'my-app'
	list read '!luci-app-firewall'
```

改完必须 `/etc/init.d/rpcd reload`（发 SIGHUP，触发 2.3 的 re-exec）才生效
——ACL 文件只在会话建立时读取，光重启前端没用。

三个务必记住的点：`read` 块和 `write` 块要分别写全（见 2.2 第 1 条）；
`"uci": ["cfg"]` 这种数组写法授出的 function 名就是 `read`/`write` 本身；
`file` 的 exec 授权如果只写裸路径，等于允许任意参数。

### 3.5 写插件

**（推荐）ucode 脚本** —— `/usr/share/rpcd/ucode/my-plugin.uc`：

```javascript
'use strict';
return {
	my_object: {
		get_status: {
			args: { detail: true },
			call: function(request) {
				return { alive: true, detail: request.args?.detail };
			}
		},
		do_thing: {
			call: function(request) {
				if (request.info.acl.user != 'root')
					exit(UBUS_STATUS_PERMISSION_DENIED);
				return { ok: true };
			}
		}
	}
};
```

注意脚本不能是 world-writable，否则 rpcd 直接跳过（`ucode.c:1091-1096`）。

**exec 插件** —— `/usr/libexec/rpcd/my-plugin`（任意可执行文件），要实现两个约定：

```sh
#!/bin/sh
case "$1" in
	list)
		# 输出方法签名：{"方法名": {"参数名": 类型示例}}
		echo '{ "hello": { "name": "str" }, "ping": {} }'
	;;
	call)
		# $2 = 方法名，stdin = 完整请求 JSON，stdout 必须是一个 JSON 对象
		read input
		case "$2" in
			hello) echo '{"greeting":"hi"}' ;;
			ping)  echo '{"pong":true}' ;;
		esac
	;;
esac
```

**.so 插件** —— 导出 `struct rpc_plugin rpc_plugin = { .init = my_init };`，
`my_init(ops, ctx)` 里 `ubus_add_object()`。需要 `rpcd` 的头文件
（`Build/InstallDev` 会把 `include/rpcd` 装到 sysroot）。

---

## 四、可以优化的点

按「架构级 → 性能 → 健壮性 → 代码卫生」排列。带【实测】的在第五章有复现输出。

### 4.1 架构级

**A1. 除 exec 外的所有慢操作都阻塞主循环。**
rpcd 是单线程 uloop，`rpc_exec()` 那套 defer 机制只被 `file.exec`、`rpc-sys` 和
exec 插件使用。而：

- `iwinfo.scan` 调 `iw->scan()`，nl80211 全信道扫描通常 2~5 秒，全程阻塞
  （`iwinfo.c` 所有 handler 都是同步的）。这段时间 LuCI 的其它请求全部排队。
- `rpc_uci_trigger_event()` 用 `ubus_invoke(..., 1000)` 同步通知 `service` 对象，
  超时 1 秒；`uci.apply` 对每个变更的 config 都调一次，N 个 config 最坏阻塞 N 秒。
- `rpc_cgi_password_set()` 里有 `nanosleep(100ms)` + `waitpid()`，同步等
  `/bin/passwd`。
- ucode 脚本的 `call` 在 VM 里同步执行完才返回。

建议：iwinfo 的 scan/survey 走 `rpc_exec()` 或独立 worker；`trigger_event` 改
`ubus_invoke_async`；`password_set` 改用 `rpc_exec()` + `stdin_cb`。

**A2. 全局单例的 apply 状态机应当按会话分离。**
`apply_sid` / `apply_timer` / `apply_ctx` 是文件级静态变量，多管理员同时操作时
第二个 apply 被静默吞掉（5.2 实测）。至少应返回 `UBUS_STATUS_PERMISSION_DENIED`
或 `RESOURCE_BUSY`，理想方案是每会话一个 apply 上下文。

**A3. exec 插件的每次调用都要 `ubus_lookup()` 全表。**

```43:53:components/rpcd/plugin.c
static bool
rpc_plugin_lookup_plugin(struct ubus_context *ctx, struct ubus_object *obj,
                         char *strptr, size_t strsize)
{
	struct rpc_plugin_lookup_context c = { .id = obj->id, .name = strptr, .namelen = strsize };

	if (ubus_lookup(ctx, NULL, rpc_plugin_lookup_plugin_cb, &c))
		return false;

	return c.found;
}
```

仅仅为了从 `obj->id` 反查出插件文件名，就向 ubusd 发一次「枚举所有对象」的同步
请求。正确做法是像 `ucode.c` 那样把 `struct ubus_object` 嵌进自己的容器结构体，
`container_of()` 直接拿到路径——零成本，且顺带解决了这次同步往返带来的阻塞。

**A4. 插件 ABI 没有版本协商。**
`struct rpc_daemon_ops` 是裸结构体，`dlopen` 后直接按偏移访问。建议在
`struct rpc_plugin` 里加 `uint32_t abi_version`（尾部追加，老插件读出 0 即可
判定为 legacy），否则将来任何字段调整都会静默破坏第三方插件。

**A5. 没有插件卸载 / 重载机制。**
`ucode.c` 的注释已经承认了这点。装一个 `rpcd-mod-*` 必须整个 re-exec。考虑到
2.3 的 freeze/thaw 已经把「优雅重启」做得很完整，这条优先级不高，但值得在文档里
说清楚，避免有人期待 `.so` 热插拔。

**A6. 认证侧缺少速率限制。**
`session.login` 每次失败只是返回 `PERMISSION_DENIED`，没有延迟、没有失败计数、
没有源标识（rpcd 根本看不到 HTTP 客户端 IP）。`crypt()` 本身的开销是唯一的减速
手段。密码比较用的还是 `strcmp()`（`session.c:852`），不是常数时间。防暴力破解
目前完全依赖 uhttpd/LuCI 层。

### 4.2 性能

**P1. `rpc_session_grant()` 的去重扫描是 O(n)，登录整体 O(n²)。**
2.1 已分析：外层宏没有前缀失配即退出的条件。一次 `login` 要为 LuCI 的几百条 ACL
逐条 grant，每条都要倒着走一遍已有条目。在小 CPU 的路由器上，登录耗时里相当一部分
花在这里。修法很小：在循环体里判断 `strncmp(acl->avl.key, object, keylen)` 失配
就 `break`。

**P2. `rc.list` 串行 fork，每个脚本 3 秒超时。**【实测】
`rc_list_readdir()` 一次只跑一个 `/etc/init.d/<x> running`，回调里再读下一个。
40 个 init 脚本就是 40 次 fork+exec 串行。我用 5 个会挂住的脚本实测，整整 15 秒：

```
### T8: rc.list forks one child per init script, serially, 3s timeout each
rc list over 5 hanging scripts took 15s (rc=0)
{
	
}
```

而且超时的条目**被静默丢弃**（`rpc_list_exec_timeout_cb` 直接进下一个，不调
`rc_list_add_table()`），所以上面返回的是空对象——调用方无从知道发生了什么。
建议：并发跑（限制并发度）+ 超时条目也返回并带 `"running": null` 之类的标记。

**P3. `file.write` 每次都 `fsync()` + 全局 `sync()`。**

```548:553:components/rpcd/file.c
out:
	if (fsync(fd) < 0)
		rv = -1;

	close(fd);
	sync();
```

`sync()` 刷的是**整个系统所有文件系统**。在 NAND/NOR + overlayfs 的设备上这非常
昂贵，而且 LuCI 上传固件是分片多次 `file.write` 的——每片一次全局 sync。
`fsync(fd)` 已经足够保证该文件落盘，`sync()` 应该去掉或做成可选参数。

**P4. iwinfo 每次调用都 `iwinfo_finish()` 拆后端。**
每个 handler 结尾都调 `rpc_iwinfo_close()`，下次调用重新 `iwinfo_backend()`
打开 netlink socket、重新探测后端类型。LuCI 的无线状态页会连续打好几个 iwinfo
调用，全部要重来一遍。可以做成惰性关闭（空闲 N 秒后再 finish）。

**P5. ucode 每次请求后做一次完整 GC。**
`rpc_ucode_script_call()` 结尾无条件 `ucv_gc(&script->vm)`。对高频轮询型对象
（LuCI 状态页 2 秒一次）可以改成按分配量或按次数触发。

**P6. `rc_list_readdir()` 用递归实现目录遍历。**
`goto next` → 递归调用自己，栈帧里有 `char path[PATH_MAX]`（4 KB）。被跳过的
脚本越多递归越深。主线程 8 MB 栈下不会真出事，但把它改成 `while` 循环是纯收益。

### 4.3 健壮性与安全

**S1. `session.create` 的 timeout 由客户端控制且未校验。**【实测，见 5.1】
`blobmsg_get_u32()` 的结果直接赋给 `int timeout`，`0xFFFFFFFF` 变成 `-1`，
`rpc_touch_session()` 因 `timeout > 0` 不成立而**根本不装定时器**——会话永不过期。
配合 `session.create` 可以无限累积不可回收的会话。虽然 HTTP 侧默认 ACL 不放开
`session.create`，但这属于典型的「输入未校验」，应当 clamp 到 `[0, 上限]`。

**S2. `file.exec` 的 ACL 无法区分参数边界。**【实测，见 5.3】
把 argv 用空格拼成一个字符串再匹配，`["one","two"]` 和 `["one two"]` 得到同一个
ACL 串。更要紧的是**只授裸可执行路径就等于放开任意参数**——`"/bin/sh": ["exec"]`
或 `"/usr/bin/curl": ["exec"]` 这样的 ACL 条目实际等于 root shell。建议 ACL 支持
按 argv 数组匹配，并在文档里显式警告裸路径授权的含义。

**S3. `rpc_check_symlink_access()` 之后仍有 TOCTOU 窗口。**【静态】
流程是 `realpath()` → 查 ACL → `open()/stat()/opendir()`。三步之间路径可能被替换。
攻击前提是攻击者能在 ACL 覆盖的目录里创建/替换符号链接（多为 `/tmp` 下的场景），
条件不算苛刻。彻底的修法是 `O_PATH`/`openat` + `fstat` 拿到 fd 后再基于 fd 操作，
让「检查的对象」和「操作的对象」是同一个 inode。

**S4. `fork()` 子进程 execv 失败后会**继续跑 rpcd 的事件循环**。**【实测，见 5.4】
`sys.c` 和 `uci.c` 里有三处同样的写法：

```419:425:components/rpcd/sys.c
	if (!fork()) {
		/* wait for the RPC call to complete */
		sleep(2);
		return execv(c[0], c);
	}

	return 0;
```

`execv` 失败时返回 −1，这个 `return` 把控制权交回 ubus 分发器，于是 **fork 出来的
子进程变成了第二个 rpcd**，和父进程共用同一个 ubus socket fd，双方都会去读同一条
连接。实测 `uci.reload_config` 在没有 `/sbin/reload_config` 的系统上必然触发。
应改成 `_exit(127)`。同样的模式在 `rpc_sys_upgrade_start`、`rpc_sys_factory`、
`rpc_uci_reload` 里各有一份。

**S5. `file.read` 对 size==0 的文件静默截断。**【实测，见 5.5】
procfs/sysfs 文件 `st_size` 为 0，代码假定 4096 字节够用；而且只调用**一次**
`read()`，短读也不补。实测 `/proc/self/maps` 真实 6010 字节，rpcd 只返回 4004
字节，且没有任何截断标志。应该循环读到 EOF 或 `RPC_FILE_MAX_SIZE`，并在截断时
返回一个 `truncated: true`。

**S6. `uci.configs` 完全不做 ACL 检查。**【实测，见 5.6】
`rpc_uci_configs()` 是唯一一个既不调 `rpc_uci_read_access()` 也不设 savedir 的
handler，任何会话（包括未认证的默认会话）都能拿到 `/etc/config` 下的完整文件列表。
信息量不大，但和其它方法的处理不一致。

**S7. `rpc_session_from_blob()` 不校验反序列化内容。**【静态】
`memcpy(ses->id, blobmsg_data(tb[RPC_DUMP_SID]), RPC_SID_LEN)` 固定拷 32 字节，
不检查该字符串是否真有 32 字符（越界读）；`avl_insert()` 的返回值被忽略，若 sid
与已有会话重复，节点没进树却装了定时器，超时后 `avl_delete()` 一个不在树里的节点
会破坏树结构。文件名走了 `rpc_validate_sid()`，但**文件内容没有**。数据源是
`/var/run/rpcd/sessions`（0700，root 写），所以只是加固项，不是可远程触发的漏洞。

**S8. iwinfo `assoclist` 在「指定 MAC 未找到」时泄漏后端。**【静态】

```592:599:components/rpcd/iwinfo.c
	if (!macaddr)
		blobmsg_close_array(&buf, c);
	else if (!found)
		return UBUS_STATUS_NOT_FOUND;

	ubus_send_reply(ctx, req, buf.head);

	rpc_iwinfo_close();
```

这条 `return` 绕过了 `rpc_iwinfo_close()`，nl80211 后端不被 `iwinfo_finish()`
释放。最近的提交 `2decaec` 正是修 `survey` 里同样形状的泄漏，`assoclist` 这处
被漏掉了。

**S9. 若干小的资源泄漏。**【静态】

| 位置 | 问题 |
|------|------|
| `session.c:352-353` | `rpc_random()` 失败时 `ses` 已 calloc 但未 free |
| `session.c:1448-1451` | `rpc_session_thaw()` 里 `uci_alloc_context()` 失败则 `DIR *d` 泄漏 |
| `plugin.c:435-436` | `rpc_plugin_register_exec()` fork 失败时 `fds[0]`/`fds[1]` 泄漏 |
| `plugin.c:503-504` | `dlsym("rpc_plugin")` 失败时不 `dlclose()` |
| `plugin.c:370-398` | `rpc_plugin_parse_exec()` 在 `!obj`/`!obj_type` 等路径上泄漏 `methods` 及其 strdup 的名字 |
| `sys.c:143-158` | `rpc_cgi_password_set()` 写失败即 return，`fds[1]` 不关、子进程不 reap（僵尸） |
| `ucode.c:762 / 811` | 注册失败路径 `free(uuobj)`，但它已经 `list_add()` 进 `uuobjs` 链表，留下悬垂节点 |

除 `sys.c` 那条外都在启动路径上，影响有限，但 `-Werror` 都开了，这些也该收掉。

**S10. exec 插件在启动时阻塞读取，没有超时。**【静态】
`rpc_plugin_parse_exec()` 用裸 `read(fd, ...)` 循环等子进程输出 JSON。一个卡住的
exec 插件会**无限期挂起 rpcd 的启动**，导致整个 ubus 侧的 uci/session/file 都不
可用。应加超时 + `SIGKILL`。

**S11. `rc.list` 与 ucode 的脚本安全检查不一致。**【静态】
`rc_check_script()` 要求 uid==0 && gid==0 && 可执行 && 非 world-writable；
`ucode.c` 只拒绝 world-writable（不检查属主）；`plugin.c` 扫 `libexec/rpcd` 时
只看 `S_ISREG && S_IXUSR`，属主和权限一概不问。三处应该统一到最严的那套。

**S12. `rpc_exec_lookup()` 的 PATH 搜索在超长路径时会拼错。**【静态】

```76:87:components/rpcd/exec.c
		plen = p - search;

		if ((plen + clen) >= sizeof(path))
			continue;

		strncpy(path, search, plen);
		sprintf(path + plen, "/%s", cmd);

		if (!stat(path, &s) && S_ISREG(s.st_mode))
			return path;

		search = p + 1;
```

`continue` 时没有推进 `search`，下一轮 `plen` 会跨过冒号，把
`"dir1:dir2"` 当成一个目录名。实际触发要求 PATH 里有超长目录，概率极低，但逻辑
是错的。这个函数在 `file.c:846-885` 还有一份**逐字复制**。

### 4.4 代码卫生

**C1. 大量复制粘贴的工具函数。**
`rpc_errno_status()` 有三份（`exec.c` `file.c` `sys.c`），
`rpc_ustream_to_blobmsg()` 两份（`exec.c` `file.c`，签名略有不同），
`ustream_for_each_read_buffer` 宏两份，PATH 搜索两份，
JSON↔blobmsg 转换在 `plugin.c` 和 `ucode.c` 各写了一套（一个走 json-c，一个走
ucode 值）。应当抽出一个 `util.c`/`libutil.a` 由主程序和插件共享。

**C2. 到处是文件级 `static` 缓冲区。**
`static struct blob_buf buf` 每个 .c 一个；`file.c` 还有 `static char *canonpath`
`static char *resolvedpath` `static char cmdstr[1024]`；
`__rpc_check_path()` 甚至返回一个 `static struct blob_attr *tb[]`。单线程下能工作，
但任何嵌套调用都会互相踩，而且直接堵死了将来引入线程或协程的可能。

**C3. `rpc_uci_verify_name()` 禁止选项名含 `-`。**【实测，见 5.8】
`uci` 命令行可以 `uci set foo.bar.some-option=x`，通过 rpcd 却会被拒
（`INVALID_ARGUMENT`），而 section **type** 里的 `-` 是允许的。这个不一致没有注释
说明理由，遇到时很难排查。

**C4. `struct rpc_session_acl::sort_len` 是死字段。**
见 2.1：声明了、用了，就是从来没赋过值。要么赋值（顺带实现 P1 的提前退出），
要么删掉。

**C5. `rpc_login_test_login()` 返回指针却写 `return false;`。**
`session.c:872-873`——语义上是 NULL，能编译过，但类型混淆。

**C6. `sys.c` 的 `packagelist` 只剩 apk 分支。**
注释里还并排列着 opkg 与 apk 的字段对照，代码却只打开
`/lib/apk/db/installed`，找不到就 `UBUS_STATUS_NOT_FOUND`。注释该更新。

**C7. `rpc_uci_trigger_event()` 里的 `strdup`/`free` 是无用功。**
`pkg` 复制出来只是为了立刻 `blobmsg_add_string()` 再释放，直接用 `config` 即可。

---

## 五、已核验的问题

以下每条都在第六章的沙箱里实际跑过，输出为原文粘贴。

### 5.1 客户端可创建永不过期的会话

```
$ ubus call session create '{"timeout":-1}'
{
	"ubus_rpc_session": "36827dc400303c08fa78a3703888f830",
	"timeout": -1,
	"expires": 0,
	"acls": { },
	"data": { }
}
```

`timeout` 原样保存为 −1，`expires` 为 0（`uloop_timeout_remaining64()` 对未装载
的定时器返回 −1，除以 1000 得 0）。对照 `rpc_touch_session()`：

```278:283:components/rpcd/session.c
static void
rpc_touch_session(struct rpc_session *ses)
{
	if (ses->timeout > 0)
		uloop_timeout_set(&ses->t, rpc_session_timeout_ms(ses->timeout));
}
```

定时器从未装载，会话不会被回收。注意代码里已经有一个 `rpc_session_timeout_ms()`
专门处理溢出（注释详细解释了 >24.85 天会翻负导致会话秒删），但**负值/超大值在进入
这个函数之前就已经让定时器完全不装载了**，clamp 的位置偏晚。

### 5.2 rollback 待确认期间的 apply 是静默空操作

```
### T3b: apply while a rollback is pending is a silent no-op
apply#1 rc=0
on disk: 	option value 'changed1'
apply#2 rc=0
on disk after apply#2: 	option value 'changed1'
```

流程：`uci.set value=changed1` → `apply {rollback:true}`（成功，磁盘变成
changed1）→ `uci.set value=changed2` → `apply {}`。第二次 apply **返回 0**，
但磁盘依旧是 changed1。原因在 `rpc_uci_apply()` 的结构：

```1598:1650:components/rpcd/uci.c
	if (!apply_sid[0]) {
		rpc_uci_set_savedir(tb[RPC_T_SESSION]);
		...
		for (i = 0; i < gl.gl_pathc; i++) {
			...
			rpc_uci_apply_config(ctx, config);
		}
		...
	}

	return 0;
```

`apply_sid` 非空时整个函数体被跳过，直接 `return 0`。调用方（LuCI）看到成功，
用户以为配置已生效。带 `rollback:true` 的第二次 apply 至少会返回
`PERMISSION_DENIED`，不带的这条则完全没有反馈。

### 5.3 file.exec 的 ACL 看不见参数边界

ACL 只授了一条：`"/bin/echo one two": ["exec"]`。

```
### T5: file.exec ACL matches the flattened command line
-- two separate params (the intended form):
{ "code": 0, "stdout": "one two\n" }
-- ONE param containing a space (different argv, same ACL string):
{ "code": 0, "stdout": "one two\n" }
-- unauthorized argv:
Command failed: ... (Permission denied)
```

`params: ["one","two"]`（argv 三个元素）和 `params: ["one two"]`（argv 两个元素）
被同一条 ACL 放行，因为二者拼出的字符串都是 `/bin/echo one two`：

```1065:1082:components/rpcd/file.c
		arglen = 2;
		p = cmdstr + sprintf(cmdstr, "%s", executable);

		blobmsg_for_each_attr(cur, arg, rem)
		{
			if (blobmsg_type(cur) != BLOBMSG_TYPE_STRING)
				continue;

			if (arglen == 255 ||
			    p + blobmsg_data_len(cur) >= cmdstr + sizeof(cmdstr))
				return UBUS_STATUS_PERMISSION_DENIED;

			p += sprintf(p, " %s", blobmsg_get_string(cur));
			arglen++;
		}

		if (!rpc_file_access(sid, cmdstr, "exec"))
			return UBUS_STATUS_PERMISSION_DENIED;
```

对 `echo` 无所谓，但对任何按参数区分行为的程序（`iptables`、`opkg`、`curl`）
这个模型都不足以表达安全策略。再加上前面那次「裸路径命中即放行、参数不限」的
检查，`file` 的 exec ACL 实际保护力度取决于 ACL 作者是否理解这两条规则。

### 5.4 execv 失败的 fork 子进程会变成第二个 rpcd

系统里没有 `/sbin/reload_config` 时调用 `uci.reload_config`：

```
### T4: fork-child fallthrough on failing execv (uci.reload_config)
rpcd processes before: 1
rc=0
rpcd processes after : 2
3646214 /tmp/rpcd-build/rpcd -s /tmp/rpcd-test/ubus.sock
3646251 /tmp/rpcd-build/rpcd -s /tmp/rpcd-test/ubus.sock
```

多出来的进程会一直活着——第一轮测试结束、namespace 销毁之后，那个子进程仍然在
宿主机上运行（下一轮测试的 "before" 计数变成了 2，我不得不手动 `pkill`）。它和
父进程共享同一个 ubus 连接 fd，两个进程会争抢同一条 socket 上的消息，是明确的
错误状态。责任代码见 4.3 的 S4，`uci.c:1727-1731`、`sys.c:419-423`、
`sys.c:446-450` 三处同形。

### 5.5 file.read 静默截断 procfs 文件

```
### T10: file.read silently truncates size-0 (procfs/sysfs) files at 4096 bytes
host read of /proc/self/maps: 6010 bytes
rpcd returned: 4004 bytes
```

两个叠加因素：`st_size == 0` 时假定 `RPC_FILE_MIN_SIZE`（4096）；
以及只 `read()` 一次不循环——所以连 4096 都没读满，实际拿到 4004 字节。
调用方完全无法区分「文件就这么大」和「被截断了」。

### 5.6 uci.configs 不检查任何权限

```
### T14: uci.configs performs no ACL check at all
-- called with the unauthenticated default session:
{ "configs": [ "rpcd", "testcfg" ] }
-- called with no session at all:
{ "configs": [ "rpcd", "testcfg" ] }
```

### 5.7 默认会话的 ACL 形状（含 access-group 元 scope）

```
$ ubus call session list '{"ubus_rpc_session":"00000000000000000000000000000000"}'
{
	"ubus_rpc_session": "00000000000000000000000000000000",
	"timeout": 0,
	"expires": 0,
	"acls": {
		"access-group": {
			"unauthenticated": [ "read" ]
		},
		"ubus": {
			"session": [ "access", "login" ]
		}
	},
	"data": { }
}
```

这印证了 2.2 第 4 条：除了 ACL 文件里写的 `ubus/session/{access,login}`，rpcd
自动补了一条 `access-group/unauthenticated/read`。另外默认会话的 `timeout` 是 0，
同样不装定时器——这是设计如此。

顺带记一笔：`session.set` / `session.get` 对**任何合法 sid** 都放行，不做 ACL
检查。默认 sid 是公开常量，所以能连到 ubus 的进程都可以往默认会话里塞数据：

```
### T15: session.get/set are unrestricted for any valid sid
set rc=0
{ "values": { "injected": "by anyone" } }
```

本地 ubus 访问本就等价 root，所以这不构成提权；但如果有人通过 ubusd ACL 把
`session` 对象暴露给非 root 连接，就要意识到 `session.grant` 同样没有自校验。

### 5.8 uci 选项名不能含 `-`，但 section type 可以

```
### T13: option names containing '-' are rejected by rpc_uci_verify_name()
plain rc=0
Command failed: ... "values":{"with-dash":"ok"} (Invalid argument)
-- but a section *type* may contain '-':
{ "section": "cfg02abda" }
add my-type rc=0
```

对应 `rpc_uci_verify_str()` 里那个三目条件：

```199:201:components/rpcd/uci.c
	for (c = name + extended; *c; c++)
		if (!isalnum((unsigned char)*c) && *c != '_' && ((!type && !extended) || *c != '-'))
			break;
```

`type == true` 时才放行 `-`。命令行 `uci` 工具没有这个限制，于是产生了「shell 能
设、rpcd 设不了」的不一致。

### 5.9 rc.list 串行执行且丢弃超时项

见 4.2 P2 的实测输出：5 个挂住的 init 脚本耗时 15 秒（= 5 × 3 秒超时），
返回值是空对象 `{}`，超时的脚本既没出现在结果里也没有任何错误提示。

---

## 六、复现环境

全部在本机完成，未使用任何交叉编译产物。

### 6.1 构建

```bash
# 依赖：/tmp/inst = libubox + blobmsg_json（分析 ubus 时留下的）
#      /tmp/jc   = json-c
#      /tmp/ubus = 已编译的 ubusd / libubus.so / ubus CLI

# 1) 补一个 libuci
mkdir -p /tmp/uci-build && cd /tmp/uci-build
cmake ~/openwrt/openwrt/components/uci -DCMAKE_INSTALL_PREFIX=/tmp/inst \
  -DCMAKE_C_FLAGS="-I/tmp/inst/include -I/tmp/jc/include" \
  -DCMAKE_SHARED_LINKER_FLAGS="-L/tmp/inst/lib -L/tmp/jc/lib" -DBUILD_LUA=OFF
make -j8 && cp libuci.so /tmp/inst/lib/
cp ~/openwrt/openwrt/components/uci/uci{,_config,_blob}.h /tmp/inst/include/

# 2) libubus 头 + 库进同一个 prefix
cp /tmp/ubus/libubus.so /tmp/inst/lib/
cp ~/openwrt/openwrt/components/ubus/{libubus.h,ubusmsg.h,ubus_common.h} /tmp/inst/include/

# 3) rpcd 本体（关掉 iwinfo / ucode，缺 libiwinfo 和 libucode）
mkdir -p /tmp/rpcd-build && cd /tmp/rpcd-build
cmake ~/openwrt/openwrt/components/rpcd \
  -DCMAKE_INSTALL_PREFIX=/tmp/rpcd-root/usr \
  -DIWINFO_SUPPORT=OFF -DUCODE_SUPPORT=OFF \
  -DCMAKE_LIBRARY_PATH="/tmp/inst/lib;/tmp/jc/lib" \
  -DCMAKE_INCLUDE_PATH="/tmp/inst/include;/tmp/jc/include" \
  -DCMAKE_C_FLAGS="-I/tmp/inst/include -I/tmp/jc/include -Wno-error"
make -j8      # → rpcd, file.so, rpcsys.so
```

`-DCMAKE_INSTALL_PREFIX=/tmp/rpcd-root/usr` 是关键：`INSTALL_PREFIX` 被编进
二进制，于是 ACL 目录变成 `/tmp/rpcd-root/usr/share/rpcd/acl.d`、插件目录变成
`/tmp/rpcd-root/usr/lib/rpcd`，不需要碰系统目录。

### 6.2 沙箱

`/var/run/rpcd`、`/etc/config` 是硬编码的，用 user + mount namespace 造出来：

```bash
unshare -Urm --propagation private ./env.sh ./tests.sh
```

`env.sh` 做的事：

```bash
mkdir -p /tmp/rpcd-test/etc/{config,init.d}
cp /etc/passwd /etc/group /tmp/rpcd-test/etc/
mount --bind /tmp/rpcd-test/etc /etc      # 换掉 /etc，拿到可写的 /etc/config
mount -t tmpfs none /var/run              # 给 rpcd 写 sessions/ 和 uci-<sid>/
mkdir -p /var/run/ubus

cat > /etc/config/rpcd <<'EOF'
config rpcd
	option socket /tmp/rpcd-test/ubus.sock
	option timeout 30

config login
	option username 'test'
	option password '$1$abcdefgh$irWbblnpmw.5z7wgBnprh0'   # crypt("test")
	list read '*'
	list write '*'
EOF

# 一个测试用 ACL 组
cat > /tmp/rpcd-root/usr/share/rpcd/acl.d/test.json <<'EOF'
{ "test-group": { "write": {
    "uci": [ "testcfg" ],
    "file": { "/bin/echo one two": ["exec"],
              "/tmp/rpcd-test/*": ["read","write"] } } } }
EOF

/tmp/ubus/ubusd -s /tmp/rpcd-test/ubus.sock &
/tmp/rpcd-build/rpcd -s /tmp/rpcd-test/ubus.sock &
```

启动后 `ubus list` 输出 `file / rc / rpc-sys / session / uci` 五个对象，说明
内建对象和两个 `.so` 插件都正常注册。

### 6.3 关于 `option password ''` 的一个坑

最初我把测试账号的密码留空（想利用「空哈希放行任意密码」的逻辑），结果 login 一直
`Permission denied`。原因是 **uci 不会存储空字符串选项**——`uci show` 里根本没有
`password` 这一行，于是 `rpc_login_test_login()` 在 `if (!ptr.o) continue;` 处跳过
了整个 login 段。改成真实 crypt 哈希后一切正常。这个现象本身也说明：想靠
`option password ''` 做「免密登录」是不成立的，必须显式给一个空哈希以外的值。
