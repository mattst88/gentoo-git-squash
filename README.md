# gentoo-git-squash
A script to create a SquashFS or EROFS snapshot of a git repo

## What

`gentoo-git-squash` is a bash script to update an arbitrary git repo and make a SquashFS or EROFS snapshot of its contents.

## Why

[Gentoo](https://www.gentoo.org/)'s ebuild repository contains more than 100 thousand files, most of which are very tiny (more than 95% are less than 4 KiB). As it's common for ebuilds of different versions of the same package to be identical except for their filename, some content in the repository is duplicated multiple times.

This has negative consequences for on-disk space utilization and for runtime performance of applications accessing the files, notably the [Portage package manager](https://wiki.gentoo.org/wiki/Portage).

### Smaller on-disk storage requirements
Storing the ebuild repository in a SquashFS snapshot greatly improves both problems because the file contents are deduplicated and compressed. A [::gentoo](https://gitweb.gentoo.org/repo/gentoo.git/) repository usually consumes 500 MiB to 1.3 GiB on-disk depending on the amount of git history retained. A SquashFS snapshot using `gzip -9` compression is 45 MiB, and the bare git repository from which it is created is 74 MiB.

### Faster runtime file access
A package manager accessing the ebuild repository potentially reads thousands of files per operation, issuing multiple syscalls to access each file: `open()`, `stat()`, `read()`, `close()`. Accessing many tiny files is often a sore spot for file system performance, and the problem is significantly worsened for networked file systems where each syscall might require a synchronous round-trip over the network.

A SquashFS image mounted locally, even if it is hosted on a network file share, avoids these problems. Files read from the SquashFS image are transparently decompressed into the page cache for fast subsequent accesses.

## How

`gentoo-git-squash` clones the repository specified by the `$repouri` variable into a directory specified by `$gitdir` (or fetches and updates it, if `$gitdir` already exists). It configures the repository to minimize the amount of history (and thus on-disk storage requirements) and executes `git gc` after each update.

From the bare git repository, `git archive` and `tar2sqfs`†  are used to generate a SquashFS snapshot of the git repository's contents (and none of the files in `.git`).

† A tool from the excellent [squashfs-tools-ng](https://github.com/AgentD/squashfs-tools-ng) package

### EROFS

Setting `$format` to `erofs` generates an [EROFS](https://erofs.docs.kernel.org/) image with `mkfs.erofs` instead, writing `$name.erofs` rather than `$name.sqfs`. EROFS is an interesting alternative for this workload:

* SquashFS compresses fixed-size *input* blocks (128 KiB by default), so reading a 300-byte ebuild requires decompressing the whole block containing it. EROFS compresses into fixed-size *output* clusters and caches the decompressed pages in the normal page cache, so per-file read amplification is much lower and there is no separate bounded cache to thrash.
* With `ztailpacking`, EROFS inlines the tails of small files into the inode block, which both saves space and avoids a second I/O for each tiny file — a good match for a repository where most files are under 4 KiB.
* `fragments` packs small files together so they can be compressed at all. This is not optional for this workload: a 1.3 KiB ebuild cannot fill a compression cluster on its own, so without it EROFS stores nearly everything uncompressed. The script therefore always passes `-Efragments,ztailpacking`.

`dedupe` is deliberately not enabled. On a ::gentoo snapshot it saved 0.1% while costing roughly 7x the build time — compressing fragments together already captures the duplication between identical ebuilds.

The tradeoffs are portability and image size. EROFS features and compression algorithms became available in the kernel at different times (LZ4 in 5.4, LZMA in 5.16, `ztailpacking` in 5.19, `fragments` in 6.1, DEFLATE in 6.6, zstd more recently still), so an image is only as portable as the oldest kernel that must mount it. SquashFS has supported zstd since 4.14 and is enabled essentially everywhere.

The `$pcluster` variable (default 65536) sets the EROFS physical cluster size. Larger values compress better but require the reader to support the big-pcluster feature.

## Usage
I find it most convenient to run the script via a cron job or systemd timer and to mount the SquashFS image on-demand with `autofs` or systemd's automount capability.

An example `/etc/fstab` entry for use with systemd's automount:

````
/root/gentoo.sqfs	/var/db/repos/gentoo	squashfs	noauto,x-systemd.automount,x-systemd.mount-timeout=30	0 0
````

Or, for an EROFS image:

````
/root/gentoo.erofs	/var/db/repos/gentoo	erofs	noauto,x-systemd.automount,x-systemd.mount-timeout=30	0 0
````
