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

### Measurements

A ::gentoo snapshot (129,703 files, 198 MiB, 32,704 `md5-cache` entries), zstd level 11 throughout. Cold cache, best of three runs.

| image                              | size      | `md5-cache` read | random 5000 | full read |
| ---------------------------------- | --------- | ---------------- | ----------- | --------- |
| SquashFS, 128 KiB block (default)  | 42.0 MiB  | 0.87s            | 1.15s       | 3.30s     |
| SquashFS, 1 MiB block              | 36.3 MiB  | 4.06s            | 4.29s       | 16.82s    |
| EROFS, 64 KiB pcluster             | 55.7 MiB  | 0.23s            | 0.27s       | 1.08s     |
| EROFS, 1 MiB pcluster              | 51.7 MiB  | 0.29s            | 0.24s       | 1.03s     |
| EROFS, 64 KiB, without `fragments` | 178.0 MiB |                  |             |           |

On raw reads EROFS is 4-15x faster, and unlike SquashFS it does not trade read latency away as the compression unit grows: its 1 MiB image reads as fast as its 64 KiB one, while SquashFS at 1 MiB is 4-5x slower than at 128 KiB. That is the difference between compressing fixed-size input blocks and fixed-size output clusters.

### What this is worth to Portage

Raw read numbers overstate the practical benefit, because dependency resolution is dominated by Portage's own CPU time rather than by file access. Measuring `emerge` against each image, with an all-warm run as the floor:

| image              | `emerge -pe @world` | over warm floor | `emerge -puDN @world` | over warm floor |
| ------------------ | ------------------- | --------------- | --------------------- | --------------- |
| warm cache (floor) | 15.20s              | —               | 17.36s                | —               |
| EROFS, 64 KiB      | 15.43s              | +0.23s          | 17.64s                | +0.28s          |
| EROFS, 1 MiB       | 15.47s              | +0.27s          | 18.05s                | +0.69s          |
| SquashFS, 128 KiB  | 16.74s              | +1.54s          | 19.25s                | +1.89s          |
| SquashFS, 1 MiB    | 19.37s              | +4.17s          | 22.79s                | +5.43s          |

EROFS essentially eliminates the filesystem from the cost of a cold `emerge` — a quarter of a second on top of the warm floor, against a second and a half for SquashFS. But since the floor is ~15s of Python either way, the end-to-end win is only about 8%, bought with a 32% larger image.

The clearer result is the SquashFS block size. Going to 1 MiB blocks saves 14% on size but costs 16-19% on every cold `emerge`, so the size-optimal SquashFS configuration is a bad trade for an image that gets read repeatedly. The 128 KiB default is the right choice.

(These timings had the images on tmpfs, so they isolate decompression and page-cache behavior rather than backing-store I/O.)

### Over NFS

Hosting the same images on an NFS export (tmpfs-backed server, so no server-side disk I/O) and loop-mounting them from the client. Over wired gigabit Ethernet:

| image                   | size     | `emerge -pe @world` | `md5-cache` read | random 5000 | full read |
| ----------------------- | -------- | ------------------- | ---------------- | ----------- | --------- |
| SquashFS, 128 KiB block | 42.0 MiB | 16.87s              | 0.95s            | 2.32s       | 3.61s     |
| SquashFS, 1 MiB block   | 36.3 MiB | 19.47s              | 4.17s            | 4.99s       | 17.41s    |
| EROFS, 64 KiB pcluster  | 55.7 MiB | 16.84s              | 0.46s            | 2.40s       | 1.95s     |
| EROFS, 1 MiB pcluster   | 51.7 MiB | **16.52s**          | 0.43s            | 2.03s       | 1.67s     |

EROFS keeps a real advantage on raw reads — roughly 2x on `md5-cache` and 1.8x on a full sweep — but `emerge` is a dead heat, because dependency resolution is CPU-bound and the filesystem is no longer the constraint. Serving over gigabit Ethernet costs a cold `emerge` almost nothing compared to a local image (16.87s against 16.74s for SquashFS).

The link matters more than the filesystem. The same measurements over WiFi:

| image                   | `emerge -pe @world` | `md5-cache` read | random 5000 | full read |
| ----------------------- | ------------------- | ---------------- | ----------- | --------- |
| SquashFS, 128 KiB block | **20.52s**          | 1.12s            | 6.22s       | 4.74s     |
| SquashFS, 1 MiB block   | 22.03s              | 4.30s            | 7.09s       | 18.26s    |
| EROFS, 64 KiB pcluster  | 22.55s              | 1.00s            | 9.00s       | 4.73s     |
| EROFS, 1 MiB pcluster   | 21.67s              | 0.86s            | 7.64s       | 3.59s     |

Here the ranking inverts and SquashFS wins: EROFS loses the random-read test outright, 9.00s against 6.22s. When per-request latency is high, EROFS's many small precise reads each cost a round-trip, while SquashFS's larger block fetches amortize that latency across neighboring files a tree walk is about to want anyway. This is the same property that makes SquashFS waste work on a local image, and it only pays off when latency is the dominant cost.

The one result that holds everywhere is SquashFS's block size: 1 MiB is worse than 128 KiB local, on wired NFS, and on WiFi.

So: **EROFS is the better choice, or tied, in every case except a high-latency link, where SquashFS wins.** But the whole spread is under 15% of a cold `emerge`, against a ~15s floor of Portage's own CPU time — the filesystem choice is a rounding error next to that, and next to the choice of block size.

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
