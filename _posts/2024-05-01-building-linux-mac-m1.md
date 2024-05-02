---
title: "Building the Linux Kernel on Apple Silicon"
categories:
  - Programming
tags:
  - apple silicon
  - linux
  - kernel
  - m1
  - m2
  - m3
  - mac
header:
  image: /assets/pelican.jpeg
  caption: "Pelicans and the Linux penguin are not very similar."
---
The following guide describes how to build the Linux kernel on Apple Silicon. This is a bit of a pain to do, especially compared to just building the kernel on a Linux OS like a regular person, but we persevere. For posterity, the below process was performed with an M1 Macbook Air, macOS Sonoma, and kernel version v6.8. Older (and future) versions of the kernel are almost guaranteed to require different changes to those described here. This post builds on the excellent work from [here][jonghoonpark].

## Prerequisites
- The [homebrew package manager][homebrew].
- The Git CLI is recommended (if you don't have Git then this may not be the exercise for you).
- Around 12GB of free space.

## Installing Dependencies
The following packages are required for building the Linux kernel on Apple Silicon:
```console
brew install cmake autoconf libtool gcc automake openssl
```
The ARM cross-compiler is also required. I had already downloaded this from the ARM website and had updated my `PATH` to include the compiler. However best practice would be to install the cross-compiler using homebrew:
```console
brew install gcc-arm-embedded --cask
```
Macs are bundled with a version of `make`. Humourously, this version is far too dated to build the current Linux kernel. Homebrew can install the latest version of `make` but for compatibility, it is installed under the alias `gmake`. Basically, although the package is called `make`, you have to type `gmake` to use it. I don't know why this confused me so much.
```console
brew install make
```
The `gmake` install can be checked easily. We can see that the system `make` is located in `/usr/bin/make` and the homebrew `gmake` is located in `/opt/homebrew/bin/gmake`. This is an exercise in redundancy but it is nice to know when something is working...

```console
username@Users-MBA:~$ which make
/usr/bin/make
username@Users-MBA:~$ which gmake
/opt/homebrew/bin/gmake
```

## Missing Header Files
There are a few system header files that the kernel depends on that are part of any Linux OS but not macOS. For older kernel versions, only `elf.h` is required. For the more recent versions, both `elf.h` and `byteswap.h` are needed. Luckily, some smart people have ported these headers over. They can be found [here][elf] and [here][byteswap]. Download these files and place them in `/usr/local/include`. Using your favourite text editor (Sublime Text), delete the last two lines of `byteswap.h` (shown below). I'm not sure why they are there. 
```c
(b) change this line 357 to
while((c = getopt(argc, argv, "hvi::o::p:k::")) != -1){
```

## Preparing the Filesystem
Linux uses a case-sensitive file system. The default filesystem on Mac is case-insensitive. We could wipe our entire disk and reformat as case-sensitive, but thankfully there is a less destructive method. We will create a mountable disk image that uses a case-sensitive filesystem, allowing us to build the Linux kernel without reformatting our computer. First, open Disk Utility. Then click `File` -> `New Image` -> `Blank Image...`. 

Set the following options:
- Image Format: `sparse disk image`
- Format: `Mac OS Extended (Case-sensitive, Journaled)`
- Size: `12GB`
- Name: `workspace`
- Save As: `linux-kernel-fs`

In my case, setting `Image Format` reset some of the other settings in the window so it is best to do this first. If the mounted name `workspace` or disk name `linux-kernel-fs` isn't good enough for you, replace these with your "better name" in the following steps. The disk image settings should look like this:

![image](/assets/building-linux-mac-m1/disk-util.png)

Locate to where the disk image was saved (under the name `linux-kernel-fs`) and double click on it to mount it. Now there should be a Location named `workspace` in the Finder sidebar. This is where the Linux kernel will be built.

## Cloning and Editing the Kernel
The case-sensitive disk image is mounted at `/Volumes/workspace`. Change to this directory in terminal:
```console
cd /Volumes/workspace
```
Clone the Linux kernel repository into `workspace`:
```console
git clone https://github.com/torvalds/linux.git
```
Switch to the latest stable release (or be brave and don't). At the time of writing this was v6.8. To check the latest release, visit the [Linux GitHub page][linux] and click on the `master` drop down menu, then select the `tags` tab. If the tag doesn't end in `rc`, it's probably stable.
```console
cd linux
git checkout v6.8
```

The file `<kernel-src>/scripts/mod/file2alias.c` needs to be modified. Open this file in your favourite editor (Sublime Text). The following snippet of code should be added in around line 42:
```c
#ifdef __APPLE__
#define uuid_t compat_uuid_t
#endif
```
Note: The line number changes depending on the kernel version - I compiled an older version and it was line 45. The reference code below will probably make it clearer. Just don't put the `#define` in the middle of a struct or after the `uuid_t` type is used or something stupid...The updated section of `file2alias.c` should now look something like this:
```c
/* backwards compatibility, don't use in new code */
typedef struct {
  __u8 b[16];
} uuidle;

#ifdef __APPLE__
#define uuid_t compat_uuid_t
#endif

typedef struct {
  __u8 b[16];
} uuid_t;
```

## Building the Kernel
Finally! The hard work is done, now the kernel can be built. Several environmental variables need to be set so the cross compiler can do its job. These are set in terminal as follows:
```console
export ARCH=arm
export CROSS_COMPILE=arm-none-eabi-
export CPATH=/opt/homebrew/include
export LIBRARY_PATH=/opt/homebrew/lib
```
Now, the kernel config can be generated. From the kernel source directory, run:
```console
make menuconfig
```

Select any options you want then save and exit. This will generate a `.config` file that is referenced by the kernel build process. Finally, to build the kernel (with 8 blazing fast cores), run:
```console
make -j 8
```
Sit back and relax (for about 10 minutes) while the kernel is built. Preferably with no errors. The kernel image will be located at `arch/arm/boot/zImage`. This process pairs well with a nice cup of tea - or the house red.

[jonghoonpark]: https://jonghoonpark.com/2023/08/21/build-linux-kernel-from-macbook-pro-m1-en
[homebrew]: https://brew.sh/
[linux]: https://github.com/torvalds/linux
[elf]: https://gist.github.com/mlafeldt/3885346
[byteswap]: https://gist.github.com/atr000/249599
