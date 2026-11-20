# Rodin hardware notes — stock boot images

Source: user-dumped `boot.img` and `vendor_boot.img` from a Redmi Turbo 4
(rodin), 64 MiB partition dumps each. Parsed from raw headers on
2026-11-15.

Test device variant: **12 GiB LPDDR5X + 256 GiB UFS**.

## boot.img

```
magic           "ANDROID!"
header_version  4
kernel_size     0x0107f89b ≈ 16.6 MiB
ramdisk_size    0 (no boot ramdisk; vendor_boot supplies it)
```

## vendor_boot.img

```
magic              "VNDRBOOT"
header_version     4
page_size          4096 (0x1000)
kernel_load_addr   0x40000000
ramdisk_load_addr  0x66f00000
vendor_ramdisk_size 0x02963011 ≈ 41 MiB
cmdline            bootopt=64S3,32N2,64N2 erofs.reserved_pages=64
```

## Vendor DTB extraction (2026-11-15)

One FDT blob found inside vendor_boot.img, right after the vendor ramdisk:

```
offset      0x2965040
totalsize   0x6c969 (~444 KiB)
fdt version 17 (min compat 16)
```

Extracted and decompiled with dtc: `.cache/firmware/rodin-vendor.dtb` /
`rodin-vendor.dts` (hashes pinned in proprietary-files.txt).

### Confirmed values (read from the blob)

| Item | Value |
| --- | --- |
| Root | model "MT6899", compatible "mediatek,MT6899" |
| CPUs | 8x Cortex-A725, enable-method psci, MPIDR 0x000–0x700 |
| CPU groups | cpu0–3 (cap 0x400, domain 0), cpu4–6 (cap 0x3bb, domain 1), cpu7 (cap 0x3cb, domain 2) |
| GIC | arm,gic-v3 at 0xc400000 (dist 0x40000, redist 0xc440000 0x200000), 4 cells |
| Timer | arm,armv8-timer, clock-frequency 13 MHz (0xc65d40) |
| PSCI | arm,psci-1.0, method "smc" |
| Memory | memory@40000000, 1 GiB window (0x40000000 + 0x40000000) |
| GPU | mali@13000000, "mediatek,mali", "arm,mali-valhall" (G720 class) |
| Display | mediatek,mt6899-dsi (disp1_dsi0), mmsys0/1 syscon, disp GCE events |
| Storage | ufsphy@112a0000, UFS referenced throughout MTK PM resources |
| Debug UART | serial@11001000 with mt6985-uart-dma |
| Wi-Fi | wifi@18000000 "mediatek,wifi", emi-connac-ver 2 (Connac gen2 silicon) |
| Chosen bootargs | console=tty0, androidboot.hardware=mt6899, vmalloc=400M, swiotlb=noforce |
| Reserved memory | MTK-specific "mediatek,reserved" nodes (vendor format, needs mapping) |

### Open questions

- The DTB declares only a 1 GiB memory window; larger DRAM (8/12/16 GiB
  variants) handling must be confirmed (bootloader memory map / MTK DRAMC).
- No plain touch node found by name search; touch driver may be probed at
  runtime or named non-obviously.
- Panel vendor not embedded in this DTB (chosen at runtime by LCM probe).

## OS profile selection (2026-11-15)

UniXNU ships four OS profiles; selected `ipados-17.7.11-21H461`:

- **arm64** target (iPad7,1 / A10X identity, arm64 dyld shared caches) —
  correct for a third-party SoC without Apple PAC specifics
- The profile supplies `[ipsw]` (URL, product, hashes), `[kernel.prelink]`
  (kernelcache sha256, slide, roots) and the ADT Apple identity, so the
  device board.toml inherits them instead of duplicating values
- XNU 10063.141.1, Darwin 23.6.0
- `dram_size` override 0x300000000 (12 GiB test device) already in
  board.toml; ADT cpu_regs and unique_chip_id remain device-side values

Alternatives considered: `ipados-26.5.2-23F84` / `ipados-27.0.1-24A446`
(newer, AEA-encrypted images need macOS tooling) and `macos-27.3-26C5035r`
(desktop profile; target after iOS boots). Revisit 26/27 after first boot.

## GPT partition layout (parsed 2026-11-15 from /dev/block/sdc)

- Logical block size: **4096 bytes** (MTK UFS 4K block); disk ≈ 246.32 GiB
- A/B scheme: every OS image has `_a`/`_b` slots; bootloader is `lk_a/lk_b`
  (MTK LittleKernel), preloader outside the GPT on the boot LU (`sda`/`sdb`)
- `super`: LBA 524288–3407871 (11 GiB, dynamic partitions)
- **`userdata`: first_lba 0x354300, last_lba 0x3d937f7 (~233 GiB)**

board.toml `[userdata]` is filled from these values. Notable other
partitions: `expdb` (MTK panic logger), `pstore`, `protect1/2` (modem NVRAM),
`seccfg`, `proinfo`, `dram_para` (MTK DRAM training parameters — useful for
the memory-map survey), `gsort`, `flashinfo`.

## Kernel source (2026-11-15)

Official Xiaomi sources, both public:

| Repo | Branch | Commit | Contents |
| --- | --- | --- | --- |
| `MiCode/Xiaomi_Kernel_OpenSource` | `bsp-rodin-v-oss` | `2cd69ee4193920840ee3c837384716a66b1faecd` | GKI kernel (android15-6.6 style); no board DTS |
| `MiCode/MTK_kernel_device_modules` | `bsp-rodin-v-oss` | `bc2dfc01f75b5fbc42d614c8c39700022cba9323` | "rodin open source commit": board DTS at `arch/arm64/boot/dts/mediatek/` |

Board DTS files: `rodin.dts`, `rodin-mt6899-common.dtsi`,
`xiaomi_mt6899_common.dts`, `mt6899.dts`, `k6899v1_64.dts`, plus
`cust_mt6899_touch_1220x2712.dtsi` and per-subsystem cust dtsi files.
Clone with the matching GKI kernel for full builds.

## Touch (from xiaomi_rodin_mt6899_touch.dtsi)

- Touch IC: **Goodix GT9916R** (`goodix,firmware-name = "goodix_firmware_rodin"`),
  fallback candidate Focaltech 3683g
- Bus: **SPI1** at 15 MHz; pins GPIO50 (CLK) / GPIO51 (CSB) / GPIO52 (MISO) /
  GPIO53 (MOSI); SCP SPI2 alternate mode
- IRQ GPIO9, reset GPIO134, AVDD enable GPIO73 (1.8 V from `mt6368_vtp`)
- Coordinates 1220x2712 (panel-max-x 121999, super-resolution factor 100)
- Panel references: `rodin_42_02_0a_dsc_vdo` and `rodin_36_02_0b_dsc_vdo`

## Wi-Fi/BT chip correction

The vendor DTB `wifi@18000000` is MediaTek Connac gen2 silicon, and the
module sources confirm the chip is **MT7902** (gen4-mt79xx driver,
`connectivity/wlan/core/gen4-mt79xx/chips/mt7902`; BT over SDIO ID 0x790A,
BT firmware `BT_RAM_CODE_MT7902_1_1_hdr.bin`). Linux mt76 porting work
exists in the community (mt7902 patch sets based on this BSP).

## Firmware dump inventory (2026-11-15)

The `turbo4分区/` working directory holds a full partition dump dated
2025-09-10. Key images for the port (hashes pinned in
`proprietary-files.txt`):

| Image | Role |
| --- | --- |
| `dtbo.img` | device-tree overlays — panel config, display GPIOs, audio |
| `init_boot.img` | GKI ramdisk |
| `connsys_wifi.img` / `connsys_bt.img` | MT7902 Wi-Fi / BT firmware |
| `gpueb.img` | Mali GPU MCU firmware (gpueb) |
| `super.img` | dynamic partitions (system/vendor/product, 9.6 GiB) |
| `modem.img` | cellular baseband |
| `vbmeta*.img` | verified-boot metadata |

## DTBO analysis (from dtbo.img)

- Panel 1: `o10_42_02_0a_dsc_vdo,lcm` (matches `rodin_42_02_0a_dsc_vdo`
  referenced by the touch node)
- Panel 2: `o10_36_02_0b_dsc_vdo,lcm` — dual-source panel strategy
- LCM GPIOs: `lcm-rst-out0/1`, `lcm-led-en0/1`, `lcm-dsi-te` pins
- Audio: 2x `awinic,aw882xx_smartpa` on I2C 0x34/0x35
- LED: `awinic,aw21024_led` on I2C 0x30
- SIM tray: `xiaomi,simtray-status`
- Wi-Fi GPIO control node: `wlan,xiaomi_wifi_gpio`

## Consequences for the port

- `[bootimg.image]` in `board.toml`: base 0x40000000, pagesize 4096,
  header_version 4 (tool support unverified).
- The MTK boot chain differs from Qualcomm targets; the bootloader handoff
  and DTB handling need a dedicated survey before anything is flashed.
- Hashes are pinned in `proprietary-files.txt`.

## Resolver-chain verification (2026-11-15)

The UniXNU resolver chain was first verified end-to-end for rodin on
this date, on a local aarch64 Linux host (Debian 12, gcc 12.2.0):

| Step | Result | Notes |
|------|--------|-------|
| `unixnu defconfig rodin_defconfig` | exit 0 | `.config` written under out/ |
| `unixnu fetch` | exit 0 | xnu d8b80295, libplatform, libdispatch, AvailabilityVersions, mu_basecore + 5 submodules, libpthread, corecrypto-2023, IOStorageFamily |
| `unixnu status --json` | exit 0 | profiles arch=arm64 + soc=mt6899-common resolved; 14 porting areas validated |
| `unixnu tree` | exit 0 | build tree assembled under out/rodin/ |

Configuration fixes that made this work:

- `feeds.toml`: mt6899 pinned to UniXNU-mt6899-common
  `da0f6103bd3529a78c4ce1a2818190a07a6734c1` (was `REQUIRED_...`).
- `board.toml`: `schema_version = 2` and `soc_source = "mt6899"`
  restored (profiles.py requires both when schema_version = 2).
- CI passes the SoC layer via `--local-module`, which bypasses the
  feeds pin until the source repository is published upstream.
- `porting.toml`: now passes `scripts/porting-record.py check` (14
  structured areas; device.summary added; canonical area IDs).

## Kernel-build chain verification (2026-11-20)

The Full Build workflow now reaches the kernel build and gets past both
bootstrap gates that blocked earlier runs:

| Gate | Requirement | rodin setting |
|------|-------------|---------------|
| ArmCpu bring-up | `XNU_ARMCPU=1` needs `XNU_SOURCE_CORECRYPTO14=1` **and** (`XNU_ARMVIRT_PLATFORM=1` or `XNU_ARM_DT_PLATFORM=1`) | all four set in `configs/rodin_defconfig` |
| pthread recipe | `XNU_SOURCE_PTHREAD=1` needs `XNU_SOURCE_CORECRYPTO14=1` | both set |
| owner volumes | `CONFIG_XNU_OWNER_*_VOLUME_UUID` must parse as UUIDs (only read when the boot manifest / scratch Data mount is enabled) | left unset for bring-up; `REQUIRED_<uuid>` placeholders abort the bootstrap |

CI notes:

- `actions/cache` only saves in its post step when the job succeeds, so a
  failing kernel build used to throw the whole toolchain away. The Full
  Build now saves the toolchain in a dedicated `actions/cache/save` step
  with `if: always()`.
- The `UEFI Config Check` runs `unixnu release --no-deps --check`: it
  validates defconfig, module resolution and the patch plan without
  fetching sources/the IPSW (whose extraction needs the Linux toolchain's
  `lzfse`) and without building anything.

## XNU patch series (2026-11-20)

`[kernel] patch_series` must be the canonical four-part list from
`docs/porting/device-repository.md`:

```toml
patch_series = ["arch:patches/xnu/series", "soc:patches/xnu/series",
                "os:patches/xnu/series", "device:patches/xnu/series"]
```

With only `soc:` and `device:` (the earlier rodin config) none of the 55
ARM64 platform patches were applied, and the kernel build failed on three
symptoms that are all fixed by the arch series:

| Failure | Real cause | Fixing patch |
|---------|-----------|--------------|
| `pmap.h: duplicate member 'reserved10'` | plain arm64 + no Rosetta activates both `#else` branches | `non-apple-pmap-field.patch` (renames the PAC-side field) |
| `'CodeSignature/Entitlements.h' file not found` | header comes from the macOS SDK, absent on the Linux toolchain | `public-entitlements-header.patch` + `offline-owner-systemos-mount.patch` |
| `vm_user.c: use of undeclared identifier 'indices'` | `(void) indices;` left behind when deferred reclaim is off | `disabled-deferred-reclaim.patch` |

The arch and OS series deliberately omit two files and expect the SoC
series to supply them (the same split the sdm710 series uses):

- `native-smp-secondary.patch` — `MAX_CPUS` 4 → 8 and MPIDR Affinity1
  core numbering; MT6899 is 8x Cortex-A725 across clusters.
- `sheng-platform-power.patch` — the `bsd_init.c` PMGR call site that
  `zzz-boot-progress.patch` quotes as context. `XNU_SHENG_PMGR` is off for
  rodin, so the block compiles to nothing; the context is still required.

Both now live in `UniXNU-mt6899-common/patches/xnu/`. The resolved plan is
61 patches for rodin, and all 61 apply cleanly to xnu `d8b8029`
(10063.141.1). `ShengPlatformPower`'s archive is only force-loaded when
`XNU_SHENG_PMGR=1`, so the off state needs no extra build input.

## Boot image status

`boot.image` is not produced yet. `unixnu build kernel` + `unixnu tree`
do not pack one; the chain is:

1. `unixnu build firmware --uefi <Mu-Silicium checkout>` — EDK II build of
   the UEFI firmware volume (`[firmware]` in `board.toml`), then
2. `unixnu bootimg --uefi <Mu-Silicium checkout>` — BootShim + firmware
   volume + DTB into `out/<board>/<os>/<config>/boot/boot.img`.

Neither repository under `dianjianhang/` contains a Mu-Silicium
(`uefi-sheng`) checkout, and the device repository still lists boot.img,
vendor_boot.img, IPSW, GPT dump and the stock DTB as missing hardware
inputs. Until those exist, CI can only report "no boot image produced".
