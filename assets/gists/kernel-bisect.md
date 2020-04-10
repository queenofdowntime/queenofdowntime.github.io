---
layout: gist
---

## Bisecting the Kernel

I'm sure there are maaany ways to bisect the ubuntu kernel, here is what I do.

**1. Space setup**

I would advise that whatever machine you are running on has a decently size persistent disk attached to act as your workdir.
This will give you more space and mean that, if you thoroughly F this up, you can trash the world without losing anything :).

If you have yet to narrow down to a particular version, you don't need to start compiling just now as you can still work with pre-compiled releases from the [ubuntu mainline](http://kernel.ubuntu.com/~kernel-ppa/mainline).

See [Switching the Kernel here](/assets/gists/switch-kernel).

Once you have found the official version which contains the problem, you can see a list of the commits which went into it on the [ubuntu mainline](http://kernel.ubuntu.com/~kernel-ppa/mainline) version page, eg: [v4.14.60 changes](http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10/CHANGES). You can also find further details of the commits at http://cdn.kernel.org/pub/linux/kernel/.

**2. Let's get started!**

```sh
# set yourself up
mkdir -p $HOME/kern/source && cd $HOME/kern/source

# takes forever
git clone git://git.launchpad.net/~ubuntu-kernel-test/ubuntu/+source/linux/+git/mainline-crack

cd mainline-crack
git log --oneline v<last-known-good>..v<current-bad-version> # eg v4.14.59..v4.14.60
# choose some sensible sha in the middle, or one that just looks suspicious if you know what you are after
git checkout <sha>

# wget any patches from http://kernel.ubuntu.com/~kernel-ppa/mainline/<VERSION>/, and then...
patch -p1 < 0001-base-packaging.patch
etc

# copy your existing kernel config so that you don't accidentally upset something else and distract you from your problem
cp /boot/config-$(uname -r) .config
make oldconfig

# start building some custom debs, DON'T forget to name the custom kernel after the sha you are building from, it will be helpful
make clean
make -j `getconf _NPROCESSORS_ONLN` deb-pkg LOCALVERSION=-custom-<sha>

# if your `/` is small and you don't have power to increase it, bind mount a modules dir backed by your persistent disk over the real one so that you don't flood it
mkdir $HOME/kern/modules/v4.14.59-custom-<sha>
mkdir /lib/modules/v4.14.59-custom-<sha>
mount --bind $HOME/kern/modules/v4.14.59-custom-<sha> /lib/modules/v4.14.59-custom-<sha>

# make modules
make modules_install -j `getconf _NPROCESSORS_ONLN`

# move your custom debs to a sensible location
mkdir $HOME/kern/custom-comp/v4.14.59-custom-<sha>
mv *.deb ../../custom-comp/v4.14.59-custom-<sha>
```

To use your newly compiled kernel just perform the same steps as in [Switching the Kernel](/assets/gists/switch-kernel).

Good luck
