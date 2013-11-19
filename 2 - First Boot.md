First Boot
------
November 04, 2013

Where do you begin when you're creating a new operating system? 
First, you're going to need a functioning base system to bootstrap the new one from--that is, if you want to do it the same way that Linux From Scratch (LFS) does it.
But do we really need to bootstrap from an existing Linux distro? I think not.

If you've ever installed Arch or Gentoo Linux before, stop for a moment and think about exactly what was happening when you installed it.

When you install Arch, you're really just using a bunch of relatively simple shell scripts, found [here](https://github.com/falconindy/arch-install-scripts) (m4 preprocessed bash scripts, so they're slightly weird to read unless you build them, just `make all` with m4 installed). These scripts create the base folders and then use the package manager, Pacman, to install packages onto the empty filesystem.  

Sounds pretty simple, right? As it turns out, Pacman does 99% of the heavy lifting in the install process--nothing fancy or inordinately complex happens behind the scenes.

When bootstrapping a Linux distribution, you need an existing installation to work from--lest it's infinitely more difficult than it has to be. So we're going to hijack the Arch install process to take what we need, namely a nicely compiled Linux Kernel, a copy of Busybox, and a bootloader (syslinux), and then doing the rest by hand. Now this might sound like we're just going to install Arch Linux by hand and create a package-manager-less derivative of Arch. Trust me, it's not--it's just far easier to bootstrap from existing binary packages than to deal with the added complexity of compiling a kernel, utilities, and bootloader. Bootstrapping from binaries just makes getting off the ground much easier, but we're still going to come back and rebuild these in the future.


### Development Environment

So what does our development environment look like?
Lets start with Virtualbox, as everything is much simpler when we don't need to worry about hardware support and can easily create and destroy virtual hard drives (VHDs) at will.
First, if you don't already have it, you'll need [Arch Linux](https://wiki.archlinux.org/index.php/Beginners%27_Guide) installed on its own VHD.
Then, create another virtual hard drive, and attach it to the virtual machine containing your Arch Linux installation.

From here, we can really begin. Partition your new VHD so it has two partitions--it should appear as `/dev/sdb/` if it's the second virtual hard drive. 
Then format one as ext4, and the other as a swap partition (`mkfs.ext4 /dev/sdb1` and `mkswap /dev/sdb5` as root, respectively).
From here, mount the primary partition (`mount /dev/sdb1 /mnt` as root), and ensure you have permissions to read/write to it. 

Before we go any further, make sure you have installed `syslinux`, `wget`, and `base-devel` on your host system.

### Busybox

> [BusyBox][] combines tiny versions of many common UNIX utilities into a single small executable. It provides replacements for most of the utilities you usually find in GNU fileutils, shellutils, etc. The utilities in BusyBox generally have fewer options than their full-featured GNU cousins; however, the options that are included provide the expected functionality and behave very much like their GNU counterparts. BusyBox provides a fairly complete environment for any small or embedded system.

[busybox]: http://www.busybox.net/about.html

Basically, Busybox is a GPLv2 licensed implementation of a Unix userland that is both highly portable and written in C.

It works in a rather interesting way: the utilities contained in it can either be called as `busybox cat` or through a [symlink](http://en.wikipedia.org/wiki/Symbolic_link). That is, `busybox` symlinked to `cat` recognizes that it's being called as `cat` and runs the appropriate utility.
This works because *nix executables are not only sent arguments from the shell, but also the name that they were called as.
To take advantage of this, inspect `argv[0]` in C, `os.Args[0]` in Go, and `@ARGV[0]` in Perl, but can be done in most languages without trouble. =
More clearly, `argv[1]` would be the first argument passed to the executable, when each argument is separated by a space. So the executable called by `cat -v example.txt` would be passed `argv[0] = cat`, `argv[1] = -v` and `argv[2] = example.txt`. 

To hijack the Arch Linux Busybox package, we need to download the package without installing it.
`sudo pacman -Syp busybox | grep http:// | xargs wget` will download a statically compiled busybox, be sure to do this while in `/mnt`. 
Now, just extract the package (`tar -xf <filename>`) and it will automatically create the neccessary folders. 

Before you continue, make sure to create symlinks from `/mnt/usr/bin/busybox` to `/mnt/usr/bin/sh` and any other utilities you may want to use from it. 
For now, we're not going to configure Busybox to provide anything more than a shell, as we're using it to aid in the process of bootstrapping a system, not as the system default. 

### Kernel Building

Unfortunately, we can't borrow the default Arch Linux kernel. We have to build our own, or else we get a kernel panic upon boot.
![kernel panic](http://i.imgur.com/EAhgtFl.png)
When Linux boots, it attempts to mount the storage device that it's installed on, but because of how the Arch-provided Kernel is built, all filesystem support and SATA drivers lie in dynamically loaded kernel modules, which we can't quite load right now. 
We can verify this by looking at the auto-generated config for the Linux kernel (`linux-lts` in the picture, but it's the same for both), as shown below.

![Ext4fs](http://i.imgur.com/yzkLpBA.png)

`CONFIG_EXT4_FS` is set to `m`, which stands for module, instead of `y`, which would compile support right into the kernel.
Therefore, we need to build our own kernel, with `CONFIG_EXT4_FS` set to `y`, so we can boot. 
However, we can't simply use the Arch Linux provided kernel. 
Therefore, obtain a copy of the Linux Kernel source with `wget -c https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.11.6.tar.xz`
Copy this file into a build directory in `/mnt`, such as `/mnt/linux-build`.  
Extract it, and then `make mrproper` to prepare for compilation.  
Now we have to configure the kernel with `make menuconfig`.  

Only two things have to be changed from the default kernel configuration to get everything working properly.

![Kirk](http://i.imgur.com/TXc357h.png)

Now type `make` and wait for the kernel to compile.  
Then copy `arch/x86_64/boot/bzImage` to `/boot/linux` (a sensible location, for once) and you're good to go! However, make sure to copy the bzImage, and not the vmlinux file in the root of the build directory, as the vmlinux file must be made bootable before being used as an operating system kernel by adding a [multiboot](http://www.gnu.org/software/grub/manual/multiboot/multiboot.html) header, bootsector and setup routines, which is already taken care of in the bzImage. Trying to boot from the vmlinux file results in an apparently undocumented Syslinux I/O error. I found this out the hard way. For more reading on vmlinuz/vmlinuz/bzImage, [this](http://www.linfo.org/vmlinuz.html) is fairly helpful.

### Installing a Bootloader

So now we have a kernel, but before we even think about booting, we need to install a bootloader.
This can be quite challenging as the GNU bootloader, GRUB, needs to be installed from a [chrooted](http://en.wikipedia.org/wiki/Chroot), and we can't do that with out a huge amount of trouble, because the system lacks all of GRUB's dependencies. 

Syslinux, however, is fun and easy to install onto a mounted partition. 
This is reasonably simple  to do:

	# Create an empty directory
	mkdir /mnt/boot/syslinux
	# Copy over the syslinux files
	cp -r /usr/lib/syslinux/bios/* /mnt/boot/syslinux
	# Install syslinux
	extlinux --install /mnt/boot/syslinux
	# umount /dev/sdb so we can install the syslinux master boot record (MBR)
	umount -R /mnt
	# Install the syslinux MBR files, where sdXX is your VHD with the new OS on it.
	dd bs=440 count=1 conv=notrunc if=/usr/lib/syslinux/bios/mbr.bin of=/dev/sdXX

Now you need to create another file, `/mnt/boot/syslinux/syslinux.cfg`. Look [here](https://wiki.archlinux.org/index.php/syslinux#Basic_configuration) if you have trouble. Mine looks like this:

	PROMPT 1
	TIMEOUT 50
	DEFAULT kirk

	LABEL kirk
    	LINUX ../linux
    	APPEND root=/dev/sdb1 rw init=/usr/bin/sh

Now we can try to boot. Logout and reset the virtual machine, then when it's rebooting, hit F12 and choose HDD 2.
If this works, then clearly you did everything right. But don't expect this to be usable for anything, as all there are quite literally 3 programs on the entire system: a kernel, a bootloader, and a barebones userland. Keep in mind that we still lack almost everything we need for even a remotely functioning system, but we'll solve those problems when the time comes. 

![Obligatory Reference](http://i.imgur.com/BoYowLd.png)

## Conclusion

Hopefully you've learned something from this post.
If I wrote something that's blatantly wrong, or could be done in a better way, PLEASE let me know so I can fix it. Next time, we'll start down the long road to usefulness, instead of merely booting to a shell. 

Development is always on [GitHub](https://github.com/kirkproject) - feel free to chime in, create an issue, and make pull requests. 

As always, feel free to contact me on Twitter (@IanChiles) or by email (ianrchiles @ gmail.com) for any reason. 

Finally, special thanks to Casey Chow (@digitxpc) for helping to edit this post. 

## References/Helpful Links

- argv[0] references - http://stackoverflow.com/questions/2050961/is-argv0-name-of-executable-an-accepted-standard-or-just-a-common-conventi

- VFS not syncing errors - http://wiki.gentoo.org/wiki/Knowledge_Base:Unable_to_mount_root_fs

- Rather old, but helpful in confirming =m / =y in kernel configuration - http://www.tldp.org/HOWTO/SCSI-2.4-HOWTO/kconfig.html

- Booting without an initramfs - http://crunchbang.org/forums/viewtopic.php?id=27213

- vmlinux vs bzImage - http://en.wikipedia.org/wiki/Vmlinux#bzImage

### Things I've been looking at recently that you might see in the future

- http://git.suckless.org/
- http://www.nico.schottelius.org/software/cinit/
- http://sta.li/
- https://github.com/limetext/lime
- https://www.docker.io/
