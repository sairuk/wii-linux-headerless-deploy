# headerless wii linux // ansible deploys
mucking around with linux on a nuber of old wii's to boot them headerless with ssh up, this repo contains that configuration

## some notes
 - you need a modified wii
 - rootfs is hardcoded in p1/gumboot/zImage.ngx to p2, can use baedit to modifiy this according to project repo
 - the archive.debian repos are added as [trusted=yes], their gpgkeys are expired and haven't been renewed so can't be updated
 - re-enabled root pass login over ssh on initial deploy, you should create a nonpriv user and disable this later on.
 - wireless will try to come up on boot configured for dhcp
 - root passwd as per main project
 - priiloader autoboot to bootmii ios works perfectly well for this

## using 
 - https://neagix.github.io/wii-linux-ngx/
 - https://github.com/neagix/wii-linux-ngx/releases

### On a machine
1. download the image
``` curl -oL wii-jessie-sd.img.xz https://github.com/neagix/wii-linux-ngx/releases/download/0.3.6/wii-jessie-sd.img.xz ```
1. unpack the images
``` $ xz -d wii-jessie-sd.img.xz ```
1. image the sd card
``` $ dd if=wii-jessie-sd.img of=<sdcard device> bs=1M ```
1. expand the second partition and add a 3rd for swap (this is for a 4G card, my sdcard device was /dev/sdf)
```
$ parted -s -a optimal -- <sdcard device> unit MB rm 2 mkpart primary ext3 43 3700 2
$ parted -s -a optimal -- <sdcard device> unit MB mkpart primary linux-swap 3700 -1 3
$ e2fsck -f <sdcard device>2
$ resize2fs <sdcard device>2 
$ mkswap <sdcard device>3
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
``` # useradd -m -s /bin/bash wii ```
1. create a password for the user
``` # passwd wii ```
1. install the sudo package
```
# apt-get update
# apt-get -y install sudo
```
1. create a sudoers file for the user in ```/etc/sudoers.d/01-wii```
```
# echo "wii	ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers/01-wii
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
 - ~~move root partition to p3 so it can be resized for whatever sd card~~
 - automate/script some of these sd operations




# modify the image to move the root partition to the end of the disk

~~(theory only) this process is untested to boot on the wii.~~ this now works but note if you follow this process you'll need to fix the fstab once it's copied over as part of the above process.

## Starting image
```
# parted -s -a optimal --  wii-jessie-sd.img unit B print free
Model:  (file)
Disk /home/user/wii-jessie-sd.img: 419430400B
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start      End         Size        Type     File system  Flags
        32256B     1048575B    1016320B             Free Space
 1      1048576B   42991615B   41943040B   primary  fat32
 2      42991616B  419430399B  376438784B  primary  ext3
```

## Add free space to the end of the image
* Add the free space
```# dd if=/dev/zero bs=512 count=819200 >> wii-jessie-sd.img ```
* Create a partition
```# parted -s -a optimal -- wii-jessie-sd.img unit B mkpart primary ext3 419430400 795869183 ```


## New disk image layout
```
# parted -s -a optimal -- wii-jessie-sd.img unit B print free
Model:  (file)
Disk /home/user/wii-jessie-sd.img: 838860800B
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start       End         Size        Type     File system     Flags
        32256B      1048575B    1016320B             Free Space
 1      1048576B    42991615B   41943040B   primary  fat32
 2      42991616B   419430399B  376438784B  primary  linux-swap(v1)
 3      419430400B  795869183B  376438784B  primary  
        795869184B  838860799B  42991616B            Free Space
```

### Start moving data around since move and cp were removed from parted 3.2
* Create loop devices for each partition
```
# losetup -o 1048576   --sizelimit 42991615  /dev/loop1001 wii-jessie-sd.img
# losetup -o 42991616  --sizelimit 419430399 /dev/loop1002 wii-jessie-sd.img
# losetup -o 419430400 --sizelimit 795869183 /dev/loop1003 wii-jessie-sd.img

```
* Make the mount points
```# mkdir -p /mnt/tmp/wiilxp{1,2,3} ```
* Mount the loops devices for access
``` 
# mount /dev/loop1001 /mnt/tmp/wiilxp1
# mount /dev/loop1002 /mnt/tmp/wiilxp2
# mount /dev/loop1003 /mnt/tmp/wiilxp3
```
* Clone partition 2 to partition 3
```# dd if=/dev/loop1002 of=/dev/loop1003 bs=1M ```
* Make working copy of the kernel before we modify it
```
# cp /mnt/tmp/wiilxp1/gumboot/zImage.ngx /mnt/tmp/wiilxp1/gumboot/zImagep2.ngx
```
* Start to modify the known references to the old root partition and point it to the new one
```
# sed -i 's/mmcblk0p2/mmcblk0p3/' /mnt/tmp/wiilxp1/gumboot/zImagep2.ngx
# sed -i 's/zImage/zImagep2/' /mnt/tmp/wiilxp1/gumboot/gumboot.lst
# sed -i 's/mmcblk0p2/mmcblk0p3/' /mnt/tmp/wiilxp3/etc/fstab
```
* Add the new swap partition to fstab
```# echo '/dev/mmcblk0p3  swap   swap   defaults	                                        0 0' | tee -a /mnt/tmp/wiilxp3/etc/fstab```
* Unmount all partitions
```# umount /mnt/tmp/wiilxp*```
* Convert the second partition to swap
```# mkswap /dev/loop1002```
* Destroy the loop devices
```# losetup -d /dev/loop100{1,2,3}```

### Final image layout
```
$ parted -s -a optimal --  wii-jessie-sd.img unit B print free
Model:  (file)
Disk /home/user/wii-jessie-sd.img: 838860800B
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start       End         Size        Type     File system     Flags
        32256B      1048575B    1016320B             Free Space
 1      1048576B    42991615B   41943040B   primary  fat32
 2      42991616B   419430399B  376438784B  primary  linux-swap(v1)
 3      419430400B  795869183B  376438784B  primary  ext3
        795869184B  838860799B  42991616B            Free Space
```

### Resizing the root paritition to use the remaining space
Use MB units so we can use -1 to fill remaiming space
```
# parted -s -a optimal -- <sdcard device> unit MB rm 3 mkpart primary ext3 419 -1 3
# e2fsck -f <sdcard device>3
# resize2fs <sdcard device>3 
```


