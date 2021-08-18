# Building TWRP for the Samsung Galaxy Tab S7 (T870)

I've already written a full guide on how to build TWRP using kernel sources acquired directly from Samsung, a modified device tree, and the TWRP minimal manifest.  This guide will skip around the gory details of hacking up the kernel and device tree, and simply explain how to reproduce the TWRP build that I ran, using the sources I've already prepared.

The general process is as follows:

1. Clone TWRP minimal manifest.
2. Modify Android manifest to pull in my device tree.
3. Build TWRP.

Note that I'll be doing this on an openSuSE Tumbleweed system, so YMMV when it comes to what you need to do to pull off the various steps here.

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
mkdir twrp_aosp
cd twrp_aosp
repo init -u git://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git -b twrp-11
repo sync
```

Note that the initial sync is going to take well over an hour, and will likely soak up 50G+ of disk space.  I had to do a lot of cleanup in flight to keep my root from filling up during the sync.

## Add my device tree repo as a remote

This was a bit weird to figure out, but I think I got it right.  You'll want to edit `.repo/manifests/twrp-extras.xml` like this:

Add this below the other remotes defined at the top:

```
    <remote name="bxlentpartyon"
            fetch="https://github.com/bxlentpartyon"
            review="https://github.com/bxlentpartyon"/>
```

And then this below the TeamWin android_bootable_recovery project:

```
    <project path="device/samsung/gts7lwifi" name="twrp_gts7lwifi" remote="bxlentpartyon" revision="master"/>
```

After that, I ran a `repo sync`, and was able to see that it created the `device/samsung` subdirectory in the root of my TWRP tree.

Ideally, all this stuff would be pushed up to github so that I didn't have to do weird hackery with bare repos, but this worked, so whatever.

## Run the build

The moment of truth.

```
. build/envsetup.sh
lunch twrp_gts7lwifi-eng
export ALLOW_MISSING_DEPENDENCIES=true
mka recoveryimage
```

This is what was in the instructions I found everywhere, but after running this, no recovery.img was built in the out/ directory.

# Conclusions

## Instructions Untested

Since I already did all this "from scratch," this is not the exact process I used to build TWRP.  I've skipped over steps that are unnecessary when using my already-prepared device tree (including the pre-built kernel).  Once somebody has successfully attempted a build using these instructions, I'll update this bit.

## Don't do this on your laptop

It was way stupid for me to try this on my ancient laptop (Lenovo G50).  At the end of the day, it all worked fine, but I think I chewed up a solid 100G of disk space.  The processor in this thing is also just not cut out for this kind of work.  Fortunately, since basically all of this stuff is self-contained in the Android tree, I can just `rsync` this off to my actual build system sometime soon.
