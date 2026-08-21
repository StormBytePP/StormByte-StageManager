# StormByte-StageManager

[![License: GPL v3](https://img.shields.io/badge/license-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Language: Bash](https://img.shields.io/badge/language-Bash-4EAA25.svg)](https://www.gnu.org/software/bash/)
[![Release](https://img.shields.io/badge/release-1.0.0-lightgrey)](https://github.com/StormBytePP/StormByte-StageManager/releases)

`StormByte-StageManager` is a Bash CLI for managing **Gentoo stage tarballs** (and similar rootfs archives). It covers the full loop: list stages, extract into temporary storage, set up bind mounts, enter a chroot, clean Portage temporary data, and optionally recompress changes — plus convert, export, download official stage3 images, and free-form notes.

Style and UX are aligned with other StormByte tools (`StormByte-GitHub`): `cmd_*` dispatch, consistent help, environment overrides, lock files, and idempotent cleanup on interrupt.

---

## Why this tool?

Manual stage work is repetitive and easy to get wrong: mounting `/proc` `/dev` `/sys`, choosing tmpfs vs zram, packing compiler caches, recompressing multi‑GB archives, and not leaving mounts behind after Ctrl‑C.

StageManager turns that into a small set of commands with:

- **Safe selection** — stages by **index** or **filename**
- **Configurable storage** — `folder`, `tmpfs`, `zram`, or `auto`
- **Optional host binds** — packages, distfiles, repos, Portage patches, extra paths
- **ccache & sccache** — host bind **or** portable sidecar (`ccache.tbz2` / `sccache.tbz2`) via a **shared tmpfs** so concurrent sessions share one hot cache
- **Progress & integrity** — `pv` for extract/compress/download; download retries + DIGESTS verification when available
- **Locking & cleanup** — per-stage locks; emergency unmount, shared-cache release, zram reset, and lock removal on EXIT/INT/TERM/HUP

---

## Features

| Area | What you get |
|------|----------------|
| **Lifecycle** | `list` → `use` / `rebase` → save or discard stage |
| **Formats** | gzip, bzip2, xz, zstd (parallel tools when installed) |
| **Convert / export** | Recompress in place or to another folder (dated names) |
| **Download** | Official Gentoo stage3 (i486, i686, amd64, x32 + profiles) |
| **Notes** | Per-stage `.notes` files (`edit` / `delete` / `list`) |
| **Config dump** | `config` shows resolved paths, cache modes, and shared workspace roots |
| **Compiler caches** | ccache + sccache; BIND or shared FILE mode; last session packs sidecars |
| **CLI polish** | Aliases (`ls`, `rm`), `DRY_RUN`, `FORCE`, `SKIP_CACHE`, `--version` |
| **Shell** | Bash completion (index **and** filename, `notes list`, `config`) |
| **Docs** | `man StormByte-StageManager`, built-in `help` |

---

## Installation

```bash
git clone https://github.com/StormBytePP/StormByte-StageManager.git
cd StormByte-StageManager
chmod +x StormByte-StageManager
```

### Configuration

```bash
sudo mkdir -p /etc/conf.d /var/lib/StormByte-StageManager/stages
sudo cp StormByte-StageManager.conf /etc/conf.d/
sudoedit /etc/conf.d/StormByte-StageManager.conf
```

Standalone: put `functions.sh` and `StormByte-StageManager.conf` next to the script (same search order the CLI uses).

### Bash completion (optional)

```bash
sudo cp StormByte-StageManager.bash-completion \
  /usr/share/bash-completion/completions/StormByte-StageManager
# new shell, or: source that file
```

### Man page (optional)

```bash
sudo cp StormByte-StageManager.1 /usr/share/man/man1/
sudo mandb   # if your distro needs it
man StormByte-StageManager
```

Root is required for mounts, chroot, and zram device operations.

---

## Dependencies

**Required**

- `bash`, `tar`, `curl`, `find`, **`pv`** (progress on extract/compress/download and FILE cache pack)
- **`flock`** (util-linux) for shared FILE cache coordination

**Recommended**

- Parallel compressors: `pigz`, `lbzip2`, `pxz`, `pzstd`
- `zramctl` (util-linux) when using zram storage
- `ccache` / `sccache` **inside the stage** when those features are enabled

**Gentoo example**

```bash
emerge --ask app-misc/pv net-misc/curl \
  app-arch/pigz app-arch/lbzip2 app-arch/pxz app-arch/zstd sys-apps/util-linux
```

---

## Quick start

```bash
StormByte-StageManager download amd64 default
StormByte-StageManager list
StormByte-StageManager use 0          # or: use stage3-amd64-openrc-….tar.xz
# inside chroot: env-update && . /etc/profile ; work…
exit                                  # prompt: save stage changes?
```

```bash
StormByte-StageManager rebase 0 my-custom.tzd
StormByte-StageManager convert my-custom.tzd xz
StormByte-StageManager export 0 /backup zstd
```

---

## Commands

Run `StormByte-StageManager help` or `help <command>` for the built-in text.

### Stage selection

Wherever a stage is required you may pass:

- **Index** — as shown by `list` (0-based), or  
- **Filename** — basename under `TARBALL_FOLDER`, or an absolute path to an existing archive  

Recognised extensions: `tar.gz`, `tgz`, `tar.bz2`, `tbz2`, `tar.xz`, `txz`, `tar.zstd`, `tzd`.  
`ccache.tbz2` / `sccache.tbz2` are never treated as stages.

---

### `list` (alias: `ls`)

List archives in `TARBALL_FOLDER`:

| Column | Meaning |
|--------|---------|
| Idx | Index for other commands |
| Size | Approximate size in MiB |
| Mtime | Last modification date |
| File | Basename + `[notes]` / `[lock]` flags |

```bash
StormByte-StageManager list
```

---

### `use <index|filename>`

Full interactive workflow:

1. Acquire per-stage lock under `/tmp/.<basename>.lock`
2. Create temporary directory; mount storage (`folder` / `tmpfs` / `zram`)
3. Extract archive (`pv` + decompressor + `tar`)
4. Bind mounts: PKG, distfiles, repos, optional Portage patches, `EXTRA_BIND_MOUNT`
5. Prepare caches (BIND and/or shared FILE — see [Shared FILE cache](#shared-file-cache-ccache--sccache))
6. Mount `/proc`, `/dev`, `/sys`, `/run` into the tree
7. Enter interactive chroot with transparent cache environment (`CCACHE_DIR`, `FEATURES+=ccache`, `SCCACHE_*` as configured) — nothing is written into the stage tree for that
8. On exit: optional ccache stats, stop sccache server if needed, unmount system/binds, clean Portage temp/logs/history
9. Prompt **Save changes?** for the **stage tarball** only
10. Release shared FILE caches (last concurrent session packs sidecars), tear down storage, unlock

```bash
StormByte-StageManager use 0
StormByte-StageManager use stage3-amd64-openrc-20260101.tar.xz
SKIP_CACHE=1 StormByte-StageManager use 0
```

---

### `rebase <index|filename> <new_file_name>`

Same as `use`, but the stage save target is `new_file_name`. Compression is taken from the **new** file’s extension. Shared FILE cache behaviour matches `use`.

```bash
StormByte-StageManager rebase 0 my-hardened.tzd
```

---

### `convert` / `export` / `delete` / `rename` / `download` / `notes` / `config` / `help`

Unchanged in spirit from the 1.0.0 CLI surface:

| Command | Role |
|---------|------|
| `convert <stage> <format>` | Recompress under `TARBALL_FOLDER` (no chroot) |
| `export <stage> <dir> <format>` | Recompress into an existing folder (dated basename) |
| `delete` / `rm` | Delete stage (+ notes sidecar) after confirm |
| `rename` | Rename stage (+ notes when present) |
| `download <arch> [profile]` | Official stage3 + DIGESTS when available |
| `notes list` / `notes <stage> edit\|delete` | Free-form notes |
| `config` | Resolved paths, storage, **ccache/sccache**, shared roots |
| `help` / `--version` | Built-in help and version |

```bash
StormByte-StageManager convert 0 xz
StormByte-StageManager export 0 /mnt/backup zstd
StormByte-StageManager download amd64 hardened
StormByte-StageManager notes list
StormByte-StageManager config
```

---

## Shared FILE cache (ccache / sccache)

When `USE_CCACHE_FILE=YES` and/or `USE_SCCACHE_FILE=YES`, each cache type uses a **shared tmpfs** workspace so several concurrent `use` / `rebase` processes can share one hot cache.

| Item | Default |
|------|---------|
| ccache workspace | `/tmp/StormByte-ccache` (`CCACHE_SHARE_ROOT`) |
| sccache workspace | `/tmp/StormByte-sccache` (`SCCACHE_SHARE_ROOT`) |
| Lock / holder metadata | `/tmp/StormByte-cache-locks` (`CACHE_LOCK_DIR`) |
| Sidecars | `TARBALL_FOLDER/ccache.tbz2`, `TARBALL_FOLDER/sccache.tbz2` |

**Lifecycle (per cache type)**

1. **First** concurrent session creates the tmpfs (size `CCACHE_TMPFS_SIZE` / `SCCACHE_TMPFS_SIZE`, defaulting to `STORAGE_SIZE` or `8G`), unpacks the `.tbz2` if it exists, and registers as a holder (`flock` + holders file of live PIDs).
2. **Later** sessions bind the same workspace into their stage tree and join the holder list.
3. On exit, each session drops itself from the list. The **last** session packs the workspace back into the sidecar and unmounts the tmpfs.
4. Packing is **independent** of “Save changes?” for the stage — declining stage save still persists compiler caches in FILE mode.
5. **sccache**: the background server is stopped before any unmount (open files on the cache dir).
6. Missing sidecars on first use are fine: the workspace starts empty and is packed on last exit.
7. `SKIP_CACHE=1` skips shared FILE acquire/pack for that run.

**BIND mode** (`USE_*_BIND=YES`) remains a simple host directory bind; no sidecar pack/unpack.

Exactly one of BIND or FILE must be `YES` when the corresponding `USE_CCACHE` / `USE_SCCACHE` is `YES`.

---

## Compression formats

| Name | Extensions | Notes |
|------|------------|--------|
| **gzip** | `.tar.gz`, `.tgz` | Uses `pigz` if available |
| **bzip2** | `.tar.bz2`, `.tbz2` | Uses `lbzip2` if available |
| **xz** | `.tar.xz`, `.txz` | Uses `pxz` if available |
| **zstd** | `.tar.zstd`, `.tzd` | Uses `pzstd` |

Levels come from the config (`GZIP_COMPRESSION_LEVEL`, …, `ZSTD_COMPRESSION_LEVEL`).

---

## Global environment

| Variable | Effect |
|----------|--------|
| `DRY_RUN=1` | Simulate heavy/destructive actions |
| `FORCE=1` | Skip interactive Y/N where supported |
| `SKIP_CACHE=1` | Skip shared FILE cache acquire/pack for this run |
| `TARBALL_FOLDER=/path` | Override stage directory |
| `STORAGE_SYSTEM=…` | Override backend (`folder` / `tmpfs` / `zram` / `auto`) |
| `STORAGE_SIZE=…` | Override size for tmpfs/zram (and default shared cache tmpfs size) |
| `CCACHE_SHARE_ROOT` / `SCCACHE_SHARE_ROOT` | Override shared workspace mountpoints |
| `CACHE_LOCK_DIR` | Override lock/holder directory |
| `CCACHE_TMPFS_SIZE` / `SCCACHE_TMPFS_SIZE` | Shared workspace sizes (e.g. `8G`) |
| `SCCACHE_CACHE_SIZE` | sccache budget exported into the chroot (default `20G`) |

```bash
DRY_RUN=1 StormByte-StageManager delete 0
SKIP_CACHE=1 StormByte-StageManager use 0
TARBALL_FOLDER=/other/stages StormByte-StageManager list
```

---

## Configuration reference

File: `/etc/conf.d/StormByte-StageManager.conf` (or next to the script).

Default example paths are generic (e.g. `TARBALL_FOLDER=/var/lib/StormByte-StageManager/stages`); adjust to your layout.

### Storage

| Variable | Values / notes |
|----------|----------------|
| `TARBALL_FOLDER` | Absolute path to stages |
| `STORAGE_SYSTEM` | `folder`, `tmpfs`, `zram`, or `auto` |
| `STORAGE_SIZE` | e.g. `20G`, `512M` (tmpfs/zram; also default FILE workspace size) |
| `STORAGE_SYSTEM_ZRAM_COMPRESSION_ALGO` | `zstd`, `lz4`, `lz4hc`, or empty |

If `zram` is requested but no free device exists, the CLI may fall back to **tmpfs** with a warning.

### Editor / notes

| Variable | Notes |
|----------|--------|
| `TARBALL_NOTE_EDITOR` | e.g. `/bin/nano` |
| `DETACH_NOTE_EDITOR` | `YES` / `NO` |

### Compression levels

- `GZIP_COMPRESSION_LEVEL`, `BZIP_COMPRESSION_LEVEL`, `XZ_COMPRESSION_LEVEL` → **1–9**  
- `ZSTD_COMPRESSION_LEVEL` → **1–19**

### ccache / sccache

| Variable | Notes |
|----------|--------|
| `USE_CCACHE` / `USE_SCCACHE` | `YES` / `NO` |
| `CCACHE_DIR` / `SCCACHE_DIR` | Path inside the stage (and host path for BIND) |
| `USE_*_BIND` | Bind-mount host directory |
| `USE_*_FILE` | Shared tmpfs + pack/unpack sidecar under `TARBALL_FOLDER` |
| `SCCACHE_CACHE_SIZE` | Budget for sccache inside the chroot |

Optional overrides (conf or environment): `CCACHE_SHARE_ROOT`, `SCCACHE_SHARE_ROOT`, `CACHE_LOCK_DIR`, `CCACHE_TMPFS_SIZE`, `SCCACHE_TMPFS_SIZE`.

### Host binds

| Variable | Role |
|----------|------|
| `PKG_DIR` / `USE_PKG_BIND` | Binary packages |
| `DISTFILES_DIR` / `USE_DISTFILES_BIND` | Distfiles |
| `REPOS_DIR` / `USE_REPOS_BIND` | Gentoo ebuild tree |
| `USE_GENTOO_PATCHES_BIND` | Host `/etc/portage/patches` |
| `EXTRA_BIND_MOUNT` | `;`-separated absolute paths (protected roots rejected) |

Paths used in mounts are validated (absolute, no `..`, restricted character set).

---

## Locking and cleanup

- **Stage lock**: `/tmp/.<stage-basename>.lock` — concurrent use of the **same** stage is refused.
- **Shared FILE caches** intentionally allow concurrent sessions (across stages or the same stage only if locks were removed by hand).
- On **EXIT / INT / TERM / HUP**: idempotent cleanup — stop sccache if needed, drop stage-side cache binds, last-holder pack when this process was a holder, recursive unmount of the temp stage tree, optional zram reset, removal of partial `.compress` sidecars, and stage lock deletion.

---

## Exit status

| Code | Meaning |
|------|---------|
| **0** | Success (including a user abort that changes nothing) |
| **1** | Usage error |
| **2** | Runtime failure (config, I/O, mount, download, integrity) |

---

## Security notes

- Intended to run as **root** for mounts and chroot — review config and bind lists before production use.
- Bind mounts expose host directories into the stage; disable unused ones.
- Download paths from the stage info file are validated before use; DIGESTS are checked when present.
- Do not point `TARBALL_FOLDER`, bind paths, or shared cache roots at untrusted locations.

---

## Related files in this repository

| File | Role |
|------|------|
| `StormByte-StageManager` | Main CLI |
| `StormByte-StageManager.conf` | Example / default configuration |
| `functions.sh` | Shared StormByte Bash helpers (or system `/lib/StormByte/functions.sh`) |
| `StormByte-StageManager.bash-completion` | Bash completion |
| `StormByte-StageManager.1` | Manual page |
| `LICENSE.txt` | GPLv3 |

---

## Contributing

Fork, branch, and open a PR. Prefer changes that keep CLI, completion, man page, conf example, and this README in sync. Report issues with command line, config (redacted), and logs.

---

## License

GNU General Public License v3.0 — see `LICENSE.txt`.

## Author

David C. Manuelda (StormByte) — [StormByte@gmail.com](mailto:StormByte@gmail.com)
