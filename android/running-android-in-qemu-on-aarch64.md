# Running Android in QEMU on aarch64

This document provides some tips about compiling Android and running it
in QEMU on aarch64 architecture. It doesn't replace the official Android
documentation in any way, it just supplements it with some practical
information.

Android virtualization is apparently focused towards
[crosvm](https://github.com/google/crosvm) and although QEMU is still
supported, it may not work as good as crosvm or may not have all the
features that crosvm has. One of the goals of this documentation is to
fill the relevant gaps.

## Which Android to use?

Android offers different kinds of images, i.e: phone, automotive, tv. We
currently use primarily the phone
[AOSP](https://source.android.com/docs/core) (Android Open Source
Project) images because they appear to be the most stable ones.

As for the version, it's usually best to use either the latest Android
release or the latest version from the master branch. But note, that not
always seems to work. If you\'re in doubt, try first using crosvm, if
the image doesn\'t work with crosvm is unlikely to work with QEMU.

## Fetching Android

In order to run Android, the following parts are needed:

1. An Android image.
2. A cvd-host package.
3. A cuttlefish-base package with some resources and scripts.

In the following steps we will learn how to generate an Android image,
the cvd-host and the cuttlefish-base packages.

### Using Binary images

The easiest way to obtain runable Android images is to fetch compiled
images from the
[Android CI](https://ci.android.com/builds/branches/aosp-master/grid)

For example, you can look into `aosp_cf_aarch64_64_phone` artifacts and
fetch `aosp_cf_aarch64_phone-*.img` + `cvd-host_package.tar.gz` from
there. *cf* here stands for
[Cuttlefish](https://source.android.com/docs/setup/create/cuttlefish),
an Android emulator trying to emulate a real physical device as close as
possible.

For now, though, we will avoid to use the binary images as, actually, we
need to patch the Android sources to have a working image for QEMU.

### Using the source code

Android is composed of [many
repositories](https://android.googlesource.com/). The whole Android tree
currently takes about 250 GB of disk space and can be fetched, depending
on your internet connection, in a couple of hours or overnight. Note
that, although, we\'re going to create aarch64 Android images, it is
perfectly fine to use your x86_64 machine for building these images. In
most cases, will probably be much faster to cross build the images than
build on aarch64 machine (it depends on the aarch64 machine of course).

The official documentation on how to fetch the sources can be found at
[upstream
documentation](https://source.android.com/docs/setup/download/downloading).
As for which branch to select, see
[above](#which-android-to-use "wikilink"). The list of available
branches can be obtained from:

- [the web](https://android.googlesource.com/platform/manifest/+refs)
- the command line: `git ls-remote -h https://android.googlesource.com/platform/manifest.git`
- the [manifest
    repo](https://android.googlesource.com/platform/manifest):
    `git branch -r`

As a quick steps, you can follow the steps below, but if you find any
problem please, refer to the official documentation.

First, enter the following to download the repo binary and make it
executable (runnable):

```sh
mkdir -p  ~/.local/bin/
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.local/bin/repo
chmod a+x ~/.local/bin/repo`
```

Then, given that repo requires you to identify yourself to sync Android,
run the following commands to configure your git identity:

```sh
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

After that, download the Android source tree:

```sh
mkdir ~/aosp && cd ~/aosp
repo init  -u https://android.googlesource.com/platform/manifest -b master
repo sync -j$(nproc)
```

Before starting the build, several packages are needed. You can install
these using the package manager with:

```sh
sudo dnf install \
       @development-tools \
       ImageMagick-devel \
       ImageMagick-c++-devel \
       SDL \
       automake \
       bc \
       bison \
       bzip2 \
       bzip2-libs \
       ccache \
       curl \
       dpkg-dev \
       flex \
       gcc \
       gcc-c++ \
       git \
       gperf \
       libstdc++.i686 \
       libxml2-devel \
       libxslt \
       lz4-libs \
       lzop \
       make \
       maven \
       ncurses-compat-libs \
       openssl \
       openssl-devel \
       patchelf \
       pngcrush \
       python27 \
       python3 \
       python3-mako \
       python3-networkx \
       readline-devel \
       readline-devel.i686 \
       rsync \
       schedtool \
       squashfs-tools \
       syslinux-devel \
       zip \
       zlib-devel \
       zlib-devel.i686
```

### Patching Android sources {#patching_android_sources}

The primary problem is that graphics doesn't work. It's necessary to
apply a cuttlefish-surfaceflinger.patch to [cuttlefish
subrepo](https://android.googlesource.com/device/google/cuttlefish/) and
compile Android in order to make SurfaceFlinger work on QEMU.

```diff
diff --git a/host/libs/vm_manager/qemu_manager.cpp b/host/libs/vm_manager/qemu_manager.cpp`\
index 5538dd89d..e80324484 100644`\
--- a/host/libs/vm_manager/qemu_manager.cpp`\
+++ b/host/libs/vm_manager/qemu_manager.cpp`\
@@ -132,6 +132,7 @@ std::vector``<std::string>`{=html}` QemuManager::ConfigureGraphics(`\
         "androidboot.cpuvulkan.version=" + std::to_string(VK_API_VERSION_1_2),`\
         "androidboot.hardware.gralloc=minigbm",`\
         "androidboot.hardware.hwcomposer=" + instance.hwcomposer(),`\
+        "androidboot.hardware.hwcomposer.display_finder_mode=drm",`\
         "androidboot.hardware.egl=angle",`\
         "androidboot.hardware.vulkan=pastel",`\
         "androidboot.opengles.version=196609"};  // OpenGL ES 3.1`\
@@ -143,6 +144,7 @@ std::vector``<std::string>`{=html}` QemuManager::ConfigureGraphics(`\
       "androidboot.hardware.gralloc=minigbm",`\
       "androidboot.hardware.hwcomposer=ranchu",`\
       "androidboot.hardware.hwcomposer.mode=client",`\
+      "androidboot.hardware.hwcomposer.display_finder_mode=drm",`\
       "androidboot.hardware.egl=mesa",`\
       // No "hardware" Vulkan support, yet`\
       "androidboot.opengles.version=196608"};  // OpenGL ES 3.0`
```

### Compiling Android images {#compiling_android_images}

Compiling Android images has some hardware requirements:

- At least 150 GB of disk space as of this writing, including the space for Android sources.
- The more CPU cores or threads, the better (use `-j` m/make option).
- At least 16 GB RAM, the more the better I think.
- Disk speed matters a lot, best to use a SSD.

Note that, depending on your hardware, the compilation may take
something between a couple of hours or tens of hours.

After compiling Android we will get an Android images and the cvd-host
package. To build an image, the official documentation can be found at
[upstream](https://source.android.com/docs/setup/build/building).

As a quick steps you can follow the below steps. First you need to
decide which target you will want to build. Some of the most common
Android aarch64 targets are:

- `aosp_cf_aarch64_phone-userdebug` (phone images)
- `aosp_auto_aarch64-userdebug` (automotive images)
- `aosp_tv_aarch64-userdebug` (tv images)

Note that running lunch without any target offers all the targets to
select from interactively.

Then, do the usual thing to build the selected target, i.e:

```sh
source build/envsetup.sh
lunch aosp_cf_aarch64_phone-userdebug
make -j$(nproc)`
```

**Note:** The userdebug build type is like user but with root access and debug capability; preferred for debugging.

Finally pack the images with:

```sh
make dist
```

If the compilation succeeds, the images are available in the following
directories, i.e:

```text
out/dist/aosp_cf_aarch64_phone-img-eng.root.zip
out/dist/cvd-host_package.tar.gz
```

### Create cuttlefish-base rpm package {#create_cuttlefish_base_rpm_package}

Before running the Android images, there are some scripts and udev rules
that needs to be installed in the host system. The code for these tools
assume a Debian-based Linux distribution and we had some problems
building it on Fedora. So, we\'re going to do a trick here and use a
podman container to create the cuttlefish-base rpm package for Fedora.

In your aarch64 machine, create a `Containerfile` to build the debian
packages:

```sh
FROM debian:bullseye

RUN set -ex \
    && sed -i -- 's/# deb-src/deb-src/g' /etc/apt/sources.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
               build-essential \
               cdbs \
               devscripts \
               equivs \
               fakeroot \
    && apt-get clean 
    && rm -rf /tmp/* /var/tmp/*
```

Now build the container that builds the Debian packages with:

```sh
podman build -t podman-deb-builder:bullseye -f Containerfile .
podman images
podman run -it <image id>
    apt install config-package-dev curl git golang
    git clone https://github.com/google/android-cuttlefish
    cd android-cuttlefish
    for dir in base frontend; do   cd $dir;   debuild -i -us -uc -b -d;   cd ..; done
    apt install alien
    alien -r *.deb
```

And get and install the package required for cuttlefish:

```sh
podman ps
podman cp <container id>:/android-cuttlefish/cuttlefish-base-0.9.25-2.arm64.rpm .
sudo rpm -i --ignorearch --force cuttlefish-base-0.9.25-2.arm64.rpm
```

Then add your `$USER` to the required groups

```sh
sudo groupadd cvdnetwork
sudo usermod -aG kvm,cvdnetwork,render $USER
```

At this point, you need to reboot to trigger loading additional kernel modules and applying udev rules.

## Running Android

The easiest way to run an Android in QEMU is using `launch_cvd`. Just
run `launch_cvd` with `-vm_manager qemu_cli` command line option. This
will use the QEMU binary provided by your system by default.

First, unpack the Android packages you already created into a directory:

```sh
tar xzf cvd-host_package.tar.gz 
unzip aosp_cf_arm64_phone-img-eng.root.zip
  inflating: vendor_boot.img
  inflating: super.img
```

Finally launch the cuttlefish virtual device with:

```sh
HOME=$PWD ./bin/launch_cvd -vm_manager qemu_cli
```

`launch_cvd` starts some services and prepares the images to boot, so
depending on your hardware, may take something between a couple of
minutes or tens of minutes.

If all works, point your Chrome browser to <https://0.0.0.0:8443> to
interact with the device.

If not, make sure that using the `crosvm` vm manager works before look
into problems with QEMU.
