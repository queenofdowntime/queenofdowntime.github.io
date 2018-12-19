---
layout: gist
---

## Switching the Kernel

I'm sure there are maaany ways to switch out the ubuntu kernel, here is what I do.
Going from 4.4 to 4.10 as an example.

**1. [Download the kernel version](http://kernel.ubuntu.com/~kernel-ppa/mainline/) that you're after**

```
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10/linux-headers-4.10.0-041000_4.10.0-041000.201702191831_all.deb
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10/linux-headers-4.10.0-041000-generic_4.10.0-041000.201702191831_amd64.deb
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10/linux-image-4.10.0-041000-generic_4.10.0-041000.201702191831_amd64.deb
```

**2. Install the .debs**

```
sudo dpkg -i *.deb
```

**3. Edit grub conf files to update the version**

```
sed -i 's/4.4.0-137-generic/4.10.0-041000-generic/g' /boot/grub/grub.conf
sed -i 's/4.4.0-137-generic/4.10.0-041000-generic/g' /boot/grub/menu.lst
```

**4. Reboot and ssh**

```
reboot
```

After a minute or two you can ssh back on and run `uname -r` to verify your kernel has changed.
