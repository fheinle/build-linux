Build yourself a Linux
======================

% TODO explain Makefile

Abstract
--------

*This started out as a personal project to build a very small Linux based
operating system that has very few moving parts but is still very complete and
useful. Along the way of figuring out how to get the damn thing to boot and
making it do something useful I learned quite alot. Too much time has been
spent reading very old and hard to find documentation, or when there was none
the source code of how other people were doing it. So I thought, why not share
what I have learned.*

The Linux Kernel
----------------

The core component of our operating system, the kernel, that which manages the
processes and talks to the hardware on our behalf. You can retrieve a copy of
the source code easily from [kernel.org](https://www.kernel.org/). There are
multiple versions to choose from, choosing one is usually a tradeoff between
stability and wanting newer features. If you look at the
[Releases](https://www.kernel.org/category/releases.html) tab, you can see how
long each version will be supported and keeps receiving updates. So you can
usually just apply the update or use the updated version without changing
anything else or having something break.

So just pick a version and download the tar.xz file and extract it with ``tar
-xf linux-version.tar.xz``. To build the kernel we obviously need a compiler and
some build tools. Installing ``build-essential`` on Ubuntu (or ``base-devel`` on
Arch Linux) will almost give you everything you need. You'll also need to
install ``bc`` for some reason.

The next step is configuring you build, inside the untarred directory you do
``make defconfig``. This will generate a default config for your currect
architecture and place it in ``.config``. You can edit it directly with a text
editor but it's much better to do it with an interface by doing ``make nconfig``
(this needs ``libncurses5-dev`` on Ubuntu). Here you can enable/disable features
and device drivers with the spacebar. ``*`` means that it will be compiled in
your kernel image. ``M`` means it will be compiled inside a seprate kernel
module. Which is a part of the kernel that will be put in a seperate file and
can be loaded in dynamically in the kernel when they are required. The default
config will do just fine for basic stuff like running in a virtual machine. But
in our case, we don't really want to deal with kernel modules so we'll just do
this: ``sed "s/=m/=y/" -i .config``. Building the kernel is now just running
``make``. Don't forget to add ``-jN`` with `N` the number of cores of this might
take a while.

Other useful/interesting ways to configure the kernel are:

  * ``make localmodconfig`` will look at the modules that are currently
      loaded in the running kernel and change the config so that only those are
      enabled as module. Useful for when you only want to build the things you
      need without having to figure out what that is. % TODO LSMOD and caveat

  * ``make localyesconfig``,the same as above but everything gets compiled in
      the kernel instead as a kernel module.

  * ``make allmodconfig`` generates a new config where all options are enabled
      and as much as possible as module.

  * ``make allyesconfig``, same as above but with everything compiled in the
      kernel.

  * ``make randconfig`` generates a random config...

Busybox Userspace
-----------------

All these tools you know and love like ``ls``, ``echo``, ``cat`` ``mv`` and
``rm`` and so on. Which are commenly referred to as the 'coreutils'. It has that
and alot more, like utilities from ``util-linux`` so we can do stuff like
``mount`` and even a complete init system. Basicly most tools to expect to be
present on a Linux system only are these somewhat simplified.

You can get the source from [busybox.net](https://busybox.net/). They also
provide prebuilt binaries which will do just fine for most use-cases. But just
to be sure we will build our own version.

Configuring busybox is very similar to configuring the kernel. It also uses a
``.config`` file and you can do ``make defconfig`` to generate one. But we are
going to use the one I provided (which I stole from Arch Linux). You can find
the config in this git repo with the name ``bb-config``. Like the ``defconfig``
version, this has most utilities enabled but with a few differences like
statically linking all libraries.  Building busybox is again done by simply
doing ``make``, but before we do this, let's look into ``musl`` first.

The C Standard Library
----------------------

The C standard library is more important to the operating system than you might
think. It provides some useful functions and an interface to the kernel. But it
also handles DNS requests and provides a dynamic linker. We don't really have to
pay attention to any of this, we can just statically link the one we are using
right know which is probably 'glibc'. This means the following part is optional.
But I thought this would make it more interesting and it also makes us able to
build smaller binaries.

That's because we are going to use [musl](https://www.musl-libc.org/), which is
a lightweight libc implementation. You can get it by installing ``musl-tools``
on Ubuntu or simply ``musl`` on Arch Linux. Now we can link binaries to musl
instead of glibc by using ``musl-gcc`` instead of ``gcc``.

Before we can build busybox with musl, we need sanitized kernel headers for use
with musl. You get get that from [this github
repo](https://github.com/sabotage-linux/kernel-headers). And set
``CONFIG_EXTRA_CFLAGS`` in your busybox config to
``CONFIG_EXTRA_CFLAGS="-I/path/to/kernel-headers/x86_64/include"`` to use them.
Obviously change ``/path/to`` to the location where you put the headers repo,
can be relative from within the busybox source directory.

If you run ``make`` now, the busybox executable will be significantly smaller
because we are statically linking a much smaller libc.

Be advised that even though there is a libc standard, musl is not always a
drop-in replacement from glibc if the application you're compiling uses glibc
specific things.

Building the Disk Image
-----------------------

Installing a OS on a file instead of a real disk complicates things but this
makes development and testing it easier.

So let's start by allocating a new file of size 100M by doing ``fallocate -l100M
image``(some distro's don't have ``fallocate`` so you can do ``dd if=/dev/zero
of=image bs=1M count=100`` instead). And then we format it like we would format
a dis with ``fdisk image``. It automatically creates a MBR partition table for
us and we'll create just 1 partition filling the whole by pressing 'n' and
afterwards just use the default options for everything and keep spamming 'enter'
untill you're done. Finally press 'w' exit and to write the changes to the
image.
```bash
$ fdisk image

Welcome to fdisk (util-linux 2.29.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x319d111f.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-204799, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-204799, default 204799):

Created a new partition 1 of type 'Linux' and of size 99 MiB.

Command (m for help): w
The partition table has been altered.
Syncing disks.
```

In order to interact with our new partition we'll create a loop device for our
image. Loop devices are block devices (like actual disks) that in our case
point to a file instread of real hardware. For this we need root so sudo up
with ``sudo su`` or however you prefer to gain root privileges and afterwards
run:
```bash
$ losetup -P -f --show image
/dev/loop0
```
The loop device probably ends with a 0 but it could be different in your case.
The ``-P`` makes sure the partition also gets a loop device, ``/dev/loop0p1`` in
my case. Let's make a filesystem on it.
```bash
$ mkfs.ext4 /dev/loop0p1
```
If you want to use something else than ext4, be sure to enable it when
configuring your kernel. Now that we have done that, we can mount it start
putting everything in place.
```bash
$ mkdir image_root
$ mount /dev/loop0p1 image_root
$ cd image_root # it's important you do the following commands from this location
$ mkdir -p usr/{sbin,bin} bin sbin boot
```
And while we're at it, we can create the rest of the file system hierarchy. This
is actually standardized and applications often assume this is the way you're
doing it, but you can often do what you want. You can find more info in this
[here](http://www.pathname.com/fhs/).
```bash
$ mkdir -p {dev,etc,home,lib}
$ mkdir -p {mnt,opt,proc,srv,sys}
$ mkdir -p var/{lib,lock,log,run,spool}
$ install -d -m 0750 root
$ install -d -m 1777 tmp
$ mkdir -p usr/{include,lib,share,src}
```
We'll copy our binaries over.
```bash
$ cp /path/to/busybox usr/bin/busybox
$ cp /path/to/bzImage boot/bzImage
```

The Boot Loader
---------------

The next step is to install the bootloader, the program that loads our kernel in
memory and starts it. For this we use GRUB, one of the most widely used
bootloaders. It has a ton of features but we are going to keep it very simple.
Installing it is very simple, we just do this:
```bash
grub-install --modules=part_msdos --target=i386-pc --boot-directory="$PWD/boot" /dev/loop0
```
The ``--target=i386-pc`` tells grub to use the simple msdos MBR bootloader. This
is often the default but this can vary from machine to machine so you better
specify it here. The ``--boot-directory`` options tells grub to install the grub
files in /boot inside the image instead of the /boot of your current system.
``--modules=part_msdos`` is a workaround for a bug in Ubuntu's grub. When you
use ``losetup -P``, grub doesn't detect the root device correctly and doesn't
think it needs to support msdos partition tables and won't be able to find the
root partition.

Now we just have to configure grub and then our system should be able to boot.
This basicly means telling grub how to load the kernel. This config is located
at ``boot/grub/grub.cfg`` (some distro's use ``/boot/grub2``). This file needs
to be created first, but before we do that, we need to figure something out
first. If you look at ``/proc/cmdline`` on your own machine you might see
something like this:
```bash
$ cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-4.4.0-71-generic root=UUID=83066fa6-cf94-4de3-9803-ace841e5066c ro
```
These are the arguments passed to your kernel when it's booted. The 'root'
options tells our kernel which device holds the root filesystem that needs to be
mounted at '/'. The kernel needs to know this or it won't be able to boot. There
are different ways of identifying your the root filesystem. Using a UUID is a
good way because it is a unique identifier for the filesystem generated when you
do ``mkfs``. The issue with using this, is that the kernel doesn't really
support it because it depends on the implementation of the filesystem. This
works on your system and I'll explain later why. But we can't use it now. We
could do ``root=/dev/sda1``, this will probably work but it has some problems.
The 'a' in 'sda' is dependant on the order the bios will load the disk and this
can change when you add a new disk or sometimes the order can change randomly.
Or when you use a different type of interface/disk it can be something entirely
different. So we need something more robust. I suggest we use the PARTUUID. It's
a unique id for the partition (and not the filesystem, like UUID) and this is a
somewhat recent addition to the kernel for msdos partition tables (GPT had this
for a while since it is quite essential for it's functionality). We'll find the
id like this:
```bash
$ fdisk -l ../image | grep "Disk identifier"
Disk identifier: 0x4f4abda5
```

So the grub.cfg should look like this:
```
linux /boot/bzImage quiet init=/bin/sh root=PARTUUID=4f4abda5-01
boot
```
