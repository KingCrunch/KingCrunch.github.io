# sudo fdisk /dev/mmcblk0

As instructed

sudo mkfs.vfat /dev/mmcblk0p1
sudo mkfs.btrfs /dev/mmcblk0p2

mkdir boot btrfs
cd btrfs
sudo btrfs subvolume create @ # /
sudo btrfs subvolume create @home # /home

bsdtar -xpf ArchLinuxARM-rpi-2-latest.tar.gz -C btrfs/@/

# Some "failed to set file flags" messages appear. Havent seen any issues with that yet...

sudo mv btrfs/@/boot/* boot

# in boot/cmdline.txt
root=/dev/mmcblk0p2 rootfstype=btrfs rootflags=subvol=@ 

# in btrfs/@/etc/fstab 

/dev/mmcblk0p2 /   btrfs   subvol=@,rw,noatime,ssd,space_cache,compress=lzo 0   1
/dev/mmcblk0p2 /home   btrfs   subvol=@home,rw,noatime,ssd,space_cache,compress=lzo        0   1

# To allow mounting the btrfs partition directly

sudo mkdir btrfs/@/mnt/btrfs

# and in btrfs/@/etc/fstab 
/mnt/btrfs   btrfs   rw,noatime,ssd,space_cache,compress=lzo,noauto      0   1

# Notice "noauto"

# Ensure everything is written
sync

sudo umount boot btrfs



# edit /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
# edit /etc/systemd/network/20-wirless.network

# stil has to enable and start wpa_supllicant@wlan0
