# HADK for Sony XZ2

This is based on https://sailfishos.org/wiki/Sailfish_X_Xperia_Android_9_Build_and_Flash and adjusted for Tama 
platform and, when needed, XZ2 device (akari) of this platform.

## .hadk.env

```
export VENDOR="sony"
export DEVICE="h8216" 
export HABUILD_DEVICE=akari 
export FAMILY=tama 
export ANDROID_FLAVOUR=pie
export HAVERSION="sony-"$FAMILY"-aosp-"$ANDROID_FLAVOUR
# Set arch to armv7hl even if you are porting a 64bit device
export PORT_ARCH="armv7hl"
```

## Setting up sources

```
HABUILD_SDK $

sudo apt-get install libssl-dev
sudo mkdir -p $ANDROID_ROOT
sudo chown -R $USER $ANDROID_ROOT
cd $ANDROID_ROOT
git clone https://github.com/sailfishos-sony-tama/droid-hal-sony-$FAMILY-$ANDROID_FLAVOUR .
git submodule init
git submodule update
cd hybris/mw/libhybris
git submodule init
git submodule update
cd -
ln -s ../dhd rpm/
mv rpm dhd-rpm
```

In contrast to official instructions, we use generic Sony AOSP repo and associate it with the device 
via local manifest:

```
repo init -u git://github.com/sailfishos-sony-tama/android.git -b sony-aosp-pie -m tagged-manifest.xml
mkdir $ANDROID_ROOT/.repo/local_manifests
```

Add the following content to $ANDROID_ROOT/.repo/local_manifests/$DEVICE.xml

```XML
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote fetch="https://github.com/sailfishos-sony-tama" name="hybris-tama"/>
  <project name="droid-src-sony-tama-pie" path="rpm" remote="hybris-tama" revision="master"/>
</manifest>
```

Continue with syncing repo and build

```
repo sync -j8 --fetch-submodules
mv rpm droid-src
```

For patches, we use a mix of Sony's repo_update and hybris patches. This allows to simplify changes
in Android's base by keeping Sony's patches in sync with Android tree and apply all patches developed 
for Hybris.

```
ln -s droid-src/patches .
droid-src/apply-patches.sh --mb
SKIP_SYNC=TRUE droid-src/repo_update/repo_update.sh
mv dhd-rpm rpm
./setup-sources.sh --mb
```

# Build hybris-hal

Start the build. As two targets are missed, we make them separately before main build

```
source build/envsetup.sh
export USE_CCACHE=1
lunch aosp_$DEVICE-user
make fec append2simg
make -j$(nproc --all) hybris-hal
make verity_key
```


# Build systemimage vendorimage

This can be done in parallel to the previous or the next sections. On your Linux host:

Follow Sony's instructions for AOSP9 builds to set up sources. Note that the same branch as used 
for tagged manifest in hybris AOSP has to be used:

```
mkdir -p $ANDROID_ROOT-syspart
cd $ANDROID_ROOT-syspart
BRANCH=android-9.0.0_r46
repo init -u https://android.googlesource.com/platform/manifest -b $BRANCH
cd .repo
git clone https://github.com/sonyxperiadev/local_manifests
cd local_manifests
git checkout $BRANCH
cd ../..
repo sync -j32
./repo_update.sh
```

On Gentoo (probably not needed on other systems):
```
unset JAVAC
# due to https://github.com/sonyxperiadev/bug_tracker/issues/136
cd prebuilts/misc/linux-x86/flex
rm flex-2.5.39
tar zxf flex-2.5.39.tar.gz
cd flex-2.5.39
./configure
make
mv flex ../
cd ../
rm -rf flex-2.5.39
mv flex flex-2.5.39
cd ../../../../
```

Build images (reduce `-j` as it is heavy on RAM if needed)

```
source build/envsetup.sh
export USE_CCACHE=1
lunch aosp_$DEVICE-user
make -j$(nproc --all) systemimage vendorimage
```


# Setup SB2

Go through chapter 6 of the HADK document.


# Build packages

In PLATFORM_SDK

```
cd $ANDROID_ROOT
sudo zypper ref
rpm/dhd/helpers/build_packages.sh --droid-hal
git clone --recursive https://github.com/sailfishos-sony-tama/droid-config-sony-$FAMILY-$ANDROID_FLAVOUR hybris/droid-configs -b master
rpm/dhd/helpers/build_packages.sh --configs
cd hybris/mw/libhybris
git checkout master
cd -
rpm/dhd/helpers/build_packages.sh --mw=https://github.com/mer-hybris/sailfish-connman-plugin-suspend.git
rpm/dhd/helpers/build_packages.sh --mw # select "all" option when asked
```

# syspart

After systemimage vendorimage have finished, in HABUILD_SDK
```
cd $ANDROID_ROOT-syspart
sudo mkdir -p $ANDROID_ROOT-mnt
export PATH=$ANDROID_ROOT-syspart/out/host/linux-x86/bin:$PATH
simg2img out/target/product/$HABUILD_DEVICE/system.img /tmp/system.img.raw
sudo mount /tmp/system.img.raw $ANDROID_ROOT-mnt
cd $ANDROID_ROOT/hybris/mw
D=droid-system-$VENDOR-$ANDROID_FLAVOUR-template
git clone --recursive https://github.com/sailfishos-sony-tama/$D
cd $D
sudo droid-system-device/helpers/copy_system.sh $ANDROID_ROOT-mnt/system rpm/droid-system-$HABUILD_DEVICE.spec
# please do not commit the binaries nor push them to github repo,
# because the licence has not been determined,
# thus they have to be built manually
sudo chown -R $USER .
sudo umount $ANDROID_ROOT-mnt
rm /tmp/system.img.raw
cd $ANDROID_ROOT-syspart
simg2img out/target/product/$HABUILD_DEVICE/vendor.img /tmp/vendor.img.raw
sudo mount /tmp/vendor.img.raw $ANDROID_ROOT-mnt
cd $ANDROID_ROOT/hybris/mw
D=droid-vendor-$VENDOR-$ANDROID_FLAVOUR-template
git clone --recursive https://github.com/sailfishos-sony-tama/$D
cd $D
sudo droid-system-device/helpers/copy_vendor.sh $ANDROID_ROOT-mnt rpm/droid-system-vendor-$HABUILD_DEVICE.spec
sudo chown -R $USER .
sudo umount $ANDROID_ROOT-mnt
rm /tmp/vendor.img.raw
```