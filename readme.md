# TL;DR

Git-managed kernels from kernel.org, with fully-automated configuration that's
better than `make oldconfig`:

```
./scripts/config --enable IKCONFIG
./scripts/config --module SENSORS_NCT6775
./scripts/config --disable AUDIT
./scripts/config --set-val SND_HDA_PREALLOC_SIZE 2048
./scripts/config --set-str UEVENT_HELPER_PATH ""
```

and various tips. No more blind copying of 4,000+ line `.config` files, no
unnecessary code, no bullshit.

# About

This repo explains how I download, configure, and install kernels on my Gentoo
Linux machines.

I claim that my methodology is better than 90% of the workflows described
online, including the official Gentoo documentation. Criticism welcome :)

Advantages of my approach (summary; see below for full explanation):

* Obtaining the kernel:
  * git > tarballs: faster updates and patching; built-in changelog viewer; git-bisect is possible
  * access to more releases, from ancient releases to latest release candidates
  * easier to get support upstream
  * simplicity & transparency
* Configuration:
  * Fully automated, comment- and git-friendly configuration. Better than
    kernel seeds or copying `.config` files around
  * Proper handling of changed default values when upgrading kernels
  * No extra code/complexity -- `scripts/config` is part of the kernel
* Building and installing the kernel
  * compilation is not done as root
  * `-march=native` is used

Disadvantages of my approach:

* ~1.5 GB extra space for `.git` (fixed cost if not using shallow clones, regardless of number of kernels)
* You don't get any patches from the Gentoo kernel team (e.g. aufs, grsecurity, small fixes/improvements)

# Target audience

I won't go over the basics of kernel configuration here -- that's already
described here:

* https://wiki.gentoo.org/wiki/Handbook:X86/Installation/Kernel
* https://wiki.gentoo.org/wiki/Kernel
* https://wiki.gentoo.org/wiki/Kernel/Configuration
* https://wiki.gentoo.org/wiki/Kernel/Upgrade

There's no need to duplicate documentation, so go and at least skim through those
pages if you haven't read them already.

# Guide

## Obtaining the kernel

How kernel development works: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/process/2.Process.rst .

TL;DR:

* `linux-next`: integration repo; might not even compile
* `mainline`: "official" kernel, managed by Linus himself
* `stable`: contains backported fixes

Distros such as Ubuntu and RHEL have their own forks of the kernel (e.g.
[here's Ubuntu's Xenial
repo](http://kernel.ubuntu.com/git/ubuntu/ubuntu-xenial.git/)). They also
backport fixes to certain "stable" and supported versions. It's possible to use
those with Gentoo, but I fail to see the appeal of doing that.

On Gentoo you have several "official" options, with various amounts of extra
patches and support available:

* `sys-kernel/gentoo-sources` -- default, slightly patched sources
* `sys-kernel/vanilla-sources` -- upstream kernel with no modifications
* `sys-kernel/hardened-sources` -- [rest in peace](https://www.gentoo.org/support/news-items/2017-08-19-hardened-sources-removal.html)
* several others -- browse the [sys-kernel category](https://packages.gentoo.org/categories/sys-kernel/)

If you use `gentoo-sources` you get:

* A bunch of required settings [automatically
enabled](https://dev.gentoo.org/~mpagano/genpatches/trunk/4.12/4567_distro-Gentoo-Kconfig.patch)
for you if you enable `CONFIG_GENTOO_LINUX`.
* Various patches. For example, you can find all the 4.12 patches here:
https://dev.gentoo.org/~mpagano/genpatches/trunk/4.12/ . For a concrete
example: without `1500_XATTR_USER_PREFIX.patch` portage will spit out a lot of
`Failed to set XATTR_PAX markings` errors if your `/var/tmp/portage` is on a
`tmpfs` mount.

The Gentoo kernel team is really good ([Greg
Kroah-Hartman](https://en.wikipedia.org/wiki/Greg_Kroah-Hartman) is a member!).
They definitely add value and are more knowledgable than I am.  However, I'd
rather get my kernel from [kernel.org](https://www.kernel.org/) directly with
`git` (yes, even though gregkh maintains `linux-stable`).

Why?

* I can `git pull` and get the latest kernel without having to wait for the
Gentoo kernel team to release an ebuild.
* Obtaining support is easier. kernel.org
[states](https://www.kernel.org/category/releases.html): "[If using a distro
kernel,] Please use the support channels offered by your distribution vendor to
obtain kernel support".
* Applying patches is trivial regardless of which approach you take, but I find
it easier to track and cherry-pick patches with git.
* It's trivial to switch to the mainline/linux-next kernels if needed, or any
other tree, without changing my workflow.
* I get access to more versions. `linux-stable` + `mainline` have 2,500+ tags,
while `vanilla-sources` currently only has 7 ebuilds.
* `git-pull` is faster than `emerge`. Portage has to parse ebuilds,
`/var/db/pkg`, etc. It also has to download a 90+ MB tarball for every new
major release. I also don't have to disable `FEATURES=buildpkg` through
`package.env` -- `git-pull` is simply less annoying.

Which kernel version to use?

Open https://www.kernel.org/ . Pick a longterm release if you want to stick
with a certain major version (e.g. `4.9`) for a long time, or instead use the
latest stable version.

Finally, the actual guide. I like having 1 "master" bare repo per remote tree:

```bash
george@george:/usr/src$ sudo mkdir -p linux-stable-git-bare && sudo chown george:george linux-stable-git-bare
george@george:/usr/src$ git clone --bare 'https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git' linux-stable-git-bare
```

For every version I'm interested in (e.g. `4.12.8`):

```
george@george:/usr/src$ git -C linux-stable-git-bare fetch --all --tags
george@george:/usr/src$ git -C linux-stable-git-bare ls-remote --refs --tags | less

george@george:/usr/src$ v="4.12.8"
george@george:/usr/src$ sudo mkdir "linux-stable-git-$v" && sudo chown george:george "linux-stable-git-$v"
george@george:/usr/src$ git clone --single-branch --branch "v$v" linux-stable-git-bare/ "linux-stable-git-$v"
```

The `.git` folders will use hardlinks (until someone/something runs `git-gc`),
so it should be pretty space efficient.

Once you no longer need the Gentoo kernels:

```bash
george@george:~$ sudo mkdir -p /etc/portage/profile/
george@george:~$ echo 'sys-kernel/vanilla-sources-4.12.8' | sudo tee --append /etc/portage/profile/package.provided
george@george:~$ sudo emerge -aC gentoo-sources vanilla-sources # ...
```

This is possibly a bad idea, because now `depclean` could uninstall some required packages:

    george@george:~$ equery depgraph '=gentoo-sources-4.12.8'
    * dependency graph for sys-kernel/gentoo-sources-4.12.8
    `--  sys-kernel/gentoo-sources-4.12.8  [~amd64 keyword] 
      `--  sys-apps/sed-4.2.2  (sys-apps/sed) amd64 
      `--  sys-devel/binutils-2.28-r2  (>=sys-devel/binutils-2.11.90.0.31) amd64 
      `--  sys-libs/ncurses-6.0-r1  (>=sys-libs/ncurses-5.2) amd64 
      `--  sys-devel/make-4.2.1  (sys-devel/make) amd64 
      `--  dev-lang/perl-5.24.1-r2  (dev-lang/perl) amd64 
      `--  sys-devel/bc-1.06.95-r1  (sys-devel/bc) amd64 

You should probably add those to `@world`.

## Configuring the kernel

Downloading, compiling, and installing the kernel is trivial: you (roughly) do
`git pull && make all install`. Kernel configuration is the most challenging
part of running your own kernel.  Depending on how far you want to deviate from
the defaults, it can take you up to several hours to manually configure a
kernel.

We can split configuration into hardware and software.

### Hardware support and options

You will likely end up with a non-fully working computer if you just use `make
defconfig`.

It's also easy to end up with partial hardware support -- perhaps
there's a hardware sensor that's available but whose support hasn't been
enabled. How do you know you have enabled everything necessary?

Options for configuring hardware support:

* Easy:
  * genkernel
  * `make allmodconfig`
  * Copy `.config` from Ubuntu/Fedora/etc. Example: http://kernel.ubuntu.com/~kernel-ppa/configs/xenial/linux/ .
  * Boot a Linux live CD with the kernel version you care about. Then use `make localmodconfig` and save the config.
  * Kernel seeds. E.g. http://www.elilabs.com/~pappy/ (untested; not recommended)
  * [AutoKernConf](https://cateee.net/autokernconf/) (untested; not recommended)
* Time-consuming:
  * Go over `lspci` / `lsusb` and use the device/vendor numbers to find the relevant configs
  * Go over all options in `make menuconfig` and enable whatever you think is relevant

genkernel, `allmodconfig`, and `localmodconfig` are useful and I recommend them
if you don't want to bother figuring out what hardware support you need
enabled.

On the other hand, I think kernel seeds are an anti-pattern. Their proponents
claim that "Seeds are a sane 'make defconfig' for the real world". I'd rather
trust upstream with my defaults and expend energy on changing said defaults
upstream if they really are suboptimal for most users of $ARCH. I think the
defaults are fine and there's no need for kernel seeds.

The few advantages of a "from-scratch" config:

* 5-10x faster compilation (depending on selections). E.g. `make -j4` on a
  quad-core Intel i7 7700K CPU @ 4.5 GHz, Linux 4.12, gcc 5.4.0, with
  everything on a ramdisk:
  * [random Ubuntu config](http://kernel.ubuntu.com/~kernel-ppa/configs/xenial/linux/4.4.0-93.116/amd64-config.flavour.generic): 3950s user, 312s system, 1166s wall
  * `defconfig` + minor changes: 624s user, 47s system, 190s wall
  * `defconfig`: 472s user, 38s system, 145s wall
  * `allnoconfig`: 55s user, 6s system, 20s wall
* You save up to 200 MB of disk space (no unnecessary modules)
* You save a few milliseconds/seconds when booting up

I'd argue that most folks would be better off sticking with gentoo-sources +
[genkernel](https://wiki.gentoo.org/wiki/Genkernel) or any of the popular
distro kernels.

### Software support and options

Software like systemd requires certain kernel features to work properly.

As a Gentoo user you've probably seen messages like this one:

    ERROR: setup                                                                                                                                                                                 │
      CONFIG_AUDITSYSCALL:   is not set when it should be.

Again, `make defconfig` won't cut it.

The traditional approach is to generate a `.config` and then keep doing `make
oldconfig` or `make olddefconfig`. `.config` files have thousands of lines.
Options that were explicitly enabled/disabled are mixed together with default
settings.

Problem 1: If defaults change (like in this [famous Linus
rant](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b4b8cbf679c4866a523a35d1454884a31bd5d8dc))
then the new defaults won't come into effect -- you'll be stuck in the past.

Don't believe me?

```bash
george@george:~/linux$ git checkout 'b4b8cbf679c4866a523a35d1454884a31bd5d8dc^' # note caret
george@george:~/linux$ make mrproper defconfig
george@george:~/linux$ grep CNN55XX .config
CONFIG_CRYPTO_DEV_NITROX_CNN55XX=m
george@george:~/linux$ git checkout 'b4b8cbf679c4866a523a35d1454884a31bd5d8dc'
george@george:~/linux$ make oldconfig
george@george:~/linux$ grep CNN55XX .config
CONFIG_CRYPTO_DEV_NITROX_CNN55XX=m
```

Problem 2: Adding comments like `CONFIG_TUN is required by openvpn` is hard and
as a result almost nobody documents their `.config` files.

Some strong opinions:

* Configuration should be fully automated. Life is short, try to automate what you can.
* Configuration should be version controlled.
* Configuration should be documented (*why* is option X enabled?).

Here's what I do:

1. `emerge` the packages you need (e.g. systemd, docker, openvpn)
2. Gather all the error/warning messages
3. Use the kernel's `scripts/config` script to enable all the settings I need.
   Because I handle hardware configuration myself, my script looks like this:

```bash
# per-machine hardware config
"$(dirname "$0")/hardware/$(hostname).sh"

# /proc/config.gz
./scripts/config --enable IKCONFIG
./scripts/config --enable IKCONFIG_PROC

# required for CHECKPOINT_RESTORE
./scripts/config --enable EXPERT

# gentoo-sources ( https://gitweb.gentoo.org/proj/linux-patches.git/tree/4567_distro-Gentoo-Kconfig.patch ):
./scripts/config --enable DEVTMPFS
./scripts/config --enable TMPFS
./scripts/config --enable UNIX
./scripts/config --enable MMU
./scripts/config --enable SHMEM
# gentoo/portage:
./scripts/config --enable CGROUPS
./scripts/config --enable NAMESPACES
./scripts/config --enable IPC_NS
./scripts/config --enable NET_NS
./scripts/config --enable SYSVIPC
# openrc/runit support
./scripts/config --enable BINFMT_SCRIPT
# gentoo/systemd:
./scripts/config --enable AUTOFS4_FS
./scripts/config --enable BLK_DEV_BSG
./scripts/config --enable CGROUPS
./scripts/config --enable CHECKPOINT_RESTORE
./scripts/config --enable CRYPTO_HMAC
./scripts/config --enable CRYPTO_SHA256
./scripts/config --enable CRYPTO_USER_API_HASH
./scripts/config --enable DEVPTS_MULTIPLE_INSTANCES # removed -- https://patchwork.kernel.org/patch/9186935/
./scripts/config --enable DMIID
./scripts/config --enable EPOLL
./scripts/config --enable FANOTIFY
./scripts/config --enable FHANDLE
./scripts/config --enable INOTIFY_USER
./scripts/config --enable IPV6
./scripts/config --enable NET
./scripts/config --enable NET_NS
./scripts/config --enable PROC_FS
./scripts/config --enable SECCOMP
./scripts/config --enable SECCOMP_FILTER
./scripts/config --enable SIGNALFD
./scripts/config --enable SYSFS
./scripts/config --enable TIMERFD
./scripts/config --enable TMPFS_POSIX_ACL
./scripts/config --enable TMPFS_XATTR
./scripts/config --enable ANON_INODES
./scripts/config --enable BLOCK
./scripts/config --enable EVENTFD
./scripts/config --enable FSNOTIFY
./scripts/config --enable INET
./scripts/config --enable NLATTR

# Recommended by the Gentoo Handbook: "Also select Maintain a devtmpfs file system to mount at /dev so that critical device files are already available early in the boot process (CONFIG_DEVTMPFS and CONFIG_DEVTMPFS_MOUNT)":
./scripts/config --enable CONFIG_DEVTMPFS
./scripts/config --enable CONFIG_DEVTMPFS_MOUNT

# chromium
./scripts/config --enable PID_NS
./scripts/config --enable NET_NS
./scripts/config --enable SECCOMP_FILTER
./scripts/config --enable USER_NS
./scripts/config --enable ADVISE_SYSCALLS

# Docker
./scripts/config --enable CONFIG_OVERLAY_FS
./scripts/config --enable CONFIG_CGROUPS
./scripts/config --enable CONFIG_NAMESPACES
./scripts/config --enable CONFIG_IPC_NS
./scripts/config --enable CONFIG_NET_NS
./scripts/config --enable CONFIG_SYSVIPC
./scripts/config --enable CONFIG_CGROUP_DEVICE
./scripts/config --enable CONFIG_MEMCG
./scripts/config --enable CONFIG_VETH
./scripts/config --enable CONFIG_BRIDGE
./scripts/config --enable CONFIG_NETFILTER_ADVANCED # implicit requirement for BRIDGE_NETFILTER
./scripts/config --enable CONFIG_BRIDGE_NETFILTER
./scripts/config --enable CONFIG_USER_NS
./scripts/config --enable CONFIG_CGROUP_PIDS
./scripts/config --enable CONFIG_MEMCG_SWAP
./scripts/config --enable CONFIG_MEMCG_SWAP_ENABLED
./scripts/config --enable CONFIG_BLK_CGROUP
./scripts/config --enable CONFIG_BLK_DEV_THROTTLING
./scripts/config --enable CONFIG_CFQ_GROUP_IOSCHED
./scripts/config --enable CONFIG_CGROUP_PERF
./scripts/config --enable CONFIG_CGROUP_HUGETLB
./scripts/config --enable CONFIG_NET_CLS_CGROUP
./scripts/config --enable CONFIG_RT_GROUP_SCHED
./scripts/config --enable CONFIG_IP_VS
./scripts/config --enable CONFIG_IP_VS_PROTO_TCP
./scripts/config --enable CONFIG_IP_VS_PROTO_UDP
./scripts/config --enable CONFIG_IP_VS_NFCT
./scripts/config --enable CONFIG_IP_VS_RR
./scripts/config --enable CONFIG_VXLAN
./scripts/config --enable CONFIG_NET_L3_MASTER_DEV # required for IPVLAN
./scripts/config --enable CONFIG_IPVLAN
./scripts/config --enable CONFIG_MACVLAN
./scripts/config --enable CONFIG_DUMMY
./scripts/config --enable CONFIG_CGROUP_NET_PRIO
./scripts/config --enable CONFIG_DM_THIN_PROVISIONING

# qemu for kernel dev
./scripts/config --enable VIRTIO_PCI
./scripts/config --enable VIRTIO_BLK
./scripts/config --enable VIRTIO_NET
./scripts/config --enable VIRTIO
./scripts/config --enable 9P
./scripts/config --enable NET_9P
./scripts/config --enable NET_9P_VIRTIO

# cryptsetup, luks (according to gentoo wiki page)
./scripts/config --enable CONFIG_BLK_DEV_DM
./scripts/config --enable CONFIG_DM_CRYPT
./scripts/config --enable CONFIG_CRYPTO_AES_X86_64
./scripts/config --enable CONFIG_CRYPTO_XTS
./scripts/config --enable CONFIG_CRYPTO_SHA256
./scripts/config --enable CONFIG_CRYPTO_USER_API_SKCIPHER

# openvpn
./scripts/config --enable CONFIG_TUN

# cups
./scripts/config --enable CONFIG_USB_PRINTER

# pulseaudio
./scripts/config --set-val CONFIG_SND_HDA_PREALLOC_SIZE 2048

# libvirt
./scripts/config --enable MACVTAP

# systemd -- https://cgit.freedesktop.org/systemd/systemd/tree/README
./scripts/config --enable CONFIG_DEVTMPFS
./scripts/config --enable CONFIG_CGROUPS
./scripts/config --enable CONFIG_INOTIFY_USER
./scripts/config --enable CONFIG_SIGNALFD
./scripts/config --enable CONFIG_TIMERFD
./scripts/config --enable CONFIG_EPOLL
./scripts/config --enable CONFIG_NET
./scripts/config --enable CONFIG_SYSFS
./scripts/config --enable CONFIG_PROC_FS
./scripts/config --enable CONFIG_FHANDLE
./scripts/config --disable CONFIG_SYSFS_DEPRECATED
./scripts/config --set-str CONFIG_UEVENT_HELPER_PATH ""
./scripts/config --disable CONFIG_FW_LOADER_USER_HELPER
./scripts/config --enable CONFIG_DMIID
./scripts/config --enable CONFIG_BLK_DEV_BSG
./scripts/config --enable CONFIG_NET_NS
# ./scripts/config --enable CONFIG_DEVPTS_MULTIPLE_INSTANCES # no longer exists!
./scripts/config --enable CONFIG_AUTOFS4_FS
./scripts/config --enable CONFIG_TMPFS_XATTR
./scripts/config --enable CONFIG_TMPFS_POSIX_ACL
./scripts/config --enable CONFIG_EXT4_FS_POSIX_ACL # CONFIG_EXT4_POSIX_ACL
./scripts/config --enable CONFIG_BTRFS_FS_POSIX_ACL
./scripts/config --enable CONFIG_SECCOMP
./scripts/config --enable CONFIG_CHECKPOINT_RESTORE
./scripts/config --enable CONFIG_CGROUP_SCHED
./scripts/config --enable CONFIG_FAIR_GROUP_SCHED
./scripts/config --enable CONFIG_CFS_BANDWIDTH
./scripts/config --enable CONFIG_SCHEDSTATS
./scripts/config --enable CONFIG_SCHED_DEBUG
./scripts/config --enable CONFIG_EFIVAR_FS
./scripts/config --enable CONFIG_EFI_PARTITION
# ./scripts/config --disable CONFIG_RT_GROUP_SCHED # docker wants this
./scripts/config --disable CONFIG_AUDIT
./scripts/config --enable CONFIG_IPV6

# sys-auth/consolekit-1.1.2
./scripts/config --enable AUDITSYSCALL

# and so on
```

Applying my config:

```
george@george:/usr/src/linux-stable-git-4.12.8$ make mrproper defconfig
george@george:/usr/src/linux-stable-git-4.12.8$ ~/kernel-config.sh
george@george:/usr/src/linux-stable-git-4.12.8$ make olddefconfig
```

Pitfall 1: Some configs are implicit -- they are enabled by other configs:

    Symbol: TAP [=n]                                                                                                                                                                       │   
    Type  : tristate                                                                                                                                                                       │   
      Defined at drivers/net/Kconfig:304                                                                                                                                                   │   
      Depends on: NETDEVICES [=y] && NET_CORE [=y]                                                                                                                                         │   
      Selected by: MACVTAP [=n] && NETDEVICES [=y] && NET_CORE [=y] && MACVLAN [=n] && INET [=y] || IPVTAP [=n] && NETDEVICES [=y] && NET_CORE [=y] && IPVLAN [=n] && INET [=y]

You have to satisfy the "Selected by" expression if you want `CONFIG_TAP` to be
enabled. `./scripts/config --enable TAP` will appear to work, but `make
olddefconfig` will nuke it.

Pitfall 2: Some configs depend on others:

    Symbol: BRIDGE_NETFILTER [=n]
    Type  : tristate
    Prompt: Bridged IP/ARP packets filtering
      Location:
        -> Networking support (NET [=y])
          -> Networking options
            -> Network packet filtering framework (Netfilter) (NETFILTER [=y])
    (1)       -> Advanced netfilter configuration (NETFILTER_ADVANCED [=n])
      Defined at net/Kconfig:187
      Depends on: NET [=y] && BRIDGE [=n] && NETFILTER [=y] && INET [=y] && NETFILTER_ADVANCED [=n]

Similarly if e.g. `NET=n`, `./scripts/config --enable BRIDGE_NETFILTER` will
not enable `NET` and you'll end up with `NET=n` and `BRIDGE_NETFILTER=n`.

Workaround for detecting broken dependencies:

    george@george:/usr/src/linux-stable-git-4.12.8$ grep -Po '(?<=--enable )[^# ]+' ~/kernel-config.sh | sed 's/^CONFIG_//' | while read kconf; do if ! grep -q "^CONFIG_$kconf=y" .config; then echo "$kconf not set"; fi; done

TODO: write an script that checks everything.

## Building and installing the kernel

`make install` requires `sys-apps/debianutils`.

```bash
george@george:/usr/src$ eselect kernel list
george@george:/usr/src$ sudo eselect kernel set "linux-stable-git-4.12.8"
george@george:/usr/src$ cd linux

george@george:/usr/src/linux$ /usr/bin/time -v make KCFLAGS="-march=native" -j "$(nproc)" olddefconfig all

root@george:/usr/src/linux# (mountpoint -q /boot || mount /boot) && make install modules_install && grub-mkconfig -o /boot/grub/grub.cfg
root@george:/usr/src/linux# emerge -avtq '@module-rebuild'
```

# Plans for the future

* A better `scripts/config` application, with proper dependency management

# Criticism? Praise?

Any feedback would be appreciated.
