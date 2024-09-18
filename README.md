# rpi raid

this document describes how to move the root directory of a RPI from the sd card onto a RAID array

## config

I enable pcie gen 3 using raspi-config (advanced section)

`sudo raspi-config`

## update firmware

`sudo rpi-eeprom-update`

## update, upgrade, install mdadm

mdadm is used to create a SW RAID array
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install mdadm
```
## create md (multiple device) disk

find the disks you want to add to your md device and change the second line accordingly
```
ls -al /dev/disk/by-path/
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/nvme0n1 /dev/nvme1n1
```
check the progress of creation, i'd suggest waiting until building is finished

´watch -n 3 cat /proc/mdstat`

## save md config

```
sudo su
mdadm --detail --scan /dev/md0 >> /etc/mdadm/mdadm.conf
exit
```

## format md device as ext4

`sudo mkfs.ext4 /dev/md0`

## rebuild initramfs

### load some modules into kernel for md

these modules will be included when building the init ram fs so the md can be mounted

`sudo nano /etc/initramfs-tools/modules`

and add
```
raid1
raid0
md_mod
ext4
```
### update initial ram filesystemm

up until now i've been able to just update:

`sudo update-initramfs -u`

## copy the root fs to the md device

### mount md device
```
sudo mkdir /mnt/clone
sudo mount /dev/md0 /mnt/clone
```
### mount rootfs 

get device from `/etc/mtab` or `ls -al /dev/disk/by-path/` and change second command accordingly

```
sudo mkdir /mnt/rootfs
sudo mount /dev/mmcblk0p2 /mnt/rootfs
```
### sync rootfs to clone

`sudo rsync -axv /mnt/rootfs/* /mnt/clone`

## update bootloader params

`sudo nano /boot/firmware/cmdline.txt`

root= becomes your md device, add rootddelay=5 (or larger) if md takes a while to be built

`console=serial0,115200 console=tty1 root=/dev/md0 rootfstype=ext4 fsck.repair=yes rootwait rootdelay=5`

## edit fstab on the clone to mount your md device on /

`sudo nano /mnt/clone/etc/fstab`

change the line to match your md device

`/dev/md0        /               ext4    defaults,noatime  0       1`

## BONUS: swapfile

striping and mirroring using nvme allows for high speed data transfer, since i'm runing some resource intensive services i've increased the size of my swap to 32GB

turn swap off and edit config

```
sudo dphys-swapfile swapoff
sudo nano /etc/dphys-swapfile
```

change config to

```
CONF_SWAPFACTOR=4
CONF_MAXSWAP=32768
```

turn swap back on

`sudo dphys-swapfile swapon`