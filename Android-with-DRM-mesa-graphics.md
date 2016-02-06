Instructions for building Android M using mesa/DRM graphics stack. Currently supported are freedreno on Dragonboard 410c and virtio-gpu on QEMU (x86 KVM and arm64 TCG).

### Install dependent packages
This is for ubuntu and assuming your machine is already setup for building kernel and QEMU.

`sudo apt-get install libgbm-dev libsdl2-dev libgtk-3-dev libgles2-mesa-dev libpixman-1-dev libtool autoconf libepoxy-dev xutils-dev`

### Build virglrenderer

- `git clone git://people.freedesktop.org/~airlied/virglrenderer`
- `./autogen.sh`
- HACK: Remove the line with CODE_COVERAGE_RULES in generated Makefile. Alternatively, install the necessary dependency.
- `make`
- `sudo make install`

### Build QEMU
- Clone mainline deo -- git clone git://git.qemu-project.org/qemu.git
- Optional performance improvement: Apply the patch “target-arm: A64: fix use 12 bit page tables for AArch64 [!UPSTREAM]”
- HACK: Mouse input with GTK 3.16 and OpenGL appears to be broken. Work-around is edit configure script and force ‘gtk_gl=”no”’.
- `./configure --target-list=aarch64-softmmu,x86_64-softmmu --enable-gtk --with-gtkabi=3.0 --enable-kvm`
- `make`

### Build kernel
QEMU:
- Clone android-4.4 branch from http://git.linaro.org/people/rob.herring/linux.git
     git clone http://git.linaro.org/people/rob.herring/linux.git -b android-4.4
- For x86, build with android_defconfig.
  `make defconfig android_defconfig`
- For arm64, build with ranchu_defconfig
  `make defconfig ranchu_defconfig`

Dragonboard 410c:
- Get http://git.linaro.org/landing-teams/working/qualcomm/kernel.git release/qcomlt-4.3
- Build defconfig + distro.config fragment + enable Android drivers
- `make defconfig distro.config; make menuconfig`
- `make`

Build dt.img and boot.img (details [here](https://github.com/96boards/documentation/wiki/Dragonboard-410c-Boot-Image) ):

`skales/mkbootimg --kernel $KBUILD_OUTPUT/arch/arm64/boot/Image --ramdisk $PATH_TO_ANDROID/out/target/product/linaro_arm64/ramdisk.img --output boot-db410c.img --dt dt.img --pagesize 2048 --base 0x80000000 --cmdline 'rw console=ttyMSM0,115200n8'`


### Build Android
The build is AOSP M plus mainline mesa and libdrm and modifications to drm_gralloc and drm_hwcomposer. 

- `repo init -u https://android.googlesource.com/platform/manifest -b android-6.0.0_r1`
- `cd .repo`
- `git clone https://github.com/robherring/android_manifest.git -b android-6.0 local_manifests`
- `cd ..`
- `repo sync -j10`
- `lunch` Select linaro_arm64-userdebug or linaro_x86_64-userdebug
- Copy Adreno firmware files a300_pfp.fw and a300_pm4.fw to device/linaro/generic
- `make -j8`

### Run QEMU
This is the script I use:

```
#!/bin/sh

QEMU_ARCH=$ARCH

if [ "$ARCH" = "arm64" ]; then
	QEMU_ARCH="aarch64"
	QEMU_OPTS="-cpu cortex-a57 -machine type=virt"
	KERNEL_CMDLINE='console=ttyAMA0,38400 debug nosmp drm.debug=0 rootwait'
	KERNEL=/home/rob/proj/git/linux-2.6/.build-arm64/arch/arm64/boot/Image
else
    # can use x86_64 if compiled Linux 64bit
    QEMU_ARCH="x86"
	KERNEL=/home/rob/proj/git/linux-2.6/.build-x86/arch/x86/boot/bzImage
	QEMU_OPTS="-enable-kvm -smp 2"
	KERNEL_CMDLINE='console=ttyS0 debug  drm.debug=0'
fi
ANDROID_IMAGE_PATH=/media/rob/robextdisk/android/aosp/out/target/product/linaro_${ARCH}

if [ ${ANDROID_IMAGE_PATH}/system.img -nt system.raw ]; then
	simg2img ${ANDROID_IMAGE_PATH}/system.img system.raw
	simg2img ${ANDROID_IMAGE_PATH}/cache.img cache.raw
	simg2img ${ANDROID_IMAGE_PATH}/userdata.img userdata.raw
fi

~/qemu/${QEMU_ARCH}-softmmu/qemu-system-${QEMU_ARCH} \
	${QEMU_OPTS} \
	-append "${KERNEL_CMDLINE}" \
	-m 1024 \
	-serial mon:stdio \
	-kernel $KERNEL \
	-initrd ${ANDROID_IMAGE_PATH}/ramdisk.img \
	-drive index=0,if=none,id=system,file=system.raw \
	-device virtio-blk-pci,drive=system \
	-drive index=1,if=none,id=cache,file=cache.raw \
	-device virtio-blk-pci,drive=cache \
	-drive index=2,if=none,id=userdata,file=userdata.raw \
	-device virtio-blk-pci,drive=userdata \
	-netdev user,id=mynet -device virtio-net-pci,netdev=mynet \
	-device virtio-gpu-pci,virgl -display gtk,gl=on \
	-device virtio-mouse-pci -device virtio-keyboard-pci \
	-d guest_errors \
	-nodefaults \
	$*
```
