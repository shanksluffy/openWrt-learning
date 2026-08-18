# 如何有效阅读和理解 OpenWrt 源码

适用对象：本仓库（OpenWrt 25.12-SNAPSHOT）
撰写日期：2026-08-06

---

## 目录

1. [先建立正确的心理模型](#一先建立正确的心理模型)
2. [当前这棵树的实际状态](#二当前这棵树的实际状态)
3. [推荐的阅读顺序](#三推荐的阅读顺序)
4. [读构建系统](#四读构建系统)
5. [读第三方包与补丁](#五读第三方包与补丁)
6. [让 IDE 真正能跳转](#六让-ide-真正能跳转)
7. [用运行时反推源码](#七用运行时反推源码)
8. [高频任务的追踪配方](#八高频任务的追踪配方)
9. [常见误区](#九常见误区)
10. [参考资料](#十参考资料)

---

## 一、先建立正确的心理模型

新手在 OpenWrt 源码里迷路，**根本原因几乎都是把四类完全不同的代码当成一类去读**。这棵树里其实混着：

| 类别 | 位置 | 本质 | 读法 |
|------|------|------|------|
| **构建系统** | `Makefile`、`rules.mk`、`include/*.mk`、各包的 `Makefile` | GNU Make 的元编程层，用宏生成规则 | 当成一门 DSL 学，不要逐行读 |
| **自研核心 C 项目** | `components/`（本树）或构建时下载 | 真正的 C 代码，OpenWrt 自己写的 | 正常读 C，自底向上 |
| **第三方包** | `package/**/Makefile` + `patches/` | **不含源码**，只有下载配方和补丁 | 先看 Makefile 定位真源码，再读 patches |
| **内核与平台支持** | `target/linux/` | 内核配置 + DTS + 平台补丁 | 按 target 隔离阅读 |

举个最容易踩的例子：`package/network/services/hostapd/` 下有 69 个补丁文件但没有 hostapd 的主体源码 —— 源码在 `make` 时才被下载解压到 `build_dir/`。如果你直接在 `package/` 里 grep 函数名，什么都找不到。

第二个心理模型是**依赖方向**。OpenWrt 的用户空间是一棵很清晰的树，理解了这棵树就理解了系统：

```
                    libubox
        （事件循环 uloop + 序列化 blobmsg + 容器）
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      ubus           uci          ustream
   （IPC 总线）    （配置存储）    （缓冲流）
        │              │
        └──────┬───────┘
               │
   ┌───────────┼───────────┬──────────┐
   │           │           │          │
 procd      netifd       rpcd      其它守护进程
（init/    （网络配置   （给 HTTP    （dnsmasq、
 进程管理）   与接口）    暴露 ubus）   hostapd…）
```

**关键认知：`ubus` 是整个系统的脊梁。** 几乎所有跨进程交互都走它。一旦你能读懂一个 ubus 方法的注册和调用，OpenWrt 的大半就通了。

---

## 二、当前这棵树的实际状态

在给出方法之前，先说清这棵树现在是什么状况 —— 这直接决定哪些手段现在可用。

| 检查项 | 状态 |
|--------|------|
| `.config` | **不存在** —— 从未配置 |
| `build_dir/` | **不存在** —— 从未构建 |
| `staging_dir/` | 仅有 `host/` |
| `feeds/` | 5 个 feed 已 clone 并建好索引 |
| `tmp/.packageinfo` | 不存在（未跑过 `make prepare`） |
| `components/` | 6 个独立 git clone：`libubox` `netifd` `procd` `rpcd` `ubus` `uci` |

两个重要推论：

**1. `components/` 不参与构建。** 我搜过整个 `include/` 和所有 `Makefile`，没有任何地方引用 `components/`。它是有人手动放进来供阅读的 git checkout，构建时走的仍是各包 Makefile 里的下载配方。

**2. `components/` 里的代码比构建实际编译的版本新。** 以 libubox 为例：

```makefile
# package/libs/libubox/Makefile
PKG_SOURCE_DATE:=2026-06-19
PKG_SOURCE_VERSION:=7dd127841e82eb1cfb61185da37dde7b9bd9ba6d
```

而 `components/libubox` 的 HEAD 是 `e7608b6`（2026-07-21），**领先 44 个提交**。这 44 个提交几乎全是安全加固（整数溢出、use-after-free、`alloca` 边界）。

这是个非常容易浪费几天的陷阱：**你在 `components/` 里读到的修复，可能并不在你构建出的固件里**。任何时候要确认"实际编译的是哪一版"，做法是：

```bash
# 查构建系统固定的版本
grep -E "PKG_SOURCE_(DATE|VERSION)|PKG_VERSION" package/libs/libubox/Makefile

# 与 components 里的 checkout 对比
cd components/libubox && git log --oneline 7dd1278..HEAD | wc -l
```

---

## 三、推荐的阅读顺序

**自底向上，一层读通再上一层。** 跳跃式阅读在 OpenWrt 上代价特别高，因为上层代码几乎每一行都在用下层的抽象。

### 第 1 层：libubox（必读，约 11500 行）

这是所有东西的基础，不读它上面全部看不懂。建议顺序：

1. `list.h` —— 内核风格侵入式链表。理解 `container_of` 是后面一切的前提。
2. `uloop.h` + `uloop.c` + `uloop-epoll.c` —— 事件循环。重点看 `uloop_run_timeout()` 主循环、`uloop_fd_add()`、以及 self-pipe 信号处理。
3. `blob.h` + `blobmsg.h` —— 序列化格式。重点理解 `struct blob_attr` 那个 32 位头如何编码类型+长度+扩展位。
4. `blobmsg.c` 的 `blobmsg_parse()` —— 所有守护进程解析参数都走它。
5. `ustream.h` —— 缓冲流，读 netifd/rpcd 时会反复遇到。
6. `avl.h`、`vlist.h` —— 容器。`vlist` 的"版本号 diff"模型是 OpenWrt 处理配置重载的核心手法，值得单独理解。

本仓库已有一份详细分析：[`libubox-analysis.md`](./libubox-analysis.md)。

### 第 2 层：ubus（IPC 总线，脊梁）

- `components/ubus/libubus.h` —— 先看 `struct ubus_object`、`struct ubus_method`、`UBUS_METHOD()` 宏。
- `libubus-obj.c` —— 对象/方法注册。
- `libubus-req.c` —— 请求-应答与超时。
- `ubusd*.c` —— 守护进程侧。**先跳过**，除非你要改总线本身。

读完这一层的验收标准：能自己写出一个注册 ubus 方法的最小程序，并说清一次 `ubus call` 在两个进程间走过的路径。

### 第 3 层：uci（配置存储）

- `components/uci/uci.h` —— `uci_package` / `uci_section` / `uci_option` 三级结构。
- `file.c` —— `/etc/config/*` 的解析。
- 重点理解 **delta/overlay 机制**：`uci set` 先写 `/tmp/.uci/`，`uci commit` 才落盘。这解释了很多"改了配置没生效"的现象。

### 第 4 层：procd（init 与进程管理）

- `components/procd/procd.c`、`instance.c`、`service/service.c`
- `initd/` —— 开机第一阶段。
- 重点是 **procd 用 ubus 接收服务定义**：`/etc/init.d/*` 脚本里的 `procd_open_instance` 等函数最终是把一段 blobmsg 发给 procd。看 `components/procd/service/instance.c` 的 `instance_config_parse()`。

### 第 5 层：netifd（网络配置）

最复杂的一个，建议只在需要时读。入口：

- `interface.c` —— `struct interface` 状态机。
- `device.c` —— 设备抽象层。
- `proto.c` / `proto-shell.c` —— 协议处理器。`proto-shell` 是理解 `/lib/netifd/proto/*.sh` 如何被调用的关键。

### 第 6 层：rpcd

把 ubus 暴露给 HTTP（LuCI 用它）。代码量小，读 `components/rpcd/session.c` 的 ACL 逻辑收益最高。

---

## 四、读构建系统

### 4.1 核心机制：BuildPackage 元编程

OpenWrt 的包 Makefile 不是普通 Makefile，而是**填一组变量和 `define` 块，然后由 `BuildPackage` 宏展开成真正的规则**。以 `package/libs/libubox/Makefile` 为例，逐段解释这个范式：

```makefile
include $(TOPDIR)/rules.mk          # ① 必须第一行，引入全局变量

PKG_NAME:=libubox                   # ② 元数据段
PKG_RELEASE=1
PKG_SOURCE_PROTO:=git               #    源码从 git 取
PKG_SOURCE_URL=$(PROJECT_GIT)/project/libubox.git
PKG_SOURCE_DATE:=2026-06-19
PKG_SOURCE_VERSION:=7dd127841e...   #    固定到这个 commit
PKG_MIRROR_HASH:=82f1a830...        #    tarball 校验和
CMAKE_INSTALL:=1

include $(INCLUDE_DIR)/package.mk   # ③ 引入标准构建流程
include $(INCLUDE_DIR)/cmake.mk     #    用 CMake 而非 autotools

define Package/libubox               # ④ 每个"输出包"一个块
  SECTION:=libs
  CATEGORY:=Libraries               #    决定它出现在 menuconfig 哪一栏
  TITLE:=Basic utility library
  ABI_VERSION:=$(PKG_ABI_VERSION)
  DEPENDS:=
endef

define Package/jshn                  #    一份源码可产出多个包
  DEPENDS:=+libjson-c +libubox +libblobmsg-json
  ...
endef

CMAKE_OPTIONS += -DLUAPATH=/usr/lib/lua   # ⑤ 传给 CMake 的选项

define Package/libubox/install       # ⑥ 从 PKG_INSTALL_DIR 拷进 $(1)
	$(INSTALL_DIR) $(1)/lib/
	$(INSTALL_DATA) $(PKG_INSTALL_DIR)/usr/lib/libubox.so.* $(1)/lib/
endef

$(eval $(call BuildPackage,libubox))  # ⑦ 展开成真正的 make 规则
$(eval $(call BuildPackage,jshn))
$(eval $(call HostBuild))             #    还要为 host 编一份
```

读懂这七段，你就能读懂 OpenWrt 里 99% 的包 Makefile。几个必须记住的约定：

- **`DEPENDS:=+foo`** 前面的 `+` 表示"自动选中 foo"，没有 `+` 表示"foo 必须已被选中，否则本包不可选"。这个区别是依赖问题的头号来源。
- **`$(1)`** 在 `install` 块里是该包的临时 rootfs 目录。
- **一份源码可以产出多个包**（这里 libubox 产出 5 个），它们共享一次编译。
- `PKG_RELEASE` 变了会触发重新打包，`PKG_SOURCE_VERSION` 变了会重新下载。

### 4.2 `include/*.mk` 的分工

不要试图通读，按需查阅：

| 文件 | 作用 |
|------|------|
| `rules.mk` | 全局变量与工具函数（`qstrip`、`confvar` 等），所有包的第一行都 include 它 |
| `include/package.mk` | 定义标准构建流程与 `BuildPackage` |
| `include/package-defaults.mk` | 各阶段的默认实现（`Build/Configure`、`Build/Compile` 等） |
| `include/cmake.mk` / `autotools.mk` / `meson.mk` | 三种构建系统的适配层 |
| `include/kernel.mk` / `kernel-build.mk` | 内核与内核模块 |
| `include/image.mk` | 固件镜像生成 |
| `include/download.mk` | 下载与校验 |
| `include/quilt.mk` | 补丁管理 |
| `include/scan.mk` | 扫描所有包生成 `tmp/.packageinfo` |

### 4.3 三板斧

**① `make V=s` 看真实命令。** 这是读构建系统最有效的手段 —— 与其推理宏怎么展开，不如直接看展开后跑的是什么：

```bash
make package/libubox/compile V=s 2>&1 | tee /tmp/build.log
```

**② 单包增量构建。** 不要动辄 `make world`：

```bash
make package/libubox/{clean,compile} V=s     # 清理后重编单个包
make package/libubox/prepare V=s             # 只下载解压打补丁
make package/libubox/install                 # 只做安装步骤
```

**③ `make printdb` 导出所有变量。** 想知道某个变量最终是什么值：

```bash
make printdb 2>/dev/null | grep -E "^TARGET_CFLAGS|^STAGING_DIR"
```

### 4.4 追踪一个 CONFIG_ 选项

这是高频需求。以任意 `CONFIG_FOO` 为例，四个地方依次查：

```bash
# ① 谁定义了它（menuconfig 里的条目）
grep -rn "config FOO" --include=Config.in .
grep -rn "FOO" --include=Config.in target/ package/

# ② 谁消费了它
grep -rn "CONFIG_FOO" --include=Makefile --include=*.mk .

# ③ 它在 .config 里的当前值（需先 make menuconfig）
grep FOO .config

# ④ 它是否传进了 C 代码
grep -rn "FOO" build_dir/target-*/*/config.h 2>/dev/null
```

注意 OpenWrt 有**三套独立的 config 命名空间**，别混淆：
- `CONFIG_*` in `.config` —— OpenWrt 构建配置
- `CONFIG_*` in `target/linux/*/config-*` —— Linux 内核配置
- `CONFIG_PACKAGE_*` —— 表示某个包是否被选中（`y`=编进固件，`m`=编成可安装包）

### 4.5 元数据缓存

`make menuconfig` 或 `make prepare` 会生成 `tmp/.packageinfo`（本树目前还没有）。它是所有包元数据的扁平文本，是"哪个 Makefile 定义了这个包"的最快答案：

```bash
make prepare                          # 先生成
grep -n "Package: jshn" -A 15 tmp/.packageinfo
```

---

## 五、读第三方包与补丁

### 5.1 先定位真正的源码

拿到一个包目录，第一件事是从 Makefile 里读出源码来源：

```bash
grep -E "PKG_NAME|PKG_VERSION|PKG_SOURCE|PKG_HASH" package/network/services/hostapd/Makefile
```

有三种情况：

1. **`PKG_SOURCE_PROTO:=git`** + `PKG_SOURCE_VERSION` —— 固定到某个 commit，可以直接去上游仓库看那个版本。
2. **`PKG_VERSION` + `PKG_SOURCE`** —— 下载 tarball，`PKG_HASH` 是校验和。
3. **有 `src/` 目录** —— 源码在树内（hostapd 就是这种，`package/network/services/hostapd/src/`）。

### 5.2 patches/ 的读法

补丁按文件名数字前缀顺序应用，前缀本身有约定：

| 前缀区间 | 惯例含义 |
|---------|---------|
| `0xx` | 上游已接受、等发版的补丁 |
| `1xx` | 功能性改动 |
| `2xx`–`4xx` | 平台/编译适配 |
| `7xx`+ | OpenWrt 特有的大改动 |

读一个包的行为差异时，**先读 patches 再读源码** —— OpenWrt 的行为往往就藏在补丁里。hostapd 有 69 个补丁，其中的 ubus 集成（`package/network/services/hostapd/src/src/ap/ubus.c`）就是 OpenWrt 自己加的，上游没有。

### 5.3 quilt 工作流（改补丁的唯一正确方式）

**不要**直接编辑 `build_dir/` 里的源码 —— 下次 `make clean` 就丢了。正确流程：

```bash
# ① 解压并应用现有补丁，进入 quilt 模式
make package/network/services/hostapd/{clean,prepare} QUILT=1

# ② 进入构建目录
cd build_dir/target-*/hostapd-*/

# ③ 应用全部现有补丁
quilt push -a

# ④ 新建一个补丁
quilt new 999-my-fix.patch
quilt add src/ap/beacon.c        # 必须先 add 再编辑！
vim src/ap/beacon.c
quilt refresh                     # 把改动写进补丁文件

# ⑤ 回到顶层，把补丁拷回 package/
cd -
make package/network/services/hostapd/update

# ⑥ 重新编译验证
make package/network/services/hostapd/{clean,compile} V=s
```

常用 quilt 命令：

```bash
quilt series      # 列出所有补丁
quilt applied     # 已应用的
quilt top         # 当前最上层补丁
quilt push/pop    # 前进/回退一个补丁
quilt files       # 当前补丁改了哪些文件
quilt diff        # 看当前补丁内容
```

内核补丁同理：

```bash
make target/linux/{clean,prepare} QUILT=1
cd build_dir/target-*/linux-*/linux-*/
```

`make package/foo/refresh` 用于在源码版本升级后批量刷新所有补丁的行号。

---

## 六、让 IDE 真正能跳转

这是提升阅读效率最大的一步，但 OpenWrt 的构建方式让它有点麻烦。三条路径按推荐度排列：

### 6.1 对 components/ 里的 CMake 项目单独生成（最简单，推荐先做这个）

`libubox`、`ubus`、`uci`、`netifd`、`procd`、`rpcd` 都是 CMake 项目，可以脱离 OpenWrt 构建体系单独生成 `compile_commands.json`：

```bash
cd components/libubox
cmake -B /tmp/cc-libubox -DCMAKE_EXPORT_COMPILE_COMMANDS=ON . 2>/dev/null
cp /tmp/cc-libubox/compile_commands.json .
```

即使 cmake 因缺少 json-c 等依赖而报错，只要走到了配置阶段，`compile_commands.json` 通常已经生成。之后 clangd 就能在这个目录里正常跳转了。

对六个项目批量处理：

```bash
for p in libubox ubus uci netifd procd rpcd; do
  (cd components/$p && cmake -B /tmp/cc-$p -DCMAKE_EXPORT_COMPILE_COMMANDS=ON . >/dev/null 2>&1
   cp /tmp/cc-$p/compile_commands.json . 2>/dev/null && echo "$p: ok" || echo "$p: 失败")
done
```

### 6.2 用 bear 包裹整个 OpenWrt 构建（覆盖全树，但慢）

```bash
bear --output compile_commands.json -- make -j$(nproc) V=s
```

代价是要跑一次完整构建（数小时）。好处是连内核和第三方包都能跳转。

### 6.3 退路：cscope / ctags

不需要构建，立即可用：

```bash
# 只索引真正的 C 源码，跳过构建产物
find components target/linux/generic package -name '*.[ch]' \
     -not -path '*/build_dir/*' > /tmp/cscope.files
cscope -bkq -i /tmp/cscope.files

# 或 ctags
ctags -R --languages=C,C++ --exclude=build_dir --exclude=staging_dir \
      components package target
```

### 6.4 一个重要提醒

**`.config` 会影响头文件可见性。** 未配置的树里，很多 `#ifdef CONFIG_*` 分支在 IDE 看来是灰的，且 `staging_dir/target-*/usr/include` 还不存在，所以跨包跳转（比如从 netifd 跳到 libubox 的头文件）会失败。想彻底解决就得先跑一次构建：

```bash
make defconfig      # 生成一个最小可用配置
make -j$(nproc) tools/install toolchain/install
make -j$(nproc) package/libubox/compile
```

---

## 七、用运行时反推源码

**这是最被低估的手段。** OpenWrt 是个高度自省的系统，在真机（或 QEMU）上花十分钟观察，往往比读两小时代码更有效。

### 7.1 ubus：看清所有跨进程接口

```bash
ubus list                      # 列出所有注册的对象
ubus -v list                   # 连方法签名和参数类型一起列出
ubus -v list network.interface # 只看某个对象

ubus call system board         # 实际调用一个方法
ubus call network.interface.lan status

ubus monitor                   # 实时抓总线上的所有报文 ★最有用
ubus listen                    # 只监听事件
```

`ubus -v list` 的输出直接对应源码里的 `struct blobmsg_policy` 数组 —— 拿着方法名去 grep，一步就能定位到实现：

```bash
# 已知 ubus 方法名 "status"，找它的实现
grep -rn '"status"' components/netifd/*.c
```

`ubus monitor` 尤其适合搞清"点一下 LuCI 的某个按钮，底层到底调了什么"。

### 7.2 uci：看清所有配置

```bash
uci show                       # 全部配置的扁平输出
uci show network               # 单个包
uci get network.lan.ipaddr     # 单项
uci changes                    # 未 commit 的改动（在 /tmp/.uci/）
```

配置项名字（如 `network.lan.ipaddr`）可以直接拿去 grep 源码里的 policy 定义。

### 7.3 procd：看清服务与启动

```bash
ubus call service list          # 所有 procd 管理的服务及其实例
ubus call service list '{"verbose":true}'
logread -f                      # 实时日志
logread -e netifd               # 过滤

/etc/init.d/network status
```

服务定义在 `/etc/init.d/*`，用的是 `procd_open_instance` 那套 shell 函数。它们在设备上是 `/lib/functions/procd.sh`，在本树里是 `package/system/procd/files/procd.sh` —— 读这个文件能看清 shell 侧是怎么把服务定义拼成 blobmsg 再发给 procd 的。想知道一个服务实际以什么参数启动，看 `ubus call service list` 的输出比读 shell 脚本可靠。

### 7.4 netifd

```bash
ubus call network.device status
ubus call network.interface dump
/etc/init.d/network restart     # 配合 logread -f 观察
```

### 7.5 没有真机时用 QEMU

```bash
# 配置 target 为 x86/64，选上 CONFIG_TARGET_ROOTFS_EXT4FS
make menuconfig
make -j$(nproc)
qemu-system-x86_64 -m 256 \
  -drive file=bin/targets/x86/64/openwrt-*-generic-ext4-combined.img,format=raw \
  -netdev user,id=n0,hostfwd=tcp::2222-:22 -device e1000,netdev=n0 -nographic
```

---

## 八、高频任务的追踪配方

### 配方 1：这个文件是哪个包装进去的？

```bash
# 在构建树上（需已构建）
grep -rn "usr/bin/jshn" package/*/*/Makefile package/*/*/*/Makefile

# 在运行的设备上（25.12 默认 apk）
apk info --who-owns /usr/bin/jshn
# 老版本 opkg
opkg search /usr/bin/jshn
```

### 配方 2：一个 ubus 方法从注册到被调用的完整链路

以 `network.interface` 的 `status` 为例：

```bash
# ① 找方法表
grep -rn "UBUS_METHOD\|ubus_method" components/netifd/ubus.c

# ② 找它的 policy（参数定义）
grep -n "blobmsg_policy" components/netifd/ubus.c

# ③ 找对象注册点
grep -rn "ubus_add_object" components/netifd/

# ④ 客户端侧怎么发起的
grep -rn "ubus_invoke" components/ | head
```

链路模型（记住这个就够了）：

```
客户端 ubus_invoke()
   → blobmsg 编码
   → 经 /var/run/ubus/ubus.sock 发给 ubusd
   → ubusd 查对象表路由
   → 服务端 uloop_fd 回调收到
   → libubus 解析 → 查方法表 → blobmsg_parse(policy)
   → 你的 handler(ctx, obj, req, method, msg)
   → ubus_send_reply() 反向走一遍
```

### 配方 3：一个配置项怎么变成运行时行为

以 `network.lan.ipaddr` 为例，四步：

1. `uci show network` 确认配置项存在
2. 在 netifd 里 grep 该 option 名：`grep -rn '"ipaddr"' components/netifd/`
3. 找到对应的 `blobmsg_policy` 与 `uci_blob_param_list`（netifd 用 `uci_to_blob()` 把 uci 转成 blobmsg）
4. 顺着使用该字段的函数往下读到真正下发的地方（通常是 `system-linux.c` 里的 netlink 调用）

`components/netifd/config.c` 是 uci → blobmsg 的转换枢纽，值得先读。

### 配方 4：一次开机都发生了什么

```
内核启动
  → /sbin/init（procd 的 initd 部分，components/procd/initd/）
  → early()：挂载 /proc /sys，加载 preinit
  → /etc/preinit → /lib/preinit/*.sh（按数字顺序）
  → 切换到 procd 主体（components/procd/procd.c）
  → procd 启动 ubusd
  → 扫描 /etc/rc.d/S* 按序启动服务
  → 各服务通过 ubus 把自己的实例定义注册回 procd
```

对应源码入口：`components/procd/initd/init.c` 的 `main()`，然后是 `components/procd/procd.c`。

### 配方 5：为什么我的改动没生效

按这个顺序排查：

```bash
# ① 构建系统固定的版本是不是你改的那个？（见第二章的陷阱）
grep PKG_SOURCE_VERSION package/libs/libubox/Makefile

# ② 包是不是真的重编了？
make package/libubox/{clean,compile} V=s

# ③ 是不是被 patches 覆盖了？
ls package/libs/libubox/patches/ 2>/dev/null

# ④ 新库进 rootfs 了吗？
find build_dir/target-*/root-*/ -name 'libubox*'

# ⑤ 设备上是不是有旧的 .so 残留？
```

---

## 九、常见误区

**1. 在 `package/` 里 grep C 函数名。** 大部分包在 `package/` 下没有源码。要 grep 就去 `components/`（自研项目）或 `build_dir/`（构建后的第三方包）。

**2. 直接改 `build_dir/` 里的代码。** 会被 `make clean` 抹掉。用 quilt（第 5.3 节）。

**3. 以为 `components/` 就是构建用的代码。** 本树里它领先 44 个提交，且完全不参与构建。

**4. 用 `make world` 做增量验证。** 单包构建（`make package/foo/compile`）快几十倍。

**5. 试图通读构建系统。** `include/*.mk` 是给机器读的元编程层。用 `make V=s` 看展开结果，用 `make printdb` 查变量值，不要逐行推导宏。

**6. 忽略 `DEPENDS` 里 `+` 的有无。** `+foo` 是自动选中，`foo` 是硬依赖前提，混用会导致包在 menuconfig 里神秘消失。

**7. 忽略 ABI 版本。** `PKG_ABI_VERSION` 变了，所有依赖它的包都得重编，否则设备上会出现符号找不到。

**8. 混淆三套 CONFIG 命名空间。** 见第 4.4 节。

**9. 光读代码不看运行时。** `ubus -v list` + `ubus monitor` 十分钟能省下几小时的推理。

---

## 十、参考资料

### 官方

- OpenWrt 开发者指南：<https://openwrt.org/docs/guide-developer/start>
- 包 Makefile 参考：<https://openwrt.org/docs/guide-developer/packages>
- 构建系统概览：<https://openwrt.org/docs/techref/buildroot>
- ubus 文档：<https://openwrt.org/docs/techref/ubus>
- UCI 文档：<https://openwrt.org/docs/guide-user/base-system/uci>
- procd init 脚本：<https://openwrt.org/docs/guide-developer/procd-init-scripts>

### 上游仓库（本树 `components/` 对应的项目）

| 项目 | 仓库 |
|------|------|
| libubox | <https://github.com/openwrt/libubox> |
| ubus | <https://github.com/openwrt/ubus> |
| uci | <https://github.com/openwrt/uci> |
| procd | <https://github.com/openwrt/procd> |
| netifd | <https://github.com/openwrt/netifd> |
| rpcd | <https://github.com/openwrt/rpcd> |

官方 git 服务器：<https://git.openwrt.org/>（GitHub 上的是镜像）

### 本树内的文档与示例

| 位置 | 内容 |
|------|------|
| `components/libubox/examples/` | `ustream-example.c`、`json_script-example.c`、`uloop-example.lua`，学 API 最快的入口 |
| `components/libubox/tests/` | 单元测试（`test-avl.c`、`test-blobmsg-parse.c` 等）外加 `fuzz/`，是理解边界条件的好材料 |
| `package/system/procd/files/procd.sh` | 设备上 `/lib/functions/procd.sh` 的源头，init 脚本那套函数都在这里 |
| `components/*/README*` | 各项目自带说明 |
| `libubox-analysis.md` | 本仓库的 libubox 深度分析 |
| `package/network/services/hostapd/README.md` | OpenWrt 对 hostapd 改动的说明 |

### 邮件列表与补丁流程

- openwrt-devel：<https://lists.openwrt.org/mailman/listinfo/openwrt-devel>
- 提交格式：首行 `<子系统>: <小写简短描述>`（≤72 字符、无句号），正文说明**为什么**，必须有 `Signed-off-by`（用 `git commit -s`）
- 代码风格检查：`./scripts/checkpatch.pl`

---

## 附：一条建议的 30 天路径

| 阶段 | 内容 |
|------|------|
| 第 1–3 天 | 读 `list.h` + `uloop.h/c`，写一个用 uloop 的定时器+socket 小程序 |
| 第 4–6 天 | 读 `blob.h`/`blobmsg.h`，写一个 blobmsg 编解码程序 |
| 第 7–10 天 | 读 `libubus.h` + `libubus-obj.c`，写一个注册 ubus 方法的守护进程 |
| 第 11–14 天 | `make defconfig && make -j$(nproc)`，跑通一次完整构建，QEMU 启动 |
| 第 15–18 天 | 用 `ubus monitor` 观察真实系统，对照源码验证理解 |
| 第 19–22 天 | 读 uci + procd，写一个带 `/etc/init.d/` 脚本和 uci 配置的服务 |
| 第 23–26 天 | 给某个包加一个 quilt 补丁，走通完整的改-编-验流程 |
| 第 27–30 天 | 挑一个真实问题（bug 或小功能）端到端解决，向上游提一个 PR |
