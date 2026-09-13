# 本地编译 JDCloud RE-CS-03 固件（手动下载版，NSS 用 git clone，含 NSS）

本文档说明如何**手动下载**绝大部分源码/软件包（不使用 `git clone`），
仅 **NSS 软件包源**例外——它直接交给 OpenWrt 的 feeds 机制从 git 自动拉取
（见第 5 节）。用本仓库的配置/补丁/自定义文件，从 `Super-Henry/openwrt` 源码编译出
京东云后羿（哪吒）RE-CS-03（IPQ5018 / ipq50xx）的 OpenWrt 固件，并**正确加入
NSS 加速软件包源**（这是 CI 工作流漏掉的一步）。

> 与 `.github/workflows/Henry.yaml` 的一致性：
> - 保留了有本地意义的步骤（依赖安装、打补丁、feeds、套配置、编译、打包）。
> - 省略了仅 CI 才需要的步骤（overlay 挂 `/mnt` 扩容、GitHub Actions 缓存恢复/清理、上传 artifact、发 Release）。
> - **补上了 CI 缺失的 NSS-PACKAGES feed**（见第 5 节）。
> - 除 NSS 外的 `git clone` 都改为「浏览器/下载工具手动下载 ZIP → 解压到指定目录」；
>   NSS 则通过 `src-git` 由 feeds 自动 `git clone`。

> ⚠️ **关键顺序（踩坑点）**：`feeds install` 必须在 `feeds update` **之后**执行，
> 否则报 `Ignoring feed 'xxx' - index missing`，包装不上。本文档已按正确顺序编排：
> 先 `feeds update -a`（一次性生成所有 feed 索引），再统一 `feeds install`。

---

## 1. 环境要求

- **系统**：64 位 Linux，推荐 **Ubuntu 24.04**（其他 Debian/Ubuntu 系亦可）。
- **硬件**：≥ 4 核 CPU、≥ 8 GB 内存；磁盘建议 **≥ 50 GB 空闲**（开启 ccache / 含 NSS 后更大）。
- **网络**：能访问 GitHub 下载源码与软件包（仅下载，不要求 git 推送）。
- **用户**：用普通用户编译（**不要**用 root，`make` 会拒绝）。

```bash
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib \
  g++-multilib gettext git libncurses5-dev libssl-dev python3-setuptools \
  rsync swig unzip zlib1g-dev file wget ccache
```

> 这里装的 `git` 仅用于在本机跑 `git apply` 打补丁（不联网），不需要你用 git 拉任何仓库。

---

## 2. 目录约定

本仓库（含 `re-cs-03.config`、`patches/`、`files/`、`luci-app-accesscontrol.zip`）放在：

```
~/JDC-RE-OS-03
```

OpenWrt 构建目录（所有手动下载内容都放进这里）放在同级：

```
~/openwrt
```

> 你实际用的是 `/data/openwrt`，下文所有 `~/openwrt` 请替换成你的实际构建目录。

---

## 3. 下载清单总表（手动下载 → 放到哪）

| # | 内容 | 分支 / 标签 | 下载地址 | 解压后放到 |
| --- | --- | --- | --- | --- |
| 1 | **OpenWrt 主源码** | `main` | https://github.com/Super-Henry/openwrt/archive/refs/heads/main.zip | `~/openwrt/`（构建根目录本身，里面要有 `Makefile`、`scripts/`） |
| 2 | 本仓库补丁 | — | 本地 `~/JDC-RE-OS-03/patches/re-cs-03-mainline.patch` | 复制到 `~/openwrt/re-cs-03-mainline.patch` |
| 3 | 编译配置 | — | 本地 `~/JDC-RE-OS-03/re-cs-03.config` | 复制到 `~/openwrt/.config` |
| 4 | 自定义文件 | — | 本地 `~/JDC-RE-OS-03/files/`（含 `etc/`、`ssh/`、`uci-defaults/`、`rc.local`） | 复制到 `~/openwrt/files/` |
| 5 | 访问控制插件 zip | — | 本地 `~/JDC-RE-OS-03/luci-app-accesscontrol.zip` | 解压到 `~/openwrt/package/feeds/luci/applications/` |
| 6 | **NSS 软件包源** | `NSS-12.5-K6.x` | `src-git`（git 自动拉取，见第 5 节，**无需手动下载**） | 由 feeds 机制自动 clone 到 `~/openwrt/feeds/nss_packages/` |
| 7 | PassWall2 (luci) | `main` | https://github.com/Super-Henry/Openwrt-Passwall2/archive/refs/heads/main.zip | `~/openwrt/package/feeds/luci/luci-app-passwall2` |
| 8 | PassWall (packages) | `main` | https://github.com/Super-Henry/Openwrt-Passwall-Packages/archive/refs/heads/main.zip | `~/openwrt/package/utils/passwall2` |
| 9 | Argon 主题 | `master` | https://github.com/jerrykuku/luci-theme-argon/archive/refs/heads/master.zip | `~/openwrt/package/feeds/luci/luci-theme-argon` |
| 10 | Argon 配置 | `master` | https://github.com/jerrykuku/luci-app-argon-config/archive/refs/heads/master.zip | `~/openwrt/package/feeds/luci/luci-app-argon-config` |

> GitHub 下载 ZIP 默认目录名形如 `<repo>-<branch>`（如 `openwrt-main`、`Openwrt-Passwall2-main`）。
> NSS（#6）不在此列——它走 `src-git` 自动 clone，无需手动下载 ZIP。
> 解压后请把**目录里面的内容**移动到上表“解压后放到”的路径，不要多包一层目录。

---

## 4. 放置 OpenWrt 主源码并打补丁

```bash
# 把 OpenWrt 主源码 ZIP 解压进 ~/openwrt（确保 Makefile 直接在 ~/openwrt 下）
mkdir -p ~/openwrt
# 例如：unzip ~/Downloads/openwrt-main.zip -d /tmp && mv /tmp/openwrt-main/* ~/openwrt/

cd ~/openwrt

# 复制并应用设备补丁（改 hostname=Houyi、时区 CST-8、nf_conntrack_max、uboot-envtools 等）
cp ~/JDC-RE-OS-03/patches/re-cs-03-mainline.patch .
git apply --verbose re-cs-03-mainline.patch
```

- 若 `git apply` 失败（源码前进导致上下文不符）：换相近提交的源码快照重下，或
  `patch -p1 --fuzz=3 < re-cs-03-mainline.patch` 再手动合并残留 `.rej`。

---

## 5. 加入 NSS 软件包源（关键：CI 漏掉的一步，用 git 自动拉取）

`re-cs-03.config` 已勾选 NSS 内核模块（`kmod-qca-nss-drv`、`kmod-qca-nss-ecm`、
`kmod-qca-nss-drv-bridge-mgr` 等），但这些包来自独立的 NSS-PACKAGES feed。
**若不加这个 feed，`make defconfig` 会把 NSS 选项当成“不存在的包”直接丢弃 → 没有 NSS 加速。**

### 5.1 用 src-git 注册 feed（从 git 自动 clone，无需手动下载）

编辑 `~/openwrt/feeds.conf.default`，在末尾加一行，让 OpenWrt 的 feeds 机制
自动 `git clone` NSS 源码（**不需要你手动下载 ZIP**）：

```bash
echo 'src-git nss_packages https://github.com/Super-Henry/NSS-PACKAGES;NSS-12.5-K6.x' >> feeds.conf.default
```

> 格式为 `src-git <feed名> <仓库URL>;<分支>`。执行第 7 节的 `feeds update -a`
> 时，OpenWrt 会自动把它 clone 到 `feeds/nss_packages/`（分支 `NSS-12.5-K6.x`），
> 再 `feeds install -a -p nss_packages` 把里面的包注册进 `package/feeds/nss_packages/`。
>
> 注意：`src-git` 会让 `feeds update -a` 真正联网 git clone 一次，所以需要能访问 GitHub。
> 仓库地址请按你实际使用的 fork/分支调整（这里沿用 CI 同款 `Super-Henry/NSS-PACKAGES`）。

---

## 6. 放置额外软件包 + 注册默认 feeds

### 6.1 手动放置 PassWall / Argon / 访问控制

```bash
cd ~/openwrt
mkdir -p package/feeds/luci package/utils

# 把下面的 <zip> 解压并把内容移入对应目录：
#   Openwrt-Passwall2-main          -> package/feeds/luci/luci-app-passwall2
#   Openwrt-Passwall-Packages-main  -> package/utils/passwall2
#   luci-theme-argon-master         -> package/feeds/luci/luci-theme-argon
#   luci-app-argon-config-master    -> package/feeds/luci/luci-app-argon-config

# 访问控制插件（本仓库自带 zip）解压到 luci applications 目录
unzip ~/JDC-RE-OS-03/luci-app-accesscontrol.zip -d package/feeds/luci/applications
```

> `luci-app-accesscontrol.zip` 解压后若多包了一层目录，请确认最终路径为
> `package/feeds/luci/applications/luci-app-accesscontrol/`，否则 `make menuconfig` 里找不到它。

### 6.2 注册默认 feeds（luci / packages 等，OpenWrt 自带机制）

默认 feeds（luci、packages、routing、telephony、video）由 OpenWrt 的 `feeds.conf.default`
自带定义，无需你手动下载，交给 `feeds update -a` 统一处理即可。

---

## 7. feeds 更新与安装（⚠️ 先 update 再 install）

```bash
cd ~/openwrt

# 1) 一次性更新【所有】feed，生成索引（关键！缺这步就会 index missing）
./scripts/feeds update -a

# 2) 安装各 feed（此时索引已存在，能正常装）
./scripts/feeds install -a -p nss_packages   # NSS（第 5 节）
./scripts/feeds install -a -p luci
./scripts/feeds install -a -p packages

# 3) 手动放置的 PassWall / Argon / 访问控制已直接位于 package/ 下，无需 install，
#    但跑一次下面这条可确保被扫描到（已在上面 luci/packages install 时涵盖）
./scripts/feeds install -a
```

验证 NSS 已接入：

```bash
ls package/feeds/nss_packages/ | grep -i nss
# 应能看到 kmod-qca-nss-drv、kmod-qca-nss-ecm、nss-firmware-* 等
```

---

## 8. 套用本项目配置与自定义文件（本项目配置如何放）

本仓库配置/文件 与 构建目录 的对应关系：

| 本仓库文件 | 放入构建目录的位置 | 作用 |
| --- | --- | --- |
| `re-cs-03.config` | `~/openwrt/.config` | 完整编译配置（目标、包、优化选项） |
| `patches/re-cs-03-mainline.patch` | `~/openwrt/re-cs-03-mainline.patch` | 第 4 节已打的补丁 |
| `files/`（含 `etc/`、`ssh/`、`uci-defaults/`、`rc.local`） | `~/openwrt/files/` | 原样打进固件 `/` 的覆盖文件 |
| `luci-app-accesscontrol.zip` | 解压到 `package/feeds/luci/applications/` | 访问控制 LuCI 应用 |

执行：

```bash
cd ~/openwrt

# 用本仓库的配置文件作为 .config
cp ~/JDC-RE-OS-03/re-cs-03.config .config

# 展开 .config（此时 NSS feed 已就绪，NSS 选项会被保留，不会被丢弃）
make defconfig

# 拷贝自定义文件到 OpenWrt 镜像覆盖目录
mkdir -p files
cp -r ~/JDC-RE-OS-03/files/* files/
```

> **顺序要点**：NSS feed 必须在 `make defconfig` **之前**装好（第 7 节已先于本节完成），
> 否则 defconfig 会把 NSS 选项丢弃。

确认 NSS 选项真生效：

```bash
grep -E 'kmod-qca-nss-drv|kmod-qca-nss-ecm|NSS_SUPPORT' .config
# 应输出带 =y 的行
```

> 想微调：跑 `make menuconfig`（目标 `Qualcomm Atheros IPQ50xx` → `ipq50xx` →
> `jdcloud_re-cs-03`），保存后再继续。改好想固化回仓库就把 `.config` 覆盖 `re-cs-03.config`。

### （可选）注入编译时间

CI 会把编译时间写进 `os-release` 与 `files/etc/build_time`，本地可手动加：

```bash
cd ~/openwrt
BUILD_DATE_FRIENDLY=$(date +"%Y-%m-%d %H:%M:%S")
echo "BUILD_DATE=\"$(date +%s)\"" >> package/base-files/files/etc/os-release
mkdir -p files/etc
echo "固件编译时间: $BUILD_DATE_FRIENDLY" > files/etc/build_time
```

---

## 9. 下载依赖并编译

```bash
cd ~/openwrt
export CMAKE_POLICY_VERSION_MINIMUM=3.5   # 对齐 CI

# 预下载所有软件包源码（避免编译中途断网；会再次 update feeds）
make download -j$(nproc)

# 正式编译；首次会先构建工具链 + NSS 模块，耗时较长
make -j$(nproc) V=s
```

- 失败重试单线程：`make -j1 V=s`。
- 加速重编：装了 `ccache` 后，`export CCACHE_DIR=~/.ccache`，并在
  `make menuconfig` 勾 `Global build settings → Use ccache`。

---

## 10. 打包快照（对应 CI 的 tar 步骤）

CI 把全部 kmod/软件包打包成 `snapshots.tar.gz` 方便离线安装，本地同样执行：

```bash
cd ~/openwrt
tar cvzf bin/targets/qualcommax/ipq50xx/snapshots.tar.gz \
  -C bin packages/aarch64_cortex-a53 targets/qualcommax/ipq50xx/packages
```

> 若某次没编出 `packages/aarch64_cortex-a53`，可去掉该路径或结尾加 `|| true`。

---

## 11. 编译产物

```
bin/targets/qualcommax/ipq50xx/
```

| 文件 | 用途 |
| --- | --- |
| `...-squashfs-factory.bin` | 首次刷机（uboot web 刷机界面用） |
| `...-squashfs-sysupgrade.bin` | Luci 升级界面更新用 |
| `...-initramfs-uImage.itb` | TTL 临时启动 / 救砖用 |
| `snapshots.tar.gz` | 全部 kmod/软件包，离线安装用（第 10 节生成） |

`kmod` 等安装包也单独在 `bin/packages/aarch64_cortex-a53/` 与
`bin/targets/qualcommax/ipq50xx/packages/` 下。

---

## 12. 刷机（简述，详见 README.md）

1. 拆机接 TTL，网线插 WAN 口，配好终端与 TFTP。
2. 上电后 TTL 终端输入 `jdqca` 中断启动。
3. 刷入 `uboot.mbn`（`flash 0:APPSBL`），无需大分区。
4. 断电按住 `JOY` 键上电，进 uboot web 刷机界面，刷入 `factory.bin`。
5. 之后可在 Luci 升级界面用 `sysupgrade.bin` 更新。

---

## 13. 注意事项 / 排错

- **`Ignoring feed 'xxx' - index missing`**：说明 `feeds install` 跑在了 `feeds update` 前面，
  或 `feeds.conf.default` 里没注册该 feed。先 `./scripts/feeds update -a` 再 install（见第 7 节）。
- **NSS 没编进去**：检查 `feeds.conf.default` 是否含
  `src-git nss_packages https://github.com/Super-Henry/NSS-PACKAGES;NSS-12.5-K6.x`，
  并确认 `feeds update -a` 已成功 clone（看 `feeds/nss_packages/` 有内容）；
  再 `feeds install -a -p nss_packages` 后 `ls package/feeds/nss_packages/ | grep -i nss` 有结果；
  并在 `make defconfig` 后确认 `grep` 有 `=y`。
- **GCC 15 / Mold**：配置启用了 `CONFIG_GCC_USE_VERSION_15` 与 `CONFIG_USE_MOLD=y`，
  工具链会自行构建 GCC 15。本机环境编不过可在 `make menuconfig` 改回主机默认 GCC、关 `Use mold`。
- **磁盘空间**：含 NSS 后构建更大，至少预留 50 GB；随时 `df -hT` 监控。
- **不要 root 编译**。
- **配置漂移**：`make defconfig` / `make menuconfig` 可能改动 `.config`；固化时覆盖回 `re-cs-03.config`。
