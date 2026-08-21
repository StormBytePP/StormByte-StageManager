# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-08-21

Initial public release of **StormByte-StageManager**: a Bash CLI for managing Gentoo stage tarballs (list, chroot, rebase, convert, export, delete, rename, download, notes), with configurable temporary storage, optional host bind mounts, ccache/sccache support (including shared FILE mode), locking, and idempotent cleanup.

### Added

#### Commands
- **`list`** (`ls`) — list stages under `TARBALL_FOLDER` with index, size, mtime, and `[notes]` / `[lock]` flags
- **`use`** — extract stage, apply binds and optional caches, enter interactive chroot, clean Portage temps, optionally save and recompress
- **`rebase`** — same as `use`, saving to a new filename (compression from the new extension)
- **`convert`** — recompress a stage in place to gzip / bzip2 / xz / zstd
- **`export`** — recompress into an existing destination folder with a dated output name
- **`delete`** (`rm`) — delete a stage (and notes sidecar) with confirmation
- **`rename`** — rename a stage within `TARBALL_FOLDER` (notes sidecar when present)
- **`download`** — fetch official Gentoo stage3 for i486 / i686 / amd64 / x32 and common profiles
- **`notes list`** / **`notes <stage> edit|delete`** — free-form per-stage notes via configured editor
- **`config`** — print resolved paths, storage, cache flags, shared workspace roots, binds, and environment overrides
- **`help`** / **`help <command>`** / **`help compression`** — built-in usage
- **`--version`** / **`-V`** — print program version

#### Stage selection
- Select stages by **zero-based index** or by **filename** (basename under `TARBALL_FOLDER` or absolute path)
- Supported extensions: `tar.gz`, `tgz`, `tar.bz2`, `tbz2`, `tar.xz`, `txz`, `tar.zstd`, `tzd`
- Sidecars `ccache.tbz2` / `sccache.tbz2` excluded from the stage list

#### Storage backends
- **`folder`**, **`tmpfs`**, **`zram`**, or **`auto`** (prefer zram, then tmpfs, then folder)
- Configurable size and optional zram compression algorithm (`zstd`, `lz4`, `lz4hc`)
- Fallback from zram to tmpfs when no free zram device is available

#### Host integration
- Optional bind mounts for PKGDIR, distfiles, Gentoo repos, `/etc/portage/patches`, and `EXTRA_BIND_MOUNT` (semicolon-separated absolute paths; protected roots rejected)
- Path validation for config paths (absolute, no `..`, restricted character set)

#### Compiler caches (ccache / sccache)
- **ccache** and **sccache** via host **BIND** mount or packed sidecars under `TARBALL_FOLDER` (`ccache.tbz2` / `sccache.tbz2`)
- **Shared FILE mode**: expand sidecars into a shared tmpfs workspace (`/tmp/StormByte-ccache`, `/tmp/StormByte-sccache` by default) so concurrent `use` / `rebase` sessions share one hot cache
- Coordination with **`flock`** and a holders list of live PIDs; dead PIDs pruned on acquire/release
- **Last session** to exit packs the workspace back into the sidecar and unmounts the tmpfs
- Sidecar pack is **independent** of stage “Save changes?” — compiler caches persist even when the stage is discarded
- Missing sidecars on first use are allowed (empty workspace, packed on last exit)
- Transparent chroot environment: `CCACHE_DIR`, `FEATURES+=ccache`, `SCCACHE_DIR`, `SCCACHE_CACHE_SIZE`, without writing cache config into the stage tree
- sccache server started on chroot entry and **stopped before unmount** (avoids busy tmpfs)
- `SKIP_CACHE=1` skips shared FILE acquire/pack for a single run
- Optional ccache statistics after leaving the chroot
- Optional overrides: `CCACHE_SHARE_ROOT`, `SCCACHE_SHARE_ROOT`, `CACHE_LOCK_DIR`, `CCACHE_TMPFS_SIZE`, `SCCACHE_TMPFS_SIZE`, `SCCACHE_CACHE_SIZE`

#### Download integrity
- Retries on transient network failures
- Best-effort verification against remote **DIGESTS** (SHA-512 preferred, else SHA-256)
- Validation of stage paths parsed from the info file

#### Safety and operations
- Per-stage lock files under `/tmp/.<basename>.lock`
- Idempotent emergency cleanup on `EXIT` / `INT` / `TERM` / `HUP` (stop sccache, drop stage-side cache binds, last-holder shared-cache pack, unmount, zram reset, partial `.compress` removal, lock removal)
- Interactive confirmations for destructive or overwrite actions; `FORCE=1` to skip where applicable
- `DRY_RUN=1` to simulate heavy or destructive actions
- Progress via **`pv`** (required) on extract, compress, download, and FILE cache pack paths
- Phase timing for extract / compress / download
- Free-space warning before large extract/compress work
- Parallel compressors when present (`pigz`, `lbzip2`, `pxz`, `pzstd`)

#### Environment overrides
- `TARBALL_FOLDER`, `STORAGE_SYSTEM`, `STORAGE_SIZE`, `DRY_RUN`, `FORCE`, `SKIP_CACHE`
- Shared cache roots and sizes as listed under Compiler caches

#### Packaging and docs
- Example configuration (`StormByte-StageManager.conf`) with generic default paths
- Bash completion (commands, aliases `ls`/`rm`, `config`, `notes list`, index and filename selection, download profiles, directory completion for export)
- Manual page (`StormByte-StageManager.1`) documenting shared FILE cache behaviour
- README documenting full CLI behaviour and shared cache lifecycle

### Notes

- This is the first stable public release of the StageManager CLI.
- Requires root for mounts, chroot, and zram operations.
- Requires `bash`, `tar`, `curl`, `find`, `pv`, and `flock`; parallel compressors and `zramctl` are recommended.
- Designed for Gentoo stage3 workflows but works with any matching rootfs tarball layout.

[1.0.0]: https://github.com/StormBytePP/StormByte-StageManager/releases/tag/1.0.0