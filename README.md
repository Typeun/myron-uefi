# myron-uefi

Mu-Silicium platform package for the **Redmi K90 Pro Max（myron，SM8850 / Canoe）**。
只用于 `fastboot boot` 的内存启动，不写任何分区。

本仓库由同 SoC、同板级的 [`songyuan-uefi`](https://github.com/WUKAWL/songyuan-uefi)
生成，生成器是 `tools/make-myron-uefi.py`：换掉平台名、SMBIOS 身份、framebuffer 尺寸，
并把设备树换成从这台 myron 上读出的原厂 `canoe.dtb`。驱动二进制与 ACPI 表仍沿用
Mu-Silicium 上游的 `aston`（一加 15，同为 SM8850）——这一层两个板子本来就共用。

## 为什么可以这样生成

`myron` 和 `songyuan` 是两个 SKU 的同一块板子。逐项比对过：

| 项 | myron（本仓库） | songyuan |
| --- | --- | --- |
| SoC | SM8850 Canoe，`qcom,chipid=0x44050A01` | 同 |
| 屏幕 | 1200×2608，stride 4800 | 同 |
| 触控 | FocalTech FT3683，QUP4 SE0，`spi19.0` | 同 |
| UFS | `ufshc@1d84000`，userdata `sda34` | 同 |
| `/reserved-memory` | 59 段 | **逐段地址与大小完全相同** |
| CPU / GIC / PSCI | 8 核，`0x17000000` / `0x17080000`，`smc` | 同 |

所以固件侧只需要身份和 DTB，物理内存表可以直接继承——并且已经用本机 DTB 复核过。

## 目录

| 路径 | 内容 |
| --- | --- |
| `Platforms/Xiaomi/myronPkg/` | 平台包：`DeviceBuild.py`、`myron.dsc/.fdf/.dec`、`Include/*.inc` 驱动清单、`Library/MemoryMapLib`、`Library/ConfigurationMapLib` |
| `Resources/Configs/myron.toml` | UEFI FD 基址/大小（`0xA7100000` + `0x300000`）、boot image 形态（`header_version = 0`、gzip 内核、追加 DTB） |
| `Resources/DTBs/myron.dtb` | 本机原厂设备树，581,416 字节，SHA-256 `42a2257670e909f82540c3a156444f17f50644702363a6b0f747b1d48e3aa61b` |
| `validate_memory_map.py` | 离线内存表校验：描述符重叠、边界钉值 |
| `.github/workflows/build.yml` | 在 Ubuntu 上克隆 Mu-Silicium、覆盖本包、产出 `Build/myronPkg/RELEASE_CLANGPDB/FV/SILICIUM_UEFI.fd` |

## 用法

```bash
# 在 UniXNU 主仓库里
./unixnu build firmware -D ../myron-device --uefi ../myron-uefi
./unixnu payload -D ../myron-device
./unixnu bootimg -D ../myron-device
```

产出的 `boot.img` 用 `fastboot boot` 临时启动。

构建前先跑离线校验：

```bash
python3 validate_memory_map.py
# PASS: 76 descriptors, 62 AddMem ranges, no overlaps
```

## 状态

**已验证（离线）**

- 内存表 76 个描述符、62 个 `AddMem` 段无重叠，FD / Kernel / XNU payload / DXE heap 四条边界都钉在保留区上；
- `myron.dtb` 的 `/reserved-memory` 与本机读出的 59 段完全一致；
- 用本机 live FDT（`/proc/device-tree`，含 ABL 叠加的 dtbo）复核：48 段实际保留区**全部**落在 `Reserv` 描述符内，
  没有一段与交给 DXE/XNU 的常规内存相交；
- 平台包内无 `songyuan` 残留，身份字段为本机 `getprop` 值。

**未验证**

- 本机是 Windows，没有 EDK2 工具链，**没有编译过这个包**；首次编译要在 Linux 或 GitHub Actions 上做；
- **没有在真机上 `fastboot boot` 过**；
- ACPI 仍是一加 15 的表，`aston` 驱动二进制是否完全适配本机外设未知；
- 面板不重新初始化（用固件 framebuffer），刷新靠上层。

## 重新生成

```bash
python tools/make-myron-uefi.py \
  --source tmp-songyuan-uefi \
  --out myron-uefi \
  --dtb device-info/myron/canoe.dtb
```

## License

BSD-2-Clause-Patent.
