The Long Road to Usability
------

We left off (far, far too long ago) with a system that consisted of nothing more than the Linux Kernel + Busybox. Today, we'll improve greatly upon what we had by installing a proper `init` and getting `/dev` populated with devices. 

In the event you missed out on the earlier posts, the introduction is [here](http://ianchiles.com/kirk-introduction) and the first real post can be found [here](http://ianchiles.com/kirk-booting-up).

### Changes to development environment

In all likelihood, it's much easier to use QEMU within the host Linux to work on the disk image, instead of rebooting the VM every time and switching hard drives. However, I'm at a bit of a loss as to how to get this to work properly, if anyone wants to contribute here, contact me and explain how, and I'll add it in.

Most programs used have seen small updates, but that should not affect anything for our purposes. 

In addition, I've overhauled the filesystem layout a bit, so all binaries now reside in `/bin`, without exception. This makes everything a great deal simpler to work with. 

### Init

>  init (short for initialization) is the first process started during booting of the computer system. Init is a daemon process that continues running until the system is shut down. It is the direct or indirect ancestor of all other processes and automatically adopts all orphaned processes. Init is started by the kernel using a hard-coded filename, and if the kernel is unable to start it, a kernel panic will result. Init is typically assigned process identifier 1 (PID 1) (wikipedia).

After looking over practically every unix init system, I finally found one that is suitable for Kirk, for now, at least. Originally created by the [suckless.org](http://suckless.org) team, a simple 92 lines of C provides all of the functionality we require (for now, at least). 

Now, I've forked this project and made some tiny modifications to it. You can find it [here](https://github.com/kirkproject/init). Clone this git repo (`git clone https://github.com/kirkproject/init`) and then compile with `gcc --static -O2 init.c -o init`, and copy the `init` binary on over to `/bin`. 

Now we simply edit our `/boot/syslinux/syslinux.cfg` so that the append line is as such `APPEND root=/dev/sda1 rw`, removing the init parameter, as Linux automatically tries to call `init` if there is no parameter (your specific /dev/sd* may vary).

> BSD init runs the initialization shell script located in /etc/rc, then launches getty on text-based terminals or a windowing system such as X on graphical terminals under the control of /etc/ttys. There are no runlevels; the /etc/rc file determines what programs are run by init. The advantage of this system is that it is simple and easy to edit manually. However, new software added to the system may require changes to existing files that risk producing an unbootable system. To mitigate this, BSD variants have long supported a site-specific /etc/rc.local file that is run in a sub-shell near the end of the boot sequence.

Our init is BSD-style, which is far simpler than the traditional SysV Runlevel style of init used by most Linux distributions. 

Using our BSD-style init, `init` runs a program located at `/bin/startup` which brings up the system, and runs `/bin/shutdown` for shutting down the system. Examples of these two files are located in the same repository as our `init`. 

One of the most fascinating things about the way `init` works is that `/bin/startup` and `/bin/shutdown` are simply executable files. Typically, these are shell scripts marked with an executable flag (`chmod +x`) and a [shebang](http://en.wikipedia.org/wiki/Shebang_%28Unix%29) on the first line. This means we could write our initscripts in any language we wanted to - such as [python](http://lionfacelemonface.wordpress.com/2009/12/29/python-init-scripts/), perl, or even something like C. As long as it's executable, it'll work. 

For now, we're just going to have to live with the shortcomings of using initscripts written in shell. 
The most notable of these shortcomings is that everything is executed sequentially, potentially slowing down our boot process.
We'll come back and change this later, but for now, simple shell scripts won't harm anyone, and the system still manages to boot insanely quickly (1.6s inside a VM running off a 5400 RPM HDD). 

This is great and all, but the question remains, what exactly do we need to put in these startup and shutdown scripts?

This is a tricky question to answer, as we can do anything we want to from within `/bin/{startup,shutdown}`. A better question would be what do we *want* to start and shutdown. 
Seeing as there aren't very many programs installed at the moment, we need to not only discover what else we need, but also build those before we need to write init scripts. 

### Getty

Before now, we've simply boot straight into a shell. As you may have noticed, control commands such as Crtl-C, Crtl-Z don't work as you'd expect them to. 
This is because the shell doesn't actually handle these control sequences. 
Rather, this is handled by a program called `getty`, which stands for get teletype. 
Looking at my computer, I noticed I don't have a serial teletype, and neither do I know of [anyone who does](http://commons.wikimedia.org/wiki/File:Teletype_DMD_5620.jpg). 
However, the two most commonly used `getty` programs, `agetty` and `mgetty` both still provide full support for actual, serial teletypes. 

Recognizing the sheer ridiculous of this, I set out to find an implementation of `getty` that still does everything it needs to, while being as lightweight and easy to understand as possible. 
Once again, the suckless.org team has a small `getty` seems to fit these requirements perfectly, it's under reasonably active development as part of a larger `ubase` (linux utilities) project, so I've pulled it out of that project and put its source code up [here](https://github.com/kirkproject/getty). 
To build this, you'll need to clone `ubase` (`git clone http://git.suckless.org/ubase`) and from there you'll need to edit the `config.mk` to include `-static` in the `LDFLAGS` so it builds statically linked. 
Then simply copy the `getty` and `respawn` binary to `/mnt/bin`.

### Device Manager

The last thing we need to do before we can write our initscripts is to find a device manager. 
A device manager is the program responsible for managing `/dev`, keeping it populated and updated with devices. 
However, the main device manager for linux, `udev` was merged into the systemd source tree nearly 2 years ago. 
Seeing as we aren't and won't be using systemd for anything, it would be a waste to include it now. 
Thankfully, there exists a tiny alternative, `smdev` that provides all the functionality we'll ever need. 

This can be found at git.2f30.org/smdev, so `git clone http://git.2f30.org/smdev`, edit the `config.mk` to have `-static` in the `LDFLAGS` and then run `make` and copy the binary over to `/mnt/bin`. 

`smdev` depends on having the standard unix `passwd` and `group` files, located at `/etc/{passwd,group}`. 
This is a problem for us, as we have neither users, nor groups. 
As a temporary measure, we can create these files and hope they work (which they will)

In `/etc/group` copy the following -

	root:x:0:root
	bin:x:1:root,bin,daemon
	daemon:x:2:root,bin,daemon
	sys:x:3:root,bin
	adm:x:4:root,daemon
	tty:x:5:
	disk:x:6:root
	cdrom:x:7:root
	video:x:8:root
	audio:x:9:root
	nogroup:x:65534:

In `/etc/passwd` copy the following - 

	root::0:0:root:/root:/bin/sh
	bin:x:1:1:bin:/bin:/bin/false
	daemon:x:2:2:daemon:/sbin:/bin/false
	nobody:x:65534:65534:nobody:/nonexistent:/bin/false

Both of these files are copied from [here](http://git.2f30.org/fs/). 


### Writing Initscripts

Now all that's left to do before we boot again is to write our initscripts. 

We need to start with a shebang line

	#!/bin/sh

PATH tells Linux where to look within the filesystem to find executables. 

	export PATH=/bin

Then we have to mount our special filesystems properly, `/tmp`, `/proc`, and `/sys`

	# make the directory if it doesn't already exist
	mkdir -p /proc
	mkdir -p /sys
	mkdir -p /dev
	
	# special mount
	mount -n -t proc -o nosuid,noexec,nodev proc /proc
	mount -n -t sysfs -o nosuid,noexec,nodev sysfs /sys
	# mounting /dev as a special tmpfs
	mount -n -t tmpfs -o nosuid,mode=0755 dev /dev
	
	# mounting psuedoterminals
	mkdir -p /dev/pts
	mount -n -t devpts -o gid=5,mode=0620 devpts /dev/pts

To read about /dev/pts, [this](http://unix.stackexchange.com/questions/93531/what-is-stored-in-dev-pts-files-and-can-we-open-those) gives an excellent explanation.

Now, we need to mount our real filesystems. However, `mount` depends on having a file in `/etc/fstab` that details what filesystems to mount where. 
Mine looks like this.

	#
	# /etc/fstab: static file system information
	#
	# <file system> <dir>   <type>  <options>       <dump>  <pass>
	/dev/sdb1       /       ext4    defaults        0       1
	tmp             /tmp    tmpfs   nosuid,noexec   0       0

Next we need to remount the filesystems originally mounted at boot. 
	
	mount -o remount,ro /

Then we can launch our device manager and set it as the [kernel hotplug](https://www.kernel.org/doc/local/hotplug-history.html)

	smdev -s

	echo "Setting smdev as the kernel hotplug"
	echo /bin/smdev > /proc/sys/kernel/hotplug

Now we have another problem! `getty` expects to start `/bin/login` and we don't seem to have one.
This problem is easily remedied by simply telling our `getty` to start `sh` instead of `login`. 

	sh -c 'getty /dev/tty1 linux sh' &>/dev/null &
	
Now that our initscript is done, save it to /etc/startup, `chmod +x startup` and reboot to your new OS and start playing with it. 

If things don't go as expected, get in touch and we'll work out the problem. 


### Conclusion

One of the primary goals of Kirk was to avoid using GNU utilities, as well as GPL licensed code. 
So far, we've had great success with that, as besides Linux, we've only used the GPLv2 licensed Busybox, which we'll look at removing next time. As a small note, statically linking non LGPL licensed programs to the GNU C Library is prohibited by its license, so I would recommend linking against the [musl c library](http://www.musl-libc.org/) as I've done throughout these posts (and didn't mention).

A lot of the utilities I'm using here come from the excellent suckless.org group. Check out all of their projects over on their website. 

From here out, I plan on trying to get at least one post done per week. Next time, we should clean up a lot of things and be well on the path to being self-hosting, and once Linux 3.16 rolls around, LLVM/Clang should be able to be used to build everything. 

Development is always on GitHub - feel free to chime in, create an issue, and send pull requests. 

Contact me with suggestions, issues, ideas, or just to chat, on freenode in #kirkproject, @IanChiles on twitter, and via email, ianrchiles (at) gmail.com. 

### References

http://www.tldp.org/LDP/sag/html/login-via-terminal.html

https://github.com/hut/minirc/blob/master/rc

http://en.wikipedia.org/wiki/Getty_%28Unix%29ly

https://bbs.archlinux.org/viewtopic.php?id=162606

### Things to look at

- http://landley.net/code/aboriginal/about.html
- https://github.com/Pardus-Linux/mudur
- https://github.com/technomancy/atreus
- https://github.com/akozumpl/hawkey
- https://github.com/hylang/hy
