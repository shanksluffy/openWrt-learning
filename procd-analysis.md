# procd 代码分析

分析对象：`components/procd`（约 18600 行 C 代码）
分析日期：2026-08-07

> **前提说明**
>
> 本仓库中的 `components/procd` 就是 **upstream 本身**：`origin` 指向官方仓库
> `https://github.com/openwrt/procd.git`，工作区干净，无本地独有提交（共 837 个提交）。
> HEAD 为 `60fdbf0`（2026-06-17，`jail: chown rootfs overlay dir to mapped root in
> user namespace`）。因此修复的正确路径是直接改本仓库、按 OpenWrt 格式提交
> （`procd: fix ...` / `jail: fix ...` + `git commit -s`），再提 PR 或发到 openwrt-devel。
> **报告缺陷前应先 `git fetch`。**
>
> **验证方式说明。** 第五章分成两部分：
>
> - **5.1 ~ 5.7 是实际编译运行复现的**，不是静态推断。做法是把 libubox 的核心
>   12 个 .c 文件（blob / blobmsg / avl / vlist / runqueue / uloop / ustream / …）
>   单独编成静态库，再把 procd 里出问题的函数**逐字照抄**成独立 PoC，用
>   `-fsanitize=address` 编译运行。文中的 ASan 报告和程序输出都是原文粘贴。
> - **5.8 ~ 5.20 是逐行核对的静态结论**，未做运行时复现，每条都注明了确信度和
>   触发前提。
>
> 第五章后半段的 ujail 部分（5.15 ~ 5.20）最初由子任务扫描得出，**文中收录的每一条
> 我都回到源码逐行复核过**；子任务报告里另有几条（`watchdog_set_cloexec` 的
> "未初始化变量"等）经复核为误报，已剔除，不在本文中。

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

`CMakeLists.txt` 切出的产物比表面上看起来多。procd 不是一个二进制，而是**一组**
围绕同一套 libubox 事件循环组织起来的程序：

| 产物 | 源文件 | 安装位置 | 角色 |
|------|--------|----------|------|
| `procd` | `procd.c signal.c state.c hotplug-dispatch.c inittab.c rcS.c ubus.c system.c sysupgrade.c service/* utils/*` (+ `watchdog.c plug/*`) | `/sbin/procd` | 主守护进程 |
| `init` | `initd/init.c initd/early.c initd/preinit.c initd/mkdev.c sysupgrade.c watchdog.c` | `/sbin/init` | PID 1，只负责把系统拉到能跑 procd 的状态 |
| `ujail` | `jail/jail.c jail/cgroups.c jail/cgroups-bpf.c jail/elf.c jail/fs.c jail/capabilities.c jail/netifd.c jail/seccomp-oci.c` | `/sbin/ujail` | 容器/沙箱启动器（6808 行，占全仓 1/3） |
| `uxc` | `uxc.c` | `/sbin/uxc` | OCI 容器管理 CLI |
| `utrace` | `trace/trace.c` | `/sbin/utrace` | 用 ptrace+seccomp 生成 seccomp 白名单 |
| `udevtrigger` | `plug/udevtrigger.c` | `/sbin/udevtrigger` | coldplug：重放 `/sys` 里已有设备的 uevent |
| `askfirst` | `utils/askfirst.c` | `/sbin/askfirst` | inittab `askfirst` 动作的"按回车启动"包装 |
| `libpreload-seccomp.so` | `jail/preload.c jail/seccomp.c jail/seccomp-oci.c` | — | LD_PRELOAD，在目标进程自己的地址空间里装 seccomp |
| `libpreload-trace.so` | `trace/preload.c` | — | LD_PRELOAD，配合 utrace |
| `libsetlbf.so` | `service/setlbf.c` | — | LD_PRELOAD，把 stdout 改成行缓冲 |
| `upgraded` | `upgraded/upgraded.c` | `/sbin/upgraded` | sysupgrade 期间接管 PID 1 |

行数分布：`jail/` 6808、根目录 4648、`service/` 3614、`plug/` 986、`initd/` 562、
`trace/` 529、`utils/` 295、`upgraded/` 104。

值得注意的是 **`init` 和 `procd` 是两个独立的可执行文件**，`init` 里没有 ubus、
没有服务管理，只有 mount + fork + exec。这个切分是刻意的，理由见 2.1。

### 1.2 主守护进程的模块划分

```
procd.c          main()、参数解析、决定自己是 procd / plugd / hotplug 处理器
  state.c        7 状态的启动/关机状态机（唯一的"主控流程"）
  signal.c       信号 → 状态机事件的翻译层
  ubus.c         ubus 连接 + 指数退避重连 + udebug 接入
  system.c       ubus `system` 对象：board/info/reboot/watchdog/signal/sysupgrade
  watchdog.c     /dev/watchdog 定时喂狗，fd 通过 WDTFD 环境变量跨 exec 传递
  rcS.c          按 glob 顺序跑 /etc/rc.d/S* 和 K*
  inittab.c      解析 /etc/inittab
  sysupgrade.c   fork upgraded、chroot 到新 rootfs
  hotplug-dispatch.c   ubus `hotplug.<subsys>` 对象，把 uevent 转发给订阅者
  plug/hotplug.c       netlink uevent 接收 + json_script 规则引擎
  plug/coldplug.c      开机时用 udevtrigger 重放已有设备
  service/
    service.c    服务注册表（AVL 树）、ubus `service` 对象
    instance.c   实例生命周期：配置解析 → fork/exec → 监督 → respawn（1820 行，最核心）
    trigger.c    JSON 脚本触发器规则引擎
    validate.c   配置项校验规则表
    watch.c      监视其它 ubus 对象出现/消失
  utils/utils.c  控制台探测、/proc/cmdline 解析、fd 重定向
```

### 1.3 三个层次的数据结构

procd 的服务模型是三层的，理解这三层是读懂 `service/` 的前提：

```
struct service                    /* AVL 树，key = 服务名，如 "network" */
 ├─ name, avl node
 ├─ struct vlist_tree instances   /* 该服务的所有实例 */
 ├─ struct blob_attr *trigger     /* 触发器定义 */
 ├─ struct list_head validators   /* 校验规则 */
 └─ bool deleted

struct service_instance           /* vlist 节点，key = 实例名，如 "instance1" */
 ├─ struct service *srv
 ├─ struct blob_attr *config      /* 原始配置，用于比较是否变化 */
 ├─ struct blob_attr *command     /* argv */
 ├─ pid_t proc.pid
 ├─ struct uloop_process proc     /* 进程退出回调 */
 ├─ struct uloop_timeout timeout  /* respawn 延时 / term 超时 */
 ├─ struct ustream_fd _stdout, _stderr   /* 转发到 syslog */
 ├─ respawn_threshold / _timeout / _retry
 └─ uid / pw_gid / gr_gid / has_jail / jail{...}

struct instance_data              /* 挂在实例上的 env / limits / jail mount 等表 */
```

`vlist_tree` 是这里的关键：**它带有"版本号"语义**。`vlist_update()` 开始一轮更新，
逐个 `vlist_add()`，最后 `vlist_flush()`。libubox 会自动调用 update 回调，告诉你
哪些节点是新增（`node_old == NULL`）、哪些是删除（`node_new == NULL`）、哪些是
更新（两者都在）。procd 的"配置变了就重启实例、没变就不动"整套逻辑就建立在这上面
（`service_instance_update()`，`instance.c`）。

### 1.4 启动状态机

`state.c` 里的 7 个状态是 procd 唯一的全局控制流：

```
STATE_EARLY ──► STATE_UBUS ──► STATE_INIT ──► STATE_RUNNING
                                                    │
                                                    ▼ (SIGTERM/SIGUSR1/SIGUSR2/SIGPWR)
                                              STATE_SHUTDOWN
                                                    │
                                                    ▼
                                              STATE_HALT ──► reboot(2)
```

每个状态的实际动作（`state_enter()`）：

| 状态 | 动作 |
|------|------|
| `STATE_EARLY` | 挂载 `/`、跑 `/etc/rc.d/S*` 之前的准备；`procd_coldplug()` 重放设备 |
| `STATE_UBUS` | 设置控制台、启动 `ubusd`、`ubus_init()` 注册 `service`/`system`/`hotplug.*` 对象 |
| `STATE_INIT` | 解析 `/etc/inittab`、跑 `rcS("S", ...)` |
| `STATE_RUNNING` | 什么都不做，纯事件驱动 |
| `STATE_SHUTDOWN` | `rcS("K", ...)`，然后 `service_stop_all()` |
| `STATE_HALT` | `perform_halt()`：SIGTERM → 睡 1s → SIGKILL → sync → `reboot()` |

状态推进靠 `procd_state_next()`，它是**单向**的——没有回退路径。rcS 的 runqueue
跑空时调用 `procd_state_next()`，这就是"脚本跑完自动进下一个状态"的机制。
**这也意味着 rcS 一旦卡住，整个启动就停在那里**（见 5.2）。

---

## 二、设计原理

### 2.1 为什么 `init` 和 `procd` 要分成两个二进制

PID 1 有两个特殊性：它不能退出（否则内核 panic），并且它在 rootfs 还没准备好时就
已经在跑了。`initd/init.c` 的职责被压缩到极限：

```c
/* initd/init.c，主流程 */
early();                    /* mount /proc /sys /dev /tmp，mknod /dev/null */
watchdog_init(1);           /* 尽早接管硬件看门狗，防止后面卡住导致复位 */
fork + exec kmodloader      /* 加载内核模块，最多等 12 秒 */
selinux_init()              /* 可选 */
preinit();                  /* 跑 /etc/preinit，并起一个 procd -h 做 hotplug */
...
execvp("/sbin/procd")       /* 用 procd 替换掉自己，procd 成为 PID 1 */
```

关键点在最后一步：`init` **不是 fork procd，而是 exec 它**。所以运行时 PID 1 是
procd 本体，`init` 只是一段一次性的引导代码。这样做的好处是 procd 不必在自己的
代码里区分"我在极早期"和"我在正常运行"两种环境——极早期的脏活已经由另一个二进制
干完了。

`watchdog_init()` 放在这么早、并且看门狗 fd 通过 **`WDTFD` 环境变量**传给 exec 后的
procd（`watchdog.c`），是为了让整条引导链上任何一步卡死都能被硬件复位救回来。这是
嵌入式设备特有的考虑。

### 2.2 为什么 ubus 连接是懒惰且可选的

`procd.c` 里：

```c
if (getpid() != 1)
    procd_connect_ubus();
```

PID 1 的 procd **不在启动时连 ubus**，而是等状态机走到 `STATE_UBUS` 时自己先把
`ubusd` 拉起来再连。这是个鸡生蛋问题：ubusd 本身是 procd 管理的服务。

连接失败后的重试用指数退避（`ubus.c`）：

```c
static void ubus_reconnect_timer(struct uloop_timeout *timeout)
{
    static int retry_ms = 50;
    ...
    if (retry_ms < 1000)
        retry_ms *= 2;
}
```

50ms 起步、翻倍、封顶 1000ms。上限存在的意义是：ubusd 崩溃重启后 procd 最迟 1 秒
内恢复，而不会在 ubusd 长期不可用时把 CPU 烧掉。

### 2.3 实例监督模型：为什么是 respawn 而不是 restart

procd 不提供无条件重启。`instance_exit()` 的逻辑是**带崩溃循环检测的** respawn：

```c
if (runtime < in->respawn_threshold)
    in->respawn_count++;        /* 存活时间太短，计入连续失败 */
else
    in->respawn_count = 0;      /* 活够久了，清零 */

if (in->respawn_count > in->respawn_retry && in->respawn_retry > 0) {
    LOG("Instance %s::%s s in a crash loop ...");
    in->restart = in->respawn = 0;
    service_event_instance_exit("instance.fail", in);   /* 放弃，发事件 */
} else {
    service_event_instance_exit("instance.respawn", in);
    uloop_timeout_set(&in->timeout, in->respawn_timeout * 1000);
}
```

三个参数 `threshold / timeout / retry` 默认是 `3600 / 5 / 5`：**存活不足 1 小时算
一次失败，连续 5 次就放弃，每次间隔 5 秒**。放弃时不是静默死掉，而是发出
`instance.fail` ubus 事件，让上层（比如 LuCI 或监控脚本）能感知。

这个设计的取舍很明确：**宁可让一个服务停摆，也不让它把整机 CPU 拖死**。嵌入式设备
上一个 busy-loop 崩溃重启的服务足以让整台路由器失去响应。

### 2.4 触发器：把"配置变了要重启谁"变成数据

procd 最有特色的设计是 trigger。传统 init 系统里"网卡起来了要重启 dnsmasq"这类
依赖写在 shell 里；procd 把它做成了一套 **JSON 表达式**，由 libubox 的
`json_script` 引擎求值：

```json
[ "if",
  [ "and", [ "eq", "SUBSYSTEM", "button" ], [ "has", "BUTTON" ] ],
  [ "button", "/etc/rc.button/%BUTTON%" ]
]
```

引擎本身在 libubox 里（`json_script.c`），procd 只负责注册**处理器**（handler）：
`exec`、`run_script`、`button`、`makedev`、`rm`、`load-firmware`、`start-console`、
`return`。同一套引擎被用在两个地方：

- `/etc/hotplug.json` —— 内核 uevent 的处理规则（`plug/hotplug.c`）
- 服务自己声明的 `triggers` —— 配置变更/接口事件的响应（`service/trigger.c`）

好处是规则可以被 ubus 动态下发、可以被非 C 代码生成（`procd.sh` 就是这么干的），
而且求值过程不需要 fork shell。代价是可读性差，并且**规则引擎的输入是不可信的
blobmsg，类型校验必须由 handler 自己做**——5.6 就是这里出的问题。

### 2.5 sysupgrade：为什么要换一个 PID 1

固件升级时要擦掉正在运行的 rootfs，这意味着 procd 自己的可执行文件、所有共享库都
会消失。procd 的解法（`sysupgrade.c`）是把 `upgraded` 这个**静态、极小（104 行）**
的程序拉起来接管：

```
procd: service_stop_all()          /* 停掉所有服务 */
procd: watchdog_set_cloexec(false) /* 看门狗 fd 要传下去 */
procd: setenv("WDTFD", ...)
procd: chdir("/")  →  chroot(prefix)   /* 切到新 rootfs */
procd: execvp("/sbin/upgraded", ...)
upgraded: 写固件、kill 剩余进程、umount、reboot()
```

注意 `chroot` 之前的 `chdir` 保留了一个"逃生舱"：如果 `execvp` 失败，还能用
`chroot(".")` 退回去。但这条恢复路径实际上是**不完整的**——`service_stop_all()`
已经把所有服务停了，退回去也只剩一个空壳 procd，见 5.14 的讨论。

### 2.6 ujail 的三阶段握手

`ujail` 的父子进程用**两条单向管道**做同步，交换三个字节。这个协议不显眼但是理解
容器启动顺序的钥匙：

| 时机 | 方向 | 字节 | 含义 |
|------|------|------|------|
| 子进程 `setns` 完成后 | 子 → 父 | `'i'` | 我活着了，你可以写 uid_map / 挂 cgroup 了 |
| 父写完 map/cgroup + 跑完 `createRuntime` hook | 父 → 子 | `'O'` | 你可以开始做挂载和 pivot_root 了 |
| 父收到 ubus `start`（或 `-i`） | 父 → 子 | `'!'` | 跑 `startContainer` hook 然后 exec |

为什么必须这样：在新 user namespace 里，子进程在父进程写 `/proc/<pid>/uid_map`
之前是 "nobody"，什么都干不了；而父进程必须先知道子进程已经进了 namespace 才能写。
这个先后关系无法用 `clone()` 的返回值表达，只能靠管道。

同理，`clone()` 的 flags 里**故意不含 `CLONE_NEWCGROUP`**：

```c
clone(exec_jail, ..., SIGCHLD | (opts.namespace & ~CLONE_NEWCGROUP), ...)
```

父进程要在宿主机的 cgroup 层级里把子进程 `cgroups_apply()` 进去；如果子进程一开始
就在新的 cgroup namespace 里，宿主机的控制器就被隐藏了。所以由子进程在收到 `'O'`
之后自己 `unshare(CLONE_NEWCGROUP)`。

### 2.7 权限剥离的顺序

`jail/jail.c` 的 `post_start_hook()` 里，最终降权顺序是精心排过的：

```
1. PR_SET_SECUREBITS(SECBIT_NO_SETUID_FIXUP)   /* 阻止 setuid 时内核自动清 caps */
2. applyOCIcapabilities(保留 SETGID|SETUID|SETPCAP)
3. initgroups() → setregid() → setreuid()
4. setgroups(additional_gids)                  /* 此时 SETGID 还在手上 */
5. 恢复并锁定 securebits
6. applyOCIcapabilities(保留 0)                /* 最终丢弃 */
7. PR_SET_NO_NEW_PRIVS
8. seccomp 过滤器
9. execve
```

`SECBIT_NO_SETUID_FIXUP` 是必须的：默认情况下 `setuid()` 到非 0 uid 会让内核清空
permitted/effective capability 集，那样第 4 步的 `setgroups` 就会失败。

**对比：非 jail 的普通服务实例走的是完全不同的一段代码**（`instance.c:546-557`），
只有 `initgroups → setgid → setuid` 三步，没有 capability 处理，也**不支持
`no_new_privs`**（这个参数只在 jail 路径下传给 ujail）。这个不对称是 5.9 的背景。

---

## 三、使用方法

### 3.1 init 脚本的形态

绝大多数使用者接触 procd 是通过 `/etc/init.d/` 下的脚本。最小骨架：

```sh
#!/bin/sh /etc/rc.common

USE_PROCD=1
START=95
STOP=10

start_service() {
    procd_open_instance
    procd_set_param command /usr/sbin/mydaemon -f
    procd_set_param respawn 3600 5 5
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_set_param file /etc/config/mydaemon
    procd_close_instance
}

service_triggers() {
    procd_add_reload_trigger "mydaemon"
}
```

`USE_PROCD=1` 让 `/etc/rc.common` 加载 `/lib/functions/procd.sh`，把
`procd_*` 一族函数注入进来。整个脚本**不直接和 procd 通信**——它构造一段 JSON，
最后由 `procd_close_service` 通过一次 `ubus call service set` 发出去。

### 3.2 `procd.sh` 做了什么

`package/system/procd/files/procd.sh`（714 行）本质上是一个 JSON 构造器。核心机制
在 `_procd_wrapper`：

```sh
_procd_wrapper() {
    procd_lock
    while [ -n "$1" ]; do
        eval "$1() { _procd_call _$1 \"\$@\"; }"
        shift
    done
}
```

它为每个 `procd_xxx` 生成一个包装函数，包装函数负责切换 jshn 的命名空间（避免和
调用者自己的 JSON 变量打架）再转调 `_procd_xxx`。`procd_lock` 用 flock 保证同一个
服务的多个 init 脚本调用不会交错。

`procd_set_param` 是入口，它按参数类型分派：

```sh
env|data|limits)                     → JSON 对象（表）
command|netdev|file|respawn|watch|watchdog)  → JSON 数组
nice|term_timeout)                   → JSON 整数
reload_signal)                       → json_add_int $(kill -l "$1")
pidfile|user|group|seccomp|capabilities|facility|extroot|...)  → JSON 字符串
stdout|stderr|no_new_privs)          → JSON 布尔
```

**注意 `respawn` 走的是 `_procd_add_array` → `json_add_string`，即三个值都是
字符串。** 这一点在 5.1 里很重要：shell 这条路径不会触发那个越界，只有绕过
`procd.sh` 直接发 ubus 请求才会。

### 3.3 ubus 接口

`ubus -v list service` 能看到全部方法。常用的：

| 对象.方法 | 用途 |
|-----------|------|
| `service set` | 注册/更新一个服务及其所有实例（init 脚本用的就是这个） |
| `service add` | 同 set，但不删除已有实例 |
| `service list` | 列出所有服务、实例、pid、运行时长 |
| `service delete` | 删除服务或单个实例 |
| `service signal` | 给实例发信号 |
| `service state` | 查询/设置服务状态 |
| `service event` | 手动投递一个事件给触发器引擎 |
| `service validate` | 查询已注册的配置校验规则 |
| `service get_data` | 取出实例声明的 `data` 表 |
| `system board` | 型号、内核版本、`/usr/lib/os-release` 内容 |
| `system info` | 运行时长、负载、内存、文件系统占用 |
| `system reboot` | 重启 |
| `system watchdog` | 查询/设置看门狗超时和喂狗频率 |
| `system signal` | 给**任意 pid** 发信号（见 5.8） |
| `system validate_firmware_image` | 校验固件镜像 |
| `system sysupgrade` | 执行升级 |
| `hotplug.<subsys> call` | 触发某个子系统的 hotplug 脚本 |

手工注册一个服务（不经过 init 脚本）：

```sh
ubus call service set '{
  "name": "demo",
  "instances": {
    "main": {
      "command": [ "/bin/sleep", "3600" ],
      "respawn": [ "3600", "5", "5" ],
      "stdout": true
    }
  }
}'
```

### 3.4 容器：uxc

```sh
uxc create mycontainer /path/to/oci/bundle    # 注册 OCI bundle
uxc start mycontainer
uxc list
uxc state mycontainer                          # OCI 风格的状态查询
uxc kill mycontainer SIGTERM
uxc delete mycontainer
```

`uxc` 把 bundle 记录在 `/etc/uxc/` 下，实际启动时是通过 `ubus call service set`
生成一个带 `jail` 参数的普通 procd 实例，再由 procd 调 `ujail -J <bundle>`。
所以**容器在 procd 眼里就是一个服务实例**，`service list` 里能看到它。

### 3.5 生成 seccomp 白名单

```sh
# 在 init 脚本里临时打开
procd_set_param seccomp /etc/seccomp/mydaemon.json

# 或者先用 utrace 录一遍
utrace /usr/sbin/mydaemon -f
```

`utrace` 用 ptrace 跑一遍目标程序，记录用到的全部系统调用号，输出可直接使用的
JSON 白名单。

### 3.6 调试

```sh
export PROCD_DEBUG=1            # procd.sh 会把生成的 JSON 打到 stderr
procd -d 4                      # procd 自身的调试级别（1-4）
ubus monitor                    # 看所有 ubus 流量
logread -f                      # 服务的 stdout/stderr（需要 procd_set_param stdout 1）
```

---

## 四、可以优化的点

### 4.1 `service list` 每次都重建整棵 blob 树

`service_handle_list()` 对每个服务、每个实例调用 `instance_dump()`，后者会把 argv、
env、limits、jail 配置全部重新序列化一遍。在实例较多的系统上（一台跑满的路由器
可以有 40+ 实例），LuCI 的状态页每次刷新都会触发一次完整重建。

配置本身存在 `in->config` 里且**只在配置变化时才会变**。可以缓存序列化结果，用
`instance_config_changed()` 已有的比较逻辑做失效判断。

### 4.2 全局 `blob_buf b` 同时用于解析和构造

`service.c` 里的文件级 `static struct blob_buf b` 既用来接收 ubus 请求的解析结果，
又用来构造回复。`service_start_early()` 这类路径上会出现"解析完之后复用同一个
buffer 构造消息"的写法。目前没有实际 bug（因为解析结果已经拷走了），但这是个
很容易在后续改动中踩到的陷阱。建议拆成 `parse_buf` / `reply_buf` 两个。

### 4.3 `find_syscall` 线性扫描

`jail/seccomp-syscalls-helpers.h` 里按名字查系统调用号是 O(n) 线性扫描，而 OCI
seccomp profile 动辄几百条规则，等于每次启动容器做几万次 `strcmp`。生成一次静态
哈希表（syscall 名字集合在编译期就固定了，`syscall-names.h` 是构建时生成的）可以
把这块降到 O(1)。

### 4.4 OCI seccomp 被解析两遍

`jail/seccomp-oci.c` 先遍历一遍算 BPF 指令条数、分配数组，再遍历一遍生成指令
（约 242-281 行和 301-402 行）。两遍之间如果有任何计数不一致就会写越界或跳转错位。
改成单遍 + 可增长 buffer，既省 CPU 又消除了两遍不一致的整类风险。

### 4.5 `build_envp` 的 64 槽静态数组

`jail/jail.c:969` 的 `MAX_ENVP 64` 是硬上限，超了就截断（而且截断本身有 bug，见
5.18）。环境变量条数在 `parseOCIenvarray` 里已经数过了，直接按实际条数
`calloc` 更简单也更安全。

### 4.6 每个依赖库一次 bind mount

`jail/fs.c` 的 `mount_all()` 会为 ELF 依赖闭包里的每一个 `.so` 单独做一次 bind
mount。一个链接了 libubox+libubus+libblobmsg+libjson-c 的服务大约 8-12 次挂载，
每次都是一个 syscall + 一个 AVL 节点。对依赖多的容器可以考虑合并挂载
`/lib` + `/usr/lib`（当然这会放宽隔离边界，需要按威胁模型取舍）。

### 4.7 hook 里嵌套 `uloop_run()`

`jail/jail.c` 的 `run_hooklist()` 在已经处于 uloop 回调里的情况下又调
`uloop_run()`（第 506 行），hook 超时后再嵌套一层（511 行）。嵌套事件循环的重入
语义很难推理，5.17 那个 bug 的存在很大程度上就是因为控制流已经绕不清楚了。改成
显式状态机会好很多。

### 4.8 `instance_config_parse` 的 blobmsg 类型检查不成体系

同一个函数里，`INSTANCE_ATTR_WATCH` 有类型检查：

```c
blobmsg_for_each_attr(cur2, tb[INSTANCE_ATTR_WATCH], rem) {
    if (blobmsg_type(cur2) != BLOBMSG_TYPE_STRING)
        continue;
    ...
}
```

而 `INSTANCE_ATTR_RESPAWN` 和 `INSTANCE_ATTR_WATCHDOG` 直接 `blobmsg_get_string()`。
`blobmsg_get_string()` 在 libubox 里只是一个 cast，不做任何校验。建议加一个
`blobmsg_check_array()` 式的统一入口，在 policy 层就把数组元素类型定死。

### 4.9 `rcS` 的错误信息丢失

`rcS.c` 里脚本的 stdout 被逐行转发到 syslog，但**脚本的退出码只在 debug 级别打印**。
一个 `S20network` 返回 1 在默认日志级别下是完全静默的。把非零退出码升到 `LOG()`
级别是个一行改动，对现场排障帮助很大。

---

## 五、已核验的潜在 bug

> **5.1 ~ 5.7 是实际编译运行复现的**，附原始 ASan 报告 / 程序输出。
> **5.8 ~ 5.20 是逐行核对的静态结论**，未做运行时复现，逐条注明确信度。

### 5.1 【高】respawn 参数解析栈越界写，可通过一次 ubus 调用触发

**位置**：`service/instance.c:1318-1333`

```c
if (tb[INSTANCE_ATTR_RESPAWN]) {
    int i = 0;
    uint32_t vals[3] = { 3600, 5, 5};

    blobmsg_for_each_attr(cur2, tb[INSTANCE_ATTR_RESPAWN], rem) {
        if ((i >= 3) && (blobmsg_type(cur2) == BLOBMSG_TYPE_STRING))
            continue;
        vals[i] = atoi(blobmsg_get_string(cur2));
        i++;
    }
```

边界检查写错了。`i >= 3` 后面**又加了一个 `&& 类型是字符串`** 的条件——本意大概是
"跳过多余的字符串项"，但实际效果是：**只要数组第 4 个及以后的元素不是字符串，
`i >= 3` 这个保护就完全失效**，`vals[3]`、`vals[4]`… 一路写下去，而 `vals` 只有
3 个元素。

同一个函数里 66 行之外的 watchdog 解析器（`instance.c:1438-1448`）写法是对的：

```c
blobmsg_for_each_attr(cur2, tb[INSTANCE_ATTR_WATCHDOG], rem) {
    if (i >= 2)
        break;                      /* 无条件、且是 break 不是 continue */
    vals[i] = atoi(blobmsg_get_string(cur2));
    i++;
}
```

两处并排放着，可以确认 respawn 那处是笔误而非设计意图。

**触发条件**：`respawn` 数组里有 4 个以上非字符串元素。JSON 里的数字会被
`blobmsg_add_json_element()` 映射成 `BLOBMSG_TYPE_INT32`，所以

```sh
ubus call service set '{"name":"x","instances":{"i":{"command":["/bin/true"],
  "respawn":[1,2,3,4,5,6,7,8]}}}'
```

就足够了——**注意这里 respawn 写的是数字不是字符串**。走 `procd.sh` 的正常
init 脚本不会触发（3.2 里说过 `_procd_add_array` 生成的全是字符串），必须直接
发 ubus 请求。

**实测结果**（把这段循环逐字抄出来，喂真实 blobmsg，ASan 编译）：

```
== case B: respawn as INT32 ("respawn":[1,2,3,4,5,6,7,8]) ==
  iteration i=0  type=5  -> writing vals[0]
  iteration i=1  type=5  -> writing vals[1]
  iteration i=2  type=5  -> writing vals[2]
  iteration i=3  type=5  -> writing vals[3]   <<< OUT OF BOUNDS
=================================================================
==1929389==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7f25f0c0926c
WRITE of size 4 at 0x7f25f0c0926c thread T0
    #0 0x401a72 in parse_respawn /tmp/pv/poc1_respawn.c:29
    #1 0x401dbe in main /tmp/pv/poc1_respawn.c:61

Address 0x7f25f0c0926c is located in stack of thread T0 at offset 44 in frame
  This frame has 1 object(s):
    [32, 44) 'vals' (line 22) <== Memory access at offset 44 overflows this variable
SUMMARY: AddressSanitizer: stack-buffer-overflow in parse_respawn
```

对照组（同样 8 个元素但都是字符串）不越界，证明确实是类型判断导致保护失效。

**影响**：`instance_config_parse()` 运行在 **procd 主进程（PID 1，root）** 里。
攻击者能控制越界写的**数量**（数组长度）和**内容**（`atoi` 的结果，即任意
32 位整数）。写入位置是 `instance_config_parse` 的栈帧，相邻的是同函数内的局部
变量和保存的寄存器。**procd 崩溃 = PID 1 崩溃 = 内核 panic**。

`vals[i] = atoi(blobmsg_get_string(cur2))` 这一句还有第二重问题：对 INT32 元素调
`blobmsg_get_string()` 拿到的是 4 字节裸整数而不是 NUL 结尾的字符串，`atoi` 会
一直读到碰巧出现的 0 字节为止，构成一次越界读。

**修法**：把条件改成和 watchdog 那处一致：

```c
if (i >= 3)
    break;
if (blobmsg_type(cur2) != BLOBMSG_TYPE_STRING)
    continue;
```

---

### 5.2 【高】rcS 收到不带换行的长输出会永久卡住启动

**位置**：`rcS.c:49-70`

```c
static void pipe_cb(struct ustream *s, int bytes)
{
    char *newline, *str;
    int len;

    do {
        str = ustream_get_read_buf(s, NULL);
        if (!str)
            break;
        newline = strchr(str, '\n');
        if (!newline)
            break;                    /* 没有换行 → 一个字节都不消费 */
        *newline = 0;
        len = newline + 1 - str;
        ulog(LOG_INFO, "%s\n", str);
        ustream_consume(s, len);
    } while (1);
}
```

libubox 的 ustream 读缓冲默认 `max_buffers = 1`、`buffer_len = 4096`。一旦攒够
4096 字节还没出现换行，`pipe_cb` 就一个字节都不消费；ustream 发现缓冲满了会停止
poll 那个 fd；于是 init 脚本写 pipe 时被永久阻塞。

而 `rcS()` 用的 runqueue 是 `max_running_tasks = 1`，并且 runqueue 跑空的回调
就是 `procd_state_next()`——**这个脚本不结束，后面的脚本不会跑，状态机也不会推进**。

`service/instance.c:722-738` 里处理服务 stdout 的同类函数写法是对的：

```c
newline = memchr(str, '\n', len);
if (!newline && (s->r.buffer_len != len))    /* 缓冲满了就先冲出去 */
    break;
```

**实测结果**（真实 pipe + 真实 ustream + 真实 uloop，向 pipe 写 64KB 无换行数据）：

```
rcS.c pipe_cb:         written=65536  consumed=0      callbacks=1   buffer_len=4096
                       writer would BLOCK FOREVER -> boot hangs

instance.c stdio:      written=65536  consumed=65536  callbacks=16  buffer_len=4096
                       writer would make progress
```

两者差异干净利落：同样的输入，rcS 的版本消费 0 字节。

**影响**：任何 `/etc/rc.d/S*` 脚本只要输出超过 4096 字节而中间没有换行，系统就停在
`STATE_INIT` 永不进入 `STATE_RUNNING`。这不需要恶意构造——`base64` 一个文件、
`printf` 一个长 hex dump、某些程序的进度条（用 `\r` 不用 `\n`）都能做到。看门狗
最终会把机器复位，形成**启动 → 卡住 → 复位**的循环。

顺带一提，同一个文件里 `rcS.c:80-87` 还有一处 fd 泄漏：`pipe()` 成功之后
`fork()` 如果失败，两个 pipe fd 都没关。

**修法**：照抄 `instance.c` 的判断，缓冲满时冲出部分行。

---

### 5.3 【中】`/etc/inittab` 里的空行导致越界读

**位置**：`inittab.c:333-337`

```c
int len = strlen(line);
while (isspace(line[len - 1]))
    len--;
line[len] = 0;
```

去尾部空白的循环**没有下界**。`fgets` 读到一个空行时 `line` 是 `"\n"`，`len` 为 1，
第一轮 `line[0]` 是 `\n`（是空白）→ `len` 变 0 → 第二轮读 `line[-1]`，即缓冲区
前面一个字节。如果那个字节恰好也是空白字符，`len` 继续变负，最后 `line[len] = 0`
往缓冲区**前面**写。

**实测结果**（把这段逐字抄出来，`line` 放在堆上，ASan）：

```
line = "\n", strlen = 1
stripping trailing whitespace exactly as inittab.c does...
=================================================================
==1929919==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x50c00000003f
READ of size 1 at 0x50c00000003f thread T0
    #0 0x40131d in main /tmp/pv/poc3_inittab.c:36

0x50c00000003f is located 1 bytes before 128-byte region [0x50c000000040,0x50c0000000c0)
SUMMARY: AddressSanitizer: heap-buffer-overflow
```

**影响**：`line` 在真实代码里是栈上数组，所以越界读的是栈上的相邻数据。要走到
"越界写"需要前面的字节恰好是空白，概率不高但不是零。触发前提是
`/etc/inittab` 里有空行——**这在手工编辑过的 inittab 里相当常见**。严重性受限于
inittab 是 root 才能写的文件，所以这是健壮性问题而非提权路径。

**修法**：`while (len > 0 && isspace(line[len - 1]))`。

---

### 5.4 【中】`askfirst` / `askconsole` 移位 argv 时丢掉 NULL 终止符

**位置**：`inittab.c:170-176` 和 `inittab.c:220-226`

解析阶段最多填 7 个 argv 槽，NULL 放在 `argv[7]`（`inittab.c:353-357`）：

```c
for (i = 0; i < (MAX_ARGS - 1) && tok; i++) {   /* MAX_ARGS = 8 */
    a->argv[i] = tok;
    tok = strtok(NULL, " ");
}
a->argv[i] = NULL;
```

`askfirst()` 要在最前面插入 `/sbin/askfirst`，于是整体右移一格：

```c
for (i = MAX_ARGS - 1; i >= 1; i--)
    a->argv[i] = a->argv[i - 1];
a->argv[0] = ask;
```

移位把 `argv[7]` 的 NULL 覆盖掉了，而新的 NULL 没有地方放（数组只有 8 个槽）。
`execvp` 于是会越过 `argv[7]` 继续往后读，读到的是 `struct init_action` 里 `argv`
后面的字段。

**实测结果**：

```
6 tokens after shift:
   argv[0] = /sbin/askfirst  ... argv[7] = NULL
   -> terminated inside argv[]: YES (safe)

7 tokens after shift:
   argv[0] = /sbin/askfirst
   argv[1] = /usr/libexec/login.sh
   argv[2..6] = p1..p5
   argv[7] = p6
   -> terminated inside argv[]: NO (BUG)
   execvp() would keep reading past argv[7]:
      argv[8] (out of bounds) = 0x504000000050
   argv[8] aliases struct init_action::line = 0x504000000050
```

**影响**：需要 inittab 里某条 `askfirst`/`askconsole` 记录的命令部分正好有 7 个
以空格分隔的词。默认 OpenWrt 的 inittab 只有
`::askconsole:/usr/libexec/login.sh`（1 个词），所以默认不触发。触发时后果是
子进程拿到一个多余的 argv 项（正好是 `->line` 指针，一个合法字符串），行为诡异
但通常不至于崩溃。

**修法**：`MAX_ARGS` 加 1，或者解析时只填 `MAX_ARGS - 2` 个槽。

---

### 5.5 【高】trigger 的 fork 失败路径绕过 runqueue，造成 use-after-free 且卡死队列

**位置**：`service/trigger.c:105-115`

```c
static void trigger_command_run(struct runqueue *q, struct runqueue_task *t)
{
    ...
    pid = fork();
    if (pid < 0) {
        trigger_command_complete(q, t);   /* 直接调 complete 回调 */
        return;
    }
```

libubox 只在 `runqueue_task_complete()` 里做真正的清理：

```c
void runqueue_task_complete(struct runqueue_task *t) {
    ...
    if (t->running) q->running_tasks--;
    safe_list_del(&t->list);
    t->queued = false;
    if (t->complete) t->complete(q, t);
}
```

手工调 `->complete` 把前面三步全跳过了，而 `trigger_command_complete()` 结尾是
`free(cmd)`。于是任务被 free 掉，但仍然挂在 `q->tasks_active` 链表里，
`running_tasks` 也没减回去。

**实测结果**（真实 libubox runqueue）：

```
adding task to runqueue
  run() entered:      queued=1 running=1 running_tasks=1
  simulating fork() == -1
  complete() called:  queued=1 running=1 running_tasks=1
  -> free(cmd)

after uloop:
  freed          = 1
  running_tasks  = 1   (expected 0 if properly completed)
  tasks_active   = STILL CONTAINS THE FREED TASK

RESULT: use-after-free - the runqueue holds a dangling pointer to freed
        memory and running_tasks is stuck at 1, so the trigger runqueue
        is wedged as well.
```

**影响**：两重。一是链表里留了悬垂指针，下一次遍历 `tasks_active` 就是 UAF；
二是 `running_tasks` 永久 +1，而 trigger 的 runqueue 是限流的，泄漏够次数之后
**所有触发器都不再执行**——热插拔、配置 reload、接口事件全部失灵，而且没有任何
日志提示。

`fork()` 在内存紧张的嵌入式设备上失败是很现实的场景。

**修法**：`runqueue_task_complete(t);` 替代直接调回调。

---

### 5.6 【高】`service validate` 对非字符串成员做 `strlen`/`strcpy`，堆越界

**位置**：`service/validate.c:145-152`

```c
blobmsg_for_each_attr(cur, tb[SERVICE_VAL_DATA], rem) {
    struct vrule *vr = calloc_a(sizeof(*vr),
        &option, strlen(blobmsg_name(cur)) + 1,
        &rule,   strlen(blobmsg_get_string(cur)) + 1);
    ...
    strcpy(vr->rule, blobmsg_get_string(cur));
```

没有 `blobmsg_type(cur) == BLOBMSG_TYPE_STRING` 检查。libubox 里
`blobmsg_get_string()` 只是一个指针 cast，对非字符串成员返回的是**没有 NUL 结尾的
裸负载**，`strlen` 会一路扫出消息边界。

**实测结果**（构造 `{"data":{"opt":1094795585}}`，`opt` 是 INT32，4 个字节全是
`0x41`，且是消息里最后一个成员，所以整条消息里没有 NUL）：

```
message is 32 bytes at 0x503000000040
member "opt" type=5 (BLOBMSG_TYPE_STRING would be 3)
calling strlen(blobmsg_get_string(cur)) as validate.c does...
=================================================================
==1931607==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x503000000060
READ of size 5 at 0x503000000060 thread T0
    #0 ... in strlen
    #1 0x401983 in main /tmp/pv/poc7_validate.c:56

0x503000000060 is located 0 bytes after 32-byte region [0x503000000040,0x503000000060)
SUMMARY: AddressSanitizer: heap-buffer-overflow
```

**影响**：`strlen` 的越界读决定了 `calloc_a` 的大小，紧接着 `strcpy` 会把同样长度
的数据拷进去。如果越界读扫过一大段非零内存，`calloc_a` 可能失败（返回 NULL，随后
解引用崩溃）；如果扫到的长度和 `strcpy` 实际拷的长度因为中途内存变化而不一致，
就是堆越界写。触发只需要一次 `ubus call service set`，`validate` 段里放一个
非字符串值。

**修法**：循环开头加类型检查，非 `BLOBMSG_TYPE_STRING` 直接 `continue`。

---

### 5.7 【中】容器控制台：空实例列表产生假指针，NULL 检查形同虚设

**位置**：`service/service.c`，`container_handle_console()`

未指定实例名时，代码取第一个实例：

```c
vlist_for_each_element(&s->instances, in, node)
    break;

if (!in)
    goto err_console_fd;
```

libubox 的 `avl_for_each_element()` 展开后是
`for (element = avl_first_element(...); ...)`，而 `avl_first_element` 在**空树**上
返回的是 `container_of(&tree->list_head, ...)`——一个由链表头地址反推出来的
**非 NULL 垃圾指针**。循环体一次都不执行，但 `in` 已经被赋成了这个值，`!in` 检查
抓不住。

**实测结果**：

```
instances.avl.count = 0 (empty)
&instances.avl.list_head = 0x7fae73d09020
after vlist_for_each_element: in = 0x7fae73d09020
RESULT: !in check PASSED with a bogus pointer -> procd would now
        dereference in->console at 0x7fae73d090a0
        offsetof(console) = 128, so the write lands 128 bytes past
        &s->instances.avl.list_head
        reading in->console.fd.fd ...
        got 0 (out-of-bounds read completed)
```

**影响**：对一个已注册但当前没有任何实例的容器服务调用 `console` 方法，procd
会去读写 `struct service` 之后 100 多字节处的堆内存。触发需要服务存在而实例为空
——`service_delete` 之后实例还没停干净、或者一次 `set` 只声明了服务没声明实例时
会出现这个窗口。

**修法**：改用 `avl_is_empty()` 判断，或者显式 `in = NULL;` 后再循环：

```c
struct service_instance *in = NULL, *tmp;
vlist_for_each_element(&s->instances, tmp, node) { in = tmp; break; }
if (!in) goto err_console_fd;
```

---

### 5.8 【高】`system signal` 不校验 pid，`pid = 4294967295` 可杀死全系统

> 静态结论，未做运行时复现。确信度：很高（C 语言语义直接可推）。

**位置**：`system.c:531-547`

```c
static const struct blobmsg_policy signal_policy[__SIGNAL_MAX] = {
    [SIGNAL_PID] = { .name = "pid",    .type = BLOBMSG_TYPE_INT32 },
    [SIGNAL_NUM] = { .name = "signum", .type = BLOBMSG_TYPE_INT32 },
};

static int proc_signal(...)
{
    blobmsg_parse(signal_policy, __SIGNAL_MAX, tb, blob_data(msg), blob_len(msg));
    if (!tb[SIGNAL_PID] || !tb[SIGNAL_NUM])
        return UBUS_STATUS_INVALID_ARGUMENT;

    kill(blobmsg_get_u32(tb[SIGNAL_PID]), blobmsg_get_u32(tb[SIGNAL_NUM]));

    return 0;
}
```

`blobmsg_get_u32()` 返回 `uint32_t`，`kill()` 的第一个参数是 `pid_t`（32 位有符号）。
传 `4294967295` 转换后就是 `-1`，而 **`kill(-1, sig)` 的语义是"给调用者有权限
发送信号的所有进程发信号"**。procd 是 root，等于全系统。

同样地 `pid = 0` 是"发给整个进程组"，负值是"发给指定进程组"。信号号也完全没有
校验，`kill()` 会用 `EINVAL` 挡掉非法值，但返回值被丢弃了，调用方永远收到成功。

**影响**：一次 `ubus call system signal '{"pid":4294967295,"signum":9}'` 就能把整机
打死。唯一的防线是 ubus ACL——也就是说这**完全依赖部署方的 ACL 配置正确**，代码
本身没有任何兜底。考虑到 rpcd 的 ACL 文件是按对象粒度授权的，任何被授予
`system` 对象访问权的角色都拿到了这个能力。

**修法**：

```c
uint32_t pid = blobmsg_get_u32(tb[SIGNAL_PID]);
uint32_t sig = blobmsg_get_u32(tb[SIGNAL_NUM]);

if (pid < 1 || pid > INT_MAX || sig >= _NSIG)
    return UBUS_STATUS_INVALID_ARGUMENT;
if (kill((pid_t)pid, (int)sig))
    return UBUS_STATUS_UNKNOWN_ERROR;
```

---

### 5.9 【高】`user` 配置写错时静默以 root 运行

> 静态结论，未做运行时复现。确信度：很高。

**位置**：`service/instance.c:1354-1371`（解析）和 `546-557`（应用）

```c
if (tb[INSTANCE_ATTR_USER]) {
    const char *user = blobmsg_get_string(tb[INSTANCE_ATTR_USER]);
    struct passwd *p = getpwnam(user);
    if (p) {
        in->user = strdup(user);
        in->uid = p->pw_uid;
        in->gr_gid = in->pw_gid = p->pw_gid;
    }
    /* 没有 else —— getpwnam 失败就什么都不做 */
}
```

`in` 来自 `calloc`，所以 `uid`/`pw_gid`/`gr_gid` 保持 0，`user` 保持 NULL。
到了执行阶段：

```c
if (!in->has_jail && in->user && in->pw_gid && initgroups(in->user, in->pw_gid)) { ... }
if (!in->has_jail && in->gr_gid && setgid(in->gr_gid))                          { ... }
if (!in->has_jail && in->uid   && setuid(in->uid))                              { ... }
```

三个条件里的 `in->user &&` / `in->gr_gid &&` / `in->uid &&` 全部为假，**三次降权
调用一个都不执行**，进程以 root 身份 exec。

**影响**：init 脚本里把用户名拼错一个字母（或者某个包的 postinst 忘了建用户、
或者用户被误删），本来应该以 `nobody` 运行的守护进程就以 **root** 跑起来了，
而且日志里没有任何提示。这是典型的"fail-open"——安全配置失效时应该拒绝启动，
而不是降级到最高权限。

同样的模式也存在于 `group`（`instance.c:1364-1371`）。

**修法**：`getpwnam` 失败时 `return false` 让整个实例配置解析失败，至少也要
`ERROR()` 出来。注意 uid 为 0 的合法用户和"未配置"目前无法区分，彻底修好需要加
一个 `bool has_user` 标志。

---

### 5.10 【中】`service_update` 失败时新分配的 service 泄漏

> 静态结论。确信度：高。

**位置**：`service/service.c` `service_handle_set()`

新服务的路径是 `service_alloc()` → `service_update()` → `avl_insert()`。如果
`service_update()` 返回错误，函数直接返回，此时 `s` 既没插进 AVL 树也没被 free。
因为没进树，后续任何清理路径都找不到它。

每次失败泄漏一个 `struct service` 加上它的名字字符串。攻击者可以用畸形请求反复
触发。

---

### 5.11 【中】fork 失败时 pipe fd 泄漏

> 静态结论。确信度：高。

**位置**：`service/instance.c:650-661`、`rcS.c:80-87`

两处都是先 `pipe()` 再 `fork()`，fork 失败时只关了一半或一个都没关。
`instance.c` 那处用的 `closefd()` 有 `if (fd > STDERR_FILENO)` 的保护，但读端根本
没进入清理路径。

在内存压力下 fork 失败会反复发生（而 respawn 机制保证它会反复重试），fd 耗尽后
procd 无法再创建任何 pipe 或 socket，等于服务管理彻底停摆。

---

### 5.12 【中】validator 插入失败后链表留下悬垂指针

> 静态结论。确信度：高。

**位置**：`service/validate.c:137-139`

```c
list_add(&vr->list, &s->validators);      /* 先进链表 */
if (avl_insert(&validators, &vr->avl))    /* 再插 AVL */
    free(vr);                             /* 失败就 free —— 但没从链表摘掉 */
```

顺序反了。`free(vr)` 之后 `s->validators` 链表里还挂着 `&vr->list`，下一次遍历
就是 UAF。正确做法是先 `avl_insert` 成功了再 `list_add`，或者 free 之前先
`list_del`。

---

### 5.13 【低】一批未检查返回值的分配

> 静态结论。确信度：高（都是显而易见的），但影响仅限 OOM 场景。

| 位置 | 内容 |
|------|------|
| `service/service.c:100` | `service_alloc()` 里 `calloc_a` 未检查就 `strcpy` |
| `service/trigger.c:215` | `calloc_a` 未检查就写 `t->type` |
| `service/watch.c:72` | `calloc_a` 未检查就赋值字段 |
| `service/instance.c:1276` | `instance_gen_setns_argstr()` 里 `malloc` 未检查 |
| `jail/jail.c:1719` | `parseOCIrlimit` 的 `malloc` 未检查 |
| `jail/jail.c:1369` | `parseOCIenvarray` 的 `strdup` 未检查 |
| `jail/fs.c:348` | `parseOCImountopts` 的 `calloc` 未检查 |
| `jail/cgroups.c:78` | `cgroups_set` 的 `strdup` 未检查 |

在 PID 1 里 NULL 解引用等于内核 panic，所以即使是"只有 OOM 才发生"也值得修。

---

### 5.14 【中】`initd/early.c` 忽略所有 mount 返回值

> 静态结论。确信度：高。

`early()` 里挂载 `/proc`、`/sys`、`/dev`、`/tmp` 的 6 个 `mount()` 调用返回值全部
丢弃。如果 `/tmp` 挂载失败（tmpfs 不可用、内存不足），系统会带着一个只读 rootfs
继续启动，后面几乎所有需要写临时文件的服务都会以各种莫名其妙的方式失败，而根因
在日志里完全看不到。

同一类问题在 `sysupgrade.c` 里更危险：`execvp("/sbin/upgraded")` 失败后代码会尝试
`chroot(".")` 退回，但 `service_stop_all()` 已经执行过了，退回去只剩一个没有任何
服务的 procd。设备既没升级成功也无法正常工作，只能靠人工干预。这条路径应该直接
`reboot()` 而不是尝试恢复。

---

### 5.15 【高】ujail：`free_hooklist` / `free_sysctl` 指针步进错误

> 静态结论，已逐行复核源码。确信度：很高。

**位置**：`jail/jail.c:199-230`

```c
static void free_hooklist(struct hook_execvpe **hooklist)
{
    struct hook_execvpe *cur;

    if (!hooklist)
        return;

    cur = *hooklist;              /* 取出第一个元素的指针 */
    while (cur) {
        free_oci_envp(cur->argv);
        free_oci_envp(cur->envp);
        free(cur->file);
        free(cur++);              /* cur += sizeof(struct hook_execvpe) */
    }
    free(hooklist);
}
```

`hooklist` 是**指针数组**（`struct hook_execvpe **`）。`cur` 是单个对象指针，
`cur++` 按 `sizeof(struct hook_execvpe)` 步进，走的根本不是数组的下一个槽位，而是
第一个 hook 对象后面的内存。而且 `while (cur)` 永远为真（一个不断递增的地址不会
变成 NULL），循环只会在解引用到未映射内存时段错误结束。

同一个文件里 `free_devices()`（232-245 行）的写法是对的，可以直接对照：

```c
struct mknod_args **cur;          /* 二级指针 */
cur = opts.devices;
while (*cur) {                    /* 解引用判空 */
    free((*cur)->path);
    free(*(cur++));               /* free 元素，指针走一格 */
}
```

`free_sysctl()`（216-230 行）有一模一样的错误。

**影响**：任何用到 OCI `hooks` 或 `linux.sysctl` 的容器，在 `free_opts()` 被调用时
崩溃。`free_opts(false)` 恰好在 `post_start_hook()` 里 **`execve` 之前**调用，所以
容器根本起不来。这两个特性目前基本处于不可用状态——大概也正因为很少有人用，这个
bug 才一直没被发现。

**修法**：照抄 `free_devices` 的二级指针写法。

---

### 5.16 【高】ujail：OCI mount 目标和 sysctl 键名可路径穿越

> 静态结论，已逐行复核源码。确信度：高（前提是 OCI bundle 不可信）。

**位置一**：`jail/fs.c:82`

```c
snprintf(new, sizeof(new), "%s%s", root, target?target:source);
```

`target` 直接来自 OCI `config.json` 的 `mounts[].destination`，**没有任何 `..`
过滤、没有要求绝对路径、没有规范化**。`root` 是 `/tmp/ujail-XXXXXX`。
destination 写成 `/../../../../etc` 就落到宿主机的 `/etc`。这段代码运行在
**`pivot_root` 之前、以 root 身份**执行。同样的路径拼接被 `maskedPaths` 和
`readonlyPaths` 复用。

**位置二**：`jail/jail.c:2159-2170` + `547`

```c
/* 解析：把 sysctl 键名里的 '.' 换成 '/' */
tc = tmp = strdup(blobmsg_name(cur));
while ((tc = strchr(tc, '.')))
    *tc = '/';
opts.sysctl[cnt]->entry = tmp;

/* 应用 */
if (asprintf(&fname, "%s/sys/%s", procdir, (*cur)->entry) < 0)
    return ENOMEM;
f = open(fname, O_WRONLY);
write(f, (*cur)->value, strlen((*cur)->value));
```

键名没有字符集限制。一个不含 `.` 的键名如 `../../../../etc/passwd` 会原样拼进
`{jail_root}/proc/sys/` 后面，穿越出去。`apply_sysctl()` 同样在 pivot_root 之前
以 root 运行，效果是**任意文件写**。

**影响**：能提供 OCI bundle 的一方等于拿到宿主机 root 写权限。OpenWrt 上 bundle
通常由管理员放置（`uxc create` 指定路径），所以威胁模型上算"管理员可信"；但
容器场景下"能提交 bundle"和"是 root"通常被认为是两回事，这是实打实的隔离逃逸。

**修法**：destination 必须是绝对路径、拒绝任何 `..` 分量；sysctl 键名限制在
`[a-zA-Z0-9_.-]`。更稳妥的做法是用 `openat2(RESOLVE_BENEATH)` 或先
`pivot_root` 再挂载。

---

### 5.17 【中】ujail：`run_hooklist` 在 stat 失败后不返回，且读未初始化的 `struct stat`

> 静态结论，已逐行复核源码。确信度：很高。

**位置**：`jail/jail.c:480-487`

```c
struct stat s;
...
if (stat(hook->file, &s))
    hook_process_handler(&hook_process, ENOENT);       /* 缺 return */

if (!((unsigned long)s.st_mode & (S_IXUSR | S_IXGRP | S_IXOTH)))
    hook_process_handler(&hook_process, EPERM);        /* 缺 return */

hook_running = 1;
hook_process.pid = fork();
if (hook_process.pid == 0) {
    execve(hook->file, hook->argv, hook->envp);
```

`hook_process_handler()` 会推进 `current_hook` 并递归调 `run_hooklist()`，跑完
**整个剩余 hook 列表**之后返回到这里，然后当前这一帧继续往下走，`fork` + `execve`
那个刚刚被判定为不存在的文件。

`stat` 失败时 `s` 从未被写过，第二个 `if` 读的是未初始化的栈内存。

**影响**：hook 重复执行、执行顺序错乱、对不存在的文件 exec。配合 5.15，OCI hook
这条特性整体不可用。

**修法**：两处都改成 `return hook_process_handler(...);`。

---

### 5.18 【中】ujail：`build_envp` 环境变量满 64 条时没有 NULL 终止符

> 静态结论，已逐行复核源码。确信度：高。

**位置**：`jail/jail.c:969-1010`

```c
#define MAX_ENVP 64
static char *envp[MAX_ENVP];
...
while (addenv && *addenv) {
    envp[count++] = *(addenv++);
    if (count >= MAX_ENVP) {
        ERROR("environment limited to %d extra records, truncating\n", MAX_ENVP);
        break;
    }
}
return envp;
```

`count` 达到 64 时 `envp[0..63]` 全被填满，跳出循环，**没有地方放 NULL**。
`execve` 要求 envp 以 NULL 结尾，于是会继续往 `envp[64]` 之后读——因为 `envp` 是
`static`，读到的是 .bss 里相邻的 `preload_var` / `seccomp_var` 等静态缓冲区。

前面无条件写入的 5 个槽位（seccomp 三项 + `container=ujail` + 可选 `LD_DEBUG`）
本身没有边界检查，不过 5 远小于 64，不构成问题。

**影响**：OCI `process.env` 有 60 条以上时（对完整的容器镜像不算罕见），容器进程
会拿到几个来路不明的环境变量，或者在 .bss 里扫不到 NUL 时崩溃。

**修法**：循环上界改成 `MAX_ENVP - 1`，退出后 `envp[count] = NULL;`。

---

### 5.19 【低】ujail：`write_uid_gid_map` 把成功当失败

> 静态结论，已逐行复核源码。确信度：很高，但当前无实际影响。

**位置**：`jail/jail.c:851-854`

```c
if (dprintf(map_file, "%s", mapstr)) {
    close(map_file);
    return -1;
}
```

`dprintf` 成功时返回**写入的字节数**（非空 map 必然 ≥ 1），所以这个条件在成功时
为真，函数总是返回 -1。同一文件里 `write_single_uid_gid_map()`（872 行）写的是
`if (dprintf(...) < 0)`，正确。

**影响**：调用点（3122-3124 行）**忽略了返回值**，所以 uid map 实际上写成功了，
目前没有可观察的错误。这是一颗定时炸弹：任何人日后加上返回值检查，userns 容器
就会立刻全部启动失败。

**修法**：`< 0`。

---

### 5.20 【低】ujail：`run_hooks` 在 hooklist 为空时缺 return

> 静态结论，已逐行复核源码。确信度：很高（缺陷确凿），但当前是潜伏状态。

**位置**：`jail/jail.c:515-523`

```c
static void run_hooks(struct hook_execvpe **hooklist, hook_return_handler return_cb)
{
    if (!hooklist)
        return_cb();          /* 缺 return */

    current_hook = hooklist;  /* = NULL */
    hook_return_cb = return_cb;

    run_hooklist();           /* 第一行就是 *current_hook → NULL 解引用 */
}
```

**影响**：目前不崩溃，因为这条链上最内层的回调（`post_start_hook`）以 `execve`
或 `exit` 结束，永远不会返回到这里。非 OCI 的普通 jail 走的正是
`hooklist == NULL` 这条路。一旦哪个回调改成正常返回，就是 NULL 解引用。

**修法**：`return return_cb();`。

---

### 5.21 【低】`system board` 解析 `os-release` 的反斜杠转义会越界

> 静态结论。确信度：高，但触发需要一个畸形的 root 属主文件。

**位置**：`system.c:289-314`

```c
dest = blobmsg_alloc_string_buffer(&b, key, strlen(val));
...
while (val && (ch = *(val++)) != 0) {
    switch (ch) {
    ...
    case '\\':
        *(dest++) = *(val++);     /* 无条件拷贝下一个字节 */
        break;
    }
}
```

如果一行以单个反斜杠结尾（`VERSION=\`），`*(val++)` 拷走的是字符串的 NUL 终止符，
`val` 随即指向 NUL **之后**——也就是 `line[]` 数组里上一行残留的、或者从未初始化
的字节。循环条件继续读这些字节，遇到 `\` 就继续拷，`dest` 一路越过
`blobmsg_alloc_string_buffer` 申请的 `strlen(val)` 字节。

**影响**：`/usr/lib/os-release` 由构建系统生成、root 属主，正常内容不会触发。
所以这是健壮性问题，不是攻击面。不过 `system board` 是 LuCI 首页必调的接口，
一个手工改坏的 os-release 会让 procd 在每次页面刷新时越界。

**修法**：`case '\\':` 里先判断 `*val` 非 0；同时给 `dest` 加上写入上限。

---

## 六、复现环境

### 6.1 依赖构建

本机没有可用的 json-c，无法完整构建 procd（`system.c` 依赖 json-c，`jail/` 依赖
libubus + libnl-tiny + libseccomp）。因此 5.1 ~ 5.7 采取的办法是**只构建 libubox
核心，再把 procd 的问题函数逐字抄成独立 PoC**——这样验证的是有问题的那段逻辑本身
加上真实的 libubox 语义（真实的 blobmsg 布局、真实的 avl/vlist 遍历宏、真实的
ustream 缓冲策略、真实的 runqueue 状态机），而不是我对这些语义的臆测。

```bash
mkdir -p /tmp/pv && cd /tmp/pv
LB=/home/m11wang/openwrt/openwrt/components/libubox
mkdir -p include && ln -sfn $LB include/libubox

for f in blob blobmsg avl-cmp utils safe_list runqueue uloop ustream ustream-fd vlist md5; do
    gcc -c -g -O0 -I$LB -Iinclude -D_GNU_SOURCE -o $f.o $LB/$f.c
done

# avl.c 要单独编：它 #include "assert.h" 会命中 libubox 自己的 assert.h，
# 而后者的 #include <assert.h> 在 -I$LB 存在时又会绕回自身（#pragma once
# 导致 assert 最终没有定义）。去掉 -I$LB 让 <assert.h> 解析到系统头文件即可。
gcc -c -g -O0 -Iinclude -D_GNU_SOURCE -o avl.o $LB/avl.c

ar rcs libubox_core.a *.o
```

### 6.2 PoC 列表

| 文件 | 对应 | 编译 | 结果 |
|------|------|------|------|
| `poc1_respawn.c` | 5.1 | `-fsanitize=address` | ASan stack-buffer-overflow |
| `poc5_rcs.c` | 5.2 | 无 sanitizer（要跑 uloop） | consumed=0，对照组 consumed=65536 |
| `poc3_inittab.c` | 5.3 | `-fsanitize=address` | ASan heap-buffer-overflow（读越界 1 字节） |
| `poc4_argv.c` | 5.4 | `-fsanitize=address` | argv[7] 非 NULL，argv[8] 别名到 `->line` |
| `poc6_trigger.c` | 5.5 | `-fsanitize=address` | `running_tasks` 卡在 1，链表里留着已 free 的 task |
| `poc7_validate.c` | 5.6 | `-fsanitize=address` | ASan heap-buffer-overflow（strlen 读出消息尾部） |
| `poc2_console.c` | 5.7 | `-fsanitize=address` | `in` = 链表头地址，`!in` 检查通过 |

统一编译命令：

```bash
gcc -g -O0 -Iinclude -D_GNU_SOURCE -fsanitize=address -o pocN pocN_xxx.c libubox_core.a
ASAN_OPTIONS=detect_leaks=0 ./pocN
```

### 6.3 在真实设备上验证 5.1 和 5.6

如果手上有 OpenWrt 设备（**只在测试机上做，5