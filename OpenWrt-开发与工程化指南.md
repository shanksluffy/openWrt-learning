# OpenWrt 开发与工程化指南

> 内容分两部分：上半部分是 OpenWrt SDK 的技术栈、构建原理与开发调试方法；下半部分是在此基础上，把开发流程标准化、自动化以提升质量和效率的落地建议。

## 目录

- [第一部分：技术栈与工程原理](#第一部分技术栈与工程原理)
  - [1. 先分清三个"SDK"](#1-先分清三个sdk)
  - [2. 构建系统原理](#2-构建系统原理)
  - [3. 运行时技术栈](#3-运行时技术栈)
  - [4. 深入开发的几条主线](#4-深入开发的几条主线)
  - [5. 调试手段](#5-调试手段)
  - [6. 厂商 SDK 的额外注意事项](#6-厂商-sdk-的额外注意事项)
- [第二部分：流程化与效率优化](#第二部分流程化与效率优化)
  - [7. 构建环境与基线管理](#7-构建环境与基线管理)
  - [8. 构建加速](#8-构建加速)
  - [9. 配置管理](#9-配置管理)
  - [10. CI 流水线](#10-ci-流水线)
  - [11. 自动化测试](#11-自动化测试)
  - [12. 调试能力工程化](#12-调试能力工程化)
  - [13. 日常迭代流程](#13-日常迭代流程)
  - [14. 知识沉淀](#14-知识沉淀)
  - [15. 落地顺序与度量指标](#15-落地顺序与度量指标)

---

# 第一部分：技术栈与工程原理

## 1. 先分清三个"SDK"

官方发布三种开发包，用途完全不同，容易被笼统地叫成 SDK：

| 形态 | 能做什么 | 典型用途 |
| --- | --- | --- |
| 完整源码树（buildroot） | 编译工具链、内核、所有包，产出固件 | 改内核、改 DTS、适配新板卡 |
| SDK（严格意义） | 预编译工具链 + 头文件 + 库，只能编包 | 只开发应用程序，不碰系统本身 |
| ImageBuilder | 无编译器，只重新打包已有的包 | 定制"预装哪些包" |

高通 QSDK、联发科 SDK 里的"SDK"是厂商叫法，实际是**完整源码树的厂商分支**，与官方定义的 SDK 不是一回事，沟通时注意区分。

## 2. 构建系统原理

OpenWrt 的构建系统叫 buildroot-ng，本质是**用 GNU make + kconfig 写的元构建系统**。

### 2.1 关键目录

- `dl/`：源码 tarball 下载缓存，可做内网镜像共享。
- `build_dir/`：实际编译目录。`host/` 存本机运行的工具，`target-<arch>_<libc>/` 存交叉编译产物。
- `staging_dir/`：交叉编译的"假根目录"。`toolchain-<arch>/` 是编译器与 libc，`target-<arch>/` 是各包 install 出的头文件和库。**排查链接错误先翻这里。**
- `target/linux/<平台>/`：内核配置、补丁、DTS、板卡定义。
- `package/`、`feeds/`：软件包定义。
- `bin/`：最终产物。
- `tmp/`：`.packageinfo`、`.targetinfo` 等元数据缓存。**改了包依赖或菜单位置却不生效时，删掉 `tmp/` 重新 menuconfig。**

### 2.2 编译流水线

整体顺序：host 工具 → 交叉工具链（binutils/gcc/libc）→ 内核 → 按依赖拓扑序编包 → 打包成 ipk/apk → `make image` 拼固件。

单个包内部固定分阶段：

```
prepare（解压 + 打补丁）→ configure → compile
  → install（装到 staging_dir，供其他包使用）
  → package/install（装进 ipk，供设备使用）
```

每阶段完成后在 build_dir 留 `.built`、`.configured` 等空标记文件，这是增量编译机制，也是"改了代码却没重新编译"的根因。解决办法是删除对应标记文件或执行 `make package/xxx/clean`。

### 2.3 包 Makefile 模板机制

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=myapp
PKG_VERSION:=1.0
PKG_RELEASE:=1
PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk

define Package/myapp
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=My application
  DEPENDS:=+libubox +libubus +libuci
endef

define Package/myapp/description
  详细描述，会显示在 menuconfig 里
endef

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
	$(CP) ./src/* $(PKG_BUILD_DIR)/
endef

define Build/Compile
	$(MAKE) -C $(PKG_BUILD_DIR) \
		CC="$(TARGET_CC)" CFLAGS="$(TARGET_CFLAGS)" \
		LDFLAGS="$(TARGET_LDFLAGS)"
endef

define Package/myapp/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/myapp $(1)/usr/bin/
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/myapp.init $(1)/etc/init.d/myapp
	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) ./files/myapp.config $(1)/etc/config/myapp
endef

$(eval $(call BuildPackage,myapp))
```

原理：`package.mk` 预定义了各阶段默认实现，用 `define` 覆盖需要定制的部分，最后 `$(eval $(call BuildPackage,...))` 展开成真正的 make 规则。`$(1)` 是构建系统传入的 install 目标目录。

按项目构建方式选择 include：`cmake.mk`、`autotools.mk`、`meson.mk`、`kernel.mk`（内核模块）。

### 2.4 feeds 机制

```bash
./scripts/feeds update -a      # 拉取/更新所有 feed
./scripts/feeds install -a     # 在 package/feeds/ 下建符号链接
./scripts/feeds install myapp  # 只装某个包
```

install 的本质是建软链接让主构建系统扫描到。**自研代码应做成独立的私有 feed**，不要直接塞进 `package/`，否则升级 OpenWrt 版本时必然冲突。

## 3. 运行时技术栈

这几个自研组件是一个整体，构成 OpenWrt 区别于普通 Linux 的核心：

- **libubox**：基础库。事件循环（uloop）、链表/哈希/AVL 树、blob/blobmsg 序列化、JSON 转换。其余组件都建在它之上。
- **ubus**：轻量系统 IPC 总线，作用类似 D-Bus。
- **uci**：统一配置系统，`/etc/config/` 下用统一的 `config`/`option`/`list` 语法。
- **procd**：init 系统（PID 1），负责服务启停、进程守护、hotplug 事件分发。
- **netifd**：网络配置守护进程，把 uci network 配置翻译成实际接口、路由和协议动作。
- **firewall4**：22.03 起取代 firewall3，底层由 iptables 换成 nftables，规则生成器用 ucode 编写。
- **ucode**：自研的类 JavaScript 脚本语言，正逐步取代 LuCI 中的 Lua。
- **包管理器**：opkg 用了十几年，较新版本已迁移到 **apk**（Alpine 的包管理器），带签名验证和更好的依赖解析。做版本升级时注意它会影响打包和 OTA 流程。
- **无线栈**：mac80211 框架 + 驱动（ath9k/ath11k/ath12k/mt76）+ hostapd/wpa_supplicant（合并为 wpad 包），上层是 `/etc/config/wireless` 与 `wifi` 脚本。

### 常用命令

```bash
# ubus
ubus list                          # 列出所有对象
ubus -v list network.interface     # 看某对象的方法和参数
ubus call network.interface.lan status
ubus monitor                       # 实时抓总线所有消息（调试神器）
ubus listen                        # 只看事件

# uci
uci show network
uci set network.lan.ipaddr='192.168.2.1'
uci commit network
/etc/init.d/network reload
```

### procd 风格的 init 脚本

```sh
#!/bin/sh /etc/rc.common
USE_PROCD=1
START=95

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/myapp -f
    procd_set_param respawn                  # 崩溃自动重启
    procd_set_param file /etc/config/myapp   # 配置变化自动 reload
    procd_set_param stdout 1                 # 日志接入 syslog，logread 可见
    procd_set_param stderr 1
    procd_close_instance
}
```

> `stdout 1` / `stderr 1` 常被遗漏，结果程序日志全部丢失，调试时无从下手。

## 4. 深入开发的几条主线

### 4.1 应用程序包

最省事的是本地源码模式：不写 `PKG_SOURCE`，在 `Build/Prepare` 里把 `./src` 拷进 build_dir。改完代码直接：

```bash
make package/myapp/{clean,compile} V=s
```

产物在 `bin/packages/<arch>/base/`，scp 到设备后 `opkg install` 或 `apk add --allow-untrusted`。更快的迭代方式是直接 scp 二进制覆盖 `/usr/bin/myapp`。

要与系统集成，还需写 uci 配置模板、procd init 脚本；需要被其他组件调用时用 libubus 的 `ubus_add_object` 注册 ubus 对象。

### 4.2 内核模块

include `$(INCLUDE_DIR)/kernel.mk`，包定义用 `KernelPackage/xxx`，结尾用 `$(eval $(call KernelPackage,xxx))`。构建系统会自动处理内核头文件路径、模块安装到 `/lib/modules/$(LINUX_VERSION)/`、生成 `kmod-xxx` 包名。

### 4.3 改内核：quilt 补丁流程

```bash
make target/linux/prepare V=s            # 解压内核并打上现有补丁
cd build_dir/target-*/linux-*/linux-6.6.x/
quilt new platform/999-my-fix.patch      # 新建补丁
quilt add drivers/net/xxx.c              # 声明要改哪些文件（必须先 add）
vim drivers/net/xxx.c
quilt refresh                            # 把改动写进补丁文件
cd -
make target/linux/refresh V=s            # 补丁同步回 target/linux/
make target/linux/compile V=s
```

要点：

- **改文件前必须先 `quilt add`**，否则 refresh 抓不到改动。
- 修改已有补丁用 `quilt push` / `quilt pop` 定位后再 `quilt refresh`。
- 直接在 build_dir 改代码而不走 quilt，下次 clean 就全部丢失。
- 改 DTS 同样走这套流程。

### 4.4 新增板卡支持

需要改动：`target/linux/<平台>/image/Makefile` 加设备定义（SoC、镜像大小、打包方式、DEVICE_PACKAGES）、写或改 DTS、在 `base-files/` 加板卡识别和默认网络配置（`/lib/functions/uci-defaults.sh`、`02_network`）、必要时改 U-Boot 引导参数。建议直接复制最相近的现有板卡再修改。

## 5. 调试手段

### 5.1 编译期

`V=s` 打印每条执行的命令，`V=sc` 额外带颜色。

工具链在 `staging_dir/toolchain-*/bin/`，加入 PATH 后可直接使用 `<arch>-openwrt-linux-gcc`、`-gdb`、`-objdump`、`-readelf`。

保留调试符号：menuconfig 里开 **Global build settings → Compile with debug information**（`CONFIG_DEBUG`）和 **Do not strip binaries**（`CONFIG_NO_STRIP`）；内核符号开 `CONFIG_KERNEL_DEBUG_INFO` 和 `KALLSYMS`。注意固件会明显变大，Flash 紧张时可只对目标包关闭 strip。

### 5.2 用户态调试

日志走 `logread`（procd 的 logd 环形缓冲）：`logread -f` 跟踪，`logread -e myapp` 过滤。

远程 gdb：

```bash
# 设备端
gdbserver :1234 /usr/bin/myapp
gdbserver :1234 --attach $(pidof myapp)      # 或附加到运行中的进程

# 主机端
staging_dir/toolchain-*/bin/<arch>-openwrt-linux-gdb
(gdb) set sysroot staging_dir/target-<arch>/
(gdb) file build_dir/target-*/myapp-1.0/myapp    # 带符号的未 strip 版本
(gdb) target remote <设备IP>:1234
```

> `set sysroot` 不做的话 gdb 找不到共享库符号，backtrace 全是问号。

其他工具：`strace`、`ltrace`、`perf`、`valgrind`（小内存设备可能跑不动）、设备本机 `gdb`。core dump 需设置 `ulimit -c unlimited` 和 `/proc/sys/kernel/core_pattern`。

组件间交互问题用 `ubus monitor` 基本一眼定位。

### 5.3 内核态调试

串口 console 是底线，硬件设计阶段务必留出串口引脚。

- `dmesg`、`printk`
- 动态调试：`echo 'file xxx.c +p' > /sys/kernel/debug/dynamic_debug/control`（需开 `CONFIG_DYNAMIC_DEBUG`）
- ftrace：`/sys/kernel/debug/tracing/`
- magic sysrq
- oops 还原源码行号：`<arch>-openwrt-linux-addr2line` 或 `scripts/decode_stacktrace.sh`
- 断点调试内核：**kgdb over serial**（启动参数加 `kgdboc=ttyS0,115200 kgdbwait`，主机端 gdb 加载 `vmlinux` 后 `target remote /dev/ttyUSB0`）或 JTAG（OpenOCD）

### 5.4 快速迭代技巧

反复刷 Flash 效率低且伤寿命，替代方案：

- **initramfs 镜像**（menuconfig 勾 `Target Images → ramdisk`）+ U-Boot `tftpboot` 从网络加载到内存启动，不写 Flash。
- **NFS rootfs**：根文件系统挂在开发机上，改完文件设备上直接生效。

刷坏了靠 **failsafe 模式**（启动时按 reset）或 U-Boot 的 tftp recovery 恢复。

### 5.5 无线专项调试

- `iw dev` / `iw list` / `iw event -t`：mac80211 层状态
- `iwinfo`：OpenWrt 的封装
- `hostapd_cli` / `wpa_cli`：交互式查询和控制
- 详细日志：手动跑 `hostapd -dd /var/run/hostapd-phy0.conf`，可看到完整的关联、认证、EAPOL 四次握手过程
- 空口抓包：`iw` 建 monitor 接口 + `tcpdump`，排查漫游和认证失败的终极手段
- mac80211 debugfs：`/sys/kernel/debug/ieee80211/phy0/`，看速率控制、统计计数、per-station 信息

## 6. 厂商 SDK 的额外注意事项

高通 QSDK / 联发科 SDK 相比官方 OpenWrt 的差异：

- 内核版本和 OpenWrt 版本**锁死**，不要指望升级，厂商补丁按特定版本制作。
- 目录结构可能被改造，多出 `qca/`、`mtk/` 之类的 feed。
- 转发加速模块（高通 NSS、联发科 PPE/hnat）通常以二进制或受限源码提供。

> **最常见的坑**：自定义转发逻辑绕过加速引擎后，功能测试全过，但性能测试发现吞吐掉了 80%，原因是报文被踢出硬件加速路径走了慢速路。

对策：

1. 开发前先搞清**加速路径的边界**——哪些流量会被 offload、什么条件下 fallback 到 CPU、如何在加速引擎里注册自己的规则。这部分文档往往不全，需要读代码并找 FAE 确认。
2. 版本管理上把厂商 SDK 作为基线，自研改动全部以 patch 或独立 feed 维护，**绝不直接改基线代码**，否则厂商发新版时合并会是灾难。

---

# 第二部分：流程化与效率优化

投入产出比最高的四件事：**构建环境容器化、配置用 diffconfig 管理、CI 里加性能回归门禁、调试符号归档**。这四项做完，团队里最常见的"我这能编你那不能编""刷了一版发现吞吐掉了但不知道哪次改的""线上崩了拿到栈但解不出来"基本就消失了。

## 7. 构建环境与基线管理

### 7.1 容器化构建环境

OpenWrt 对 host 侧 gcc 版本、Python 版本、各种 `-dev` 包有隐式依赖，厂商 SDK 尤其挑剔（老 QSDK 在新发行版上编不过是常态）。把环境固化成 Dockerfile，镜像 tag 与 SDK 版本绑定，本地和 CI 用同一镜像。

收益不只是"能不能编"，更是**构建可复现性**——半年后要在旧版本上出补丁版本时，仍能编出字节级接近的固件。

### 7.2 三层代码结构

强制约定 **厂商基线 / 自研 feed / 补丁集** 三者分离：

- 厂商 SDK 作为 vendor 分支原样导入，永不直接修改。
- 自研代码全部放独立的私有 feed 仓库，通过 `feeds.conf` 引入。
- 必须改基线的地方一律走 patch 文件。

配套建立**补丁台账**，每条 patch 记录：为什么改、改了什么、是否已推给厂商、厂商哪个版本会合入。没有台账，SDK 升版本时几十个 patch 冲突，没人说得清哪些还需要、哪些可以丢。

### 7.3 制品可追溯

固件内嵌 git commit id、构建时间、构建号，放到 `/etc/openwrt_release` 或自定义的 `/etc/build_info`，设备上一条命令可查。CI 归档用同一标识命名。"这台设备跑的到底是哪版代码"不该靠猜。

## 8. 构建加速

全量编译动辄一两小时，是效率的最大黑洞。按收益排序：

1. **ccache**：menuconfig 里开 `CONFIG_CCACHE`，缓存目录挂到容器外持久化，重复构建省一半以上时间。
2. **dl/ 内网镜像**：内部源码缓存服务器或共享 NFS。既提速，又避免上游源码消失导致老版本编不出来（这事真的会发生）。
3. **分层编译**：工具链和内核很少变，把 `staging_dir` + `build_dir/toolchain-*` 做成缓存产物。CI 上以"toolchain 相关文件哈希"为缓存 key，命中直接复用。
4. **给应用开发者分发 SDK**：多数写业务功能的同事不需要编内核。用 `make target/sdk/compile` 生成官方形态的 SDK 包发到内部制品库，他们的编译时间从一小时降到一分钟——这是团队级提效。
5. **清理策略脚本化**：把"什么时候该 clean 什么"写成脚本而非口口相传。改包源码用 `make package/x/{clean,compile}`；改包 Makefile 元数据要 `rm -rf tmp/`；改 kconfig 要 `make defconfig`。新人在这上面浪费的时间超乎想象。

## 9. 配置管理

**不要把完整 `.config` 提交进仓库**——上万行、随 kconfig 变动疯狂 diff、合并必冲突。改用差异化配置：

```bash
./scripts/diffconfig.sh > configs/ap-ipq6018.config
```

生成的只有几十行有效差异。每个产品型号一个 seed 文件，构建时：

```bash
cp configs/ap-ipq6018.config .config && make defconfig
```

配套写 `build.sh <product>` 把流程封起来，CI 和本地共用同一入口。这样"产品 A 和产品 B 配置差在哪"就变成一次 diff 能看清的事。

## 10. CI 流水线

分两级设计：

**MR 级（要求快，10 分钟内出结果）**：静态检查 + 增量编译。

- shellcheck 查 init 脚本和 uci-defaults 脚本
- clang-format 或 checkpatch 查 C 代码风格
- cppcheck 或 clang static analyzer 查空指针和越界
- 自定义检查：**扫描是否有人直接改了厂商基线目录**
- 编译只编本次改动涉及的包

**每日 / 合入主干级**：全量编译所有产品型号、产出固件、跑板级自动化测试、归档制品和符号。

**编译告警必须治理**：定一个基线告警数，只允许下降不允许上升；新增模块直接上 `-Wall -Wextra -Werror`。OpenWrt 环境下 warning 淹没在几万行日志里，不设门禁等于没有。

## 11. 自动化测试

这是从"能开发"到"开发质量高"的分水岭，也是 OpenWrt 项目里最容易被跳过的部分。

### 11.1 单元测试要能脱离硬件

业务逻辑与硬件/驱动交互分层，纯逻辑部分（配置解析、状态机、协议编解码）编成 host native 测试程序，在 CI 容器里直接跑，用 CMocka、Unity 之类的轻量框架，秒级反馈。做不到这一点，说明分层本身有问题。

### 11.2 板级测试台架（test farm）

基本配置：几块目标板 + USB 转串口 + **可远程控制的电源**（网络 PDU 或继电器板）+ TFTP/NFS 服务器 + 一台调度机。

有了它，CI 就能自动断电上电、U-Boot 里自动 tftp 加载新固件、串口捕获完整启动日志、跑测试脚本、失败自动抓日志归档。刷坏了断电重来，无需人工干预。

**冒烟测试**至少覆盖：能否正常启动到 shell、关键进程是否都在、`ubus list` 里自研对象是否注册齐全、`/etc/config` 配置是否正确加载、网口是否 up、内存占用是否在阈值内。

### 11.3 无线专项自动化

AP 项目必备：用支持 STA 模式的网卡或专门的测试仪作为客户端，脚本化执行关联/去关联、各种加密方式、多 SSID、VLAN、漫游切换用例。吞吐用 iperf3 打，覆盖上下行、TCP/UDP、不同带宽和空间流配置。

### 11.4 性能回归门禁（最值得做的一条）

厂商 SDK 上最典型的坑是"报文掉出硬件加速路径导致吞吐腰斩"，可怕之处在于**功能测试完全发现不了**，往往到系统测试甚至客户现场才暴露，回溯定位极其痛苦。

对策：把吞吐、CPU 占用、内存占用做成**带基线的自动化用例**，每日构建跑一次，偏离基线超过阈值（如 5%）就报警并阻断。任何一次让性能掉下去的提交，第二天就能被抓到，二分定位只需看一天的提交。

配套的还有：

- 长稳测试中定期采集 `/proc/meminfo`、`slabtop`、进程 RSS，抓内存泄漏。
- Flash 剩余空间和镜像大小门禁——镜像超出分区大小若到发布前才发现就来不及了。

## 12. 调试能力工程化

### 12.1 双轨镜像

每次构建同时出 **release 镜像**（strip、精简、体积优化）和 **debug 镜像**（带符号、内置 gdbserver、strace、tcpdump、valgrind）。开发和测试默认用 debug 版，性能测试和发布用 release 版。不要临时改 menuconfig 重编，既慢又不可复现。

### 12.2 符号归档

每个版本的构建产物必须归档：`vmlinux`、所有未 strip 的用户态二进制、`System.map`、内核 `.config`。配一个解析脚本，输入版本号 + 崩溃栈地址，自动 `addr2line` 出源码行号。

价值在现场问题回来时体现——没有符号归档，一个客户现场的 oops 日志就是一堆没用的十六进制数。

### 12.3 崩溃信息自动落盘

- 内核侧开 **pstore/ramoops**，panic 日志写入保留内存，重启后从 `/sys/fs/pstore/` 读出。
- 用户态设置 `core_pattern`，把 core dump 收集到指定分区并限制大小。
- 加开机自检脚本，发现上次有崩溃就自动上报或打日志。

设备现场重启一次什么线索都没有，是最常见的排查困境。

### 12.4 一键调试脚本

把"启动 gdbserver + 主机端 gdb 加载符号 + set sysroot + connect"封成一条命令，参数只要设备 IP 和进程名。这几步每人每次手敲，错一步就是十分钟的困惑。

## 13. 日常迭代流程

写成团队标准并配好脚本：

- **开发期一律用 NFS rootfs 或 initramfs 网络启动**，改完文件立即生效，不刷 Flash。单次迭代从五分钟压到十秒，且不消耗 Flash 寿命。
- **一键部署**：`deploy.sh myapp <device-ip>` 完成编译、传输、重启服务、跟踪日志。
- **设备资源池**：哪怕只是共享表格或简单的占用锁脚本，也比"谁在用 3 号板"在群里喊强。

## 14. 知识沉淀

两份文档的收益远高于其他所有文档：

1. **厂商 SDK 加速路径边界说明**——哪些流量走硬件、什么条件下 fallback、如何在加速引擎里注册规则。这是原厂文档最不全、踩坑最多、口口相传最易失真的部分，必须写下来并随版本更新。
2. **新板卡适配 checklist**——从 DTS、board 识别、默认网络配置、镜像打包规则、U-Boot 参数到冒烟验证，逐项打勾。适配新板卡是低频高难度动作，每次靠回忆必然遗漏。

再加一份新人上手指南（环境搭建到编出第一个包），可省掉大量重复的一对一指导。

## 15. 落地顺序与度量指标

从零开始的推进顺序，每阶段都能独立产生收益：

- **第一阶段（1–2 周）**：构建环境容器化、ccache 和 dl 缓存、diffconfig 配置管理、`build.sh` 统一入口。纯工程配置，见效最快，是后续一切的地基。
- **第二阶段（2–4 周）**：搭 CI，先做静态检查加全量编译，定下编译告警基线；同时建立符号归档和双轨镜像。
- **第三阶段（1–2 月）**：搭板级测试台架，先跑通自动刷机和冒烟测试，再逐步补无线功能用例。
- **第四阶段**：上性能回归门禁和长稳测试，需要前面的台架做基础，但它是防住最贵的那类缺陷的关键。
- **贯穿全程**：代码分层规范和补丁台账。越早立规矩越好，晚了改造成本翻倍。

衡量效果的指标：

- 全量构建时长 / 增量构建时长
- CI 从提交到出结果的时间
- 编译告警数
- 冒烟通过率
- 性能相对基线的偏差
- 现场问题的平均定位时长
