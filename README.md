# RHEL Study Guide: Manage Storage Stack

[RHEL Study Guide - Table of Contents](https://github.com/pslucas0212/RHEL-Study-Guide) 

High level steps for LVM extension/creation
1. lsblk - Identify unused disks with lsblk command.  lsblk list block devices connected to your system
2. parted print - Confirm a disk does not have a label with parted /dev/vdb unit MiB print
3. Make a partition to expand the LVM - parted /dev/vdb mkpart primary 514MiB 1024MiB
4. Set the new partition to LVM - parted /dev/vdb set 2 lvm on
5. Register new partition with kernel - udevadm settle
6. Initialize the physical volume for use with LVM - pvcreate /dev/vdb2
7. Extend the volume group to the new partition - vgextend serverb_01_vg /dev/vdb2
8. Extend the the logical volume - lvextend -L 768M /dev/serverb_01_vg/serverb_01_lv   The -L switch is used to pass the size of to grow
9. Extend the file system format to the new space on the lvm - xfs_growfs /storage/data1
10. Create a new logical volume in an existing volume group - lvcreate -n serverb_02_lv -L 128M serverb_01_vg
11. Create file system on new lvm - mkfs -t xfs /dev/serverb_01_vg/serverb_02_lv
12. Make new mount point - mkdir /storage/data2
13. Update fstab - /dev/serverb_01_vg/serverb_02_lv    /storage/data2    xfs   defaults   0 0
14. Reload deamon - systemctl daemon-reload
15. Mount nvew volume - mount /storage/data2
16. Check size with df -h /storage/datax or lvdisplay /dev/serverb_01_vg/server_0x_lv

In this example we have an existing volume group with a defined logical volume.  We the logical volume is running out of space and the volume group has space we can add to the current logical volume

Let's get the layout of the land so to speak
```
# lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda                             252:0    0   10G  0 disk 
├─vda1                          252:1    0    1M  0 part 
├─vda2                          252:2    0  200M  0 part /boot/efi
├─vda3                          252:3    0  500M  0 part /boot
└─vda4                          252:4    0  9.3G  0 part /
vdb                             252:16   0    5G  0 disk 
└─vdb1                          252:17   0  512M  0 part 
  └─serverb_01_vg-serverb_01_lv 253:0    0  256M  0 lvm  /storage/data1
vdc                             252:32   0    5G  0 disk 
vdd                             252:48   0    5G  0 disk 
```

Now let's see in MiB where the first partition ends
```
# parted /dev/vdb unit MiB print
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 5120MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start    End     Size    File system  Name     Flags
 1      1.00MiB  513MiB  512MiB               primary
```

Let's add or create an additional 512MIB partition set that partition as a Logical Volume.  We create a primary partion whic becomes the second partition on /dev/vdb and we set the that partition to have lvm on
```
# parted /dev/vdb mkpart primary 514MiB 1026MiB
Information: You may need to update /etc/fstab.
# parted /dev/vdb set 2 lvm on                            
Information: You may need to update /etc/fstab.
# udevadm settle
```
Next use the pvcreate command to initialize a block device to be used as a physical volume.
```
# pvcreate /dev/vdb2
  Physical volume "/dev/vdb2" successfully created.
```

Let's add or extend that partition to existing volume group
```
# vgextend serverb_01_vg /dev/vdb2
  Volume group "serverb_01_vg" successfully extended
```

Now we will grow or extend the existing Logical volume to our new target size.  We use -L to define the logical volume size
```
# lvextend -L 768M /dev/serverb_01_vg/serverb_01_lv
  Size of logical volume serverb_01_vg/serverb_01_lv changed from 256.00 MiB (64 extents) to 768.00 MiB (192 extents).
  Logical volume serverb_01_vg/serverb_01_lv successfully resized.
```

Usese the xfs_growfs command to increase the size of a mounted XFS file system.  Or we extend the XFS file system to the remaining space on the logical volume
```
# xfs_growfs /storage/data1
meta-data=/dev/mapper/serverb_01_vg-serverb_01_lv isize=512    agcount=4, agsize=16384 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1
data     =                       bsize=4096   blocks=65536, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=1368, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 65536 to 196608
```
All done!


Now let's create a second logical volume on the existing volume group
