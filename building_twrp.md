# Building TWRP for the Samsung Galaxy Tab S7 (T870)

There seems to be a surprising lack of quality documentation available on how to build TWRP, and a Linux kernel that supports it.  Here I'll cover how I was able to do this from start to finish.

The general process is as follows:

1. Acquire the appropriate Linux kernel source code from Samsung.
2. Import Samsung's kernel into Ian Macdonald's kernel source tree.
3. Modify the kernel config supplied by Samsung to build a TWRP-compatible kernel.
4. Build the kernel.
5. Copy the kernel, dtb, and stock recovery_dtbo into the TWRP device tree.
6. Clone TWRP.
7. Add device tree to TWRP source as a submodule (need to understand Android terms/methods for this).
8. Build TWRP.

Note that I'll be doing this on an openSuSE Tumbleweed system, so YMMV when it comes to what you need to do to pull off the various steps here.

# Building a TWRP-compatible Linux Kernel

Building a kernel for Android has a few added layers of complexity, due to the fact that most of us are building on an x86 system (and therefore do not have an ARM toolchain available by default), and that the kernel configuration needs to be tweaked slightly in order to make it TWRP-compatible.

## Toolchain

An ARM toolchain is needed to cross-compile the Linux kernel for ARM on an x86-64 machine.  From what I can tell, Ian Macdonald grabbed pre-built binaries for clang+llvm from here:

https://releases.llvm.org/download.html

Specifically, he used:

https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/clang+llvm-10.0.0-x86_64-linux-gnu-ubuntu-18.04.tar.xz

And then this prebuilt GNU toolchain from here:

https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9

These are both intended to be stashed under a `toolchains` subdirectory, one level above the kernel tree.

### Clang+LLVM

To get this stuff set up, first create the `toolchains` directory, preferably in some clean working directory where all subsequent steps will be performed:

```
mkdir toolchains
```

Now enter the directory, then download and extract the tarball:

```
cd toolchains
wget https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.0/clang+llvm-10.0.0-x86_64-linux-gnu-ubuntu-18.04.tar.xz
tar -x -f clang+llvm-10.0.0-x86_64-linux-gnu-ubuntu-18.04.tar.xz
```

### aarch64-linux-android-4.9

Go back up one level, to your `toolchains` directory, and clone the aarch64-linux-android toolchain:

```
cd ..
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9
```

## Acquiring Linux Kernel Source

In order to build a kernel that will function properly on our device, we need to grab the kernel sources that Samsung used, and build a custom kernel based on that.

They have a search tool to locate open source code that they've released here:

https://opensource.samsung.com/

In order to find the code for the T870, I just entered "T870" into the search box.  At the time of writing, this came back with 3 results, corresponding to the following Android versions they'd released:

T870XXU1ATJ4
T870XXU1BUAC
T870XXU2BUC6

My device came from the factory with T870XXU2BUC6 installed, so this is the source I grabbed.  You click the link with the downward pointing arrow in the "Source" column to download the source.

This will download a SM-T870_EUR_RR_Opensource.zip file, which contains 4 other files:

- Kernel.tar.gz           
- README_Kernel.txt       
- Platform.tar.gz         
- README_Platform.txt    

For our purposes, we're really only interested in the 2 Kernel files.  README_Kernel.txt contains some basic information on how to build the kernel.  There's some interesting info here, however, we don't really need it, as Ian was kind enough to write a build script for us (I'll get to this later).

## Merging the Kernel Source into Ian's Tree

I'm unsure of exactly how Ian was going about this, but, from what I can tell, he was using a `stock-11` branch that contains _only_ the code acquired from Samsung, and then regularly merging this into his master branch, which contains his various changes to get the kernel functioning properly for use with TWRP.

In an attempt to mimic this process, I've done the following:

### Check out Ian's `gts7xl` repo from GitHub

```
git clone https://github.com/ianmacd/gts7xl
```

### Rename repo folder to `gts7l`

This step is not strictly necessary, but I want the naming to actually reflect the device I'm building for.

```
mv gts7xl gts7l
```

### Check out the `stock-11` branch in the `gts7l` repo

```
cd gts7l
git checkout stock-11
cd ..
```

### Expand Samsung's kernel source zip file to a separate directory

```
mkdir samsung-kernel
mv SM-T870_EUR_RR_Opensource.zip samsung-kernel/
cd samsung-kernel
unzip SM-T870_EUR_RR_Opensource.zip
```

### Extract the kernel source tarball into the gts7l repo directory

```
cd samsung-kernel
tar -p -x -C ../gts7l -f Kernel.tar.gz
```

This will take a little bit (30s on my old laptop).  Note that the permissions on _every_ file in the tree will be changed during this operation, as apparently the executable bit is set on literally everything in the gts7l tree.  I think this is a Windows thing or something - not sure.  In any case, the permissions that get pulled in from the Samsung tree appear "correct" to me, so I accepted them as-is, even though it makes the diff look super-nasty.

If you want to see just the code (et al.) changes that actually matter you can do:

```
git diff -G.
```

This will show the diff, but ignore all the permissions changes.

### Commit the changes from the source tarball

```
git add -A
git commit -m 'Import T870XXU2BUF5 from SM-T870_EUR_RR_Opensource.zip.'
git tag BUC6
```

### Merge changes into Ian's master branch

```
git checkout master
git merge stock-11
```

I modified the merge commit to include the commit description from the import commit as well, just to make everything match as closely as possible with what Ian did.

### Modify the kernel config in the same manner as Ian

This is a somewhat tricky step.  I either had the option of copying over Ian's gts7xl_eur_openx_caliban_defconfig to a gts7lwifi_eur_open_caliban_defconfig, and then modifying the appropriate bits to build for the T870, or taking the T870 defconfig that was provided by Samsung and making the same changes to it that Ian made in this commit:

```
commit 770deb5eb27edafe09f1f385b4d7d281cdca954e
Author: Ian Macdonald <ian@caliban.org>
Date:   Mon Sep 7 01:05:42 2020 +0200

    This is a kernel configuration suitable for a TWRP-compatible kernel.
    
    Relative to gts7xl_eur_openx_defconfig, this config disables the
    following features and any dependent features:
    
    CONFIG_SLUB_DEBUG
    CONFIG_UH
    CONFIG_UH_RKP
    CONFIG_KDP_CRED
    CONFIG_KDP_NS
    CONFIG_KDP_DMAP
    CONFIG_UH_LKM_BLOCK
    CONFIG_CFP
    CONFIG_CFP_JOPP
    CONFIG_CFP_ROPP
    CONFIG_CFP_ROPP_SYSREGKEY
    CONFIG_DM_VERITY
    CONFIG_WATCHDOG_HANDLE_BOOT_ENABLED
    CONFIG_SOFT_WATCHDOG
    CONFIG_USB_GSPCA
    CONFIG_MMC_TEST
    CONFIG_FIVE
    CONFIG_PROCA
    CONFIG_GAF
    CONFIG_GATOR
    CONFIG_KPERFMON
    CONFIG_SECURITY_DEFEX
    CONFIG_DEFEX_KERNEL_ONLY
    CONFIG_CORESIGHT
    
    Additionally, the following features are built into the kernel, rather
    than as separate modules:
    
    CONFIG_TCP_CONG_WESTWOOD
    CONFIG_TCP_CONG_HTCP
    CONFIG_BRIDGE_NETFILTER
```

In order to try and make sure that I didn't inadvertently leave anything configured for the T97XX, I chose to go with the latter and use `make menuconfig` to disable each unneeded feature.  This is a bit tedious, but doesn't take that long overall.  I can't/won't go through how to do that for every single feature, but I'll give the general process so that it can be reproduced for other devices.

#### Copy defconfig to .config

```
cp arch/arm64/configs/vendor/gts7lwifi_eur_open_defconfig ./.config
```

#### Run menuconfig

I hacked up Ian's build_t976b script to create a run_menuconfig script, which sets the appropriate variables and runs `make menuconfig` for you.  You can run this simply by doing:

```
./run_menuconfig
```

In the root of your kernel tree.

Note that I apparently hadn't done `make menuconfig` on this laptop previously (somehow), so I had to install `ncurses-devel` and `flex`.

I also repeatedly ran into an error like this at first:

```
scripts/kconfig/mconf  Kconfig
init/Kconfig:17: syntax error
init/Kconfig:16: invalid option
make[1]: *** [scripts/kconfig/Makefile:29: menuconfig] Error 1
```

Poking around on Google, most people ran into this when they had their CROSS_COMPILE variable set incorrectly.  I'm not sure if this was entirely my problem or not, but I _know_ one problem I was having was that Clang couldn't find the appropriate libtinfo.so on my system.  I finally realized that the issue was that I had `libncurses6` installed, but not `libncurses5`.

The error from Clang looked like this:

```
./clang --version
./clang: error while loading shared libraries: libtinfo.so.5: cannot open shared object file: No such file or directory
```

This is the typical error you get when a library isn't installed, but I believe this was getting swallowed by make, so I didn't end up actually seeing what was wrong until I tried to run `clang --version` manually.  This was easily fixed by installing `libncurses5`, but the real problem was not immediately obvious from the `make` errors I was seeing.

Anyway, after hacking up the `run_menuconfig` script and getting this library installed, I was finally able to get menuconfig running properly.

#### Remove unwanted features

In order to remove a single feature, you need to search for the feature in menuconfig, and disable it.

With menuconfig running, press the `/` key to begin a search.  Looking at Ian's commit from above, the first thing to disable is `SLUB_DEBUG`, so enter that as your search string and press `Enter`.

This will give a screen something like this:

```
Symbol: SLUB_DEBUG [=y]
Type  : bool
Prompt: Enable SLUB debugging support
  Location:
(1) -> General setup
  Defined at init/Kconfig:1809
  Depends on: SLUB [=y] && SYSFS [=y]
  Selected by [n]:
  - KASAN_GENERIC [=n] && <choice> && HAVE_ARCH_KASAN [=y] && CC_HAS_KASAN_GENERIC [=y] && (SLUB [=y] && SYSFS [=y] || SLAB [=n] && !DEBUG_SLAB [=n]) && SLUB [=y]
  - KASAN_SW_TAGS [=n] && <choice> && HAVE_ARCH_KASAN_SW_TAGS [=y] && CC_HAS_KASAN_SW_TAGS [=y] && (SLUB [=y] && SYSFS [=y] || SLAB [=n] && !DEBUG_SLAB [=n]) && SLUB [=y]

...
```

Here you'll press the `1` key to edit the entry, and you'll be brought to another screen with

```
[*] Enable SLUB debugging support
```

Highlighted.  Here you press `N` to disable the feature.

Rinse and repeat for remaining features.

Now I know this could be scripted, and probably somewhat easily, but there weren't really that many features to disable, and I already understand this method, so this is how I did it.  I'm sure someone out there will call me an idiot for doing it this way, and to that person I would say, "write your own guide then."

Note that some features will not be able to be disabled, due to dependent features being enabled.  Those features look like this in menuconfig:

```
-*- Enable micro hypervisor feature of Samsung
```

With dashes (`-`) around the star, instead of brackets (`[]`).  For those, you'll need to find the dependent feature and disable it first.  It's usually going to be right below the "dashed" feature, and indented to indicate that it depends on the dashed feature.

### Build the Kernel

At this point, all that needs to be done is to run the build script in the root of the kernel tree:

```
./build_t870
```

Or one would hope, at least...  In my case, I was missing some packages, since I hadn't built a kernel on this machine before.  I usually build stuff on my desktop build server in my basement, but I already started working on the laptop, and didn't want to move everything.  Anyway, I'll enumerate the errors I hit and how I corrected them below.

#### Missing openssl-devel

The first error I hit was this:

```
  HOSTCC  scripts/sign-file
scripts/sign-file.c:25:10: fatal error: openssl/opensslv.h: No such file or directory
   25 | #include <openssl/opensslv.h>
      |          ^~~~~~~~~~~~~~~~~~~~
compilation terminated.
make[1]: *** [scripts/Makefile.host:90: scripts/sign-file] Error 1
```

This was remedied easily, by installing `libopenssl-devel`.1

#### Missing bc

The second error I hit was:

```
  CC      kernel/bounds.s
/bin/sh: line 1: bc: command not found
make[1]: *** [Kbuild:42: include/generated/timeconst.h] Error 127
make: *** [Makefile:1256: prepare0] Error 2
make: *** Waiting for unfinished jobs....
```

Not sure how bc isn't installed by default on my distro, but whatever.  Fixed by installing bc.

Surprisingly enough, these were the only 2 hard errors I hit.  There were some other errors that I saw early in the build, but they were apparently not fatal.  I'll revisit those if necessary.

#### Symbol versioning failure for `gsi_write_channel_scratch`

I'm not sure what caused this one:

```
  MODPOST vmlinux.o
WARNING: EXPORT symbol "gsi_write_channel_scratch" [vmlinux] version generation failed, symbol will not be versioned.
WARNING: modpost: Found 1 section mismatch(es).
To see full details build your kernel with:
'make CONFIG_DEBUG_SECTION_MISMATCH=y'
  KSYM    .tmp_kallsyms1.o
  KSYM    .tmp_kallsyms2.o
  LD      vmlinux
  SORTEX  vmlinux
  SYSMAP  System.map
  FIPS : Generating hmac of crypto and updating vmlinux...
HMAC for "crypto" module is: b'8aa5b5cd209dec2fbe85b32ae48bb4110230aa331fb3190a3f91c5205b325595'
FIPS integrity procedure has been finished for crypto
  OBJCOPY arch/arm64/boot/Image
  Building modules, stage 2.
  MODPOST 0 modules
echo -n 'UNCOMPRESSED_IMG' > arch/arm64/boot/Image-dtb-hdr && \
printf \\014\\130\\355\\002 >> arch/arm64/boot/Image-dtb-hdr
WARNING: EXPORT symbol "gsi_write_channel_scratch" [vmlinux] version generation failed, symbol will not be versioned.
  CAT     arch/arm64/boot/Image-dtb
```

I don't really know what that function does, so I'm unsure of whether or not this will be an issue at this point.  Considering the fact that symbol versioning only matters when loading kernel modules, I doubt this matters, but it's something to keep an eye on.

# Extract the dtb and recovery_dtbo from your firmware

This took me quite a while to figure out.  There's a good bit of information on what these files _are_, but basically nothing on where the hell you're supposed to get them from.

I'll skip over all the thrashing around I did to figure this crap out, and just tell you what you need to know to get your hands on them.

## Acquire firmware

I can't really give good instructions on this.  I got my T870XXU2BUC6 firmware from samfw.com, but firmware can also be grabbed using tools like samloader and samfirm.js.  I think these will only allow you to grab the _latest_ firmware though, so you'll have to hunt around if you want to build for an older one.

Anyway, after searching around for a while, I was finally able to acquire a Samfw.com_SM-T870_XAR_T870XXU2BUC6_fac.zip that contained the firmware I was after.

## Extract recovery.img from firmware

This part should be self-explanatory.  Just extract the zip file somewhere on your disk.  Once extraced, you'll see 4 files like this:

```
AP_T870XXU2BUC6_CL21033099_QB38876786_REV00_user_low_ship_MULTI_CERT_meta_OS11.tar.md5
BL_T870XXU2BUC6_CL21033099_QB38876786_REV00_user_low_ship_MULTI_CERT.tar.md5
CSC_OMC_OXM_T870OXM2BUC6_CL21033099_QB38876786_REV00_user_low_ship_MULTI_CERT.tar.md5
HOME_CSC_OMC_OXM_T870OXM2BUC6_CL21033099_QB38876786_REV00_user_low_ship_MULTI_CERT.tar.md5
```

And potentially some .txt file with a bit of info about where the firmware came from.

Anyway, the file we're interested in here is the AP_<blah> file.  I created and `AP` directory and extracted that tarball there:

```
mkdir AP
tar -x -C AP -f AP_T870XXU2BUC6_CL21033099_QB38876786_REV00_user_low_ship_MULTI_CERT_meta_OS11.tar.md5
```

Inside this directory, you'll find a bunch of stuff, but the file we're interested in is `recovery.img.lz4`:

```
-rw-r--r-- 1 <censored> users  41030218 Mar 16 03:07 recovery.img.lz4
```

I created another subdirectory called recovery, and extracted this file there:

```
mkdir recovery
cp recovery.img.lz4 recovery
cd recovery
lz4 -d recovery.img.lz4
```

You should now have a file in the directory called `recovery.img`.

## Clone Android's mkbootimg repo

This is a bit of a detour from where we were (and should probably be moved up in the guide), but I'm sticking it here for now.

Anyway, you need the `mkbootimg` tools to properly unpack everything inside this image.  You clone those like this:

```
git clone https://android.googlesource.com/platform/system/tools/mkbootimg
```

Just put them wherever, but you'll need to know the path in order to run the `unpack_bootimg.py` script in the next step.

## Unpack the recovery.img

Go back to wherever you extracted your `recovery.img`, and run the following commands:

```
mkdir out
<path_to_mkbootimg_repo>/unpack_bootimg.py --out out --boot_img recovery.img
```

You should now have the following files in your `out` directory:

```
dtb
kernel
ramdisk
recovery_dtbo
```

The two we're interested in are `dtb` and `recovery_dtbo`.  These will be placed in the `prebuilt` directory of Ian's `twrp_gts7*` tree, along with the custom kernel that you built in the previous step.

# Building TWRP

## Install Repo

This is something that others who have built Android stuff a lot probably won't have to do, but I'm including instructions here for those who (like me) haven't done this before.

I followed these instructions to get repo installed:

https://source.android.com/setup/develop#installing-repo

Essentially, all you need to do is this:

```
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

## Clone the "minimal manifest" TWRP source

After looking at some posts on the TWRP Telegram group, I finally figured out that I need to build up a full TWRP source tree from scratch, as opposed to cloning the tree that Ian posted (I think).

Anyway, for Android 11 this looks like this:

```
mkdir android_bootable_recovery
cd android_bootable_recovery
repo init -u git://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git -b twrp-11
repo sync
```

I did this in the same "work" directory where I cloned my kernel source, and placed my `toolchains` directory.  Note that the initial sync is going to take well over an hour, and will likely soak up 50G+ of disk space.  I had to do a lot of cleanup in flight to keep my root from filling up during the sync.

## Clone Ian's original device tree branch

Do this in your root working directory:

```
git clone https://github.com/ianmacd/twrp_gts7xl
```

Now rename it to match what we're working on:

```
mv twrp_gts7xl twrp_gts7lwifi
```

## Modify Ian's original branch

### Copy in necessary prebuilt files

You'll need to copy in the kernel you built, and the two device tree files that you gathered from your firmware:

```
cd twrp_gts7lwifi/prebuilt
cp ../../gts7l/arch/arm64/boot/Image ./
cp ../../recovery_dtbo ./
cp ../../dtb dtb/dtb.dtb
```
### Modify various build files

I won't attempt to enumerate all the necessary changes here.  It wasn't much.  Here's a diff:

```
commit a796c2f81ebd280eba718b7eed9dcfc5e79391bf (HEAD -> master)
Author: Alex Thorlton <bxlentpartyon@gmail.com>
Date:   Fri Jul 30 16:52:23 2021 -0500

    Modify build files for gts7lwifi BUC6 firmware.

diff --git a/Android.mk b/Android.mk
index fc67ee1..9d280e9 100755
--- a/Android.mk
+++ b/Android.mk
@@ -1,6 +1,6 @@
 LOCAL_PATH := $(call my-dir)

-ifneq ($(filter gts7xl, $(TARGET_DEVICE)),)
+ifneq ($(filter gts7lwifi, $(TARGET_DEVICE)),)

 include $(call all-makefiles-under,$(LOCAL_PATH))

diff --git a/AndroidProducts.mk b/AndroidProducts.mk
index 9305eeb..4f4a6d2 100755
--- a/AndroidProducts.mk
+++ b/AndroidProducts.mk
@@ -1,2 +1,2 @@
 PRODUCT_MAKEFILES := \
-       $(LOCAL_DIR)/omni_gts7xl.mk
+       $(LOCAL_DIR)/omni_gts7lwifi.mk
diff --git a/BoardConfig.mk b/BoardConfig.mk
index b0a16cd..e0b8e5c 100755
--- a/BoardConfig.mk
+++ b/BoardConfig.mk
@@ -12,7 +12,7 @@
 #
 PLATFORM_VERSION := 11

-DEVICE_PATH := device/samsung/gts7xl
+DEVICE_PATH := device/samsung/gts7lwifi

 # Architecture
 TARGET_ARCH := arm64
@@ -55,11 +55,11 @@ TARGET_BOARD_PLATFORM_GPU := qcom-adreno650
 QCOM_BOARD_PLATFORMS += kona

 # Recovery
-TARGET_OTA_ASSERT_DEVICE := gts7xl
+TARGET_OTA_ASSERT_DEVICE := gts7lwifi
 BOARD_HAS_LARGE_FILESYSTEM := true
 TARGET_RECOVERY_FSTAB := $(DEVICE_PATH)/recovery/root/etc/recovery.fstab
 TARGET_COPY_OUT_VENDOR := vendor
-PLATFORM_SECURITY_PATCH := 2021-01-01
+PLATFORM_SECURITY_PATCH := 2021-03-01

 # Portrait orientation:
 #
diff --git a/README.md b/README.md
index 61bff4f..22e283c 100755
--- a/README.md
+++ b/README.md
@@ -1,3 +1,3 @@
-# Samsung Galaxy Tab S7+ 5G device tree for TWRP
+# Samsung Galaxy Tab S7 device tree for TWRP

-twrp_android_device_samsung_gts7xl
+twrp_android_device_samsung_gts7lwifi
diff --git a/omni_gts7xl.mk b/omni_gts7lwifi.mk
similarity index 92%
rename from omni_gts7xl.mk
rename to omni_gts7lwifi.mk
index d37cafa..ed2a983 100755
--- a/omni_gts7xl.mk
+++ b/omni_gts7lwifi.mk
@@ -21,9 +21,9 @@
 $(call inherit-product, $(SRC_TARGET_DIR)/product/aosp_base_telephony.mk)

 ## Device identifier. This must come after all inclusions
-PRODUCT_NAME := omni_gts7xl
-PRODUCT_DEVICE := gts7xl
-PRODUCT_MODEL := SM-T976B
+PRODUCT_NAME := omni_gts7lwifi
+PRODUCT_DEVICE := gts7lwifi
+PRODUCT_MODEL := SM-T870
 PRODUCT_BRAND := samsung
 PRODUCT_MANUFACTURER := samsung
 PRODUCT_GMS_CLIENTID_BASE := android-samsung
```

## Create a local bare repo to house your device tree branch

You have to do this, due to the way that repo pieces together remote repository URLs.  See: https://gerrit.googlesource.com/git-repo/+/master/docs/manifest-format.md

To do this I did the following in my root work directory:

```
mkdir klugman-device-tree
cd klugman-device-tree
mkdir twrp_gts7lwifi.git
cd twrp_gts7lwifi.git
git init --bare
```

Now you need to add this remote to your device tree checkout, and push master to it.

(from your work area root again)

```
cd twrp_gts7lwifi
git remote add klugman-device-tree <path_to_workarea>/klugman-device-tree/twrp_gts7lwifi.git
git push klugman-device-tree HEAD
```

## Add your local device tree repo as a remote

You'll want to make the file `.repo/local_manifests/gts7lwifi.xml` like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>

  <remote name="klugman-device-tree"
          fetch="file://<path_to_workarea>/klugman-device-tree"
          review="file:///<path_to_workarea>/klugman-device-tree"/>

  <project path="device/samsung/gts7lwifi"
          name="twrp_gts7lwifi"
          remote="klugman-device-tree"
          revision="master"/>

</manifest>
```

After that, I ran a `repo sync`, and was able to see that it created the `device/samsung` subdirectory in the root of my TWRP tree.

Ideally, all this stuff would be pushed up to github so that I didn't have to do weird hackery with bare repos, but this worked, so whatever.

## Run the build

The moment of truth.

```
. build/envsetup.sh
lunch omni_gts7lwifi-eng
export ALLOW_MISSING_DEPENDENCIES=true
mka recoveryimage
```

This is what was in the instructions I found everywhere, but after running this, no recovery.img was built in the out/ directory.

### Fix the build

After poking around a bit, I found that I was not the only person running into the issue that I hit.  It had already been documented here:

https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp/issues/2

Knowing that it wasn't something I had screwed up gave me some motivation to try to pin down the issue.  I won't go through all of the details, but I was eventually able to determine that I needed to set the `BOARD_RECOVERYIMAGE_PARTITION_SIZE` variable in my BoardConfig.mk.  After doing this the build went through smoothly (well it filled up my disk once, but finished successfully after I made room).

Note that I'm not confident that this is the "correct" solution, and I'm also not entirely positive that I got the partition size right.  I came up with this value after poking around a bit in `abd shell`.  Here's what I looked at:

First find the recovery partitions actual block device:

```
gts7lwifi:/ $ ls -l /dev/block/by-name/recovery 
lrwxrwxrwx 1 root root 16 2019-04-13 00:58 /dev/block/by-name/recovery -> /dev/block/sda23
```

Now see how big it is:

```
gts7lwifi:/ $ head -1 /proc/partitions && grep sda23 /proc/partitions 
major minor  #blocks  name
 259        7      84852 sda23
```

Since the recovery partition isn't mounted when the device is operating normally, I couldn't just pull the size directly from `df`, but I compared the `/proc/partitions` info to `df` for a few things that were mounted and I'm fairly confident that our block size here is 1K.

That being the case, we multiply the `#blocks` from `/proc/partitions` by 1024 to get our partition size in bytes:

```
84852 * 1024 = 86888448
```

So I added:

```
BOARD_RECOVERYIMAGE_PARTITION_SIZE := 86888448
```

After the other partition related info in the BoardConfig.mk in my device tree.  This seems to have done the trick.

# Conclusions

## Still untested

At the time of writing this, I have not yet been able to test the recovery image, as my tablet isn't quite hacked up enough yet.  I will update this document when I have some verfied test results.  I won't be surprised if more hackery is needed to make this work, but we at least have a development environment to work in.

## Don't do this on your laptop

It was way stupid for me to try this on my ancient laptop (Lenovo G50).  At the end of the day, it all worked fine, but I think I chewed up a solid 100G of disk space.  The processor in this thing is also just not cut out for this kind of work.  Fortunately, since basically all of this stuff is self-contained in the Android tree, I can just `rsync` this off to my actual build system sometime soon.
