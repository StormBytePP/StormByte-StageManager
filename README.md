This is an script to help the managing of the Gentoo stages, chroot, package and export.

Its requirements are:
Mandatory:
* bash
* btrfs-progs
* curl
* tar

Configuration based (depends on the chosen compression algorithm)
* pigz
* pbzip2
* pxz
* zstd

Depending on the storage methos:
* zram

A detailed help is available by calling the script with help command, like

```bash
Stormbyte-StageManager help
```

Please remember to fill StormByte-StageManager.conf file prior its usage!

Hint: If you "use" a Gentoo tarball, you will be automatically chrooted into it, once you type ```exit``` chroot will end and you will be prompted to save or discard your changes.
