---
title: "Debian on Devterm R-01"
summary: ""
authors: ["thomas"]
tags: ["linux", "debian", "riscv"]
categories: []
date: 2022-06-08
---
I recently found out about [DevTerm Kit R-01](https://www.clockworkpi.com/product-page/devterm-kit-r01) via [Bryan Lunduke](https://lunduke.substack.com/p/the-first-risc-v-portable-computer). I've been interested in RISC V for ages and so far have resisted buying any dev boards as they would just sit in boxes. However because this is an entire portable computer it's a great way to play, there is hope it can become a useful piece of kit, so I immediately decided to get one.

The [official images](http://dl.clockworkpi.com/) for the R-01 is based on [Ubuntu 22.04](http://dl.clockworkpi.com/DevTerm_R01_v0.1.img.bz2). This is great to build on something solid, however really wanted something based on Debian, with the intention of getting as much of this upstream and packaged as possible. This is not possible to perform in one step. There is a [wiki page](https://github.com/clockworkpi/DevTerm/wiki/Create-DevTerm-R01-OS-image-from-scratch) that documents some of this, however it self admits that it is more of a guidance that exact instructions. Clockwork have done a great job of building a working device from the ground up, eg hardware, case, schematics, pcb, firmware, and software (for multiple archs). Just the process of creating a working image from scratch is an epic piece of work, so clockworkpi's "principal of minimal changes" is really great to see. However there are a few sharp edges in the official image, I expect mostly down to just the speed of implementation. Some things are not implemented very "Debiany", eg it's nicer to configure /etc/default/u-boot correctly and removed the non working kernels work so that u-boot-update does not leave the system unbreakable.

This is the result of that I have found out and a set of instructions that can be followed to create a Debian image that works. I expect it to be a working document that I will use going forward if I need to reinstall my Devterm-01. It's also fair to say this is slightly opinionated to my style of Debian systems.

## Resources
 * https://fedoraproject.org/wiki/Architectures/RISC-V/Allwinner
 * https://open.allwinnertech.com/
 * https://andreas.welcomes-you.com/boot-sw-debian-risc-v-lichee-rv/
 * https://linux-sunxi.org/Allwinner_Nezha

## Prerequisites and prepare environment
Obviously this needs Linux to run, I run and use Debian Testing for this. I expect Debian based distros might work with a little work, other distros will probably need more work. Step one is to install some prerequisite packages, create a work dir and rootfs dir eg /home/thomas/devterm and /home/thomas/devterm/rootfs. About 15G of disk space is required.

```
sudo apt-get install binfmt-support binutils-riscv64-linux-gnu debian-ports-archive-keyring debootstrap gcc-11-riscv64-linux-gnu qemu-user-static qemu-user-static fdisk swig uuid
mkdir -p ~/devterm/rootfs
cd ~/devterm
```

## Create new disk image file, partition and make filesystems
Use dd to create a new 4G disk image. The bootloader is in the first 64M of the disk so the /boot does not start at the beginning, then one big remaining / partition.

```
dd if=/dev/zero bs=1M count=4096 of=disk.img
echo -en "label: dos\n disk.img : start=65536, size=204800, type=83\n disk.img : start=270336, size=8118272, type=83\n" | /sbin/sfdisk disk.img

d=$(sudo losetup --show -f -P disk.img)

sudo mkfs.ext4 ${d}p1
sudo mkfs.ext4 ${d}p2

mkdir -p disk.img.d
sudo mount ${d}p2 disk.img.d
sudo mkdir -p disk.img.d/boot
sudo mount ${d}p1 disk.img.d/boot
sudo umount disk.img.d/boot disk.img.d
sudo losetup -d $d

```

## Compile bootloader
The bootloader consists of three suites of software. The first is "sun20i_d1_spl", perhaps spl means something as the Chip is a D1 and the platform(?) is sun20i from Allwinner. It needs to be compiled with the cross compiler and gets installed into the mbr gap, 8k into the block device, eg between the partition table and first partition. Which means that GPT partitions can't be used. I think the SoC reads the disk and starts running this code and it does print some messages to the serial debug console. It then jumps to OenSBI/u-boot, which also needs to be compiled with the cross compiler. Both OpenSBI and u-boot need to be compiled separately, afterwards a u-boot utility is used to combine both into a single image that gets installed 16M into the block device. When excicution passed from the SPL to OpenSBI it shows a banner and later shows some u-boot output. Unfortunately I've not been able to get the debug uart working well, with a very short USB cable connected to a powered hub it seems to show output but does not allow any input to the uart and which interrupting and inspecting u-boot is not possible. U-Boot then reads the first partition and has the smarts to read ext4 file systems, it loads /extlinux/extlinux.conf. This is a syslinux style bootloaded file that u-boot uses to load and then run the linux kernel with the right command line parameters. There is no initrd/initramfs (initial ramdisk), the root filesystem it just mounted

```
git clone https://github.com/smaeul/sun20i_d1_spl
(cd sun20i_d1_spl; make CROSS_COMPILE=riscv64-linux-gnu- p=sun20iw1p1 mmc)

git clone https://github.com/tekkamanninja/opensbi -b allwinner_d1
(cd opensbi; CROSS_COMPILE=riscv64-linux-gnu- PLATFORM=generic FW_PIC=y BUILD_INFO=y make)

git clone https://github.com/tekkamanninja/u-boot -b allwinner_d1
(cd u-boot; make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv nezha_defconfig)
echo "CONFIG_MMC_BROKEN_CD=y" >> u-boot/.config
(cd u-boot; make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv oldconfig)
(cd u-boot; make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv u-boot.bin u-boot.dtb)

cat << EOF > u-boot/toc1.cfg
[opensbi]
file = fw_dynamic.bin
addr = 0x40000000
[dtb]
file = u-boot.dtb
addr = 0x44000000
[u-boot]
file = u-boot.bin
addr = 0x4a000000
EOF

cp opensbi/build/platform/generic/firmware/fw_dynamic.bin u-boot/
(cd u-boot; tools/mkimage -T sunxi_toc1 -d toc1.cfg u-boot.toc1)

sudo dd if=sun20i_d1_spl/nboot/boot0_sdcard_sun20iw1p1.bin of=disk.img bs=512 seek=16
sudo dd if=u-boot/u-boot.toc1 of=disk.img bs=512 seek=32800
```

## Compile Linux
The Linux kernel is somewhat harder, it seems to be some sort of "Android Common Kernels" as the README mentions this. The wiki page gives https://github.com/cuu/last_linux-5.4.git, which has no git history. So one assumes it's just been put together in a rather ad-hoc manor. Searching about with references to "tina-d1-h", the tree might be a combination of https://gitlab.com/weidongshan/tina-d1-h/-/tree/main/lichee/linux-5.4 with patches from clockwork here https://github.com/clockworkpi/DevTerm/tree/main/Code/patch/d1. Also because (I assume) of the age of this release (5.4 is an LTS releases in 2019-11-24 supported will 01-12-2025) as it uses assembly opcodes that GCC-11 errors on where as does compile with an older GCC-8.


Get the GCC-8 cross compiler.
```
#https://github.com/cuu/toolchain-thead-glibc -> mega.nz download -> riscv64-glibc-gcc-thead_20200702.tar.gz
tar xvf riscv64-glibc-gcc-thead_20200702.tar.gz
export OLDPATH=$PATH
export PATH=`pwd`/riscv64-glibc-gcc-thead_20200702/bin:$PATH
```

Grab the kernel config from the clockworkpi image.
```
wget -c ttp://dl.clockworkpi.com/DevTerm_R01_v0.1.img.bz2
bunzip2 DevTerm_R01_v0.1.img.bz2
u=$(sudo losetup --show -f -P DevTerm_R01_v0.1.img)
mkdir ubuntu
sudo mount ${d}p4 ubuntu

sudo cp ubuntu/boot/config-5.4.61 .
sudo umount ubuntu
sudo losetup -d $u
```

Clone the kernel source, update a few files which seem like obvious git commit errors, copy in the config, run oldconfig and compile linux. Then install the kernel image, modules, config and dtd into a dir for installation later.
```
git clone https://github.com/cuu/last_linux-5.4.git
sed -i '159d' last_linux-5.4/drivers/base/Kconfig
sed -i '29d' last_linux-5.4/drivers/ntb/Kconfig
sed -i '27d' last_linux-5.4/drivers/base/Makefile
sed -i 's^test/^^' last_linux-5.4/drivers/ntb/Makefile

cp config-5.4.61 last_linux-5.4/.config
sed -i 's/CONFIG_VECTOR=y/CONFIG_VECTOR=n/' last_linux-5.4/.config
(cd last_linux-5.4/; make CROSS_COMPILE=riscv64-unknown-linux-gnu- ARCH=riscv oldconfig)
(cd last_linux-5.4/; make CROSS_COMPILE=riscv64-unknown-linux-gnu- ARCH=riscv)
mkdir -p last_linux-5.4/rootfs/boot
(cd last_linux-5.4/; make CROSS_COMPILE=riscv64-unknown-linux-gnu- ARCH=riscv INSTALL_MOD_PATH=rootfs modules_install)
(cd last_linux-5.4/; make CROSS_COMPILE=riscv64-unknown-linux-gnu- ARCH=riscv INSTALL_PATH=rootfs/boot zinstall)
cp last_linux-5.4/.config last_linux-5.4/rootfs/boot/config-5.4.61
cp last_linux-5.4/arch/riscv/boot/dts/sunxi/board.dtb last_linux-5.4/rootfs/boot

export PATH=$OLDPATH
```
## Create root file system, install packages and configure
Just use debootstrap to create a clean install. Set a few debconfig values, install locales and then install a bunch of useful packages.
```
sudo debootstrap --arch=riscv64 --keyring /usr/share/keyrings/debian-ports-archive-keyring.gpg --include=debian-ports-archive-keyring unstable /thomas/home/devterm/rootfs http://deb.debian.org/debian-ports

sudo chroot rootfs apt-get update

echo -e "locales\tlocales/default_environment_locale\tselect\ten_GB.UTF-8" | sudo chroot rootfs debconf-set-selections
echo -e "locales\tlocales/locales_to_be_generated\tmultiselect\ten_GB.UTF-8 UTF-8" | sudo chroot rootfs debconf-set-selections
sudo chroot rootfs apt-get -y install locales

echo -e "ferm\tferm/enable\tboolean\ttrue" | sudo chroot rootfs debconf-set-selections
echo -e "keyboard-configuration\tkeyboard-configuration/model\tselect\tGeneric 105-key PC" | sudo chroot rootfs debconf-set-selections
echo -e "keyboard-configuration\tkeyboard-configuration/variant\tselect\tEnglish (UK)" | sudo chroot rootfs debconf-set-selections

sudo chroot rootfs apt-get -y install aptitude apt-utils bash-completion bc bind9-host build-essential busybox curl debconf-utils dstat elinks ferm file git gkrellm iw mesa-utils mtr netcat-traditional network-manager network-manager-gnome nmap psmisc pv qutebrowser rsync screen ssh sudo systemd-timesyncd tcpdump telnet twm u-boot-menu unattended-upgrades vim-gtk3 wireless-regdb wireless-tools wpasupplicant x11-apps xdm xfonts-100dpi xinit xserver-xorg-video-fbdev xterm
sudo chroot rootfs apt-get clean
```

Set a hostname and create a fstab.
```
echo devterm | sudo tee rootfs/etc/hostname

cat <<EOF | sudo tee rootfs/etc/fstab
/dev/mmcblk0p2 /     ext4    defaults,noatime 0 0
/dev/mmcblk0p1 /boot ext4    defaults,noatime 0 0
EOF
```

## Install Linux kernel
Install Linux kernel, modules, and other files. Update u-boot configuration and run u-boot-update.
```
sudo mkdir -p rootfs/lib/modules/5.4.61+
sudo rsync -a last_linux-5.4/rootfs/lib/modules/5.4.61+/. rootfs/lib/modules/5.4.61+/.
sudo cp -a last_linux-5.4/rootfs/boot/* rootfs/boot
sudo chown -R root:root rootfs/boot rootfs/lib/modules/

cat <<EOF | sudo tee -a rootfs/etc/default/u-boot
U_BOOT_ALTERNATIVES="default"
U_BOOT_TIMEOUT="1"
U_BOOT_PARAMETERS="earlyprintk=sunxi-uart,0x02500000 clk_ignore_unused initcall_debug=0 console=ttyS0,115200 console=tty0 cma=8M fbcon=rotate:1"
U_BOOT_FDT="board.dtb"
U_BOOT_FDT_DIR="noexist"
EOF
sudo chroot rootfs u-boot-update
sudo sed -i 's^/boot^^' rootfs/boot/extlinux/extlinux.conf
echo "        fdt /board.dtb" | sudo tee -a rootfs/boot/extlinux/extlinux.conf
```

## Install Firmware
The Wifi requires some firmware, I was not able to find this, I assume it's behind the allwinner site. I got the 3 files from the clockworkpi image.
```
wget -c http://dl.clockworkpi.com/DevTerm_R01_v0.1.img.bz2
bunzip2 DevTerm_R01_v0.1.img.bz2
u=$(sudo losetup --show -f -P DevTerm_R01_v0.1.img)
mkdir ubuntu
sudo mount ${d}p4 ubuntu

sudo mkdir -p rootfs/lib/firmware/brcm
sudo cp ubuntu/lib/firmware/brcm/brcmfmac43456-sdio.* rootfs/lib/firmware/brcm

sudo umount ubuntu
sudo losetup -d $u
```

## Configure Xorg
Xorg needs some config to rotate screen and set res
```
cat <<EOF | sudo tee rootfs/etc/X11/xorg.conf.d/10-d1.conf
Section "Device" 
        Identifier "FBDEV" 
        Driver "fbdev" 
        Option "fbdev" "/dev/fb0" 
        Option "Rotate" "cw" 
        Option "SwapbuffersWait" "true" 
EndSection 
 
Section "Screen" 
        Identifier "Screen0" 
        Device "FBDEV" 
        DefaultDepth 24 
        Subsection "Display" 
                Depth 24 
                Modes "1280x480" "480x1280" 
        EndSubsection 
EndSection
EOF

```

## Disable some services
To speed up booting disable tome services: apt-daily, apt-daily-upgrade, ModemManager, avahi-daemon, NetworkManager-wait-online:
```
rm rootfs/etc/systemd/system/timers.target.wants/apt-daily.timer
rm rootfs/etc/systemd/system/timers.target.wants/apt-daily-upgrade.timer
rm rootfs/etc/systemd/system/dbus-org.freedesktop.ModemManager1.service
rm rootfs/etc/systemd/system/multi-user.target.wants/ModemManager.service
rm rootfs/etc/systemd/system/dbus-org.freedesktop.Avahi.service
rm rootfs/etc/systemd/system/sockets.target.wants/avahi-daemon.socket
rm rootfs/etc/systemd/system/multi-user.target.wants/avahi-daemon.service
rm rootfs/etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service
```

## Create user
Create a user, set the password and add to the sudo group.
```
sudo chroot rootfs adduser --gecos thomas --add_extra_groups --disabled-password thomas
pass=$(mkpasswd -m sha-512 CorrectHorseBatteryStaple)
sudo chroot rootfs usermod -a -G sudo -p "$pass" thomas
```

## Install clockworkpi repos
The official image contains some extra software for audio, backlight, fan, keyboard, printer and wiringpi. The below are the steps to install them, however it does not work as one of the debian packages depends on python2.7. However it's not clear how these package were created. For example devterm-keyboard-firmware installs /usr/local/bin/devterm_keyboard_firmware_flash.sh which is a self extracting sh, extracting that clones https://github.com/cuu/stm32duino_bootloader_upload.git along with devterm_keyboard.ino.bin. The firmware then seems to come from https://github.com/cuu/devterm_keyboard.

```
curl https://raw.githubusercontent.com/clockworkpi/apt/main/debian/KEY.gpg | sudo tee rootfs/etc/apt/trusted.gpg.d/clockworkpi.asc
echo "deb https://raw.githubusercontent.com/clockworkpi/apt/main/debian/ stable main" | sudo tee rootfs/etc/apt/sources.list.d/clockworkpi.list  
sudo chroot rootfs apt-get update
sudo chroot rootfs apt-get install devterm-audio-patch devterm-backlight-cpi devterm-fan-temp-daemon-rpi devterm-keyboard-firmware devterm-thermal-printer devterm-thermal-printer-cups devterm-wiringpi-cpi
```

## Official image autologin, and home dir files
The official image implements auto login with a getty systemd override. It then has a .bash_profile to startx and an .xinit to start gkrelm and twm. There is a copy of the home dir and screenshots of the default twm theme here: https://github.com/clockworkpi/DevTerm/tree/main/Code/R01. This is how the getty systemd override, I have not implement this, but am including as I think it's quite a nice solution.
```
sudo mkdir -p rootfs/etc/systemd/system/getty@tty1.service.d
cat <<EOF | sudo tee rootfs/etc/systemd/system/getty@tty1.service.d/override.conf
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin thomas %I $TERM
EOF
```

I chose not to do this and just installed xdm and expect to play with different window managers going forward.

## Copy rootfs to disk image and sd card
Then the disk image needs to be mounted, the root filesystem copied into it, unmounted, then copied to an sd card and then the filesystem increased to cover the rest of the sd card.
```
d=$(sudo losetup --show -f -P disk.img)
sudo mount ${d}p2 disk.img.d
sudo mount ${d}p1 disk.img.d/boot
sudo rsync -vax --delete rootfs/. disk.img.d/.
sudo umount disk.img.d/boot disk.img.d
sudo losetup -d $d
pv disk.img | sudo dd bs=1M of=/dev/sdX
echo "resizepart 2 100%" | sudo parted /dev/sdX
sudo partprobe
sudo fsck -C -t ext4 -f -y /dev/sdX2
sudo resize2fs /dev/sdX2
```

## update sd card
If updates are needed directly from the rootfs directory to the sd card without rewriting entire sd card
```
sudo umount /dev/sdX1 /dev/sdX2; sudo mount /dev/sdX2 /mnt; sudo mount /dev/sdX1 /mnt/boot
sudo rsync -vaxP --delete rootfs/. /mnt/.
sudo umount /mnt/boot /mnt
```

## WiFi
To configure the wifi either use nmtui of nmcli:
```
sudo nmcli dev wifi connect "MyWifi" password "my-password"
```

## Jobs

 * How to package SPL
 * How to package OpenSBI/u-boot
 * Find what patches were applied are needed in mainline Linux
 * Package Linux
 * Repackage clockworkpi debs

