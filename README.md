# documentation
UBports Ubuntu Touch documentation for the OnePlus 5/5T.

### Table of Contents
* [Install prerequisites for building](#install-prerequisites-for-building)
* [Initializing local repo](#initializing-local-repo)
* [Syncing local repository](#syncing-local-repository)
* [Applying patches](#applying-patches)
* [Building HAL parts](#building-hal-parts)
* [Firmware & TWRP](#firmware-twrp)
* [Installing using Erfan GSI](#installing-using-erfan-gsi)
* [Installing Ubuntu Touch manually](#installing-ubuntu-touch-manually)
* [Helpful tips for Ubuntu Touch](#helpful-tips-for-ubuntu-touch)

## Install prerequisites for building

### Arch based systems
1. Enable [multilib repositories](https://wiki.archlinux.org/index.php/Official_repositories#multilib) & run:
```
sudo pacman -Syyu --needed base-devel git
cd /tmp/
git clone https://aur.archlinux.org/halium-devel.git
cd halium-devel/
makepkg -si --needed
```

### Debian based systems
```
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install git gnupg flex bison gperf build-essential \
  zip bzr curl libc6-dev libncurses5-dev:i386 x11proto-core-dev \
  libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-glx:i386 \
  libgl1-mesa-dev g++-multilib mingw-w64-i686-dev tofrodos \
  python-markdown libxml2-utils xsltproc zlib1g-dev:i386 schedtool \
  repo liblz4-tool bc lzop imagemagick libncurses5 rsync libssl-dev
```

## Initializing local repo
```
mkdir ~/Halium/ && cd ~/Halium/
repo init -u https://github.com/Halium/android -b halium-9.0 --depth=1
git clone https://github.com/ubports-oneplus5/local_manifests -b halium-9.0 .repo/local_manifests/
```

## Syncing local repository
```
cd ~/Halium/
repo sync --no-clone-bundle --no-tags -c -j`nproc`
```

## Applying patches
```
hybris-patches/apply-patches.sh --mb
```
**NOTE:** This will fail if you've already applied them; to revert the patches **and all other local changes** run `repo sync -l`

## Building HAL parts
Type the following to initilize the current environment for building:
```
. build/envsetup.sh
breakfast cheeseburger         # use "dumpling" for 5T builds
export LANG=C LC_ALL=POSIX
```
To produce the required `halium-boot.img` kernel image for Ubuntu Touch, execute:
```
mka halium-boot
```
**NOTE:** If you've decided to install manually (without Erfan GSI) you also need to `mka systemimage`!

Assuming your device is in fastboot mode you can now flash this image by simply using:
```
fastboot flash boot out/target/product/cheeseburger/halium-boot.img
```

## Firmware & TWRP<a name="firmware-twrp"></a>
You may need to install the latest O<sub>2</sub>OS 9.0.11 firmware ([OP5](https://sourceforge.net/projects/crdroid/files/cheeseburger/6.x/firmware/firmware_9.0.11_oneplus5.zip/download) / [OP5T](https://sourceforge.net/projects/crdroid/files/dumpling/6.x/firmware/firmware_9.0.11_oneplus5T.zip/download)) on your device (if you aren't already on it) & flash a [TWRP >=3.3.x image](https://github.com/engstk/android_device_oneplus_cheeseburger/releases) on your recovery partition & reboot to TWRP again to proceed.

**NOTE:** Full backup of **all data** (as per usual) & format of `userdata` partition is also highly recommended.

## Installing using Erfan GSI

1. Download the latest GSI zip from [here](https://t.me/ErfanGSI)
2. Ensure your `/vendor` (after mounting) is populated with content from an Android 9 ROM (LineageOS or otherwise)
3. Flash the GSI zip file
4. Flash the `halium-boot.img` from before to your boot partition (if you didn't yet):
```
adb push ~/Halium/out/target/product/cheeseburger/halium-boot.img /tmp/
adb shell "dd if=/tmp/halium-boot.img of=/dev/block/bootdevice/by-name/boot"
```

## Installing Ubuntu Touch manually
**TODO:** Explain setting up rootfs using [rootfs-builder-debos-android9](https://github.com/ubports-on-fxtec-pro1/rootfs-builder-debos-android9
) & deploy along `system.img` using `halium-install`.
## Helpful tips for Ubuntu Touch
On the Erfan GSI the default password is `phablet`.

### SSH from host via USB
```
ssh phablet@10.15.19.82
```

### Gain root user access
```
sudo -s
```

### RootFS as read-write
```
sudo mount / -o remount,rw
```

### Running commands inside the Android HAL LXC container
```
sudo lxc-attach -n android -- /system/bin/logcat
```
**NOTE:** If this fails and you're using an `armhf` rootfs (like on Erfan's GSI) run the following command instead:
```
sudo env LD_LIBRARY_PATH=/system/lib64:/vendor/lib64 lxc-attach -n android -- /system/bin/logcat
```

### Networking on-device via USB
Host:
```
sudo sysctl net.ipv4.ip_forward=1
sudo iptables -P FORWARD ACCEPT
sudo iptables -A POSTROUTING -t nat -j MASQUERADE -s 10.15.19.0/24
```
Device:
```
sudo ip route add default via 10.15.19.100
echo nameserver 1.1.1.1 | sudo tee /etc/resolv.conf > /dev/null
```

### Expand the GSI RootFS size
This is an example on how to expand the rootfs image to 16 GiB. Begin by booting your device back to TWRP, connect the USB cable and run the command below from your host:
```
adb shell "resize2fs /data/rootfs.img 16G"
```

### Anbox setup
Start by enabling USB networking using above instructions and then run the following:
```
sudo mount -o rw,remount /
sudo apt update && sudo apt install -y anbox-ubuntu-touch android-tools-adb
mkdir ~/anbox-data
wget http://cdimage.ubports.com/anbox-images/android-armhf-64binder.img -O ~/anbox-data/android.img
touch ~/anbox-data/.enable
sudo chmod -R o+wrx /home/phablet/anbox-data/data
sudo start -q anbox-container
```
After these just reboot twice, waiting about two minutes before doing so again :)

### Interacting with the RootFS from TWRP
```
mount /data/rootfs.img /mnt
chroot /mnt /bin/bash
. /etc/environment
<execute what you need to>
exit
umount /mnt && sync
```
