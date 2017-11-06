# Ubuntu 16.04 LTS on Xserve 1,1

by [frusdelion](https://github.com/frusdelion/xserve-guides)

## Introduction
We will be assuming the following as your work environment:
- macOS Sierra with Docker for Mac installed (as of November 2017)


## Contents
1. [Prepare USB Drive for Ubuntu Installation](#step-1)
2. [Prepare the GRUB2 Bootloader to Generate EFI-IA32 Image](#step-2)
3. [Copy the Ubuntu Server installer files to the USB Stick](#step-3)
4. [Boot and install Ubuntu Server on Xserve 1,1]($step-4)
5. [Fix EFI System Partition on the installed Ubuntu system](#step-5)

<a name="step-1"></a>
## 1. Prepare USB Drive for Ubuntu Installation

**Requirements**
- 4GB or more Storage Space

**Note**

Backup the contents of the USB Storage Device you will be using. Your USB Storage Device will be formatted, **all data on that device will be lost**.

**Steps**
1. Open `Disk Utility`, and format your USB Drive in this manner:
  - GUID Partition Map (aka GUID Partition Table)
  - MS-DOS (FAT) partition
  - Give it a name (suggestion: **UBUNTU**)
  - Apply the changes to the drive

<a name="step-2"></a>
## 2. Prepare the GRUB2 Bootloader to Generate EFI-IA32 Image

**Requirements**
- An Ubuntu Linux system, OR
- Mac OS X with Docker for Mac installed

This guide will assume you are using macOS Sierra with Docker for Mac installed.

If you wish to use your own distribution of Linux, please translate the following instructions to suit the distribution variant of Linux you are using.

**Steps**
1. Run an Ubuntu Docker container as your workspace. This command will use your current directory as your work directory in the container. See `$(pwd)`.
  ```bash
  $ docker run -it --rm -v "$(pwd)":/root --workdir /root ubuntu
  ```
2. Download or clone GRUB2 from the grub website.
  - [GRUB2 source](ftp://ftp.gnu.org/gnu/grub/) (2.02 as of November 2017)
  - Or you can use the following to clone the GRUB2 repository.
  ```bash
  # In Docker Container
  $ git clone git://git.savannah.gnu.org/grub.git
  $ cd ./grub
  ```
3. Run either of the following environment variable commands for your desired EFI variant.

  ```bash
  # In Docker Container

  #32-bit EFI (for Xserve 1,1)
  $ export EFI_ARCH=i386

  # 64-bit EFI
  $ export EFI_ARCH=x86_64
  ```
3. Run the following commands
  ```bash
  # In Docker Container

  # Dependencies
  $ sudo apt-get install bison libopts25 libselinux1-dev autogen m4 autoconf help2man libopts25-dev flex libfont-freetype-perl automake autotools-dev libfreetype6-dev texinfo make

  # Compile GRUB2
  $ ./configure --with-platform=efi --target=${EFI_ARCH} --program-prefix=""
  $ make
  ```
4. Then run the following commands to build the EFI image.
  ```bash
  # In Docker Container
  $ cd ./grub-core

  $ ../grub-mkimage -d . -o bootia32.efi -O i386-efi -p /boot/grub ntfs hfs appleldr boot cat efi_gop efi_uga elf fat hfsplus iso9660 linux keylayouts memdisk minicmd part_apple ext2 extcmd xfs xnu part_bsd part_gpt search search_fs_file chain btrfs loadbios loadenv lvm minix  minix2 reiserfs memrw mmap msdospart scsi loopback normal configfile gzio all_video efi_gop efi_uga gfxterm gettext echo boot chain eval
  ```
5. Take note of the `bootia32.efi`. You will need it later.

<a name="step-3"></a>
## 3. Copy the Ubuntu Server installer files to the USB Stick
**Requirements**
- A copy of the [Ubuntu Server 16.04.3 LTS](http://releases.ubuntu.com/16.04/ubuntu-16.04.3-server-amd64.img) `.img` (as of November 2017)

**Steps**
1. Run the following command.
  ```bash
  $ hdiutil attach -nomount <PATH TO YOUR UBUNTU SERVER .IMG FILE>
  ```
2. Take note of the `/dev/diskN` portion of your output.
3. Run the following commands.
  ```bash
  $ mkdir ./ubuntu-server-mount
  $ sudo mount -t cd9660 </dev/diskN> ./ubuntu-server-mount

  # When you're done
  $ sudo umount ./ubuntu-server-mount
  $ rmdir ./ubuntu-server-mount
  ```
4. Copy **ALL** the files in this folder-mount into your USB Storage Device.
5. Go to your `grub` folder and run the following commands.
  ```bash
  $ cd ./grub-core

  $ mkdir /Volumes/<USB STORAGE MOUNT PATH>/boot/grub/i386-efi
  $ sudo cp *.mod *.lst /Volumes/<USB STORAGE MOUNT PATH>/boot/grub/i386-efi

  $ sudo cp bootia32.efi /Volumes/<USB STORAGE MOUNT PATH>/EFI/BOOTIA32.EFI
  ```

<a name="step-4"></a>
## 4. Boot and install Ubuntu Server on Xserve 1,1
**Steps**
1. Plug in your USB Storage Device into Xserve 1,1
2. Boot your Xserve 1,1 and hold down `C`
3. Select `Install Ubuntu Server`
4. At the partition manager, select the `Guided Mode - use entire disk`
5. Once the installation completes, you should not be able to immediately boot from your newly installed Ubuntu Server.

**Note**

If *Installation CD could not be found* error occurs, plug out and in your USB Storage Device and press *Continue*.


<a name="step-5"></a>
## 5. Fix EFI System Partition on the installed Ubuntu system
**Credits**
- [The Slightly Disgruntled Scientist](http://heeris.id.au/2014/ubuntu-plus-mac-pure-efi-boot/#fixing-efi)

**Steps**
1. Reboot your Xserve 1,1 and hold down `C`.
2. You should see *Welcome to GRUB*
3. When presented with the GRUB menu, press `C`. You should be presented with the GRUB Console.
  ```bash
  grub>
  ```
4. Run the following commands on the GRUB Console prefixed with `grub>`.
  ```bash
  grub> ls
  (memdisk) (hd0) (hd0,msdos) (hd1) (hd2) (hd2,gpt3) (hd2,gpt2) (hd2,gpt1)
  ```
5. Go through each `(hdN,gptN)` to find your installed Ubuntu Server's GRUB folder. I suggest that you use the `/home` directory as your marker.
  ```bash
  grub> ls (hdN,gptN)/home
  jason/
  ```
6. Now, go through each partition `gptN` in the hard drive `hdN` with the `/home` directory, to look for the `/boot/grub` folder. NOTE: The output may not be the same, but it should look similar.
  ```bash
  grub> ls (hdN,gptN)/boot/grub
  unicode.pf2 [...] grub.cfg
  ```
7. Enter the UUID of the correct drive, into the following commands and boot into your Ubuntu Server. *NOTE*: Grub Console supports auto-completion. Press the Tab key when you see `<TAB>` below.
  ```bash
  grub> ls -l (hdN,gptN)
  Partition hd2,gpt2: Filesystem type ext* 〈...snip...〉 UUID e86c20b9-83e1-447d-a3be-d1ddaad6c4c6 - Partition start at [...]

  grub> linux /boot/vmlinuz<TAB>.efi.signed root=UUID=<UUID OF THE PREVIOUS COMMAND> ro nomodeset radeon.modeset=0
  grub> initrd /boot/initrd<TAB>
  grub> boot
  ```
8. Run the following commands to prepare to fix the EFI Partition. Note that `sdXN` refers to the `X: alphabet` and `N: number` of the partition that your system returns to you. i.e. `/dev/sda1`
  ```bash
  # In Xserve 1,1
  $ sudo add-apt-repository ppa:detly/mactel-utils
  $ mount | grep '/boot/efi'

  # You should see a line that looks like this:
  /dev/sdXN on /boot/efi type vfat (rw)

  $ sudo umount /dev/sdXN
  ```
9. Enter the gdisk console. Ensure that `GPT` is `present`.
  ```bash
  $ sudo gdisk /dev/sdXN
  GPT fdisk (gdisk) version 0.8.8

  Partition table scan:
    MBR: hybrid
    BSD: not present
    APM: not present
    GPT: present

  Found valid GPT with hybrid MBR; using GPT.

  Command (? for help):
  ```
10. See all partitions, and ensure that you have the first partition of the type `EF00`.
  ```bash
  Command (? for help): p
  Disk /dev/sda: 976773168 sectors, 465.8 GiB
  Logical sector size: 512 bytes
  Disk identifier (GUID): 717BD65E-A514-4FD9-A4A7-1BE01C2F31E0
  Partition table holds up to 128 entries
  First usable sector is 34, last usable sector is 976773134
  Partitions will be aligned on 2048-sector boundaries
  Total free space is 4077 sectors (2.0 MiB)

  Number  Start (sector)    End (sector)  Size       Code  Name
     1            2048          194559   94.0 MiB    EF00
     2          194560       968574975   461.8 GiB   8300
     3       968574976       976771071   3.9 GiB     8200
  ```
11. Delete the `EF00` partition
  ```bash
  Command (? for help): d
  Partition number (1-3): 1
  ```
12. Create a new HFS+ one in its place
  ```bash
  Command (? for help): n
  Partition number (1-128, default 1): 1

  # Just press enter for first and last sector options.
  First sector (34-976773134, default = 2048) or {+-}size{KMGTP}:
  Last sector (2048-194559, default = 194559) or {+-}size{KMGTP}:
  ```
13. Enter `AF00` for filesystem code:
  ```bash
  Current type is 'Linux filesystem'
  Hex code or GUID (L to show codes, Enter = 8300): AF00
  Changed type of partition to 'Apple HFS/HFS+'
  ```
14. Press `p` to double check your changes, and press `w` to write.
  ```bash
  Command (? for help): w

  Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
  PARTITIONS!!

  Do you want to proceed? (Y/N): Y
  OK; writing new GUID partition table (GPT) to /dev/sda.
  Warning: The kernel is still using the old partition table.
  The new table will be used at the next reboot.
  The operation has completed successfully.
  ```
15. Format the newly created partition.
  ```bash
  $ sudo mkfs.hfsplus /dev/sda1 -v Ubuntu
  Initialized /dev/sda1 as a 94 MB HFS Plus volume
  ```
16. Enter the filesystem table editor
  ```bash
  sudoedit /etc/fstab
  ```
17. Remove the existing lines that looks like this:
  ```
  # /boot/efi was on /dev/sda1 during installation
  UUID=C59D-1B30  /boot/efi       vfat    defaults        0       1
  ```
18. Run this command to push the new /boot/efi UUID into the filesystem table.
  ```bash
  $ sudo bash -c 'echo $(blkid -o export -s UUID /dev/sda1) /boot/efi auto defaults 0 0 >> /etc/fstab'
  ```
19. Ensure that it does not have anything before `UUID=` on the last line:
  ```bash
  $ sudoedit /etc/fstab
  ```
20. Test mount the new `/boot/efi` folder. The output should be 0.
  ```bash
  $ sudo mount /boot/efi
  echo $?
  # 0
  ```
21. Run the following commands to prepare and install GRUB
  ```bash
  $ sudo mkdir -p "/boot/efi/EFI/$(lsb_release -ds)/"
  $ sudo bash -c 'echo "This file is required for booting" > "/boot/efi/EFI/$(lsb_release -ds)/mach_kernel"'
  $ sudo bash -c 'echo "This file is required for booting" > /boot/efi/mach_kernel'

  # if 32-bit EFI
  $ sudo grub-install --target i386-efi --boot-directory=/boot --efi-directory=/boot/efi --bootloader-id="$(lsb_release -ds)"

  # if 64-bit EFI
  $ sudo grub-install --target x86_64-efi --boot-directory=/boot --efi-directory=/boot/efi --bootloader-id="$(lsb_release -ds)"

  # Apply this
  $ sudo hfs-bless "/boot/efi/EFI/$(lsb_release -ds)/System/Library/CoreServices/boot.efi"
  ```
22. Add the Mactel Boot Logo
  ```bash
  $ sudo apt-get install mactel-boot-logo
  $ sudo cp /usr/share/mactel-boot-logo/ubuntu.icns /boot/efi/.VolumeIcon.icns
  ```
23. Reboot
24. When you see the `GRUB` menu, press E and add the following to your `linux /boot/vmlinuz...`. *NOTE*: Not necessary if you do not need to access your server using the mini-DVI dongle.
  ```bash
  # See note for more information.
  linux /boot/vmlinuz.... ro nomodeset radeon.modeset=0
  ```
25. Boot into your new Xserve 1,1. Enjoy.

**Note**
- If you are not using the stock X1300 card, exclude `nomodeset radeon.modeset=0` from the command. This additional command is meant for Xserve 1,1 with the stock Radeon X1300 card, to prevent the framebuffer from getting stuck while booting.
