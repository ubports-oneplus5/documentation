# documentation
UBports Ubuntu Touch documentation for the OnePlus 5.

### Table of Contents
* [Install prerequisites for building](#install-prerequisites-for-building)
* [Initializing local repo](#initializing-local-repo)
* [Syncing local repository](#syncing-local-repository)
* [Building HAL parts](#building-hal-parts)
* [Downgrading firmware & TWRP](#downgrading-firmware-twrp)
* [Installing Ubuntu Touch](#installing-ubuntu-touch)
* [Miscellaneous steps](#miscellaneous-steps)
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
2. Setup a [Python 2 virtual environment](https://wiki.archlinux.org/index.php/Python/Virtual_environment#virtualenvwrapper):
```
sudo pacman -S --needed python-virtualenvwrapper
export WORKON_HOME="~/.virtualenvs"
. /usr/bin/virtualenvwrapper.sh
mkdir $WORKON_HOME
mkvirtualenv -p /usr/bin/python2.7 python2
```
**NOTE:** Remember to use `workon python2` in the future to enter this Python 2 virtualenv and `deactivate` to leave it.

### Debian based systems
```
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install git gnupg flex bison gperf build-essential \
  zip bzr curl libc6-dev libncurses5-dev:i386 x11proto-core-dev \
  libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-glx:i386 \
  libgl1-mesa-dev g++-multilib mingw-w64-i686-dev tofrodos \
  python-markdown libxml2-utils xsltproc zlib1g-dev:i386 schedtool \
  repo liblz4-tool bc lzop imagemagick libncurses5 rsync
```

## Initializing local repo
```
mkdir ~/Halium/ && cd ~/Halium/
repo init -u https://github.com/Halium/android.git -b halium-7.1 --depth 1
git clone https://gitlab.com/JBBgameich/halium-install/ halium/scripts/ --depth 1
```

## Syncing local repository
```
cd ~/Halium/
repo sync --no-clone-bundle --no-tags -c -j`nproc`
curl https://git.io/JeNka -Lo halium/devices/manifests/oneplus_cheeseburger.xml
halium/devices/setup cheeseburger
```

## Building HAL parts
Type the following to initilize the current environment for building:
```
. build/envsetup.sh
breakfast cheeseburger
export LANG=C LC_ALL=POSIX
```
To produce the required `halium-boot.img` & `system.img` image files for Ubuntu Touch, execute:
```
mka halium-boot systemimage
```

## Downgrading firmware & TWRP<a name="downgrading-firmware-twrp"></a>
If needed you'll want to downgrade to [O<sub>2</sub>OS 4.5.15 pre-treble firmware](http://download940.mediafire.com/eqz30j0gbc0g/df2wbe8ix0xfiw9/OnePlus+5+OxygenOS+4.5.15+Firmware+%2B+Radio.zip) on your device & flash a [TWRP <3.2.x image](https://forum.xda-developers.com/devdb/project/dl/?id=27442) on your recovery partition & reboot to TWRP again to proceed.

**NOTE:** Full backup of **all data** (as per usual) & format of `userdata` partition is also highly recommended.

## Installing Ubuntu Touch
Copy the URL link to your desired Ubuntu Touch RootFS from below:
* [Edge arm64 rootfs](https://ci.ubports.com/job/xenial-hybris-edge-rootfs-arm64/lastSuccessfulBuild/artifact/out/ubuntu-touch-hybris-xenial-edge-arm64-rootfs.tar.gz) ([Jenkins CI site](https://ci.ubports.com/job/xenial-hybris-edge-rootfs-arm64/))
* [Edge armhf rootfs](https://ci.ubports.com/job/xenial-hybris-edge-rootfs-armhf/lastSuccessfulBuild/artifact/out/ubuntu-touch-hybris-xenial-edge-armhf-rootfs.tar.gz) ([Jenkins CI site](https://ci.ubports.com/job/xenial-hybris-edge-rootfs-armhf/))
* [Xenial arm64 rootfs](https://ci.ubports.com/job/xenial-rootfs-arm64/lastSuccessfulBuild/artifact/out/ubports-touch.rootfs-xenial-arm64.tar.gz) ([Jenkins CI site](https://ci.ubports.com/job/xenial-rootfs-arm64/))
* [Xenial armhf rootfs](https://ci.ubports.com/job/xenial-rootfs-armhf/lastSuccessfulBuild/artifact/out/ubports-touch.rootfs-xenial-armhf.tar.gz) ([Jenkins CI site](https://ci.ubports.com/job/xenial-rootfs-armhf/))

Next let's deploy the Ubuntu Touch `rootfs` & built `system.img` using [@JBBgameich](https://gitlab.com/JBBgameich)'s [`halium-install`](https://gitlab.com/JBBgameich/halium-install) scripts:
```
URL="<insert rootfs url here>"
ROOTFS="$HOME/Halium/ubuntu-touch-rootfs.tar.gz"
SYSIMG="$HOME/Halium/out/target/product/cheeseburger/system.img"
curl $URL -o $ROOTFS
halium/scripts/halium-install -p ut $ROOTFS $SYSIMG
```
Finally you'll need the `halium-boot` image flashed on your `boot` partition:
```
adb push ~/Halium/out/target/product/cheeseburger/halium-boot.img /tmp/
adb shell "dd if=/tmp/halium-boot.img of=/dev/block/bootdevice/by-name/boot"
```

## Miscellaneous steps
Currently to get SSH we need a patched `hybris-usb` DEB package with ConfigFS support; deploy it on the rootfs like so:
```
DEB_PKG="https://ci.ubports.com/job/ubports/job/hybris-usb-packaging/job/xenial_-_edge_-_android8/lastSuccessfulBuild/artifact/hybris-usb_0.2ubports1+0~20191013221700.1~1.gbpbfa9ad_all.deb"
curl $DEB_PKG -o $HOME/Halium/hybris-usb.deb

adb shell "mount /data/rootfs.img /mnt" && adb push $HOME/Halium/hybris-usb.deb /mnt/

adb shell "chroot /mnt /bin/bash" <<EOF
  . /etc/environment
  dpkg -i hybris-usb.deb
  rm -f hybris-usb.deb
EOF

adb shell "umount /mnt; sync; reboot"
```
Now your device should reboot to Ubuntu Touch :)

## Helpful tips for Ubuntu Touch

### SSH from host via USB
```
ssh phablet@10.15.19.82
```

### Gain root user access
```
sudo -i
```

### RootFS as read-write
```
sudo mount / -o remount,rw
```

### Interacting with the RootFS from TWRP
```
mount /data/rootfs.img /mnt
chroot /mnt /bin/bash
. /etc/environment
<execute what you need to>
exit
umount /mnt && sync
```
