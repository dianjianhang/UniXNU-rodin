# AGENTS.md — UniXNU-rodin

Device repository for Redmi Turbo 4 / POCO X7 Pro (rodin), SoC
MT6899 (Dimensity 8400-Ultra). Read this before editing.

## Responsibilities

- `board.toml` — format 2 board configuration; identity must match
  `porting.toml` (name "rodin", soc "mt6899").
- `porting.toml` — authoritative device record; update findings or
  evidence for every changed code path, then run the checker.
- `feeds.toml` — pinned SoC source (UniXNU-mt6899-common) and component
  sources.
- `configs/`, `patches/xnu/series` — ordered configuration fragments and
  patch series; every referenced file must exist.
- `dts/rodin.dts` — board DTS; includes the SoC description resolved from
  the `soc:` include directory.
- `package/rodin.kit` — payload package; add `[[driver]]`, `[[overlay]]`
  and `[[kext]]` entries as drivers are adapted.
- `proprietary-files.txt`, `.cache/firmware` — required firmware inputs
  and hashes.

## Required reading

- https://unixnu.com/porting/steps
- https://unixnu.com/porting/device-repository
- https://unixnu.com/porting/records
- https://unixnu.com/reference/components

## Rules

- Keep hardware values sourced: record where each value came from in the
  owning porting.toml area.
- `porting.toml` status levels: planned, implemented, host-verified,
  device-verified, not-applicable.
- Never write local paths into tracked files; use the git-ignored
  `local.toml` (copy from `local.toml.example`).
