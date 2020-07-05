# headerless wii linux // ansible deploys
mucking around with linux on a nuber of old wii's to boot them headerless with ssh up, this repo contains that configuration

## some notes
 - you need a modified wii
 - rootfs is hardcoded in p1/gumboot/zImage.ngx to p2, can use baedit to modifiy this according to project repo
 - the archive.debian repos are added as [trusted=yes], their gpgkeys are expired and haven't been renewed so can't be updated
 - re-enabled root pass login over ssh on initial deploy, you should create a nonpriv user and disable this later on.
 - wireless will try to come up on boot configured for dhcp
 - root passwd as per main project

## using 
 - https://neagix.github.io/wii-linux-ngx/
 - https://github.com/neagix/wii-linux-ngx/releases

### On a linux machine
1. download the image
``` curl -oL wii-jessie-sd.img.xz https://github.com/neagix/wii-linux-ngx/releases/download/0.3.6/wii-jessie-sd.img.xz ```
1. unpack the images
``` $ xz -d wii-jessie-sd.img.xz ```
1. image the sd card
``` $ dd if=wii-jessie-sd.img of=<sdcard device> bs=1M ```
1. expand the second partition and add a 3rd for swap (this is for a 4G card, my sdcard device was /dev/sdf)
```
$ /sbin/parted -s -a optimal -- <sdcard device> unit MB rm 2 mkpart primary ext3 43 3700 2
$ /sbin/parted -s -a optimal -- <sdcard device> unit MB mkpart primary linux-swap 3700 -1 3
$ /sbin/e2fsck -f <sdcard device>2
$ /sbin/resize2fs <sdcard device>2 
$ /sbin/mkswap <sdcard device>3
```

### Update the configuration before copying on the sdcard
1. add your own wpa_supplicant.conf so wlan1 can come up during boot
1. cp this reconfig archive to the WII-LINUX partition on the sdcard
``` sudo cp -R config/* <WII-LINUX> ```

### On the wii
1. boot the wii with the sdcard

### find and ssh into the wii as the wii user
1. use whatever method (nmap, check router, etc) to find the wii-linux-ngx host
2. ssh into the wii
``` ssh root@<wiihost> ```

# bonus -- deploy to the wii with ansible (because why not)
a basic playbook to make some changes to the wii's is included in the cluster folder

### Install the dependancies on the wii over ssh
1. login as the root user
1. install the required packages for ansible to the wii
```
# apt-get update
# apt-get -y install python
```

### create a nonpriv user to work with
(optional) you can do all this stuff as root if you really want
1. create a nonpriv user wii
``` useradd -m -s /bin/bash wii ```
1. create a password for the user
``` passwd wii ```
1. install the sudo package
```
# apt-get update
# apt-get -y install sudo
```
1. create a sudoers file for the user in ```/etc/sudoers.d/01-wii```
```
echo "wii	ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers/01-wii
```

### On a machine
1. test the login on another machine
``` ssh wii@<wiihost> ```
1. escalate to root
``` $ sudo -i ```
1. (optional) install a ssh key
``` $ ssh-copy-id wii@<host> ```
1. deploy to the wii inventory items with ansible
``` ansible-playbook playbook.yml -u root ```


the playbook with install and start nginx atm

thats it, a bit of fun.
- sairuk

# the maybe pile
 - find something useful to do with this.
 - move root partition to p3 so it can be resized for whatever sd card
 - automate/script some of these sd operations




# modify the image to move the root partition to the end of the disk

(theory only) this process is untested to boot on the wii

## Starting image
```
~$ parted -s -a optimal --  wii-jessie-sd.img unit s print free
Model:  (file)
Disk /home/sairuk/wii-jessie-sd.img: 819200s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End      Size     Type     File system  Flags
        63s     2047s    1985s             Free Space
 1      2048s   83967s   81920s   primary  fat32
 2      83968s  819199s  735232s  primary  ext3
```

## Add free space to the end of the image
* Add the free space
``` dd if=/dev/zero bs=1M count=700 >> wii-jessie-sd.img ```
* Create a partition
``` parted -s -a optimal -- wii-jessie-sd.img unit MB mkpart primary ext3 419 $((419+376+1)) ```


## New disk image layout
```
$ parted -s -a optimal -- wii-jessie-sd.img unit s print free
Model:  (file)
Disk /home/sairuk/wii-jessie-sd.img: 1638400s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start     End       Size     Type     File system  Flags
        63s       2047s     1985s             Free Space
 1      2048s     83967s    81920s   primary  fat32
 2      83968s    819199s   735232s  primary  ext3
 3      819200s   1554431s  735232s  primary
        1554432s  1638399s  83968s            Free Space
```

## Start moving data around since move and cp were removed from parted 3.2
* Create loop devices for each partition
```
sudo losetup -o $((2048*512)) /dev/loop1001 wii-jessie-sd.img 
sudo losetup -o $((83968*512)) /dev/loop1002 wii-jessie-sd.img 
sudo losetup -o $((819200*512)) /dev/loop1003 wii-jessie-sd.img 
```
* Make an ext3 file system on the new partition
``` sudo mkfs.ext3 /dev/loop1003 ```
* Make the mount point dirs
``` sudo mkdir -p /mnt/tmp/wiilxp{1,2,3} ```
* Mount the loops devices for access
``` 
sudo mount /dev/loop1001 /mnt/tmp/wiilxp1
sudo mount /dev/loop1002 /mnt/tmp/wiilxp2
sudo mount /dev/loop1003 /mnt/tmp/wiilxp3
```
* rsync the data from the old root to new
```sudo rsync -avzP /mnt/tmp/wiilxp2/* /mnt/tmp/wiilxp3/```
* Make working copy of the kernel before we modify it
```
sudo cp /mnt/tmp/wiilxp1/gumboot/zImage.ngx /mnt/tmp/wiilxp1/gumboot/zImagep2.ngx
```
* Start to modify the known references to the old root partition and point it to the new one
```
sudo sed -i 's/mmcblk0p2/mmcblk0p3/' /mnt/tmp/wiilxp1/gumboot/zImagep2.ngx
sudo sed -i 's/zImage/zImagep2/' /mnt/tmp/wiilxp1/gumboot/gumboot.lst
sudo sed -i 's/mmcblk0p2/mmcblk0p3/' /mnt/tmp/wiilxp3/etc/fstab
```
* Add the new swap partition to fstab
```sudo echo '/dev/mmcblk0p3  swap   swap   defaults	                                        0 0' >> /mnt/tmp/wiilxp3/etc/fstab```
* Unmount all the partition mounts
```sudo umount /mnt/tmp/wiilxp*```
* Convert the second partition to swap
```sudo mkswap /dev/loop1002```
* Destroy the loop devices that are no longer needed
```sudo losetup -d /dev/loop100{1,2,3}```

## Final image layout
```
$ parted -s -a optimal -- wii-jessie-sd.img unit s print free
Model:  (file)
Disk /home/sairuk/wii-jessie-sd.img: 1638400s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start     End       Size     Type     File system     Flags
        63s       2047s     1985s             Free Space
 1      2048s     83967s    81920s   primary  fat32
 2      83968s    819199s   735232s  primary  linux-swap(v1)
 3      819200s   1554431s  735232s  primary  ext3
        1554432s  1638399s  83968s            Free Space
```
