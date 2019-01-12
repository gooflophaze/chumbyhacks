# chumbyhacks

I had a little python project I made for a raspi and screen to show me my current dhcp leases - too many IoT devices to remember all the ip's, too lazy to set up a local DNS server. After I made it I wanted to use my pi screen for something else, and thought putting it on my long neglected chumby would be good.

Well - that turned out to be a bit more work than I anticipated. But I've got wifi, the screen, and the touchscreen working and I thought I'd throw up a how to and some patches - hopefully someone that's more experienced can give it more love and time than I can afford right now.

Please keep in mind these are unpolished dirty hacks. Do not attempt. Standard disclaimers etc etc.

Prerequisites

At minimum, you'll need a serial console to do the configuration post SD card install. I'm sure there are some posts to reference setting up. 3.3v FTDI at 115200. You'll also need some way to write to the SD card - and an upgrade from the stock 1-2gb sdcard would be wise as well. I'm using an 8gb card that's a little too small for my rpi project now.

I originally used my ubuntu desktop to compile - but found it was pretty bad at getting the proper libraries set up. I ended up running a debian minimum machine under a virtualbox VM on my desktop. ARM cross compiling has come a ways since the old tutorials were made. Specifically I used https://www.digikey.com/eewiki/display/linuxonarm/iMX233-OLinuXino as a rough guide. I skipped using the linaro environment, instead relying on straight debian to do what I needed.

apt-get install gcc-arm-linux-gnueabi build-essential bison flex libssl-dev libncurses-dev

Problem 1: Bootloader

Boot process is well described at http://wiki.chumby.com/index.php?title=Falconwing_boot#Boot_process

[code]
1. The boot rom examines the OCOTP fuses and determines it needs to boot from MBR
2. The first 2048 bytes of the SD card are examined, and the boot image is located
3. sdram_prep, clocks, power, and chumby_stub are all loaded from a Bootstream image into OCRAM
4. sdram_pre, clocks, and power are all executed to set up registers and DRAM
5. chumby_stub is executed
6. chumby_stub loads chumby_boot from disk into DRAM
7. chumby_stub jumps to chumby_boot
8. chumby_boot checks for someone touching the screen
9. chumby_boot sets up the LCD and draws a boot logo.
10. chumby_boot waits 3 sec for the user to press a key to enter shell
11. chumby_boot draws the appropriate "now booting" logo
12. chumby_boot loads the Linux kernel from disk into DRAM
13. chumby_boot sets up the Linux tags
14. chumby_boot jumps to the Linux kernel
[/code]

The problem in trying to get a newer kernel is two fold - one, the reserved space for a kernel is 4mb in size. Secondly - the initial setup data is encrypted by a key that's burned into the imx233 processor - so it can't be easily manipulated. The solution is to take step 14 - which is outside the encrypted boot area - and run das u-boot in its place. From here we can basically chain load uboot - and use u-boot to set up all the cmdline variables and devicetree data needed for a new kernel.

But it's not quite as easy as running "config_util --cmd=putblock --dev=/dev/mmcblk0p1 --block=krnA < u-boot.bin" - it's pretty close. U-boot expects to be run close to bare metal assembly - and would normally run steps 2-4 as part of the SPL - Secondary Program Loader - before u-boot runs. The SPL also gathers some basic information about the system to pass onto uboot - like ram size and address range. We'll need to modify u-boot to hard code those values as a dirty hack.

First step - snag u-boot.
https://github.com/u-boot/u-boot/archive/v2018.11.zip 

Then create a new board config - these are copied off of pre-existing imx23-based boards. I began to make a more pi-like system where uEnv and cmdline could be stored in a fat32 partition on the SD card. U-Boot can also read it's config off a SD card block which I copied off another config, but didn't do the math to make sure it's outside the stock bootloaders range - so beware using it. In the defconfig file you set CONFIG_SYS_TEXT_BASE=0x40008000 - this is the entry point for the kernel.

The second major change is to edit arch/arm/cpu/arm926ejs/mxs/mxs.c - this is where the SPL normally sets the memory size. Since chumbys have 64MB, we hardcode it here and disable the hang(); when it returns a 0 mem size - which it will.

 if (data->mem_dram_size == 0) {
                printf("MXS:\n"
                        "Error, the RAM size passed up from SPL is 0!\n");
        //      hang();
        }
        data->mem_dram_size = 0x04000000;
        gd->ram_size = data->mem_dram_size;
        return 0;

Aside from that, there's some setting up of the iomux pins and a disabling of the watchdog. 

https://github.com/openhoon/chumbyhacks/blob/master/uboot-2018.11-chumby.patch

Extract v2018.11.zip and the github patch an apply it by cd'ing into the directory and patch < uboot-2018.11-chumby.patch - that should create the new config files. From there

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- mx23_chumby_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-

That will produce uboot.bin. Copy that (either through SSH or a usb memstick) and run

config_util --cmd=putblock --dev=/dev/mmcblk0p1 --block=krnA < u-boot.bin  

on the chumby.

Reboot - and now you should see a u-boot prompt available through serial.

Now though u-boot you can load up a kernel image and boot it - if you're running this off the madox rom you can run 

setenv bootargs console=ttyAM0,115200 root=/dev/mmcblk0p2 rw rootwait chumbyrev=08 ssp1=mmc sysrq_always_enabled logo.brand=chumby
ext2load mmc 0:2 0x42000000 /boot/zImage
bootz 0x42000000 - 

Next up is setting up the the SD card - just make sure you leave the first partition alone. Remove it from the chumby, slap it into your machine, and fdisk to create a small 16mb fat32 partition, and then the rest for the root fs. Snag a rootfs from https://rcn-ee.com/rootfs/eewiki/minfs/ - anything listed as armel will work - basically we're following the same steps from https://www.digikey.com/eewiki/display/linuxonarm/iMX233-OLinuXino - set up etc/fstab, /etc/wpa/wpa_supplicant.conf, and etc/interfaces .

2nd Problem: Kernel and Device Tree

It's not that hard to compile a new kernel - a make CROSS_COMPILE=arm-linux-gnueabi- mxs_defconfig will give you a pretty basic kernel. Key to making things work is a new devicetree - this provides system information similar to the bios for the kernel. I was able to get the system booted with the olinuxino devicetree, but had a heck of a time making the graphics work. Eventually I had to bust out the schematics to map the gpio pins - I think it's mostly accurate. 

The framebuffer console has changed a bit in newer kernels, with a new mxsfb driver that works on the DRM subsystem - which now needs an additional devicetree node for the LCD panel. The LCD panel in the chumby is a nanovision nma35qv65 320x240 screen - I found the datasheet at http://www.warf.com/download/5466_9662_1.pdf - which has a typo - the HX8328-A device doesn't exist - but the http://orientdisplay.com/pdf/HX8238-D.pdf does. Still couldn't get it to work - until I manually inserted the initialization sequence from the chumby-boot code into the driver initialization. Looking at the registers the MXSFB sets -

mxsfb ctrl1: 00070000
chumby ini  010f0701
mxsfb ctrl: 00080320
chumby ini: 000b0821

These values don't appear to have any external flags to pass so the mxsfb_crtc.c file had to be modified directly. Most of the register value differences appear to be related to IRQ over and underflows.

https://github.com/openhoon/chumbyhacks/blob/master/chumby-4.20-kernelhacks.patch includes the mxsfb changes, along with a new devicetree, and a chumby.config that I'm currently using. A couple of notes - the USB wifi card is rt73usb, DRM requires some CMA DMA ram (I just gave it 2mb). Once the patch is applied - run

make CROSS_COMPILE=arm-linux-gnueabi- mxs_defconfig 
cp chumby.config .config
make CROSS_COMPILE=arm-linux-gnueabi- menuconfig 
make -j8 CROSS_COMPILE=arm-linux-gnueabi- 

That'll pop the kernel out in arch/arm/boot - cp that to your /boot directory on the flash cart, cp arch/arm/dts/imx23-chumby.dtb to /boot/dtbs/ . Uboot is currently set up to run 

setenv bootargs console=ttyAMA0,115200 root=/dev/mmcblk0p3 rootfstype=ext4 rw rootwait ssp1=mmc ram=64M net.ifnames=0

I'm not sure if ssp1=mmc is still used by the 4.20 kernel. 

With only 64mb of ram, even an apt-get update will run out of memory. While it's generally not advisable to run a swapfile on an SD card - wants must.

fallocate -l 512M /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo 'vm.swappiness=10'
echo 'vm.swappiness=10' >> /etc/sysctl.conf
echo '/swapfile swap swap defaults 0 0' >> /etc/fstab

Hopefully I've written these instructions out clear enough - and maybe someone can use them and make it better. I haven't attempted to get the sound working and I don't think the backlight is 100% tuned correctly. I might throw up a dd'able image up somewhere once it's more polished. On the surface the alterations I made aren't much - but they actually took 3 weeks of poking around during the holiday break, learning about device trees and poking around in source files. 








