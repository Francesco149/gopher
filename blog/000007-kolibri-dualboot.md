I recently discovered [KolibriOS](http://wiki.kolibrios.org/wiki/Main_Page), a very unique tiny
OS written in assembly that isn't windows or unix-like. it boots as a ramdisk and manages to pack
drivers for common NICs, AMD GPUS, Intel GPUs and more into its tiny ~60MB distro (30MB compressed)

since some of my machines happen to have supported hardware, I decided to try and run it on
bare metal, and I got it running on both my q9650 and xeon 2420v2 machines. the radeon driver
works great on my r9 270x

# BIOS install
* you need a drive with a BIOS linux install on it
* you need a machine that can boot legacy BIOS stuff
* ideally when you install linux leave a few gigs for the kolibri partition
* use grub as a bootloader, other bootloaders are untested
* if your sata controller is in AHCI mode, set it to IDE mode. you can try AHCI, but I couldn't
  get it to work even with the AHCI driver. enabling or disabling the controllers can also make
  it work (I had to enable both and set them to enhanced mode)

create a fat32 partition for kolibri, I resized my linux partition with gparted to make space
for that. to resize or create partitions you can't be booted into your drive unless you already
have free space to create the partition in so you probably want to do it either on a different
drive or from gparted live.

now you can boot into the target drive's linux and start working on the kolibri partition

format it (replace with your disk and partition)

    sudo mkfs.vfat -F32 /dev/disk/by-id/ata-Netac_SSD_720GB_AA000000000000000591-part4

mount it

    sudo mkdir /mnt/kolibri
    sudo mount /dev/disk/by-id/ata-Netac_SSD_720GB_AA000000000000000591-part4 /mnt/kolibri

download the [latest distro](http://builds.kolibrios.org/eng/) and extract it to the partition:

    wget http://builds.kolibrios.org/eng/latest-distr.7z
    sudo 7z x -o/mnt/kolibri latest-distr.7z

if you don't have the 7z command, install p7zip. if you don't have wget, install wget

install syslinux through your package manager and copy its memdisk to the kolibri partition

    sudo cp /usr/lib/syslinux/bios/memdisk /mnt/kolibri/

edit `/etc/grub.d/40_custom` with your favorite text editor (as root) and add the following:

    menuentry "KolibriOS" {
      insmod part_msdos
      set root='(hd0,4)'
      search --no-floppy --fs-uuid --set=root XXXX-XXXX
      linux16 /memdisk iso
      initrd16 /kolibri.img
    }

change the hd0,4 part to your drive and partition number. 0,4 means drive 0, partition 4, so sda4

find your kolibri partition UUID with `sudo blkid` and replace the XXXX-XXXX with the UUID. this
will find your partition even if the disk/partition indices change

now update the grub config

    sudo update-grub

you should now be able to reboot and selecy KolibriOS in grub and it should boot up fine

have fun!

to save any changes to the ramdisk, run RDSAVE and select the location of your kolibri.img .
your drive should be one of the hd*/*/

# radeon drivers and others

right click desktop -> open system panel -> driver installer

to load the driver at boot, edit /sys/settings/autorun.dat and add this line

    /kolibrios/drivers/atikms/atikms -l/kolibrios/atikms.log 0

then use RDSAVE to save it to /hd*/*/kolibri.img (browse to wherever your kolibri.img is located)

by default it will set the same resolution you set in the blue screen at boot, but you can change
it by adding a parameter such as `-m1920x1080x60` after the log file parameter

# UEFI install
NOTE: the uefi install didn't actually work for me, these are the steps I attempted

as with the bios install, make a fat32 partition for kolibri (refer to the bios guide) and extract
the KolibriOS files to it

as with the bios install, I assume you already installed linux on the target drive

mount your kolibri partition as demonstrated in the BIOS install

I am assuming you have a grub install and /boot contains your linux kernels and grub folder

edit /mnt/kolibri/boot/kolibri.ini (as root)

put the following in it (adjust to your liking)

    ; Place this file to ESP:/EFI/BOOT/kolibri.ini, near kolibri.efi and kolibri.krn

    ; Screen resolution
    resolution=1920x1080

    ; Duplicate debug output to the screen
    debug_print=0

    ; Start LAUNCHER app after kernel is loaded
    launcher_start=1

    ; Configure MTRR's
    mtrr=1

    ; 3: use ramdisk loaded from kolibri.img
    ; 5: don't use ramdisk
    imgfrom=3

    ; Path to /sys directory
    syspath=/rd/1

    ; Interrupt booting to ask the user for boot params
    ask_params=0

now do:

    wget http://builds.kolibrios.org/eng/data/data/kolibri.img
    wget http://builds.kolibrios.org/eng/data/kernel/trunk/bootloader/uefi4kos/kolibri.efi
    wget http://builds.kolibrios.org/eng/data/kernel/trunk/kolibri.krn
    sudo mv kolibri.* /mnt/kolibri/boot/
    sudo umount /mnt/kolibri

it's fine if you get ownership errors

now boot into linux on the target drive.

assuming you use grub2, add this to `/etc/grub.d/40_custom`

    menuentry "KolibriOS" {
      insmod part_gpt
      insmod chain
      chainloader /boot/kolibri.efi
    }

now do

    sudo update-grub

you should now be able to boot KolibriOS from grub

... except it just blackscreens. apparently you need to generate a devices.dat files by booting
an image with the [devman](http://ftp.kolibrios.org/users/Serge/new/ACPI/devman.zip) utility
and running it, but it doesn't generate anything on the bios install for me
