# StormByte-StageManager

[![License: GPL v3](https://img.shields.io/badge/license-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Language: Bash](https://img.shields.io/badge/language-Bash-4EAA25.svg)](https://www.gnu.org/software/bash/)
[![Release](https://img.shields.io/badge/release-2.0.0-blue.svg)](https://github.com/StormBytePP/StormByte-StageManager/releases)
[![helpers bash](https://img.shields.io/badge/functions.sh-%E2%89%A5%201.3.0-555)](https://github.com/StormBytePP/StormByte.Repository)

CLI for Gentoo stage tarballs: list them, live inside one, save or throw the
tree away, recompress without unpacking, fetch official stage3 images, leave
notes. The parts that used to eat an afternoon — `/proc` `/dev` `/sys`, tmpfs
vs zram, packing ccache, not leaving mounts after Ctrl-C — are the script’s
job, not yours.

Needs **StormByte-functions-bash ≥ 1.3.0** (`confirm_yes`, `loadConfig`,
`require_cmd`, `path_normalize`, `handleCommandWithOutput`, …). Same library
StormByte-GitHub uses.

---

## Why 2.0.0 exists

1.x listed `.tzd` and then refused to *open* `.tzd`. Convert extracted a
multi-gigabyte tree just to change gzip → xz. Everything demanded root,
including `list`.

2.0.0:

- One codec table. Read the old names *and* the short ones. Write short names.
- `convert` / `export` stream (or `cp` when the codec does not change). No
  extract, no tmpfs, no root.
- `use` / `rebase` still need root. If you are not root they re-exec
  `sudo -E`. Artefacts written under sudo go back to `SUDO_UID`.
- The EXIT/INT/TERM/HUP trap only tears down what *this* PID mounted or
  locked. It will not `rm -rf` a directory that is still a mountpoint.

`use` and `rebase` stay. That workflow is the point of the tool.

---

## Install

```bash
git clone https://github.com/StormBytePP/StormByte-StageManager.git
cd StormByte-StageManager
chmod +x StormByte-StageManager
```

```bash
sudo mkdir -p /etc/conf.d /var/lib/StormByte-StageManager/stages
sudo cp StormByte-StageManager.conf /etc/conf.d/
sudoedit /etc/conf.d/StormByte-StageManager.conf
```

`loadConfig` looks at `${workdir}/${self}.conf` first, then
`/etc/conf.d/${self}.conf`. Do not vendor `functions.sh` next to the script
unless you are hacking; the packaged `/lib/StormByte/functions.sh` is the
one that must satisfy ≥ 1.3.0.

```bash
sudo cp StormByte-StageManager.bash-completion \
  /usr/share/bash-completion/completions/StormByte-StageManager
sudo cp StormByte-StageManager.1 /usr/share/man/man1/
sudo mandb
```

On Gentoo the ebuild already drops the binary, conf.d file and completion.

---

## Dependencies

**Required:** `bash`, `tar`, `curl`, `find`, `pv`, `flock` (util-linux),
**StormByte-functions-bash ≥ 1.3.0**.

**Recommended:** `pigz`, `lbzip2`, `pxz`, `pzstd`, `zramctl`, `sudo`.
`ccache` / `sccache` only need to exist *inside the stage* if you enable them.

```bash
emerge --ask app-misc/pv net-misc/curl \
  app-arch/pigz app-arch/lbzip2 app-arch/pxz app-arch/zstd sys-apps/util-linux \
  sys-libs/StormByte-functions-bash
```

---

## Quick start

```bash
StormByte-StageManager download amd64 default
StormByte-StageManager list
StormByte-StageManager use 0
# inside: env-update && . /etc/profile
# work…
exit
# Save changes? [y/N]
```

```bash
StormByte-StageManager rebase 0 Wesker+base.tzd
StormByte-StageManager convert 0 xz          # → <stem>.txz next to the original
StormByte-StageManager export 0 /backup      # → <stem>-YYYYMMDD.txz  (always xz)
```

`use` as a normal user:

```
Privileges    re-exec via sudo -E
[sudo] password for stormbyte:
```

Same flags and `TARBALL_FOLDER` survive. The new tarball is not left
`root:root`.

---

## Stage names

Index from `list` (0-based) or a filename (basename, path under
`TARBALL_FOLDER`, or absolute).

| Codec | Written by convert | Also read |
|-------|--------------------|-----------|
| gzip  | `.tgzip`           | `.tar.gz` `.tgz` |
| bzip2 | `.tbzip2`          | `.tar.bz2` `.tbz2` |
| xz    | `.txz`             | `.tar.xz` |
| zstd  | `.tzd`             | `.tzstd` `.tar.zst` `.tar.zstd` |

`ccache.tbz2` / `sccache.tbz2` are sidecars, never stages.

`convert` / `export` take **format names** (`gzip` `bzip2` `xz` `zstd`), not
extensions. Same codec as the source → copy (`cp --reflink=auto`, else `cp`).
Different codec → `pv | decompress | compress > .partial && mv`. Ctrl-C
deletes the partial; the original stage is untouched.

---

## Commands

| Command | Root? | What it does |
|---------|-------|----------------|
| `list` (`ls`) | no | Index, size, mtime, `[notes]` `[lock]` |
| `use <sel>` | yes (`sudo -E`) | Extract, binds, chroot, optional save |
| `rebase <sel> <new>` | yes (`sudo -E`) | `use`, save to `<new>` (codec from its extension) |
| `convert <sel> <fmt>` | no | Stream or copy next to the original |
| `export <sel> <dir>` | no | Always xz; `<stem>-YYYYMMDD.txz` in an existing dir |
| `delete` (`rm`) | no* | Stage + notes, after confirm |
| `rename <sel> <new>` | no* | Stage + notes inside `TARBALL_FOLDER` |
| `download <arch> [profile]` | no* | Official stage3 + DIGESTS when present |
| `notes list` / `notes <sel> edit\|delete` | no | Sidecar `.notes` |
| `config` | no | Resolved paths and flags |
| `help` / `help <cmd>` / `--version` | no | Built-in text |

\*Needs write access to `TARBALL_FOLDER`, not uid 0.

`STORAGE_SYSTEM` / `STORAGE_SIZE` apply to `use` / `rebase` only.

---

## A day with it

Download once, live in it many times:

```bash
StormByte-StageManager download amd64 hardened
StormByte-StageManager notes 0 edit          # what this tree is for
StormByte-StageManager use 0
# emerge -uDN @world, break nothing on the host
exit
# n → throw the tree away; FILE ccache still packed if you were last holder
```

Promote a session to a named stage:

```bash
StormByte-StageManager rebase 0 Wesker+base+system.tzd
StormByte-StageManager list
```

Ship a copy without unpacking 20G:

```bash
StormByte-StageManager export 2 /StormWarehouse/backup
# Wesker+base+system+Cinnamon-20260901.txz
```

Recompress in place-ish (new file, same folder):

```bash
StormByte-StageManager convert Wesker+base.tzd gzip
# Wesker+base.tgzip
StormByte-StageManager convert Wesker+base.tzd zstd
# same codec → copy, or no-op if the target path is the source
```

Dry-run before you delete a week of work:

```bash
DRY_RUN=1 StormByte-StageManager delete 2
FORCE=1 StormByte-StageManager delete 2      # CONFIRM=1, no prompt
```

Skip the shared compiler cache for one shot:

```bash
SKIP_CACHE=1 StormByte-StageManager use 0
```

---

## Shared FILE cache

`USE_CCACHE_FILE=YES` and/or `USE_SCCACHE_FILE=YES`:

| Piece | Default |
|-------|---------|
| ccache workspace | `/tmp/StormByte-ccache` |
| sccache workspace | `/tmp/StormByte-sccache` |
| locks / holders | `/tmp/StormByte-cache-locks` |
| sidecars | `$TARBALL_FOLDER/ccache.tbz2`, `sccache.tbz2` |

First session mounts the tmpfs and unpacks the sidecar if it exists. Later
sessions bind the same workspace. Last session out packs and unmounts.
Independent of “Save changes?”. sccache’s server is stopped before any umount.
`SKIP_CACHE=1` skips acquire/pack for that run.

BIND mode is the other exclusive: host directory, no sidecar.

---

## Storage backends (`use` / `rebase`)

| `STORAGE_SYSTEM` | Meaning |
|------------------|---------|
| `folder` | Plain directory under `/tmp` |
| `tmpfs` | RAM, size `STORAGE_SIZE` |
| `zram` | Compressed RAM + ext2. No free device → tmpfs + warning |
| `auto` | zram, else tmpfs, else folder |

---

## Environment

| Variable | Effect |
|----------|--------|
| `DRY_RUN=1` | Print what would happen |
| `FORCE=1` | Skip `[y/N]` (`CONFIRM=1` for `confirm_yes`) |
| `SKIP_CACHE=1` | No FILE acquire/pack |
| `TARBALL_FOLDER=/path` | Override conf |
| `STORAGE_SYSTEM=` / `STORAGE_SIZE=` | Override backend (`use`/`rebase`) |
| `CCACHE_SHARE_ROOT` / `SCCACHE_SHARE_ROOT` / `CACHE_LOCK_DIR` | FILE workspace |
| `CCACHE_TMPFS_SIZE` / `SCCACHE_TMPFS_SIZE` | FILE tmpfs size |
| `SCCACHE_CACHE_SIZE` | Budget inside the chroot (default `20G`) |

---

## Locking and Ctrl-C

Lock: `/tmp/.<basename>.lock`. Two `use` of the *same* basename refuse.
Different stages can share the FILE cache.

Trap (EXIT / INT / TERM / HUP), only what this PID created:

1. stop sccache
2. umount stage binds and `/proc` `/dev` `/sys` `/run`
3. last-holder FILE pack
4. umount storage
5. reset zram if this PID allocated it **and** the mount is gone
6. delete `.partial` / `.compress`
7. drop the lock

No `rm -rf` on a live mountpoint. `list` / `help` / `config` never set the
inventory flags, so the trap is a no-op there.

---

## Configuration

`/etc/conf.d/StormByte-StageManager.conf` (or next to the script). Keys are
the same as 1.x — paths, `USE_*`, compression levels, binds. `config` dumps
what the process actually resolved.

Binds (`PKG_DIR`, `DISTFILES_DIR`, `REPOS_DIR`, `/etc/portage/patches`,
`EXTRA_BIND_MOUNT` as `;`-separated absolute paths) are validated: absolute,
no `..`, restricted alphabet, protected roots rejected.

---

## See also

`StormByte-StageManager help`  
`StormByte-StageManager help compression`  
`man StormByte-StageManager`

---

## License

GPLv3. David C. Manuelda (StormByte) — [StormByte@gmail.com](mailto:StormByte@gmail.com)
