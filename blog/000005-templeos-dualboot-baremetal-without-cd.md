this is the simplest way to install TempleOS to dualboot with linux on bare-metal without a cd/dvd
drive

if you want to hang out with other TempleOS users, come join our community on matrix:
https://chat.tchncs.de/#/room/#TempleOS:matrix.kiwifarms.net

# requirements
you need a linux machine with qemu installed (doesn't need to be the target machine) and an extra
hdd/sdd to install TempleOS + linux on.

your target machine must support legacy boot, so if you have a modern motherboard that uses uefi
you need to make sure it's configured to also boot legacy stuff.

you will also need a ps/2 mouse and keyboard to use in TempleOS.

# preparing the partitions
before you start, boot any live or non-live linux on the target machine and do

    lspci -v

look for something like this in the output:

    00:1f.2 SATA controller: Intel Corporation 6 Series/C200 Series Chipset Family 6 port Desktop SATA AHCI Controller (rev 05) (prog-if 01 [AHCI 1.0])
            Subsystem: Intel Corporation 6 Series/C200 Series Chipset Family 6 port Desktop SATA AHCI Controller
            Flags: bus master, 66MHz, medium devsel, latency 0, IRQ 31
            I/O ports at f070 [size=8]
            I/O ports at f060 [size=4]
            I/O ports at f050 [size=8]
            I/O ports at f040 [size=4]
            I/O ports at f020 [size=32]
            Memory at fbf05000 (32-bit, non-prefetchable) [size=2K]
            Capabilities: <access denied>
            Kernel driver in use: ahci

write down those I/O ports for later.

now, connect the blank ssd/hdd to a linux machine and start setting up the partitions. this is also
the part where you install linux, so ideally you would boot into your preferred linux install
environment (for me it's just my main artix linux install).

first, you need to set up 2 partitions for templeos in the first 8 gigs of the drive. rest can
be used for linux or whatever. you must use legacy boot.

I'm doing these steps from an artix linux live iso. I'm assuming you are root, otherwise use sudo

WARNING: this wipes your data! double and triple check that you are doing this on the right drive

wipe partition table if the ssd/hdd is not blank:

    dd if=/dev/zero of=/dev/disk/by-id/your-disk-id-here count=10 && sync

start partitioning:

    cfdisk /dev/disk/by-id/your-disk-id-here

* select dos and press enter
* select new and press enter
* change partition size to 4G and press enter
* select primary and press enter
* select type and press enter
* select "88 Linux plaintext" and press enter. this is for the redsea TempleOS partition
* press down to select the free space, select new and press enter
* change partition size to 4G and press enter
* select primary and press enter
* select type and press enter
* select "b W95 FAT32" and press enter. this is for the fat32 TempleOS partition
* press down to select the free space, select new and press enter
* press enter to confirm the size, which should take up the rest of the space
* select primary and press enter. this will be your linux partiton
* if you want more than one linux partition, adjust to your liking
* select write and press enter
* type yes and press enter to confirm
* select quit and press enter

format and mount the linux partition:

    mkfs.ext4 /dev/disk/by-id/your-disk-id-here-part3
    mount /dev/disk/by-id/your-disk-id-here-part3 /mnt

install linux normally, in my case I would continue from after the mount partitions part here:
https://wiki.artixlinux.org/Main/Installation

I only tested grub as the bootloader, so I recommend using that

NOTE: you can also install linux from a VM by passing through the disk to qemu similarly to how
we do the TempleOS install from below, but it's best to do it from the target machine to avoid
incompatibilities

NOTE: the linux install must use a MBR bootloader, no EFI

to avoid incorrect clock on TempleOS, set linux to use localtime. this varies between distributions

# installing TempleOS

the following steps can be done either from a different machine or from the target machine itself
using the linux install we set up in the previous steps

some qemu tips before you start:
* CTRL+ALT+G releases the mouse/keyboard from qemu
* CTRL+ALT+T opens a new terminal in TempleOS
* CTRL+ALT+X closes the current terminal
* CTRL+ALT+C stops the current task

start qemu with the TempleOS iso and your disk as the drive
(change your-disk-here and the path to the templeos iso)

    sudo qemu-system-x86_64 \
      -enable-kvm -cpu host -smp 4 -m 1G \
      -soundhw pcspk \
      -rtc base=localtime -machine kernel_irqchip=off \
      -drive if=ide,format=raw,file=/dev/disk/by-id/ata-YOUR-DISK-HERE,cache=none,format=raw,aio=native \
      -cdrom '/path/to/TempleOS.ISO' \
      -boot d

* press Y to install to hard drive
* say no when it asks if you're installing from a VM
* y to continue install wizard
* press enter
* enter the auto detected i/o ports for the hard drive (NOT the ones from lspci)
* select master
* destination letter: C
* y to format
* select redsea as the file system
* y to install the boot loader
* when it asks to reboot, say n
* when it asks to take tour, say n

type this and press enter:

    BootHDIns('C');

* boot drive C
* drive letter C
* partition number (enter for all)
* s to skip probing

remember the lspci output we saved earlier? enter 2 of the I/O ports from there, prefixed with 0x.
this will take trial and error. just keep trying different combinations until it boots, for me it
was the ones with size=8, so 0xF070 and 0xF050 . order matters, try all combinations. depending on
which sata port it's connected to, it might want different i/o ports

note: if your bios has the option, you can change your sata to operate in IDE mode, and then check
the I/O ports again in case they changed and try again.

switching the compatible/enhanced setting and disabling one of the two controllers might also help

* select master
* enter to exit
* enter
* enter
* kill the virtual machine.
* connect the drive to your target machine and see if it boots. if it doesn't, repeat these steps
  with different i/o ports. some modern motherboard just will not work, but many should

once your install works, boot into it and do

    Fmt('D');

Y to confirm, this will format the other partition to FAT32

let's now copy the install to the FAT partition

    CopyTree("C:/","D:/");
    BootHDIns('D');

entering the same info you did for `BootHDIns('C')` except swap every C with D

if D is not mounted for whatver reason you can always mount with `Mount;`

now do

    BootMHDIns('C',"CD");

to regenerate the bootloader with both C and D as bootable TempleOS installs

now `Reboot;` and test both installs as well as the old mbr option which should boot into linux

note that BootMHDIns generates a backup of your old MBR at `C:\0000Boot\OldMBR.BIN.C` if it doesnt
already exist. if you delete it, you won't be able to boot into linux anymore. if that happens you
will have to reinstall grub and then reinstall the TempleOS bootloader on top of it

# mounting the TempleOS partition at boot on linux

find out the UUID of your FAT32 TempleOS partition

    sudo blkid | grep vfat

create the mountpoint

    sudo mkdir /tos

add it to /etc/fstab (change UUID=xxx to your uuid)

    echo "UUID=xxx /tos vfat user,owner,rw,umask=000 0 0" | sudo tee -a /etc/fstab

mount it

    sudo mount -a

your TempleOS files are now accessible at /tos

# installing supplementals

boot into linux and get your supplemental iso's from whatever source you prefer

copy the ISO's (or ISO.C) files to your TempleOS partition which should be at /tos

    cp TOS_Supplementals.ISO.C /tos/Sup1.ISO.C

make sure to rename the files so they don't have unsupported characters such as `:`

reboot into TempleOS

    MountFile("D:/Sup1.ISO.C");
    DrvRep;

the DrvRep should show the iso mounted at M. change the letter in the following commands if it
doesn't match

    CopyTree("M:/","C:/Home/Sup1");
    Umount('M');

repeat for the other ISO's

# changing framerate to 60fps

type `WINMGR_FPS` and press CTRL+SHIFT+F1 to jump to the code for the first autocomplete result

edit the two `30000` 's to `60000`

`Reboot;`
