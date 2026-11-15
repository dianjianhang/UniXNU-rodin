# UniXNU-rodin — Redmi Turbo 4 / POCO X7 Pro

UniXNU device repository for **rodin** (Redmi Turbo 4, China; identical to
POCO X7 Pro): MediaTek **Dimensity 8400-Ultra (MT6899)**, 8x Cortex-A725,
Mali-G720, UFS storage.

> **Status: new port scaffold.** No value is hardware-verified yet. Fill
> every `REQUIRED_` / TODO entry before the first build, following
> [Port a device](https://unixnu.com/porting/steps).

## Layout

```
UniXNU-rodin/
  AGENTS.md
  README.md
  .gitignore
  board.toml          # format 2 board configuration (REQUIRED_ values pending)
  porting.toml        # device porting record, schema 1, all areas "planned"
  feeds.toml          # SoC source pin (UniXNU-mt6899-common)
  configs/rodin_defconfig
  dts/rodin.dts
  patches/xnu/series
  scripts/package.sh
  proprietary-files.txt
  package/rodin.kit/Kit.toml
  local.toml.example
  docs/hardware-notes.md
```

## Prerequisites

- Linux host with the UniXNU toolchain
- Unlocked bootloader
- Firmware inputs: boot.img, vendor_boot.img (collected), IPSW, GPT dump,
  stock DTB
- Main UniXNU checkout beside this repository

## Build

From the main UniXNU checkout:

```sh
./unixnu defconfig rodin_defconfig -D ../UniXNU-rodin
./unixnu fetch -D ../UniXNU-rodin
./unixnu status --json -D ../UniXNU-rodin
./unixnu tree -D ../UniXNU-rodin
python3 scripts/porting-record.py check --device ../UniXNU-rodin
```

## References

- https://unixnu.com/porting/steps
- https://unixnu.com/porting/device-repository
- https://unixnu.com/devices/
