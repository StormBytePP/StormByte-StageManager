# StormByte-StageManager

![License: GPL v3](https://img.shields.io/badge/license-GPLv3-blue.svg)
![Language: Bash](https://img.shields.io/badge/language-Bash-4EAA25.svg)
![Release](https://img.shields.io/badge/release-3.1.0-lightgrey)

`StormByte-StageManager` is a focused Bash utility to simplify maintenance of Linux stage tarballs (primarily Gentoo stage3 archives). It automates extracting, preparing, chrooting, cleaning, and recompressing stage tarballs so you can manage chroot environments quickly and reproducibly.

**Why this exists**
- Managing stage tarballs manually (mounts, tmpfs/zram, caches, chroot upkeep, and recompression) is error-prone and repetitive. This tool bundles common operations into a safe, scriptable workflow.

**Use cases**
- Prepare an editable chroot from a stage tarball and re-export it after changes.
- Rebase or clone an existing stage tarball into a new file name/format.
- Convert or export stage tarballs between compression formats.
- Maintain ccache/sccache, pkgdir and distfiles bindings while chrooting.
- Download official Gentoo stage tarballs for standard architectures/profiles.

**Supported compression formats**: gzip, bzip2, xz, zstd

**Storage backends for temporary mounts**: folder (plain directory), tmpfs, zram (with btrfs + zstd optional).

**Important**: Many operations require root privileges (mounting, chroot, manipulating loop/zram devices).

**Contents**
- `StormByte-StageManager` — main executable script
- `StormByte-StageManager.conf` — example configuration shipped with the project

## Quick Start

1. Make the script executable:

```bash
chmod +x StormByte-StageManager
```

2. Edit `StormByte-StageManager.conf` to point `TARBALL_FOLDER` and set desired storage behavior.

3. Run help to explore commands:

```bash
./StormByte-StageManager help
```

## Common Commands

- `list` — list available tarballs in `TARBALL_FOLDER` with indexes.
- `use <index>` — extract selected tarball into temporary storage, mount system binds, chroot into it, and prompt to save changes after exit.
- `rebase <index> <new_file_name>` — copy and rebase an archive into a new file.
- `convert <index> <compression_format>` — recompress an archive into another format.
- `export <index> <destination> <compression_format>` — export to a different folder/format.
- `delete <index>` — remove an archive from the store.
- `rename <index> <new_file_name>` — rename an archive.
- `download <arch> <profile>` — fetch an official stage tarball for the specified arch/profile.
- `notes <index> edit|delete` — manage notes for a specific stage file.

See `./StormByte-StageManager help <command>` for detailed usage and examples.

## Configuration highlights

- `TARBALL_FOLDER`: directory that contains stage tarballs.
- `STORAGE_SYSTEM`: `folder`, `tmpfs` or `zram` — controls temporary extraction/mount.
- `STORAGE_SIZE`: size for tmpfs/zram (e.g., `20G`).
- Compression level variables: `GZIP_COMPRESSION_LEVEL`, `BZIP_COMPRESSION_LEVEL`, `XZ_COMPRESSION_LEVEL`, `ZSTD_COMPRESSION_LEVEL`.
- Cache and bind options: `USE_CCACHE`, `USE_CCACHE_BIND`, `USE_SCCACHE`, `USE_SCCACHE_BIND`, `USE_PKG_BIND`, `USE_DISTFILES_BIND`.

Edit the provided `StormByte-StageManager.conf` to match your environment and copy it to `/etc/conf.d/` for system-wide usage if desired.

## Requirements

- bash (POSIX-compatible)
- tar
- curl (for downloads)
- btrfs-progs (when using zram with btrfs)
- Optional accelerated compressors: `pigz`, `pbzip2`/`lbzip2`, `pxz`, `zstd`

## Security & Privileges

Many operations require root. Be cautious when running scripts that mount filesystems or call `chroot`. Review configuration settings (especially bind mounts like `PKG_DIR` and `DISTFILES_DIR`) before running on production systems.

## Contributing

Contributions and bug reports are welcome. Fork the repository, create a branch, and open a pull request. Please include a clear description and minimal repro steps for any bug report.

## License

This project is licensed under the GNU General Public License v3.0 — see the bundled `LICENSE.txt`.

## Author

Developed and maintained by David C. Manuelda (StormByte) — StormByte@gmail.com
