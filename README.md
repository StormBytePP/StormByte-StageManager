# StormByte-StageManager

**StormByte-StageManager** is a Bash script designed to manage Gentoo chroot environments from compressed tarballs. It provides functionality for listing, using, rebasing, converting, exporting, deleting, renaming, downloading, and managing notes for tarballs. The script also handles mounting/unmounting system structures and temporary storage.

## Features

- Manage Gentoo chroot environments with ease.
- Support for multiple compression formats (gzip, bzip2, xz, zstd).
- Automatic mounting and unmounting of system directories.
- Export and convert tarballs to different formats.
- Download Gentoo stage tarballs for various architectures and profiles.
- Manage notes for tarballs.

## Requirements

### Mandatory
- `bash`
- `btrfs-progs`
- `curl`
- `tar`

### Compression Tools (based on configuration)
- `pigz` (for gzip)
- `pbzip2` (for bzip2)
- `pxz` (for xz)
- `zstd` (for zstd)

### Storage Methods
- `zram` (if using ZRAM-based temporary storage)

## Installation

To install **StormByte-StageManager**, clone the repository from GitHub and set up the necessary configuration:

1. Clone the repository:
   ```bash
   git clone https://github.com/StormBytePP/StormByte-StageManager.git
   cd StormByte-StageManager
   ```

2. Make the script executable:
   ```bash
   chmod +x StormByte-StageManager
   ```

3. (Optional) Move the script to a directory in your `PATH` for global access:
   ```bash
   sudo mv StormByte-StageManager /usr/local/bin/
   ```

4. Copy the configuration file to `/etc/conf.d/`:
   ```bash
   sudo cp conf.d/StormByte-StageManager.conf /etc/conf.d/
   ```

5. Edit the configuration file to suit your environment:
   ```bash
   sudo nano /etc/conf.d/StormByte-StageManager.conf
   ```

## Configuration

Before using the script, ensure that the configuration file `/etc/conf.d/StormByte-StageManager.conf` is properly filled out. This file contains important settings such as the tarball folder, compression levels, and storage options.

## Usage

The script provides several commands to manage Gentoo tarballs. Below are examples of how to use the script:

### Display Help
To view the general help or help for a specific command:
```bash
StormByte-StageManager help
StormByte-StageManager help list
StormByte-StageManager help use
```

### List Available Tarballs
To list all tarballs in the configured folder:
```bash
StormByte-StageManager list
```

### Use a Tarball
To use a tarball and automatically chroot into it:
```bash
StormByte-StageManager use <index>
```
- Replace `<index>` with the index of the tarball from the `list` command.
- After exiting the chroot (`exit`), you will be prompted to save or discard your changes.

### Rebase a Tarball
To create a new tarball based on an existing one:
```bash
StormByte-StageManager rebase <index> <new_file_name>
```
- Replace `<index>` with the index of the tarball from the `list` command.
- Replace `<new_file_name>` with the desired name for the new tarball.

### Convert a Tarball
To convert a tarball to a different compression format:
```bash
StormByte-StageManager convert <index> <compression_format>
```
- Replace `<compression_format>` with one of the supported formats: `gzip`, `bzip2`, `xz`, `zstd`.

### Export a Tarball
To export a tarball to a specific folder with a new compression format:
```bash
StormByte-StageManager export <index> <destination_folder> <compression_format>
```

### Delete a Tarball
To delete a tarball:
```bash
StormByte-StageManager delete <index>
```

### Rename a Tarball
To rename a tarball:
```bash
StormByte-StageManager rename <index> <new_file_name>
```

### Download a Gentoo Stage Tarball
To download a Gentoo stage tarball for a specific architecture and profile:
```bash
StormByte-StageManager download <arch> <profile>
```
- Replace `<arch>` with the desired architecture (e.g., `amd64`, `i686`).
- Replace `<profile>` with the desired profile (e.g., `default`, `hardened`, `systemd`).

### Manage Notes
To edit or delete notes for a tarball:
```bash
StormByte-StageManager notes <index> edit
StormByte-StageManager notes <index> delete
```

## Example Workflow

1. **List available tarballs**:
   ```bash
   StormByte-StageManager list
   ```
   Output:
   ```
   [0] 1.2G stage3-amd64.tar.xz
   [1] 1.3G stage3-hardened.tar.xz
   ```

2. **Use a tarball**:
   ```bash
   StormByte-StageManager use 0
   ```
   This will chroot into the environment. After exiting (`exit`), you will be prompted to save or discard changes.

3. **Rebase the tarball**:
   ```bash
   StormByte-StageManager rebase 0 stage3-amd64-updated.tar.xz
   ```

4. **Convert the tarball to a different format**:
   ```bash
   StormByte-StageManager convert 0 gzip
   ```

5. **Download a new Gentoo stage tarball**:
   ```bash
   StormByte-StageManager download amd64 default
   ```

## Notes

- Ensure that all required tools and dependencies are installed before using the script.
- The script requires root privileges for certain operations (e.g., mounting system directories, chrooting).
- **Important**: After adding or removing a tarball, the indexes may change. Always run the `list` command again before referencing a tarball to avoid unintended operations.

## License

This project is licensed under the **GNU General Public License v3.0**. See the [LICENSE.txt](LICENSE.txt) file for details.

## Author

Developed by **David C. Manuelda** a.k.a **StormByte** (<StormByte@gmail.com>).

## Contribution

Contributions are welcome! Feel free to submit issues or pull requests to improve the script.
