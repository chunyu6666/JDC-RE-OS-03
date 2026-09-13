# 本地编译 JDCloud RE-CS-03 固件（手动下载版，不使用 git clone）

本文档假设你**不使用任何 `git clone`**，所有源码/软件包都通过浏览器或下载工具
手动下载压缩包，再解压到指定位置。流程对应 `.github/workflows/Henry.yaml`，
但去掉了 CI 专用的缓存清理、overlay 扩容、Release 发布等步骤。

---

## 1. 环境要求

- **系统**：64 位 Linux，推荐 Ubuntu 24.04（Debian/Ubuntu 系均可）。
- **硬件**：≥ 4 核、≥ 8 GB 内存；磁盘 **≥ 40 GB 空闲**。
- **用户**：普通用户（不要 root，`make` 会拒绝）。

```bash
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib \
  g++-multilib gettext git libncurses5-dev libssl-dev python3-setuptools \
  rsync swig unzip zlib1g-dev file wget ccache
```

> 注：`git` 这里只用来跑 `git apply` 打补丁，不需要联网；下载全部手工完成。

---

## 2. 目录约定

把你当前的这个仓库（含 `re-cs-03.config`、`patches/`、`files/`、`luci-app-accesscontrol.zip`）
放在：

```
~/JDC-RE-OS-03
```

新建一个**构建根目录**，所有“下载并解压”的内容都放进这里：

```
~/openwrt
```

后续所有路径都相对于 `~/openwrt`。

---

## 3. 需要手动下载的内容与放置位置

下表每一项都对应工作流里的一条 `git clone` / 仓库来源。
“下载地址”是 GitHub 页面 → 右上角 **Code → Download ZIP**，或用表格里的直链。

| # | 内容 | 下载地址（默认分支） | 解压后放到 |
| --- | --- | --- | --- |
| 1 | **OpenWrt 主源码** | https://github.com/Super-Henry/openwrt （切到 `main` 分支下载 ZIP） | `~/openwrt/`（即构建根目录本身，里面应有 `Makefile`、`scripts/`、`package/` 等） |
| 2 | **本仓库补丁** | 已在本地 `~/JDC-RE-OS-03/patches/re-cs-03-mainline.patch` | 复制到 `~/openwrt/re-cs-03-mainline.patch` |
| 3 | **编译配置** | 已在本地 `~/JDC-RE-OS-03/re-cs-03.config` | 复制到 `~/openwrt/.config` |
| 4 | **自定义文件** | 已在本地 `~/JDC-RE-OS-03/files/`（含 `etc/`、`ssh/`、`uci-defaults/`、`rc.local`） | 复制到 `~/openwrt/files/` |
| 5 | **访问控制插件 zip** | 已在本地 `~/JDC-RE-OS-03/luci-app-accesscontrol.zip` | 解压到 `~/openwrt/package/feeds/luci/applications/` |
| 6 | **PassWall2 (luci)** | https://github.com/Super-Henry/Openwrt-Passwall2 | `~/openwrt/package/feeds/luci/luci-app-passwall2` |
| 7 | **PassWall2 (packages)** | https://github.com/Super-Henry/Openwrt-Passwall-Packages | `~/openwrt/package/utils/passwall2` |
| 8 | **Argon 主题** | https://github.com/jerrykuku/luci-theme-argon | `~/openwrt/package/feeds/luci/luci-theme-argon` |
| 9 | **Argon 配置** | https://github.com/jerrykuku/luci-app-argon-config | `~/openwrt/package/feeds/luci/luci-app-argon-config` |

### 下载直链写法（可选，用 wget/curl 也可）

GitHub 分支 ZIP 直链格式：

```
https://github.com/<owner>/<repo>/archive/refs/heads/<branch>.zip
```

例如 PassWall2：

```
https://github.com/Super-Henry/Openwrt-Passwall2/archive/refs/heads/master.zip
```

> 各仓库没指定分支时取默认分支（PassWall 系列多为 `master`，Argon 系列多为 `master`）。
> 若解压后顶层目录带 `-master` 后缀，把**目录里面的内容**移动到上表“解压后放到”的路径即可。

### 关于主源码的版本（重要）

补丁 `re-cs-03-mainline.patch` 是基于 `main` 分支某一时刻的提交生成的。
下载 ZIP 时若 `main` 已经前进，可能导致 `git apply` 失败。建议：
- 优先下载与补丁时间相近的提交快照；或
- 直接下 `main` 分支最新 ZIP，若打补丁报错再用 `patch -p1 --fuzz=3` 或手动合并 `.rej`。

---

## 4. 放置与解压操作示例

```bash
# —— 构建根目录 ——
mkdir -p ~/openwrt
# 把 OpenWrt 主源码 ZIP 解压进 ~/openwrt（确保 Makefile 直接在 ~/openwrt 下）

cd ~/openwrt

# —— 本仓库文件复制进来 ——
cp ~/JDC-RE-OS-03/patches/re-cs-03-mainline.patch .
cp ~/JDC-RE-OS-03/re-cs-03.config .config
cp -r ~/JDC-RE-OS-03/files .

# —— 额外软件包：把各自下载的 ZIP 解压到下表目录 ——
# （下面路径需与你实际解压出的目录名对应，假设解压出的顶层目录名为 <repo>-master）
mkdir -p package/feeds/luci package/utils
# 例：unzip 后把目录内容移入对应路径
#   Openwrt-Passwall2        -> package/feeds/luci/luci-app-passwall2
#   Openwrt-Passwall-Packages -> package/utils/passwall2
#   luci-theme-argon         -> package/feeds/luci/luci-theme-argon
#   luci-app-argon-config    -> package/feeds/luci/luci-app-argon-config

# —— 访问控制插件 zip（本仓库自带）——
unzip ~/JDC-RE-OS-03/luci-app-accesscontrol.zip -d package/feeds/luci/applications
```

目录放置检查（应能看到）：

```
~/openwrt/Makefile
~/openwrt/scripts/feeds
~/openwrt/.config
~/openwrt/re-cs-03-mainline.patch
~/openwrt/files/etc/rc.local
~/openwrt/package/feeds/luci/luci-app-passwall2/
~/openwrt/package/utils/passwall2/
~/openwrt/package/feeds/luci/luci-theme-argon/
~/openwrt/package/feeds/luci/luci-app-argon-config/
~/openwrt/package/feeds/luci/applications/luci-app-accesscontrol/
```

---

## 5. 打补丁

```bash
cd ~/openwrt
git apply --verbose re-cs-03-mainline.patch
```

- 成功无输出错误即可。
- 若报 `patch does not apply`，说明主源码版本与补丁不匹配：
  - 换用相近提交的源码快照重下；或
  - `patch -p1 --fuzz=3 < re-cs-03-mainline.patch`，再手动处理残留 `.rej`。

---

## 6. feeds（这一项可保留自动，或全手动）

工作流里的 `./scripts/feeds update -a` 会按 `feeds.conf.default` 拉取 luci / packages 等
官方 feed。这是 OpenWrt 自带的机制，**内部也是 git 拉取**，如果你坚持“完全不碰 git”：

### 方案 A（推荐，仍自动）：保留这一条
```bash
cd ~/openwrt
./scripts/feeds update -a
./scripts/feeds install -a -p luci
./scripts/feeds install -a -p packages
```

### 方案 B（全手动，离线）：自己把每个 feed 下载后放进 `feeds/`
`feeds.conf.default` 中每一行 `src-git <name> <url>` 对应目录 `feeds/<name>/`。
把对应仓库下载解压到 `feeds/<name>/`，然后：
```bash
./scripts/feeds install -a -p luci
./scripts/feeds install -a -p packages
```
> 官方 feed 数量多（luci、packages、routing、telephony 等），手动较繁琐，一般建议用方案 A。

---

## 7. 套用配置 → 下载依赖 → 编译

```bash
cd ~/openwrt

# 展开 .config（解析选中包、丢弃不可用选项）
make defconfig

# 预下载软件包源码（需联网，非 git clone）
make download -j$(nproc)

# 正式编译；首次会先构建工具链，耗时较长
make -j$(nproc) V=s
```

- 失败重试单线程：`make -j1 V=s`
- 想加速重编：装了 `ccache` 后，在 `make menuconfig` 勾
  `Global build settings → Use ccache`，并设置 `export CCACHE_DIR=~/.ccache`。

---

## 8. 编译产物

```
bin/targets/qualcommax/ipq50xx/
```

| 文件 | 用途 |
| --- | --- |
| `...-squashfs-factory.bin` | 首次刷机（uboot web 刷机界面） |
| `...-squashfs-sysupgrade.bin` | Luci 升级界面更新 |
| `...-initramfs-uImage.itb` | TTL 临时启动 / 救砖 |

kmod 等全部安装包在 `bin/packages/aarch64_cortex-a53/` 与
`bin/targets/qualcommax/ipq50xx/packages/`，可单独安装不缺依赖。

---

## 9. 刷机与注意事项

刷机步骤见 `README.md`（拆机 TTL → 刷 uboot → JOY 进 web 刷 `factory.bin`）。

注意事项：
- **NSS 加速**：`re-cs-03.config` 勾了 `kmod-qca-nss-drv` 等项，但本流程**未**加入
  `Super-Henry/NSS-PACKAGES` feed，这些 kmod 大概率不会被编入，与 README
  “没有 NSS 加速功能”一致。需要 NSS 时请手动下载 NSS-PACKAGES 并加入 `feeds.conf.default`。
- **GCC 15 / Mold**：配置启用了 `CONFIG_GCC_USE_VERSION_15` 与 `CONFIG_USE_MOLD=y`，
  工具链会自行构建 GCC 15；本机环境编不过可在 `make menuconfig` 改用主机 GCC、关 `Use mold`。
- **磁盘**：至少预留 40 GB，用 `df -hT` 监控。
- **不要 root 编译**。
- 改好 `.config` 想固化回仓库：把 `~/openwrt/.config` 覆盖 `~/JDC-RE-OS-03/re-cs-03.config`。
