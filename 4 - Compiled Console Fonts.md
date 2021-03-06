Compiled Console Fonts
------
June 04, 2014

The default Linux console font is not particularly pretty, and there's not a particularly good way to compile a different one in by default. Sure the `setfont` utility exists, but that has problems persisting across reboots, and doesn't address the original problem - the default compiled in font is ugly. 
![default font](http://i.imgur.com/BoYowLd.png)

A particularly good 8x16 monospace font (shown above) is [Terminus](http://terminus-font.sourceforge.net/). Terminus also happens to be OFL/GPLv2 dual licensed (as far as I can tell), meaning we can embed it into the Linux kernel without any licensing issues. But how are we going to compile that into the kernel to replace the default font?
![terminus](http://terminus-font.sourceforge.net/img/8x16n.gif)

The original Linux console fonts were generated using a utility from AmigaOS named `cpi2fnt` which generates a C source code from a `.psf` font. Much to my dismay, this program is impossible to obtain today.  There's an excellent bug report on the [kernel bugtracker](https://bugzilla.kernel.org/show_bug.cgi?id=15635) that claims the issue was "resolved and documented", which doesn't seem to be the case. 

However, another program `psf2inc` is also capable of converting a `.psf` font to a C source file. You can find a copy of this program [here](http://www.seasip.info/Unix/PSF/), which you'll need to build with `./configure` and `make`. 

After downloading a copy of the Terminus font, we run into a small problem. The font is distributed as `.bdf` files, not `.psf` ones. However, this is easy to change via the included `Makefile`. Just run `make` from within the Terminus source directory and you should now have `.psf` copies of all the Terminus fonts (to learn why there are so many, read the included README). Move `ter-116n.psf` to the same place you built `psftools` and run `./psf2inc -psf1 ter-116n.psf > ter-116n.c`. Now we have a bit of C that supposedly represents a console font. 

However, this file has a bunch of extra information in it, and we can only use up to 255 characters in a console font, and we have 500+ in this file. This is easy to fix, simply by removing any past 255, following the comments in the generated file. You should land up with a file that looks like [this](https://gist.github.com/IanChiles/87b59eb8f33c0fef70fb). 

Replace the contents of the file located within the Linux source at `lib/fonts/font_8x16.c` with the contents of the new font. However, the way that Linux fonts work seems to have changed somewhat recently, meaning that simply overwriting won't work anymore. ONLY replace the `static const unsigned char fontdata_8x16[FONTDATAMAX] = ` variable within the c file, as the rest of the file generated by `psf2inc` is not correct and will not work. 

The final step is to compile our kernel and ensure that it was compiled in. Luckily, there's no special changes we need to make to our Kernel configuration to do so, so just run `make mrproper` and `make` from within the linux source directory to generate a new bzImage. Swap out your old bzImage for this new one, and you should be greeted by your new console font.

Oh no! Even after completely replacing the default Linux kernel font with Terminus, once booted back up, we're greeted by... 

![ugly console font](http://i.imgur.com/AYic4B1.png)

The same ugly console font we had prior to changing it. How is this possible, considering we completely removed the old font from our copy of the kernel source code? 

![not working](http://i.imgur.com/jBwKgxn.png)

Hmmm. It certainly looks like our bootloader has exactly the same font as our kernel. This certainly isn't what we wanted to have happen, but it's easy to fix once you discover what's causing the problem.  

Edit your `/boot/syslinux/syslinux.cfg` to include the line `FONT ter-116n.psf` before the `LABEL` section, add `vga=current` to the append line, and copy `ter-116n.psf` to your `/boot/syslinux/` directory. Reboot, and everything should be perfect.

![Iflawless](http://i.imgur.com/uRicvHm.png)

All of the "greatest stack depth" warnings are a result of me not disabling This is the result of `CONFIG_DEBUG_STACK_USAGE` when I built the kernel this time around - nothing is wrong. 

### Conclusion

This was way harder than it should've been - I spent far too much time thinking I did something wrong building the kernel, only to eventually figure out that it was Syslinux causing problems all along. 

However, without `vga=current` in the kernel APPEND line, it defaults back to a font that wasn't compiled into the kernel... meaning that we're doing something wrong here - I cannot figure this out; if anyone can, get in touch :)

As always, development is always on GitHub - feel free to chime in, create an issue, and send pull requests. 

Feel free to contact me with suggestions, issues, ideas, or just to chat, on freenode in #kirkproject, @IanChiles on twitter, or via email, ianrchiles (at) gmail.com. 


### References
Lots of helpful information here - but nothing about Syslinux causing problems.

- http://kaktyc.wordpress.com/2006/10/23/terminis-in-kernel/

ForeSight Linux seems to have tried something similar, but doesn't seem to use this currently

- https://issues.foresightlinux.org/jira/browse/FL-2302

Kernel.org Bugzilla regarding cpi2fnt... Disappointing, I think. But it turns out you don't need cpi2fnt.

- https://bugzilla.kernel.org/show_bug.cgi?id=15635

Other sources

- http://www.spinics.net/lists/newbies/msg11448.html
- https://lkml.org/lkml/2002/8/29/30

Note for Arch Users, [this](https://aur.archlinux.org/packages.php?ID=22491) AUR package for psftools doesn't build as of 6/4/2014 so you're going to need to just build psftools on your own. 
