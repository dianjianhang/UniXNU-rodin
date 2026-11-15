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
