# Cudy TR3000 v1 256MB — 一个 U-Boot 通吃（原厂固件 + OpenWrt）

给 **Cudy TR3000 v1 256MB**（256 MiB SPI NAND，SN 前缀 `R103`）云端编译一份 dhcpd U-Boot（FIP）：

* 能引导 **原厂 stock 固件**（Cudy 官方 R103 系列）
* 能**直刷原厂整包** `*-sysupgrade.bin`（不用手工切割）
* 能按不同布局刷 **OpenWrt / ImmortalWrt**（`default` 与 `112m` 两套分区）
* 编译全在 **GitHub Actions** 上跑，本机不用装交叉编译环境

上游源码：<https://github.com/weekdaycare/bl-mt798x-dhcpd>（U-Boot 2025.07）

---

## 一、先说结论：这块板刷原厂固件起不来，根因是 `mtdparts`

256MB 版原厂分区表（从原厂 FIP 里挖出来的）是：

```
nmbm0:1024k(bl2),512k(u-boot-env),2048k(factory),256k(bdinfo),2048k(fip),235520k(ubi)
```

而 bl-mt798x-dhcpd 给这块板**没有**定义 `mtd-layout`，`-(ubi)`（把 NMBM 剩余空间全给 ubi）
算出来的值取决于本 U-Boot 预留了多少 NMBM 块 —— 实测比原厂多 **3072 KiB**（`238592k`）。

后果是一条很隐蔽的连锁：

```
U-Boot 用 238592k 的范围 attach UBI（比真实 UBI 大 3072 KiB）
  -> 尾部那批空白擦除块被收编成 free PEB
  -> U-Boot 顺手挪动了 volume table
  -> 内核按自己设备树里的 235520k 去找那张表，找不到：
        ubi0 error: ubi_read_volume_table: the layout volume was not found
        ubi0 error: ubi_attach_mtd_dev: failed to attach mtd6, error -22
  -> panic -> 复位 -> 循环
```

`235520k` 由**三处独立证据**共同确认：

1. 原厂 FIP 内置的 `mtdparts` 字符串
2. 原厂内核 DTB：`partition@5C0000 { reg = <0x5c0000 0xe600000> }` = 235520 KiB
3. 原厂/OpenWrt 的镜像是按这个尺寸建的（下面第五节）

> ⚠️ **还有一个更隐蔽的坑：存盘 env 里的 `mtdparts` 优先级高于设备树。**
> 只改 DTS 是白改的 —— 如果 `u-boot-env` 分区里躺着一行旧的
> `mtdparts=...2048k(fip),-(ubi)`，它会**整个盖住** DTS 里的布局。
> 症状是 `mtd erase ubi` 报出的擦除块数一直对不上。
> 处理办法见第六节第 2 步。

---

## 二、布局一览

`arch/arm/dts/mt7981-cudy-tr3000-v1-256mb.dts` 里新增 `/mtd-layout`：

| label | `mtdparts`（ubi 部分） | ubi 大小 | 用途 |
|---|---|---|---|
| `default` | `235520k(ubi)` | 235520 KiB（230 MiB） | **原厂 stock 固件**、**OpenWrt/ImmortalWrt 官方 256MB 镜像** |
| `112m` | `114688k(ubi)` | 114688 KiB（112 MiB） | 128MB 机型那套 `112m-nmbm` / ubootmod 风格镜像 |

**前五个分区（bl2 / u-boot-env / Factory / bdinfo / fip）在两套布局里完全一致** ——
所以 BL2 和 FIP 永远不会挪位，只有 ubi 大小变化；引导链上除了 U-Boot 谁都不用知道
现在用的是哪套布局。

切换方式：

| 场景 | 怎么做 |
|---|---|
| 网页 | 高级功能 → MTD 布局（读 `/getmtdlayout`，写入 `mtd_layout_label`） |
| 命令行 | `setenv mtd_layout_label 112m; saveenv` |
| 刷机时 | `/upload` 请求里带 `mtd_layout=112m` |

> 上游 `CONFIG_MEDIATEK_MULTI_MTD_LAYOUT` 会 `select SYS_MTDPARTS_RUNTIME` +
> `CMD_SHOW_MTD_LABEL`，所以 `board_mtdparts_default()` 一定被调用，DTS 里的布局一定生效。
> 网页上还有 `showlayout` 命令可以随时打印当前所有布局。

---

## 三、和社区「中文三分区 U-Boot」的差异

社区版（fry2022 的 `dhcp-mt7981_cudy_tr3000-fip-fixed-parts-multi-layout-256M.bin`）
内置三套写死布局：

| 社区版 label | ubi | 本仓库对应 |
|---|---|---|
| `default` | `65536k`（64 MiB） | ❌ 这是 128MB 机型的原厂尺寸，用在这块 256MB 板上会浪费 3/4 flash |
| `mod-112m` | `114688k`（112 MiB） | ✅ 同一档，本仓库 label 叫 `112m` |
| `maximum-240m` | `245760k`（240 MiB） | ⚠️ 240 MiB，**与 OpenWrt 官方 256MB 机型的 235520k 不一致** |

本仓库把 `default` 钉成 **`235520k`**，与 Cudy 原厂 **以及 OpenWrt 官方** 逐字节一致 ——
好处是「网页刷机时不用纠结选哪套布局，闭眼选 `default` 就同时满足原厂和 OpenWrt」。

---

## 四、六个补丁

| 补丁 | 解决的问题 |
|---|---|
| `0001` | **写得进去**：U-Boot 在 attach 失败时会 `*** Rebuilding UBI ***` 把整个 ubi 分区擦掉，把刚刷进去的固件抹了。改成「检测到有效 UBI 镜像就拒绝自动重建」，并新增 `generic_mtd_write_oem()` 裸写路径 |
| `0002` | **起得来**：这块 U-Boot 树里没有任何地方定义 `CONFIG_BOOTCOMMAND`，`bootcmd` 一旦丢失就永远进不了系统。补回默认值，并加 `vendor_bootargs` 钩子做现场命令行覆盖 |
| `0003` | **不用手切**：原厂 `-sysupgrade.bin` = 一个 FIP + 一整份 UBI，偏移 0 是 FIP 魔数，解析器认不出就报 `*** Image not supported! ***`。改成自动逐字节找出 UBI 载荷、按 PEB 对齐切割后裸写；顺手修掉一处误删卷的隐患 |
| `0004` | **通吃**：补齐 256MB 机型的多布局支持（新增 `mt7981_cudy_tr3000-v1-256mb_multi_layout_defconfig` + DTS `/mtd-layout`），并把 vendor 引导命令行从硬编码改成「由布局提供 + env 可现场覆盖」 |
| `0005` | **对齐原厂命令行**：删掉 0004 给 `default` 布局写的那条 `cmdline`。原厂 U-Boot 压根不设 `bootargs`，原厂内核收到的就是 FIT 自带的 `/chosen/bootargs`（只有 `console` + `earlycon`）；env 一旦有 `bootargs`，`fdt_chosen()` 就会整体替换掉 FIT 那条 |
| `0006` | **钉死 ubi 尺寸**：`default` 布局从 `-(ubi)` 改为写死的 `235520k(ubi)` —— 就是第一节那个根因 |

### 为什么命令行必须留空（0005 的依据）

`boot/fdt_support.c` 的 `fdt_chosen()` 只看 `board_fdt_chosen_bootargs()`（= `env_get("bootargs")`）：
**env 一旦非空就整体替换掉 FIT 那条，env 为空则一个字都不动。**

```
cmdline == NULL
  -> board_mtdparts_default() 里 env_set("bootargs", NULL)（等于删除该变量）
  -> board_fdt_chosen_bootargs() 返回 NULL
  -> fdt_chosen() 里 `if (str)` 不成立，FIT 的 /chosen/bootargs 原封不动
```

结果就是与原厂 U-Boot **逐位一致**。

### 命令行仍然可以现场试，不用重编

`vendor_bootargs` 这个出口还在，换一组参数试一次 = 一条 `setenv` + 一次 `mtkboardboot`：

```text
setenv vendor_bootargs "console=ttyS0,115200n1 loglevel=8 ubi.mtd=ubi"
mtkboardboot

setenv vendor_bootargs          # 清掉，回到默认
mtkboardboot
```

---

## 五、OpenWrt / ImmortalWrt 怎么刷

**也是进 U-Boot 网页直接刷，布局选 `default`。** 理由：

OpenWrt 官方对这块板的设备定义里（`target/linux/mediatek/`）：

```dts
/* mt7981b-cudy-tr3000-256mb-v1.dts */
&ubi {
	reg = <0x5c0000 0xe600000>;      /* = 235520 KiB，与本仓库 default 逐字节一致 */
};
```

```makefile
# image/filogic.mk
define Device/cudy_tr3000-256mb-v1
  IMAGE_SIZE := 235520k
  KERNEL_IN_UBI := 1
  ...
  IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata
endef
```

* `IMAGE_SIZE := 235520k` ⇒ **必须选 `default`**（`112m` 只有 112 MiB，装不下）
* `sysupgrade-tar` ⇒ 镜像是 **tar** 格式，U-Boot 的 `parse_image_ram()` 认得它
  （`IMAGE_TAR`），`fw` 路径会把 tar 里的 `kernel` / `rootfs` 分别写成两个 UBI 卷，
  再建一个 `rootfs_data` 空卷 —— 正是 OpenWrt NAND 的标准布局
* 官方**没有**给 256MB 版做 ubootmod 变体（`cudy_tr3000-v1-ubootmod` 是 128MB 版才有的），
  因为 256MB 的原厂布局本来就已接近用满 NAND，没有「扩容」的余地

所以流程是：

1. 按住 `reset` 上电 → 浏览器 `<http://192.168.1.1>`
2. **固件升级** 页签 → 上传 `openwrt-mediatek-filogic-cudy_tr3000-256mb-v1-squashfs-sysupgrade.bin`
   （或对应的 ImmortalWrt 同名镜像）→ 核对 MD5 → Update
3. 布局框选 **`default`**
4. 等它自己重启。首次启动较慢（内核要建 `rootfs_data`），耐心等 2~5 分钟

> 如果之后想换回原厂固件：同样进 U-Boot 网页，用 `default` 布局刷原厂整包即可。

---

## 六、刷机步骤

### 1. 编译（或直接下载 Release）

仓库页面 → **Actions** → 左侧 **Build OEM-flashable U-Boot (FIP)** → **Run workflow**。
1~3 分钟后在运行页面底部 **Artifacts** 下载
`fip-cudy-tr3000-v1-256mb-multi-layout.zip`，解压得到 `fip-*.bin`。

> **只需要刷 `fip-*.bin`。** 同一个 zip 里那个 `bl2-*.bin` 是上游 `build.sh` 顺手产出的，
> 本仓库所有补丁都**不涉及 BL2**，不要去刷它。

### 2. 先检查 env 里有没有旧的 `mtdparts`（关键一步）

按住 `reset` 上电 → 进 `<http://192.168.1.1>` → **Console**：

```text
printenv mtdparts
```

如果输出里 ubi 那段不是 `235520k(ubi)`（比如是 `-(ubi)` 或 `65536k`），
**必须先把这行改掉**，否则它会盖住新 FIP 的设备树：

```text
setenv mtdparts nmbm0:1024k(bl2),512k(u-boot-env),2048k(Factory),256k(bdinfo),2048k(fip),235520k(ubi)
saveenv
```

改完可以再用 `mtd list` 确认 ubi 那行是 `0x5c0000 - 0xebc0000`（= `0xE600000` = 235520 KiB）。

> **不要点网页上的「重置为默认」，也不要 `mtd erase u-boot-env`。**
> 那会把 env 换成 OpenWrt ubootmod 那一套（指向不存在的 `fit`/`recovery` 卷），
> 落进无限 TFTP 轮询 `192.168.1.254` 卡死。
> 已经有网页可用的情况下，用 `setenv` 覆盖才是安全做法。

### 3. 刷 FIP

**更新 U-Boot** 页签 → 选 `fip-*.bin` → 刷入 → 等它自己重启。

刷完 ubi 若还是空的，`mtkboardboot` 引导失败会**自动回到网页**
（`do_mtkboardboot()` 里那句 `httpd` 在 `if (ret)` 之外），不用按键。

### 4. 刷固件

**固件升级** 页签 → 上传固件 → 核对 MD5 → 布局选 `default` → Update。

| 想刷什么 | 上传什么 | 布局 |
|---|---|---|
| 原厂 stock | Cudy 官方 `R103-256MB` 整包 `*-sysupgrade.bin` | `default` |
| OpenWrt | `openwrt-mediatek-filogic-cudy_tr3000-256mb-v1-squashfs-sysupgrade.bin` | `default` |
| ImmortalWrt | `immortalwrt-*-mediatek-filogic-cudy_tr3000-256mb-v1-squashfs-sysupgrade.bin` | `default` |

---

## 七、救砖与回退

* **FIP 是独立分区** —— 刷错 U-Boot 也能重新按住 `reset` 上电进网页，刷回来就行
* **BL2 不用动**：这块板原厂 BL2 正常，本仓库的补丁也不碰它
* **彻底回原厂**：
  1. **更新 U-Boot** 刷回原厂 FIP（2 MB）
  2. 电脑静态 IP `192.168.1.88/24`、网关留空，用 tftpd64 提供 `recovery-256MB.bin`
  3. 按住 `reset` 上电，等指示灯红/粉交替闪烁、TFTP 开始传输，松手
  4. 刷完自动重启，地址回到 `192.168.10.1`

---

## 八、排障

| 现象 | 看哪里 |
|---|---|
| 「依序应用补丁」红 | 补丁上下文对不上。上游 commit 已钉死在 workflow 的 `UPSTREAM_REF`，正常不会发生；要升级上游就改它 |
| 「校验补丁已生效」红 | 补丁没真正落地，同上 |
| 「编译 FIP」红 | 交叉编译报错，日志里搜 `error:` |
| 刷完还是引导失败 | 先 `printenv mtdparts` 看 env 有没有盖住设备树；再 `mtd list` 看 ubi 是不是 `0x5c0000-0xebc0000` |
| 想看完整构建输出 | 下载 Artifacts 里的 `build.log` |

---

## 九、说明

`build.sh` 是上游脚本，不改。CI 里用的参数：

| 变量 | 值 | 说明 |
|---|---|---|
| `BOARD` | `cudy_tr3000-v1-256mb` | 机型 |
| `SOC` | `mt7981` | SoC，显式给出省得脚本猜 |
| `VERSION` | `2025` | U-Boot 2025.07（`uboot-mtk-20250711`） |
| `VARIANT` | `default` | 256MB 版 **不要**用 `ubootmod` |
| `MULTI_LAYOUT` | `1` | 用 `configs/mt7981_cudy_tr3000-v1-256mb_multi_layout_defconfig` |
| `FIXED_MTDPARTS` | `1` | 固定分区表；多布局要求它必须为 1 |
| `SILENT` | `Y` | 找不到 multi-layout 配置时别 `read` 等输入（CI 里没有 stdin） |

日志里出现 `Features: fixed-mtdparts: 1, multi-layout: 1` 就说明参数对了，
产物名里也会带 `-multi-layout`。

---

## 许可

补丁均针对 U-Boot 源码，随 U-Boot 本体按 **GPL-2.0-or-later** 授权。
上游 `bl-mt798x-dhcpd` 与 U-Boot 的版权归各自作者所有。
