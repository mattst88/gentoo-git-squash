# gentoo-git-squash
A script to create a SquashFS snapshot of a git repo

## What

`gentoo-git-squash` is a bash script to update an arbitrary git repo and make a SquashFS snapshot of its contents.

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

## Usage
I find it most convenient to run the script via a cron job or systemd timer and to mount the SquashFS image on-demand with `autofs` or systemd's automount capability.

An example `/etc/fstab` entry for use with systemd's automount:

````
/root/gentoo.sqfs	/var/db/repos/gentoo	squashfs	noauto,x-systemd.automount,x-systemd.mount-timeout=30	0 0
````
