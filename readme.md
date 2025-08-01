# Manifest of the Advantech Linux TSU Yocto project
The goal of this project is to release opensource Linux source code that runs on Advantech Hardware

## Download the Yocto Linux for ECU-150-12A1 & ECU-150-12A1U and setup environment variables
For downloading the source code and setting up the environment, follow the instructions below:
```console
foo@bar:~/yocto$ repo init -u https://github.com/saurontech/Advantech-LinuxTSU-manifest.git -b main -m ecu-150a1-6.6.52-2.2.0.xml
foo@bar:~/yocto$ repo sync
foo@bar:~/yocto$ MACHINE=imx8mq-ecu150a1 DISTRO=fsl-imx-xwayland source ./imx-setup-release.sh -b build
foo@bar:~/yocto/build$ bitbake-layers add-layer ../sources/meta-ecu-150a1/
```
After the commands, not only was the Yocto project for ECU-150-A1 downloaded, the operating console were also setup to operate bitbaker.
Please also notice, that after the commands, your current position has been changed to the build directory!
If, in the future, to operate bitbake from another console; in that spacific console, use the command:
```console
foo@bar:~/yocto$ source ./setup-environment build
foo@bar:~/yocto/build$
```
## Build Yocto
>[!NOTE]
> On Ubuntu 24.04 hosts, AppAromor settings needs to be adjusted before building Yocto.
>```console
> foo@bar:~/$ sudo sh -c 'echo 0 > /proc/sys/kernel/apparmor_restrict_unprivileged_userns'
>```

>[!TIP]
> Edit __local.conf__ based on your host resource.  
> Building Yocto, with the default configure, is very memory consuming. At least 32 GBytes of RAM will be needed.  
> With insufficient RAM, the building process will fail.
> Therefore, limiting the maximum parallel processes allowed, migth be a good idea.
> One may do so by adding the following parameters to **"build/config/local.conf"**
> ```sh
> PARALLEL_MAKE = "-j 2"
> BB_NUMBER_THREADS = "2"
> ```

In a console that has been setup properly, use the following command to build Linux and the Yocto rootfs
```console
foo@bar:~/yocto/build$ bitbake core-image-minimal

```
- The __Linux kernel__ will be located at : __./build/tmp/deploy/images/imx8mq-ecu150a1/Image__
- The __dtb__ will be located at: __./build/tmp/deploy/images/imx8mq-ecu150a1/fsl-imx8mq-ecu150a1.dtb__
- The __rootfs__ will be located at: __./build/tmp/deploy/images/imx8mq-ecu150a1/core-image-minimal-imx8mq-ecu150a1.rootfs.tar.gz__

To build only the Linux kernel, use the following command instead:
```console
foo@bar:~/yocto/build$ bitbake linux-imx
```
## Deploy the Yocto Image
The easiest way to delpy the yocto image is to dump the wic file to a SD card.  
To create a bootable SD card, use the following commands:
> [!CAUTION] 
>  __BEWARE!!!__ The following example assumes that the __SD card__ was located as **/dev/sdb**, change the location accordingly. otherwise, /dev/sdb would be ruined.

> [!NOTE]
>  1. The wic image could be found at ./build/tmp/deploy/images/imx7mq-ecu150a1/core-image-minimal-imx8mq-ecu150a1.rootfs-*.wic.gz
>  2. To deploy the image to the on board EMMC, copy the wic.gz file to the __root/__ partition on the SD, boot from SD and follow the same commands with the following parameters swapped out:
>        1. __/dev/sdb__ swapped to __/dev/mmcblk0__
>        2. __/dev/sdb2__ swapped to __/dev/mmcblk0p2__
>  3. Use the on board hardware switch __SW2__ to select between the boot devices.

```console
foo@bar:~/yocto/build/tmp/deploy/images/imx8mq-ecu150a1/$ gunzip -c ./core-image-minimal-imx8mq-ecu150a1.rootfs-*.wic.gz | sudo dd of=/dev/sdb bs=1M iflag=fullblock oflag=direct conv=fsync
1181+1 records in
1181+1 records out
1238877184 bytes (1.2 GB, 1.2 GiB) copied, 73.1945 s, 16.9 MB/s
foo@bar:~/yocto/build/tmp/deploy/images/imx8mq-ecu150a1$ sudo parted -s -a opt /dev/sdb "resizepart 2 100%"
foo@bar:~/yocto/build/tmp/deploy/images/imx8mq-ecu150a1$ sudo e2fsck -f /dev/sdb2 
e2fsck 1.47.0 (5-Feb-2023)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
root: 16790/107296 files (0.1% non-contiguous), 126901/214396 blocks
foo@bar:~/yocto/build/tmp/deploy/images/imx8mq-ecu150a1$ sudo resize2fs /dev/sdb2
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/sdb2 to 7545600 (4k) blocks.
The filesystem on /dev/sdb2 is now 7545600 (4k) blocks long.
```
## Programmable LEDs and Watchdog Timer
The Programmable LEDs and Watch Dog Timer are accessed via GPIO; therefore, it should be accessed via the standard libGPIOD API.
```console
root@imx8mq-ecu150a1:~/$ gpioinfo
...
gpiochip3 - 32 lines:
        ...
        line   11:    "WDT_EN"       unused   input  active-high
        ...
        line   20:       "WDT"       unused   input  active-high
gpiochip4 - 32 lines:
        line   0:      unnamed       unused   input  active-high
        line   1:      unnamed       unused   input  active-high
        line   2:      unnamed       unused   input  active-high
        line   3:        "PL1"       unused   input  active-high
        line   4:        "PL2"       unused   input  active-high
        line   5:        "PL3"       unused   input  active-high
...
root@imx8mq-ecu150a1:~/$ gpioset -t0 PL1=1	//switch on Programmable LED 1
root@imx8mq-ecu150a1:~/$ gpioset -t0 PL1=0        //switch off Programmable LED 1
```
The following bash-script gives an idea on how the watchdog timer could be enabled and updated.
Run the script and terminate it with control-c and the WDT will reset the system.
```sh
#!/bin/bash
i=0
# clear the wdt a few times before enabling it.
# It has left alone for too long,
# enabling it right away will reset the system immediately.
for j in {0..5}
do
	gpioset -t0 WDT=$i
	if [ $i -eq 0 ]; then 
		i=1
	else
		i=0
	fi
	sleep 0.3
done
# enable the wdt
gpioset -t0 WDT_EN=0
while true
do
	gpioset -t0 WDT=$i
	if [ $i -eq 0 ]; then 
		i=1
	else
		i=0
	fi
	sleep 0.3
done
```

## Create SDK for Yocto
```console
foo@bar:~/yocto/build$ bitbake -c populate_sdk core-image-minimal
foo@bar:~/yocto/build$ sh ./tmp/deploy/sdk/fsl-imx-xwayland-glibc-x86_64-core-image-minimal-armv8a-imx8mq-ecu150a1-toolchain-6.6-scarthgap.sh
NXP i.MX Release Distro SDK installer version 6.6-scarthgap
===========================================================
Enter target directory for SDK (default: /opt/fsl-imx-xwayland/6.6-scarthgap): ~/my_sdk
You are about to install the SDK to "/home/foo/test/my_sdk". Proceed [Y/n]? y
Extracting SDK...................................................................................................................................................................................................................done
Setting it up...done
SDK has been successfully set up and is ready to be used.
Each time you wish to use the SDK in a new shell session, you need to source the environment setup script e.g.
 $ . /home/foo/my_sdk/environment-setup-armv8a-poky-linux
foo@bar:~/yocto/build$ . /home/foo/my_sdk/environment-setup-armv8a-poky-linux
foo@bar:~/yocto/build$ make modules_prepare -C $SDKTARGETSYSROOT/usr/src/kernel

```
## Build out-of-tree kernel modules
The following example shows how to build a out-of-tree kernel module with the Yocto SDK.  
We use the Advantech USB-4604B, a USB to serial converter, as an example.  
>[!NOTE]
>Notice that this spacific drivers makefile uses **$(KERNELDIR)** to represent the position of the kernel header files; therefore, we use **export KERNELDIR=$SDKTARGETSYSROOT/usr/src/kernel** before we build the driver with **make**.

```console
foo@bar:~/example$ git clone https://github.com/saurontech/USB-4604-BE-linux-driver.git
foo@bar:~/example$ cd ./USB-4604-BE-linux-driver/driver
foo@bar:~/example/USB-4604-BE-linux-driver/driver$ . /home/foo/my_sdk/environment-setup-armv8a-poky-linux
foo@bar:~/example/USB-4604-BE-linux-driver$ cat ./Makefile
obj-m := adv_usb_serial.o
adv_usb_serial-objs := xr_usb_serial_common.o

KERNELDIR ?= /lib/modules/$(shell uname -r)/build
PWD       := $(shell pwd)

EXTRA_CFLAGS	:= -DDEBUG=0

all:
	$(MAKE) -C $(KERNELDIR) M=$(PWD)

modules_install:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules_install

clean:
	rm -rf *.o *~ core .depend .*.cmd *.ko *.mod.c .tmp_versions vtty *.symvers *.order *.a *.mod
foo@bar:~/example/USB-4604-BE-linux-driver/driver$ export KERNELDIR=$SDKTARGETSYSROOT/usr/src/kernel
foo@bar:~/example/USB-4604-BE-linux-driver$ make
```
## Kernel Development
To modify/develop the in-tree kernel source code, the **devtool** command provided by Yocto is a good base to start.
The following command will prepare a kernel source tree, which is managed by git, under **"build/workspace/sources/linux-imx"**.
```console
foo@bar:~/yocto/build$ devtool modify linux-imx
foo@bar:~/yocto/build$ ls ./workspace/sources/
linux-imx
```
To test the modified source code, build the modified kernel with the following command.
```console
foo@bar:~/yocto/build$ devtool build linux-imx
```
To modify the default configure, use the following command.
```console
foo@bar:~/yocto/build$ devtool menuconfig  linux-imx
```
During the process, use **"git add, git commit"** to source-control the development.
To save all the changes controled by git into a new layer, use the following commands
```console
foo@bar:~/yocto/build$ bitbake-layers create-layer ../sources/meta-mylayer
foo@bar:~/yocto/build$ devtool update-recipe -a ../sources/meta-mylayer linux-imx
```
To finish the process, use the following procedure to clean the current workspace and add the newly created layer to the yocto project.
```console
foo@bar:~/yocto/build$ devtool reset linux-imx
foo@bar:~/yocto/build$ bitbake-layers add-layer ../sources/meta-mylayer/
foo@bar:~/yocto/build$ bitbake linux-imx
```

## Build Debian/Ubuntu based rootfs
follow the instructions below to build the rootfs with debootstrap, qemu, and chroot
```console
foo@bar:~/work$ sudo apt-get install qemu-user-static debootstrap debian-archive-keyring
foo@bar:~/work$ nano ch-rootfs.sh
foo@bar:~/work$ chmod +x ./ch-rootfs.sh
```
the content of ch-rootfs.sh is listed as below:
```sh
#!/bin/bash
#
function mnt() {
 echo "MOUNTING"
 sudo mount -t proc /proc ${2}proc
 sudo mount -t sysfs /sys ${2}sys
 sudo mount -o bind /dev ${2}dev
 sudo mount -o bind /dev/pts ${2}dev/pts
 sudo chroot ${2}
}
function umnt() {
 echo "UNMOUNTING"
 sudo umount ${2}proc
 sudo umount ${2}sys
 sudo umount ${2}dev/pts
 sudo umount ${2}dev
}

function pack() {
 echo "Packing rootfs to rootfs.tar.gz ...."
 sudo rm -f ../rootfs.tar.gz
 echo '=== tar rootfs start ==='
 cd $2 && sudo tar zcvf ../rootfs.tar.gz *
 echo '=== tar rootfs finish ==='
}

if [ "$1" == "-m" ] && [ -n "$2" ] ;
then
 mnt $1 $2
 umnt $1 $2
elif [ "$1" == "-u" ] && [ -n "$2" ];
then
 umnt $1 $2
elif [ "$1" == "-z" ] && [ -n "$2" ];
then
 pack $1 $2
else
 echo ""
 echo "Either 1'st, 2'nd or both parameters were missing"
 echo ""
 echo "1'st parameter can be one of these: -m(mount) OR -u(umount) or -z(pack)"
 echo "2'nd parameter is the full path of rootfs directory(with trailing '/')"
 echo ""
 echo "For example: ./ch-rootfs.sh -m /media/sdcard/"
 echo ""
 echo 1st parameter : ${1}
 echo 2nd parameter : ${2}
fi

```
> [!NOTE]
> The following commands will slightly differe between Debian 12 and Ubuntu 24.04; therefore, we seperate the instuctions into two subsections.  
> Choose the instructions based on your target distro.
> The "deb" files mentioned below could be found in "yocto/build/tmp/deploy/deb/all/".
### Debian 12 
```console
foo@bar:~/work$ sudo debootstrap --arch arm64 bookworm my_rootfs http://deb.debian.org/debian
foo@bar:~/work$ sudo tar xvf ./modules-imx8mq-ecu150a1.tgz -C ./my_rootfs/usr/
foo@bar:~/work$ cp firmware-imx-sdma-imx7d*.deb ./my_rootfs/tmp/
foo@bar:~/work$ cp linux-firmware-rtl*.deb ./my_rootfs/tmp/
foo@bar:~/work$ cp linux-firmware-whence-license_*.deb ./my_rootfs/tmp/
foo@bar:~/work$ ./ch-rootfs.sh -m ./my_rootfs/
root@imx:~/$ apt-get update
root@imx:~/$ apt-get install sudo ssh net-tools iputils-ping rsyslog bash-completion htop resolvconf dialog gpiod vim locales netplan.io systemd-timesyncd systemd-resolved
root@imx:~/$ dpkg-reconfigure locales
root@imx:~/$ cd /tmp  && dpkg -i *.deb
root@imx:~/$ rm /tmp/*
```
### Ubuntu 24.04
```console
foo@bar:~/work$ sudo debootstrap --arch arm64 noble my_rootfs http://tw.archive.ubuntu.com/ubuntu/
foo@bar:~/work$ sudo tar xvf ./modules-imx8mq-ecu150a1.tgz -C ./my_rootfs/usr/
foo@bar:~/work$ cp firmware-imx-sdma-imx7d*.deb ./my_rootfs/tmp/
foo@bar:~/work$ cp linux-firmware-rtl*.deb ./my_rootfs/tmp/
foo@bar:~/work$ cp linux-firmware-whence-license_*.deb ./my_rootfs/tmp/
foo@bar:~/work$ ./ch-rootfs.sh -m ./my_rootfs/
root@imx:~/$ apt-get update
root@imx:~/$ add-apt-repository universe
root@imx:~/$ apt install sudo ssh net-tools iputils-ping rsyslog bash-completion htop vim nano netplan.io software-properties-common gpiod
root@imx:~/$ cd /tmp  && dpkg -i *.deb
root@imx:~/$ rm /tmp/*
```
On Ubuntu, root login is disabled by default; therefore it is a good idea to add a user with sudo privilage.
```console
root@imx:~/$ adduser admin
root@imx:~/$ usermod -aG sudo admin
```
> [!NOTE]
> The difference between Debian 12/Ubuntu 24.04 stops here.  
The following instructions can be shared between two target distros.
```console
root@imx:~/$ ln -s /dev/null /etc/systemd/network/99-default.link
root@imx:~/$ echo 'imx8mq-ecu150a1' > /etc/hostname
root@imx:~/$ cat <<EOF >> /etc/udev/rules.d/localextra.rules
# Microchip Technology USB2740 Hub
KERNEL=="rtc1", SYMLINK+="rtc"

KERNEL=="ttymxc2", GROUP="dialout", MODE="0664", SYMLINK+="ttyAP0"
KERNEL=="ttymxc3", GROUP="dialout", MODE="0664", SYMLINK+="ttyAP1"
KERNEL=="fram0", GROUP="dialout", MODE="0664", SYMLINK+="fram"
EOF
root@imx:~/$ systemctl enable systemd-networkd.service
root@imx:~/$ systemctl enable systemd-resolved.service
root@imx:~/$ systemctl enable systemd-timesyncd.service
root@imx:~/$ nano /etc/profile.d/custom.sh
root@imx:~/$ exit
foo@bar:~/work$ ./ch-rootfs.sh -z ./my_rootfs/
```
The constent of custom.sh is listed as below:
```sh
# Set the prompt for bash and ash (no other shells known to be in use here)
[ -z "$PS1" ] || PS1='\u@\h:\w\$ '

# Use the EDITOR not being set as a trigger to call resize later on
FIRSTTIMESETUP=0
if [ -z "$EDITOR" ] ; then
    FIRSTTIMESETUP=1
fi

if [ -t 0 -a $# -eq 0 ]; then
    if [ ! -x /usr/bin/resize ] ; then
        if [ -n "$BASH_VERSION" ] ; then
            # Optimized resize funciton for bash
resize() {
    local x y
    IFS='[;' read -t 2 -p $(printf '\e7\e[r\e[999;999H\e[6n\e8') -sd R _ y x _
    [ -n "$y" ] && \
    echo -e "COLUMNS=$x;\nLINES=$y;\nexport COLUMNS LINES;" && \
    stty cols $x rows $y
}
        else
# Portable resize function for ash/bash/dash/ksh
# with subshell to avoid local variables
resize() {
    (o=$(stty -g)
    stty -echo raw min 0 time 2
    printf '\0337\033[r\033[999;999H\033[6n\0338'
    if echo R | read -d R x 2> /dev/null; then
        IFS='[;R' read -t 2 -d R -r z y x _
    else
        IFS='[;R' read -r _ y x _
    fi
    stty "$o"
    [ -z "$y" ] && y=${z##*[}&&x=${y##*;}&&y=${y%%;*}
    [ -n "$y" ] && \
    echo "COLUMNS=$x;"&&echo "LINES=$y;"&&echo "export COLUMNS LINES;"&& \
    stty cols $x rows $y)
}
        fi
    fi
    # only do this for /dev/tty[A-z] which are typically
    # serial ports
    if [ $FIRSTTIMESETUP -eq 1 -a ${SHLVL:-1} -eq 1 ] ; then
        case $(tty 2>/dev/null) in
            /dev/tty[A-z]*) resize >/dev/null;;
        esac
    fi
fi

```
## Create Secure Boot Images
To create secure boot images, the first step would be setting up the CST tool, provided by NXP.  

1. Download the Code-Singing-Tool From NXP:https://www.nxp.com/webapp/Download?colCode=IMX_CST_TOOL_NEW&appType=license  
2. Create fuse.bin and table.bin
```console
foo@bar:~/Download$ cd ./cst-3.4.0/keys
foo@bar:~/Download/cst-3.4.0/keys$ vi serial
12345678
foo@bar:~/Download/cst-3.4.0/keys$ vi key_pass.txt 
my_pass_phrase
my_pass_phrase
foo@bar:~/Download/cst-3.4.0/keys$ ./hab4_pki_tree.sh
...
Do you want to use an existing CA key (y/n)?: n

Key type options (confirm targeted device supports desired key type):
Select the key type (possible values: rsa, rsa-pss, ecc)?: rsa
Enter key length in bits for PKI tree: 2048
Enter PKI tree duration (years): 10
How many Super Root Keys should be generated? 1
Do you want the SRK certificates to have the CA flag set? (y/n)?: y
...
foo@bar:~/Download/cst-3.4.0/keys$ ls SRK*
SRK1_sha256_2048_65537_v3_ca_key.der  SRK1_sha256_2048_65537_v3_ca_key.pem

foo@bar:~/Download/cst-3.4.0/keys$ cd ../crts/
foo@bar:~/Download/cst-3.4.0/crts$ ../linux64/bin/srktool -h 4 -t SRK_1_2_3_4_table.bin \
    -e SRK_1_2_3_4_fuse.bin -d sha256 -c \
    ./SRK1_sha256_2048_65537_v3_ca_crt.pem, \
    ./SRK2_sha256_2048_65537_v3_ca_crt.pem, \
    ./SRK3_sha256_2048_65537_v3_ca_crt.pem, \
    ./SRK4_sha256_2048_65537_v3_ca_crt.pem -f 1
Number of certificates    = 1
SRK table binary filename = SRK_1_2_3_4_table.bin
SRK Fuse binary filename  = SRK_1_2_3_4_fuse.bin
SRK Fuse binary dump:
SRK HASH[0] = 0x3D06A4A9
SRK HASH[1] = 0x4BC55D12
SRK HASH[2] = 0xA5F45E7F
SRK HASH[3] = 0x1F1F68FC
SRK HASH[4] = 0x3B9B4AE8
SRK HASH[5] = 0xFC658293
SRK HASH[6] = 0x40A706C9
SRK HASH[7] = 0x94A9139E

foo@bar:~/Download/cst-3.4.0/keys$ ls SRK_*
SRK_1_2_3_4_fuse.bin  SRK_1_2_3_4_table.bin
```
3. Add & setup the "security reference design" meta-layer to the yocto project.
```console
foo@bar:~/yocto$ source ./setup-environment build
foo@bar:~/yocto/build$ bitbake-layers add-layer ../sources/meta-nxp-reference-design/meta-secure-boot
foo@bar:~/yocto/build$ vim ./conf/local.conf     #add    CST_PATH = "<absolute path to cst-3.4.0 folder>"  
```
4. Use one of the following commands to build the desired signed image:
```console
foo@bar:~/yocto/build$ bitbake core-image-minimal-secure-boot
foo@bar:~/yocto/build$ bitbake imx-boot-signature                # for building a signed uboot only
foo@bar:~/yocto/build$ bitbake linux-imx-signature               # for building a signed linux only
```
## Program eFUSE
In the "cst-3.4.0/keys" folder, extract the values needed to program the eFUSE.
```console
foo@bar:~/Download/cst-3.4.0/keys$ hexdump -e '/4 "0x"' -e '/4 "%X""\n"' SRK_1_2_3_4_fuse.bin
0x9A842534
0xB0491AB4
0xD5B6A07B
0xFD92DCE7
0xC10DC87C
0xD8BD04A9
0x704E9FE4
0x9B025359
```
On a ECU device running the signed uboot, check the HAB status:
```console
u-boot=> hab_status
Secure boot disabled
HAB Configuration: 0xf0, HAB State:
0x66
No HAB Events Found!
```
> ### Warning! for each CPU, the fuse can only be programmed once
Burn the eFUSE with the values extracted from the previous command:
```console
u-boot=> fuse prog 6 0 0x9A842534
u-boot=> fuse prog 6 1 0xB0491AB4
u-boot=> fuse prog 6 2 0xD5B6A07B
u-boot=> fuse prog 6 3 0xFD92DCE7
u-boot=> fuse prog 7 0 0xC10DC87C
u-boot=> fuse prog 7 1 0xD8BD04A9
u-boot=> fuse prog 7 2 0x704E9FE4
u-boot=> fuse prog 7 3 0x9B025359
```
## Verify the signature included in flash.bin
The next step is to verify that the signatures included in flash.bin image is
successfully processed without errors. HAB generates events when processing
the commands if it encounters issues.

The hab_status U-Boot command call the hab_report_event() and hab_status()
HAB API functions to verify the processor security configuration and status.
This command displays any events that were generated during the process.
```console
u-boot=> hab_status
Secure boot disabled
HAB Configuration: 0xf0, HAB State: 0x66
```
## Closing the device

After the device successfully boots a signed image without generating any HAB
events, it is safe to close the device. This is the last step in the HAB
process, and is achieved by programming the SEC_CONFIG[1] fuse bit.

Once the fuse is programmed, the chip does not load an image that has not been
signed using the correct PKI tree.
```console
u-boot=> fuse prog 1 3 0x2000000
```
