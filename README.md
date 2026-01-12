# StormByte-StageManager

[![License: GPL v3](https://img.shields.io/badge/license-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Language: Bash](https://img.shields.io/badge/language-Bash-4EAA25.svg)](https://www.gnu.org/software/bash/)
[![Release](https://img.shields.io/badge/release-3.2.0-lightgrey)](https://github.com/StormBytePP/StormByte-StageManager/releases/tag/v3.2.0)

`StormByte-StageManager` is a streamlined Bash script designed to simplify the management of Linux stage tarballs, with a focus on Gentoo stage3 archives. It automates the entire workflow: extracting tarballs, setting up chroot environments, handling mounts and binds, cleaning up, and recompressing changes. This tool is ideal for developers, system administrators, and Gentoo enthusiasts who need reproducible, efficient chroot operations without manual hassle.

## Why Use This Tool?

Managing stage tarballs manually involves repetitive tasks like mounting system directories, handling temporary storage (e.g., tmpfs or zram), managing caches (ccache/sccache), and recompressing archives. `StormByte-StageManager` encapsulates these into safe, configurable commands, reducing errors and saving time. Key benefits:
- **Portability**: Self-contained operations with optional file-based caching for easy backups or multi-machine use.
- **Flexibility**: Supports multiple compression formats, storage backends, and bind mounts for packages/distfiles/caches.
- **Efficiency**: Uses accelerated compressors (e.g., pigz, pxz) when available and provides progress indicators via `pv`.

## Key Features

- Extract, chroot, modify, and recompress stage tarballs seamlessly.
- Rebase/clone tarballs to new files with automatic compression based on extension.
- Convert/export tarballs between formats (gzip, bzip2, xz, zstd).
- Download official Gentoo stage3 tarballs for common architectures (i486, i686, amd64, x32) and profiles (default, hardened, systemd, etc.).
- Manage notes for individual tarballs.
- Optional bind mounts or file-based storage for ccache/sccache, pkgdir, and distfiles.
- Temporary storage options: plain folder, tmpfs, or zram (with btrfs checksum and zstd compression).

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/StormBytePP/StormByte-StageManager.git
   cd StormByte-StageManager
   ```

2. Make the script executable:
   ```bash
   chmod +x StormByte-StageManager
   ```

3. Copy and customize the configuration:
   ```bash
   sudo mkdir -p /etc/conf.d/
   sudo cp StormByte-StageManager.conf /etc/conf.d/
   sudo nano /etc/conf.d/StormByte-StageManager.conf  # Edit as needed
   ```

For standalone use, place `functions.sh` and `StormByte-StageManager.conf` in the same directory as the script.

## Dependencies

- **Required**: bash, tar, curl (for downloads), pv (for progress bars).
- **Optional**:
  - Accelerated compressors: pigz (gzip), lbzip2 (bzip2), pxz (xz), pzstd (zstd).
  - btrfs-progs (for zram with btrfs).
  - zramctl (for zram storage).
- Root privileges are needed for mounts, chroot, and device operations.

Install on Gentoo:
```bash
emerge --ask app-arch/pigz app-arch/lbzip2 app-arch/pxz app-arch/zstd sys-fs/btrfs-progs sys-apps/util-linux app-misc/pv
```

## Usage

Run `./StormByte-StageManager help` for an overview or `./StormByte-StageManager help <command>` for details.

### Common Commands

| Command | Description | Example |
|---------|-------------|---------|
| `list` | List tarballs in `TARBALL_FOLDER` with indexes and sizes. | `./StormByte-StageManager list` |
| `use <index>` | Extract, chroot, and optionally save changes. | `./StormByte-StageManager use 0` |
| `rebase <index> <new_file>` | Clone and rebase to a new file (compression from extension). | `./StormByte-StageManager rebase 0 new-stage.txz` |
| `convert <index> <format>` | Recompress to a new format (e.g., xz). | `./StormByte-StageManager convert 0 xz` |
| `export <index> <dest_folder> <format>` | Export to a folder with new format and dated name. | `./StormByte-StageManager export 0 /backup xz` |
| `delete <index>` | Delete a tarball. | `./StormByte-StageManager delete 0` |
| `rename <index> <new_name>` | Rename a tarball. | `./StormByte-StageManager rename 0 renamed-stage.txz` |
| `download <arch> <profile>` | Fetch official Gentoo stage3. | `./StormByte-StageManager download amd64 hardened` |
| `notes <index> edit\|delete` | Manage notes for a tarball. | `./StormByte-StageManager notes 0 edit` |

### Examples

- Download and use an amd64 stage:
  ```bash
  ./StormByte-StageManager download amd64 default
  ./StormByte-StageManager list  # Note the index
  ./StormByte-StageManager use <index>
  # Inside chroot: make changes, e.g., emerge packages
  exit  # Prompt to save
  ```

- Rebase with new compression:
  ```bash
  ./StormByte-StageManager rebase 0 my-custom-stage.tzd
  ```

## Configuration

The config file (`/etc/conf.d/StormByte-StageManager.conf` or local) allows customization. Key options:

- `TARBALL_FOLDER`: Path to store tarballs (default: `/StormWarehouse/Software/Gentoo`).
- `STORAGE_SYSTEM`: `folder`, `tmpfs`, or `zram`.
- `STORAGE_SIZE`: e.g., `20G` (for tmpfs/zram).
- Compression levels: `GZIP_COMPRESSION_LEVEL=9`, etc.
- Caches: `USE_CCACHE=YES`, `USE_CCACHE_BIND=YES` (host bind) or `USE_CCACHE_FILE=YES` (file-based .txz).
- Binds: `USE_PKG_BIND=YES`, `PKG_DIR=/var/db/repos/packages`.
- Checksum for zram+btrfs: `BTRFS_CHECKSUM=sha256`.

Note: For caches, bind is faster for repeated use; file-based is portable but involves compress/decompress overhead.

## Security Considerations

- Runs as root for mounts/chroot—review code and configs before use.
- Bind mounts expose host directories to chroot; disable if unneeded.
- Lockfiles prevent concurrent ops on the same tarball.

## Contributing

Fork, branch, and PR! Include tests or examples for new features. Report issues with steps to reproduce.

## License

GNU General Public License v3.0 (see `LICENSE.txt`).

## Author

David C. Manuelda (StormByte) — [StormByte@gmail.com](mailto:StormByte@gmail.com)  