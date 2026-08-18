# OpenWrt 构建工程化：容器化 · 缓存 · 配置管理 · 统一入口

> 把 OpenWrt 的构建从"能跑"变成"可复现、可协作、可进 CI"的四件事：构建环境容器化、ccache 与 dl 缓存、基于 diffconfig 的配置管理、`build.sh` 统一入口。
>
> 配套文档：[`ac-ap-software-architecture.md`](./ac-ap-software-architecture.md)（软件分层）、[`openwrt-reading-guide.md`](./openwrt-reading-guide.md)（代码阅读路线）
>
> 代码引用基于本仓库（OpenWrt 25.12 分支）。方案的前提是**尽量不改上游构建系统**——四项能力上游都已留好接入点，我们只做外围封装。

## 目录

- [零、总体设计](#零总体设计)
- [一、构建环境容器化](#一构建环境容器化)
- [二、ccache 与 dl 缓存](#二ccache-与-dl-缓存)
- [三、diffconfig 配置管理](#三diffconfig-配置管理)
- [四、build.sh 统一入口](#四buildsh-统一入口)
- [五、执行顺序清单](#五执行顺序清单)
- [六、验收标准](#六验收标准)
- [七、已知坑](#七已知坑)

---

## 零、总体设计

### 0.1 先看仓库现状

动手之前先确认上游已经给了什么，避免重复造轮子：

| 现状 | 位置 | 对方案的影响 |
|---|---|---|
| 官方 buildbot 容器镜像已被引用 | `.devcontainer/ci-env/devcontainer.json` | 直接复用为基础镜像，不用从 Debian 自己装依赖 |
| ccache 接线完备 | `rules.mk:347`、`tools/Makefile:158`、`config/Config-devel.in:135` | 只需开开关 + 挂目录，不改代码 |
| `diffconfig` 脚本已存在 | `scripts/diffconfig.sh` | 配置管理的核心工具现成 |
| 缓存目录已被忽略 | `.gitignore` 含 `/dl`、`.ccache`、`/.config`、`/feeds.conf` | 用 bind mount 覆盖进仓库内是干净的 |
| dl 镜像源机制 | `scripts/download.pl`、`scripts/projectsmirrors.json` | 内网加速走 `DOWNLOAD_MIRROR` 环境变量即可 |

两个必须提前知道的反直觉点：

1. **`rules.mk` 会覆盖外部的 `CCACHE_DIR`**（`export CCACHE_DIR:=` 是无条件赋值），所以从 docker 传这个环境变量无效。
2. **`make diffconfig` 不输出到 stdout**，而是写 `$(BIN_DIR)/config.buildinfo`，做配置管理要直接调 `scripts/diffconfig.sh`。

### 0.2 新增文件清单

全部可入 git：

```
docker/Dockerfile          # 构建镜像：官方 buildworker + UID 对齐
.dockerignore
build.sh                   # 统一入口，容器内外同一份
configs/base.config        # 公共 kconfig 片段
configs/fragments/*.config # 可复用特性片段
configs/<profile>.config   # 每个产品形态一个最小种子配置
board/<profile>/files/     # 每个形态的 rootfs 覆盖文件（可选）
```

### 0.3 运行期数据流

```
宿主 $HOME/.cache/openwrt/dl      ──bind──> <repo>/dl        （跨分支/跨 target 共享）
宿主 $HOME/.cache/openwrt/ccache  ──bind──> <repo>/.ccache   （跨 target 共享）
宿主 <repo>                       ──bind──> 容器内同一绝对路径（关键约束，见 1.1）

configs/<profile>.config ──assemble──> .config ──make defconfig──> 完整 .config
完整 .config ──scripts/diffconfig.sh──> configs/<profile>.config    （回写闭环）
```

---

## 一、构建环境容器化

### 1.1 原理

**为什么要容器化。** OpenWrt 对宿主机有大量隐式依赖：host gcc/g++ 版本（`config/check-hostcxx.sh` 就是在挡这个）、perl 模块、`gawk` 而非 `mawk`、GNU `getopt`、libncurses、locale。这些不一致时的表现是构建到一半某个包 configure 失败，排查成本极高。

**最关键的约束：路径。** OpenWrt 的 `staging_dir` 里存着编译好的 toolchain，绝对路径被硬编码进了 gcc specs、libtool 的 `.la` 文件、pkg-config 的 `.pc` 文件以及各种 sysroot 引用。同时顶层 Makefile 明确禁止路径含空格：

```5:13:Makefile
TOPDIR:=${CURDIR}
LC_ALL:=C
LANG:=C
TZ:=UTC
export TOPDIR LC_ALL LANG TZ

empty:=
space:= $(empty) $(empty)
$(if $(findstring $(space),$(TOPDIR)),$(error ERROR: The path to the OpenWrt directory must not include any spaces))
```

如果容器里挂到 `/builder` 而宿主是 `/home/m11wang/openwrt/openwrt`，那么容器构建完的树在宿主上直接 `make` 会炸，反之亦然，只能 `make dirclean` 重来。

> **结论：把宿主仓库路径原样挂到容器内同一个绝对路径。** 这样容器与宿主可以自由混用同一棵树，代价是零。这一条比镜像里装什么都重要。

**第二个坑：UID。** 容器内以 root 构建会在宿主留下 root 所有的 `build_dir/`、`staging_dir/`、`bin/`，之后宿主的 IDE、git、清理脚本全部失效。必须做 UID/GID 对齐。

**不需要的东西：`--privileged` 和 loop 设备。** OpenWrt 生成 squashfs/ubifs/ext4 全部用用户态 mkfs 工具（`mksquashfs`、`genext2fs`），不 mount 任何东西，所以普通非特权容器足够。这点和很多发行版镜像构建不同。

### 1.2 步骤

**1) 写 `docker/Dockerfile`**

```dockerfile
FROM ghcr.io/openwrt/buildbot/buildworker-v3.8.0:v9

ARG UID=1000
ARG GID=1000
ARG USER=builder

USER root

# 把镜像自带的 buildbot 用户改名/改号到与宿主一致；
# 若该 UID/GID 已被占用则复用，否则新建。
RUN set -eux; \
    if getent group "${GID}" >/dev/null; then \
        groupmod -n "${USER}" "$(getent group "${GID}" | cut -d: -f1)"; \
    else \
        groupadd -g "${GID}" "${USER}"; \
    fi; \
    if getent passwd "${UID}" >/dev/null; then \
        old="$(getent passwd "${UID}" | cut -d: -f1)"; \
        [ "$old" = "${USER}" ] || usermod -l "${USER}" -g "${GID}" "$old"; \
    else \
        useradd -u "${UID}" -g "${GID}" -m -s /bin/bash "${USER}"; \
    fi; \
    mkdir -p /home/${USER} && chown "${UID}:${GID}" /home/${USER}

# 容器身份标记：build.sh 靠它判断自己是否已在容器内，避免嵌套
RUN echo "openwrt-builder" > /.owrt-container

USER ${USER}
ENV LC_ALL=C LANG=C
```

配套 `.dockerignore`（镜像构建不需要任何仓库内容，全部排除即可让 context 为空、秒建）：

```
*
```

**2) 构建镜像（每台机器一次）**

```bash
docker build -t openwrt-builder:25.12 \
  --build-arg UID=$(id -u) --build-arg GID=$(id -g) \
  -f docker/Dockerfile docker/
```

**3) 运行时的挂载与参数**（`build.sh` 会封装，这里说明每一项的理由）

```bash
docker run --rm -it \
  --user "$(id -u):$(id -g)" \
  -v "$TOPDIR:$TOPDIR" \
  -v "$CACHE_ROOT/dl:$TOPDIR/dl" \
  -v "$CACHE_ROOT/ccache:$TOPDIR/.ccache" \
  -v "$CACHE_ROOT/home:/home/builder" \
  -e HOME=/home/builder \
  -e CCACHE_MAXSIZE=50G \
  -w "$TOPDIR" \
  openwrt-builder:25.12 ./build.sh "$@"
```

| 参数 | 理由 |
|---|---|
| `-v "$TOPDIR:$TOPDIR"` | 满足 1.1 的路径一致约束，容器/宿主可混用同一棵树 |
| `--user` | 双保险：即使镜像烘的 UID 与当前不符（CI runner），文件所有权仍正确 |
| `-e HOME=...` + 对应挂载 | 必须给可写 HOME，否则 `git`（`scripts/getver.sh` 要调）和 ccache 在只读 `/` 上失败 |
| `CCACHE_MAXSIZE` | `rules.mk` 不碰这个变量，是从外部调容量的干净入口（见 2.1） |

用 podman 时改用 `--userns=keep-id` 替代 `--user`。

**4) 更新 devcontainer 复用同一镜像**

把 `.devcontainer/ci-env/devcontainer.json` 改成 build 本地 Dockerfile，并让工作区挂到与宿主同路径：

```json
{
  "name": "OpenWrt build",
  "build": {
    "dockerfile": "../../docker/Dockerfile",
    "args": { "UID": "${localEnv:UID}", "GID": "${localEnv:GID}" }
  },
  "remoteUser": "builder",
  "updateRemoteUserUID": true,
  "workspaceMount": "source=${localWorkspaceFolder},target=${localWorkspaceFolder},type=bind",
  "workspaceFolder": "${localWorkspaceFolder}",
  "mounts": [
    "source=${localEnv:HOME}/.cache/openwrt/dl,target=${localWorkspaceFolder}/dl,type=bind",
    "source=${localEnv:HOME}/.cache/openwrt/ccache,target=${localWorkspaceFolder}/.ccache,type=bind"
  ],
  "containerEnv": { "CCACHE_MAXSIZE": "50G" }
}
```

`workspaceMount` 用 `${localWorkspaceFolder}` 同时做 source 和 target，正是为了满足路径一致这条硬约束。

---

## 二、ccache 与 dl 缓存

### 2.1 ccache 原理

接入点上游已经做完，不需要改代码：

```347:355:rules.mk
ifneq ($(CONFIG_CCACHE),)
  TARGET_CC:= ccache $(TARGET_CC)
  TARGET_CXX:= ccache $(TARGET_CXX)
  HOSTCC:= ccache $(HOSTCC)
  HOSTCXX:= ccache $(HOSTCXX)
  export CCACHE_NOCOMPRESS:=true
  export CCACHE_BASEDIR:=$(TOPDIR)
  export CCACHE_DIR:=$(if $(call qstrip,$(CONFIG_CCACHE_DIR)),$(call qstrip,$(CONFIG_CCACHE_DIR)),$(TOPDIR)/.ccache)
endif
```

三个要点：

1. **`CCACHE_BASEDIR=$(TOPDIR)`** 会把 TOPDIR 下的绝对路径重写成相对路径再算哈希，所以换目录、换 worktree 也能命中缓存。这是它能跨 target、跨分支复用的基础。
2. **`export CCACHE_DIR:=` 是无条件赋值 + 导出**，会覆盖 docker 传进来的同名环境变量。想改位置只有两条路：改 `.config` 里的 `CONFIG_CCACHE_DIR`，或者把宿主缓存 bind-mount 到 `<repo>/.ccache`。**选后者**——`.config` 里不出现机器相关的绝对路径，种子配置才能在团队和 CI 之间通用。
3. **命令行里写的是裸 `ccache`**，走 `PATH`，用的是 `tools/ccache` 编译出来的 `staging_dir/host/bin/ccache`（`tools/Makefile:158` 以 `CONFIG_CCACHE` 为条件），不是宿主的 ccache。所以第一次必须先让 tools 阶段跑一遍。

**`CONFIG_CCACHE` 藏在 DEVEL 菜单下**：

```135:145:config/Config-devel.in
	config CCACHE
		bool "Use ccache" if DEVEL
		help
		  Compiler cache; see https://ccache.samba.org/

	config CCACHE_DIR
		string "Set ccache directory" if CCACHE
		default ""
		help
		  Store ccache in this directory.
		  If not set, uses './.ccache'
```

必须先 `CONFIG_DEVEL=y` 才能设 `CONFIG_CCACHE`。好消息是 `diffconfig.sh` 显式把 `CONFIG_DEVEL=y` 保留进种子配置，所以它不会在配置往返中丢掉。

**ccache 收益在哪。** 增量改一个包时收益为零（`build_dir` 的 stamp 机制已经跳过了编译）。真正的收益场景是：`make dirclean` 后重建、切 target/subtarget、切分支导致 toolchain 重建、CI 上的干净构建。这些场景里 toolchain（gcc/binutils，走 `HOSTCC`）和大包（openssl、boost、gcc 自身）是大头，命中率高时全量构建能省一半以上时间。

> 别拿 ccache 当"改一行重编快"的方案，那是 `make package/xxx/{clean,compile}` 的活。

### 2.2 dl 缓存原理

`dl/` 存的是带 `PKG_HASH` 校验的源码 tarball，文件名含版本号，内容与 target、架构、配置**完全无关**，天然全局可共享：

```149:149:rules.mk
DL_DIR=$(if $(call qstrip,$(CONFIG_DOWNLOAD_FOLDER)),$(call qstrip,$(CONFIG_DOWNLOAD_FOLDER)),$(TOPDIR)/dl)$(if $(DL_SUBDIR),/$(DL_SUBDIR))
```

同样，用 bind mount 而不是 `CONFIG_DOWNLOAD_FOLDER`，理由和 ccache 一致：保持 `.config` 与机器无关。

**并发是唯一的真坑。** `scripts/download.pl` 下载到 `$target/$filename.dl` 再改名，这个临时文件名不带随机后缀。两个构建同时下载同一个包（多分支并行、CI 多 job 共享缓存卷）会互相踩。缓解办法：把 `make download` 单独提出来做预取，并用 `flock` 串行化；构建阶段就只读命中，不再触发下载。

### 2.3 步骤

**1) 建缓存目录（每台机器一次）**

```bash
mkdir -p ~/.cache/openwrt/{dl,ccache,home}
```

**2) 打开开关**（进 `configs/base.config`，见第三章）

```
CONFIG_DEVEL=y
CONFIG_CCACHE=y
```

不要设 `CONFIG_CCACHE_DIR` 和 `CONFIG_DOWNLOAD_FOLDER`，保持默认值 `$(TOPDIR)/.ccache` 和 `$(TOPDIR)/dl`，由挂载接管。

**3) ccache 调参**：写 `~/.cache/openwrt/ccache/ccache.conf`

```ini
max_size = 50G
sloppiness = locale,time_macros,include_file_mtime,include_file_ctime,file_stat_matches
```

- **容量是第一优先级。** 默认 5G 在 OpenWrt 场景下会不断淘汰，命中率上不去。
- `compression` 不用配，`rules.mk` 已强制 `CCACHE_NOCOMPRESS=true`（换 CPU 时间为磁盘）。
- `sloppiness` 里 `time_macros` 允许含 `__DATE__`/`__TIME__` 的文件命中，代价是这类时间戳不再准确——对日常固件构建可接受，但做可复现构建时应去掉，改用 `scripts/get_source_date_epoch.sh` + `SOURCE_DATE_EPOCH`。
- **不建议开 `hash_dir = false`**：它能进一步提升跨路径命中，但会让 `__FILE__` 和调试信息里的路径失真，gdb / 远程调试会受影响。

**4) 首次初始化 ccache 二进制**

```bash
make tools/ccache/compile              # 产出 staging_dir/host/bin/ccache
staging_dir/host/bin/ccache -M 50G     # 兜底，若没写 ccache.conf
```

**5) 预取 dl（带锁）**

```bash
flock ~/.cache/openwrt/dl.lock make download -j"$(nproc)"
```

**6) 观测与回收**

| 动作 | 命令 | 说明 |
|---|---|---|
| 看命中率 | `make world` 结尾自带，或 `staging_dir/host/bin/ccache -s` | 见 `Makefile:137-139` |
| 清 ccache | `make cacheclean` | 内部就是 `ccache -C`，见 `Makefile:75-78` |
| 清 dl 老版本 | `scripts/dl_cleanup.py -d dl` | 按包名只保留最新版本；**先用 `-d` 干跑确认删除清单**，再去掉 `-d` 实删 |

**7) 国内网络加速（可选）**

`scripts/download.pl` 读 `DOWNLOAD_MIRROR` 环境变量，镜像列表在 `scripts/projectsmirrors.json`。企业内网可以搭一个 HTTP 镜像，然后 `docker run` 加 `-e DOWNLOAD_MIRROR=https://mirror.corp/openwrt/sources`。同一处还支持 `CURL_OPTIONS`/`WGET_OPTIONS`（走代理）和 `DOWNLOAD_CHECK_CERTIFICATE`。

---

## 三、diffconfig 配置管理

### 3.1 原理

完整 `.config` 有 4000+ 行，其中 99% 是 kconfig 从 target 选择推导出的默认值。直接把它入 git 有三个致命问题：review 不可读；上游改一个 default 就产生几百行无意义 diff；不同 target 之间无法复用。

`scripts/diffconfig.sh` 解决的正是这个——输出**相对于 defconfig 的最小增量**，通常几十行。它的算法值得理解，因为决定了你能怎么用它：

```3:15:scripts/diffconfig.sh
grep \^CONFIG_TARGET_ .config | head -n3 > tmp/.diffconfig.head
grep \^CONFIG_TARGET_DEVICE_ .config >> tmp/.diffconfig.head
grep '^CONFIG_ALL=y' .config >> tmp/.diffconfig.head
grep '^CONFIG_ALL_KMODS=y' .config >> tmp/.diffconfig.head
grep '^CONFIG_ALL_NONSHARED=y' .config >> tmp/.diffconfig.head
grep '^CONFIG_DEVEL=y' .config >> tmp/.diffconfig.head
grep '^CONFIG_TOOLCHAINOPTS=y' .config >> tmp/.diffconfig.head
grep '^CONFIG_BUSYBOX_CUSTOM=y' .config >> tmp/.diffconfig.head
grep '^CONFIG_TARGET_PER_DEVICE_ROOTFS=y' .config >> tmp/.diffconfig.head
./scripts/config/conf --defconfig=tmp/.diffconfig.head -w tmp/.diffconfig.stage1 Config.in >/dev/null
./scripts/kconfig.pl '>+' tmp/.diffconfig.stage1 .config >> tmp/.diffconfig.head
./scripts/config/conf --defconfig=tmp/.diffconfig.head -w tmp/.diffconfig.stage2 Config.in >/dev/null
./scripts/kconfig.pl '>' tmp/.diffconfig.stage2 .config >> tmp/.diffconfig.head
```

1. **head**：先固定 target/subtarget/device 三行 + 若干"元开关"（`ALL`、`DEVEL`、`TOOLCHAINOPTS`、`BUSYBOX_CUSTOM`、`PER_DEVICE_ROOTFS`）。这些必须先定，因为它们会展开出巨量的默认值。
2. **stage1**：拿 head 跑一次 `conf --defconfig`，得到"只选了 target 时的完整默认配置"，再用 `kconfig.pl '>+'` 求出你的 `.config` 里与之不同的符号，追加进 head。
3. **stage2**：因为第 2 步新加的符号又会改变别的符号的默认值，所以**再展开一次**，用 `'>'` 捞出残余差异。

> 这就是为什么种子配置**必须由脚本生成、不能手写重排**——它的行序隐含依赖关系。也解释了为什么两轮就够：kconfig 的默认值依赖深度在 OpenWrt 里实际不超过两层。

注意 `make diffconfig` 这个 target 是写文件而非 stdout：

```123:125:Makefile
diffconfig: FORCE
	mkdir -p $(BIN_DIR)
	$(SCRIPT_DIR)/diffconfig.sh > $(BIN_DIR)/config.buildinfo
```

所以配置管理要直接调 `./scripts/diffconfig.sh`。

**关于 `scripts/env`。** 上游自带一个 git 支持的环境管理器（`scripts/env new/switch/save`，管 `.config` + `files/`），`.gitignore` 里的 `/env` 就是它。它适合单人在一棵树上频繁切换多套配置。但主线建议用 `configs/*.config` 明文种子：能在 PR 里 review、CI 能直接消费、能做片段组合。两者不冲突，`scripts/env` 可以当个人临时工具用。

### 3.2 步骤

**1) 目录结构**

```
configs/
  base.config                    # 所有 profile 共享
  fragments/
    ai-telemetry.config          # 遥测 agent 相关包
    debug.config                 # gdbserver/perf/strace + CONFIG_DEBUG
  ap-mt7981.config               # 具体产品形态（扁平，由 diffconfig 生成）
  ap-ipq807x.config
```

`configs/base.config`：

```
CONFIG_DEVEL=y
CONFIG_CCACHE=y
CONFIG_BUILD_LOG=y
CONFIG_AUTOREMOVE=y
```

**2) 片段组合机制**

profile 文件里允许一行 `#include fragments/xxx.config`，`build.sh` 递归展开后拼接，再 `make defconfig`。`#` 开头对 kconfig 是注释，所以即使不展开也不会出错，向后兼容。

**3) 从零创建一个 profile**

```bash
./build.sh config ap-mt7981     # 若种子不存在则从 base 起
./build.sh menuconfig           # 交互勾选 target/device/packages
./build.sh save ap-mt7981       # diffconfig.sh 回写最小种子
git diff configs/               # 应该是几十行可读的增量
```

**4) 消费一个 profile**

```bash
./build.sh config ap-mt7981     # 组装 → cp .config → make defconfig
./build.sh build ap-mt7981
```

**5) 关键的校验步骤（`config-check`）**

`make defconfig` 会**静默丢弃**不可满足的符号——依赖没选中、feed 没 install、上游改名了，它都不报错，只是那一行悄悄消失。这是这套方案最容易踩的坑：你以为装了某个包，构建出来固件里没有。

所以 `build.sh config` 之后必须自动做一次回查：把组装出的种子里每一行 `CONFIG_X=y` 在展开后的 `.config` 里逐行核对，缺失的打印为 `DROPPED`。这个检查在 CI 里应该是硬失败。

```bash
while IFS= read -r line; do
  case "$line" in
    CONFIG_*=*) grep -qxF -- "$line" .config || echo "DROPPED: $line" ;;
  esac
done < tmp/seed.config
```

**6) `save` 的防护**

`diffconfig.sh` 输出是扁平的，会冲掉 `#include` 结构。规则定为：**扁平种子是每个 profile 的唯一真相来源，片段只用于共享基座**；`save` 时如果目标文件含 `#include`，写到 `.new` 并提示人工归并，不直接覆盖。

**7) rootfs 覆盖文件**

`.gitignore` 忽略了 `/files` 和 `/overlay`，所以自定义文件要放在跟踪目录里，比如 `board/<profile>/files/`，由 `build.sh config` 同步到 `files/`。

---

## 四、build.sh 统一入口

### 4.1 原理

三个设计要点：

1. **容器内外同一份脚本，靠 re-exec 自举。** 脚本启动先检测 `/.owrt-container` 标记文件（比环境变量可靠，`sudo`/`su` 也不会丢）。不在容器里就把自己连同原始参数丢进 `docker run` 再跑一遍。好处是使用者永远只记一条命令，不需要知道自己在哪。
2. **失败自动降级复现。** OpenWrt 的 `-j42` 并行输出交错，几乎无法定位。标准做法是失败后用 `-j1 V=sc` 重跑到出错点。把这个动作自动化，能省掉每次失败后的手工重试。
3. **共享缓存加锁。** `dl` 的 `.dl` 临时文件、`staging_dir` 都不耐并发，用 `flock` 保证同一份缓存串行。

### 4.2 脚本

```bash
#!/usr/bin/env bash
set -euo pipefail

TOPDIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
IMAGE="${OWRT_IMAGE:-openwrt-builder:25.12}"
CACHE_ROOT="${OWRT_CACHE_ROOT:-$HOME/.cache/openwrt}"
CONFIGS_DIR="$TOPDIR/configs"
JOBS="${JOBS:-$(nproc)}"

log()  { printf '\033[1;34m==>\033[0m %s\n' "$*" >&2; }
die()  { printf '\033[1;31mERR\033[0m %s\n' "$*" >&2; exit 1; }

in_container() { [ -f /.owrt-container ]; }

reexec_in_container() {
    command -v docker >/dev/null || die "docker not found"
    mkdir -p "$CACHE_ROOT"/{dl,ccache,home}
    local tty=(); [ -t 0 ] && tty=(-it)
    exec docker run --rm "${tty[@]}" \
        --user "$(id -u):$(id -g)" \
        -v "$TOPDIR:$TOPDIR" \
        -v "$CACHE_ROOT/dl:$TOPDIR/dl" \
        -v "$CACHE_ROOT/ccache:$TOPDIR/.ccache" \
        -v "$CACHE_ROOT/home:/home/builder" \
        -e HOME=/home/builder \
        -e CCACHE_MAXSIZE="${CCACHE_MAXSIZE:-50G}" \
        -e JOBS="$JOBS" \
        ${DOWNLOAD_MIRROR:+-e DOWNLOAD_MIRROR="$DOWNLOAD_MIRROR"} \
        -w "$TOPDIR" "$IMAGE" ./build.sh "$@"
}

# ---------- 配置组装 ----------
expand_seed() {   # $1=输入文件 $2=输出文件
    local line
    while IFS= read -r line || [ -n "$line" ]; do
        case "$line" in
            "#include "*) expand_seed "$CONFIGS_DIR/${line#\#include }" "$2" ;;
            *)            printf '%s\n' "$line" >> "$2" ;;
        esac
    done < "$1"
}

cmd_config() {
    local profile="${1:?usage: build.sh config <profile>}"
    local seed="$CONFIGS_DIR/$profile.config"
    [ -f "$seed" ] || die "no such profile: $seed"
    mkdir -p tmp
    : > tmp/seed.config
    expand_seed "$CONFIGS_DIR/base.config" tmp/seed.config
    expand_seed "$seed" tmp/seed.config

    log "expanding kconfig for profile $profile"
    cp tmp/seed.config .config
    make defconfig >/dev/null

    # 同步 rootfs 覆盖文件
    if [ -d "board/$profile/files" ]; then
        rm -rf files && cp -a "board/$profile/files" files
    fi

    cmd_config_check
    echo "$profile" > tmp/.current-profile
}

cmd_config_check() {
    local dropped=0 line
    while IFS= read -r line; do
        case "$line" in
            CONFIG_*=*)
                grep -qxF -- "$line" .config || { echo "DROPPED: $line"; dropped=1; } ;;
        esac
    done < tmp/seed.config
    [ "$dropped" = 0 ] || die "some options were silently dropped by kconfig (see above)"
    log "config check passed"
}

cmd_save() {
    local profile="${1:-$(cat tmp/.current-profile 2>/dev/null)}"
    [ -n "$profile" ] || die "usage: build.sh save <profile>"
    local out="$CONFIGS_DIR/$profile.config"
    if grep -q '^#include ' "$out" 2>/dev/null; then
        ./scripts/diffconfig.sh > "$out.new"
        log "profile uses #include; wrote $out.new — merge manually"
    else
        ./scripts/diffconfig.sh > "$out"
        log "saved $out ($(wc -l < "$out") lines)"
    fi
}

# ---------- 构建 ----------
cmd_feeds() {
    ./scripts/feeds update -a
    ./scripts/feeds install -a
}

cmd_download() {
    flock "$CACHE_ROOT/dl.lock" -c "make download -j$JOBS" 2>/dev/null \
        || flock "$TOPDIR/dl/.lock" make download -j"$JOBS"
}

cmd_build() {
    [ -n "${1:-}" ] && cmd_config "$1"
    [ -f .config ] || die "no .config; run: build.sh config <profile>"
    make tools/ccache/compile >/dev/null 2>&1 || true
    cmd_download
    log "building with -j$JOBS"
    if ! make -j"$JOBS" world; then
        log "parallel build failed — replaying single-threaded for a readable log"
        mkdir -p logs
        make -j1 V=sc world 2>&1 | tee logs/last-failure.log
        die "build failed, see logs/last-failure.log"
    fi
    cmd_stats
}

cmd_stats() {
    [ -x staging_dir/host/bin/ccache ] && staging_dir/host/bin/ccache -s || true
    du -sh dl .ccache 2>/dev/null || true
}

cmd_shell()      { exec bash -l; }
cmd_menuconfig() { make menuconfig; }
cmd_clean()      { make clean; }
cmd_dirclean()   { make dirclean; }
cmd_cacheclean() { make cacheclean; }
cmd_dlgc() {   # 默认干跑，避免误删共享缓存
    if [ "${1:-}" = "--force" ]; then
        ./scripts/dl_cleanup.py dl
    else
        ./scripts/dl_cleanup.py -d dl
        log "dry-run only — pass --force to actually delete"
    fi
}

cmd_image() {
    docker build -t "$IMAGE" \
        --build-arg UID="$(id -u)" --build-arg GID="$(id -g)" \
        -f docker/Dockerfile docker/
}

cmd_ci() {
    local profile="${1:?usage: build.sh ci <profile>}"
    cmd_feeds; cmd_config "$profile"; cmd_build
}

usage() {
    cat >&2 <<'EOF'
usage: ./build.sh <command> [args]
  image                 build the builder docker image (host only)
  shell                 interactive shell inside the container
  feeds                 feeds update -a && feeds install -a
  config <profile>      assemble configs/<profile>.config -> .config, verify
  config-check          re-verify current .config against the seed
  menuconfig            ncurses config editor
  save [profile]        write minimal seed back via scripts/diffconfig.sh
  download              prefetch sources into the shared dl cache (locked)
  build [profile]       full build; auto-replays -j1 V=sc on failure
  ci <profile>          feeds + config + build, non-interactive
  stats                 ccache statistics and cache sizes
  dl-gc [--force]       prune superseded tarballs from dl (dry-run by default)
  clean|dirclean|cacheclean
EOF
    exit 1
}

cmd="${1:-}"; [ -n "$cmd" ] || usage; shift || true
[ "$cmd" = image ] && { cmd_image "$@"; exit; }
in_container || reexec_in_container "$cmd" "$@"

case "$cmd" in
    shell)        cmd_shell ;;
    feeds)        cmd_feeds ;;
    config)       cmd_config "$@" ;;
    config-check) cmd_config_check ;;
    menuconfig)   cmd_menuconfig ;;
    save)         cmd_save "$@" ;;
    download)     cmd_download ;;
    build)        cmd_build "$@" ;;
    ci)           cmd_ci "$@" ;;
    stats)        cmd_stats ;;
    clean)        cmd_clean ;;
    dirclean)     cmd_dirclean ;;
    cacheclean)   cmd_cacheclean ;;
    dl-gc)        cmd_dlgc "$@" ;;
    *)            usage ;;
esac
```

### 4.3 多 profile 并行

不想为每个 profile 开一棵树时，用 `CONFIG_BUILD_SUFFIX`——它会给 `build_dir`/`staging_dir` 加后缀（进 `TARGET_DIR_NAME`），而 `dl` 和 `.ccache` 仍然共享：

```96:96:rules.mk
BUILD_SUFFIX:=$(call qstrip,$(CONFIG_BUILD_SUFFIX))
```

在每个 profile 的种子里加一行 `CONFIG_BUILD_SUFFIX="ap-mt7981"` 即可，几乎零成本。真要**同时**跑多个构建，用 `git worktree` 开独立树、共享同一份缓存挂载更稳（避开 `staging_dir` 竞争）。

---

## 五、执行顺序清单

| # | 动作 | 验证点 |
|---|---|---|
| 1 | `mkdir -p docker configs/fragments && mkdir -p ~/.cache/openwrt/{dl,ccache,home}` | — |
| 2 | 写 `docker/Dockerfile`、`.dockerignore`、`build.sh`（`chmod +x`）、`configs/base.config` | — |
| 3 | `./build.sh image` | 镜像建成 |
| 4 | `./build.sh shell`，内部跑 `id && pwd && gcc --version && touch a.tmp && ls -l a.tmp` | UID 与宿主一致、`pwd` 与宿主路径完全相同、写出的文件宿主可读写 |
| 5 | `./build.sh feeds` | 首次约几分钟 |
| 6 | `./build.sh config <profile>` → `./build.sh menuconfig` → `./build.sh save <profile>` | — |
| 7 | `git add configs/ && git diff --cached configs/` | diff 是几十行可读增量，不含任何绝对路径 |
| 8 | `./build.sh download` | 之后可断网构建 |
| 9 | `./build.sh build <profile>` | 首次全量成功 |
| 10 | `./build.sh stats` 记基线；`./build.sh dirclean && ./build.sh build <profile>` | ccache 命中率与耗时明显下降 |
| 11 | 更新 `.devcontainer/ci-env/devcontainer.json`，提交 | IDE 内可直接开发 |
| 12 | CI 一行 `./build.sh ci <profile>`，配 `actions/cache` 缓存 `~/.cache/openwrt/{dl,ccache}` | 干净环境可复现 |

---

## 六、验收标准

| 项 | 指标 |
|---|---|
| 容器化 | 容器与宿主交替 `make` 不需要 `dirclean`；构建产物无 root 所有权文件 |
| ccache | 第二次 `dirclean` 后全量构建 hit rate > 60%，耗时降 40% 以上 |
| dl | `./build.sh download` 后断网可完成 `build` |
| diffconfig | 每个 `configs/*.config` < 100 行；`config-check` 零 DROPPED |
| build.sh | 新人从 clone 到出固件只需 `./build.sh image && ./build.sh ci <profile>` |

---

## 七、已知坑

- **路径一致性是硬约束。** 一旦容器内外路径不同，`staging_dir` 只能重建。改挂载路径前先 `dirclean`。
- **`dl` 不要放在多写者共享的 NFS 上并发构建。** `download.pl` 的 `<file>.dl` 临时名无随机后缀，会互相覆盖。用预取 + `flock`，或每个 job 独立 `dl` 加一个只读镜像源。
- **`make defconfig` 静默丢选项。** 这是 `config-check` 存在的唯一理由，别省。feed 未 install 时最容易发生。
- **`CONFIG_CCACHE` 依赖 `CONFIG_DEVEL=y`**，且它会出现在 `bin/.../config.buildinfo` 里。做正式发布 / 可复现构建的 profile 建议单独一份不开 ccache 的种子。
- **ccache 对增量改包无收益。** 别拿它当"改一行重编快"的方案。
- **UID 不匹配时 git 会报 `dubious ownership`**，`scripts/getver.sh` 随之失效导致版本号变 `unknown`。真遇到就在 entrypoint 里加 `git config --global --add safe.directory "$PWD"`。
- **时间戳与可复现性。** 若要求可复现，去掉 ccache 的 `time_macros` sloppiness，并用 `scripts/get_source_date_epoch.sh` 设 `SOURCE_DATE_EPOCH`。
- **不要用 `--privileged`。** OpenWrt 不需要 loop 设备，加了只是扩大攻击面。

---

## 附录：关键位置速查

| 用途 | 路径 |
|---|---|
| ccache 接线 | `rules.mk:347-355` |
| ccache 开关定义 | `config/Config-devel.in:135-145` |
| ccache 工具构建条件 | `tools/Makefile:158` |
| ccache 统计 / 清理 | `Makefile:75-78`、`Makefile:137-139` |
| DL_DIR 定义 | `rules.mk:149` |
| 下载与镜像逻辑 | `scripts/download.pl`、`scripts/projectsmirrors.json`、`include/download.mk` |
| dl 清理 | `scripts/dl_cleanup.py` |
| 最小配置生成 | `scripts/diffconfig.sh`、`scripts/kconfig.pl` |
| diffconfig make target | `Makefile:123-125` |
| BUILD_SUFFIX（多变体共存） | `rules.mk:96` |
| 内置配置环境管理器 | `scripts/env` |
| 现有 devcontainer | `.devcontainer/ci-env/devcontainer.json` |
| 宿主编译器检查 | `config/check-hostcxx.sh`、`config/check-uname.sh` |
