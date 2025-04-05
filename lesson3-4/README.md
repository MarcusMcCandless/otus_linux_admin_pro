## Занятие 3-4. Файловые системы и LVM

**Домашнее задание**
### 1. Создать Physical Volume, Volume Group и Logical Volume
```
root@ubuntuserv2402:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0 11,5G  0 lvm  /
sdb                         8:16   0    4G  0 disk 
├─sdb1                      8:17   0  200M  0 part 
├─sdb2                      8:18   0  200M  0 part 
├─sdb3                      8:19   0  200M  0 part 
└─sdb4                      8:20   0  200M  0 part 

root@ubuntuserv2402:~# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.

root@ubuntuserv2402:~# vgcreate otus /dev/sdb1 
  Volume group "otus" successfully created

root@ubuntuserv2402:~# lvcreate -l+80%FREE -n test otus
  Logical volume "test" created.

root@ubuntuserv2402:~# vgdisplay otus
  --- Volume group ---
  VG Name               otus
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               196,00 MiB
  PE Size               4,00 MiB
  Total PE              49
  Alloc PE / Size       39 / 156,00 MiB
  Free  PE / Size       10 / 40,00 MiB
  VG UUID               2Sz1Wz-kKK1-Tifp-omez-4am2-YMFh-TiC9c3

root@ubuntuserv2402:~# vgdisplay -v otus | grep 'PV Name'
  PV Name               /dev/sdb1

root@ubuntuserv2402:~# lvdisplay /dev/otus/test
  --- Logical volume ---
  LV Path                /dev/otus/test
  LV Name                test
  VG Name                otus
  LV UUID                0mXpah-VZL2-vhCy-PnXB-GrEH-HPpa-S3035K
  LV Write Access        read/write
  LV Creation host, time ubuntuserv2402, 2025-03-31 16:16:50 +0000
  LV Status              available
  # open                 0
  LV Size                156,00 MiB
  Current LE             39
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:1

root@ubuntuserv2402:~# vgs
  VG        #PV #LV #SN Attr   VSize   VFree 
  otus        1   1   0 wz--n- 196,00m 40,00m
  ubuntu-vg   1   1   0 wz--n- <23,00g 11,50g

root@ubuntuserv2402:~# lvs
  LV        VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test      otus      -wi-a----- 156,00m                                                    
  ubuntu-lv ubuntu-vg -wi-ao---- <11,50g

root@ubuntuserv2402:~# lvcreate -L20M -n small otus
  Logical volume "small" created.
```

### 3. Отформатировать и смонтировать файловую систему

```
root@ubuntuserv2402:~# mkfs.ext4 /dev/otus/test 
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 39936 4k blocks and 39936 inodes
Filesystem UUID: a4c43814-459a-4344-a10b-b29b0fbdfc30
Superblock backups stored on blocks: 
	32768

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

root@ubuntuserv2402:~# mkdir /data
root@ubuntuserv2402:~# mount /dev/otus/test /data/
root@ubuntuserv2402:~# mount | grep /data
/dev/mapper/otus-test on /data type ext4 (rw,relatime)
```

#### 4, 5. Расширить файловую систему за счёт нового диска, выполнить resize

```
root@ubuntuserv2402:~# pvcreate /dev/sdb2 
  Physical volume "/dev/sdb2" successfully created.

root@ubuntuserv2402:~# vgextend otus /dev/sdb2
  Volume group "otus" successfully extended

root@ubuntuserv2402:~# vgdisplay -v otus | grep 'PV Name'
  PV Name               /dev/sdb1     
  PV Name               /dev/sdb2     

root@ubuntuserv2402:~# vgs
  VG        #PV #LV #SN Attr   VSize   VFree  
  otus        2   2   0 wz--n- 392,00m 216,00m
  ubuntu-vg   1   1   0 wz--n- <23,00g  11,50g

root@ubuntuserv2402:~# dd if=/dev/zero of=/data/test.log bs=1M status=progress
dd: error writing '/data/test.log': No space left on device
127+0 records in
126+0 records out
133091328 bytes (133 MB, 127 MiB) copied, 0,117248 s, 1,1 GB/s

root@ubuntuserv2402:~# df -Th /data
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  131M  127M     0 100% /data

root@ubuntuserv2402:~# lvextend -l+80%FREE /dev/otus/test
  Size of logical volume otus/test changed from 156,00 MiB (39 extents) to 332,00 MiB (83 extents).
  Logical volume otus/test successfully resized.

root@ubuntuserv2402:~# lvs /dev/otus/test
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test otus -wi-ao---- 332,00m   

# Но файловая система при этом осталась прежнего размера:
root@ubuntuserv2402:~# df -Th /data
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  131M  127M     0 100% /data

root@ubuntuserv2402:~# resize2fs /dev/otus/test
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/otus/test is mounted on /data; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/otus/test is now 84992 (4k) blocks long.


root@ubuntuserv2402:~# df -Th /data
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  302M  127M  162M  45% /data

root@ubuntuserv2402:~# umount /data/

# Проверка корректности
root@ubuntuserv2402:~# e2fsck -fy /dev/otus/test
e2fsck 1.47.0 (5-Feb-2023)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/otus/test: 12/59904 files (8.3% non-contiguous), 40388/84992 blocks

```

### Уменьшение lv

```
root@ubuntuserv2402:~# resize2fs /dev/otus/test 250M
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/otus/test to 64000 (4k) blocks.
The filesystem on /dev/otus/test is now 64000 (4k) blocks long.

root@ubuntuserv2402:~# lvreduce /dev/otus/test -L 250M
  Rounding size to boundary between physical extents: 252,00 MiB.
  WARNING: Reducing active logical volume to 252,00 MiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce otus/test? [y/n]: y
  Size of logical volume otus/test changed from 332,00 MiB (83 extents) to 252,00 MiB (63 extents).
  Logical volume otus/test successfully resized.

root@ubuntuserv2402:~# mount /dev/otus/test /data/

root@ubuntuserv2402:~# df -Th /data/
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  225M  127M   85M  61% /data

root@ubuntuserv2402:~# lvs /dev/otus/test
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test otus -wi-ao---- 252,00m                                                 
```

### Работа со снапшотами

```
root@ubuntuserv2402:~# lvcreate -L 100M -s -n test-snap /dev/otus/test

root@ubuntuserv2402:~# vgs -o +lv_size,lv_name
  VG        #PV #LV #SN Attr   VSize   VFree  LSize   LV       
  otus        2   3   1 wz--n- 392,00m 20,00m 252,00m test     
  otus        2   3   1 wz--n- 392,00m 20,00m  20,00m small    
  otus        2   3   1 wz--n- 392,00m 20,00m 100,00m test-snap
  ubuntu-vg   1   1   0 wz--n- <23,00g 11,50g <11,50g ubuntu-lv

root@ubuntuserv2402:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0 11,5G  0 lvm  /
sdb                         8:16   0    4G  0 disk 
├─sdb1                      8:17   0  200M  0 part 
│ ├─otus-small            252:2    0   20M  0 lvm  
│ └─otus-test-real        252:3    0  252M  0 lvm  
│   ├─otus-test           252:1    0  252M  0 lvm  
│   └─otus-test--snap     252:5    0  252M  0 lvm  
├─sdb2                      8:18   0  200M  0 part 
│ ├─otus-test-real        252:3    0  252M  0 lvm  
│ │ ├─otus-test           252:1    0  252M  0 lvm  
│ │ └─otus-test--snap     252:5    0  252M  0 lvm  
│ └─otus-test--snap-cow   252:4    0  100M  0 lvm  
│   └─otus-test--snap     252:5    0  252M  0 lvm  
├─sdb3                      8:19   0  200M  0 part 
└─sdb4                      8:20   0  200M  0 part 
sr0                        11:0    1 1024M  0 rom  

root@ubuntuserv2402:~# mkdir /data-snap
root@ubuntuserv2402:~# mount /dev/otus/test-snap /data-snap/
root@ubuntuserv2402:~# ll /data-snap/
total 130000
drwxr-xr-x  3 root root      4096 мар 31 16:55 ./
drwxr-xr-x 25 root root      4096 апр  1 12:55 ../
drwx------  2 root root     16384 мар 31 16:22 lost+found/
-rw-r--r--  1 root root 133091328 мар 31 16:55 test.log

root@ubuntuserv2402:~# umount /data-snap
root@ubuntuserv2402:~# mount /dev/otus/test /data/
root@ubuntuserv2402:~# rm /data/test.log
root@ubuntuserv2402:~# umount /data

# Восстанавливаем snapshot
root@ubuntuserv2402:~# lvconvert --merge /dev/otus/test-snap 
  Merging of volume otus/test-snap started.
  otus/test: Merged: 100,00%

root@ubuntuserv2402:~# ll /data
total 130000
drwxr-xr-x  3 root root      4096 мар 31 16:55 ./
drwxr-xr-x 25 root root      4096 апр  1 12:55 ../
drwx------  2 root root     16384 мар 31 16:22 lost+found/
-rw-r--r--  1 root root 133091328 мар 31 16:55 test.log
```

### Работа с LVM-RAID

```
root@ubuntuserv2402:~# pvcreate /dev/sdb{3,4}
  Physical volume "/dev/sdb3" successfully created.
  Physical volume "/dev/sdb4" successfully created.
root@ubuntuserv2402:~# vgcreate vg0 /dev/sdb{3,4}
  Volume group "vg0" successfully created
root@ubuntuserv2402:~# lvcreate -l+80%FREE -m1 -n mirror vg0
  Logical volume "mirror" created.
root@ubuntuserv2402:~# lvs
  LV        VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  small     otus      -wi-a-----  20,00m                                                    
  test      otus      -wi-ao---- 252,00m                                                    
  ubuntu-lv ubuntu-vg -wi-ao---- <11,50g                                                    
  mirror    vg0       rwi-a-r--- 156,00m                                    100,00          
```


### Уменьшить том под / до 8G

```
root@ubuntuserv2402:~# pvcreate /dev/sdc
  Physical volume "/dev/sdc" successfully created.
root@ubuntuserv2402:~# vgcreate vg_root /dev/sdc
  Volume group "vg_root" successfully created
root@ubuntuserv2402:~# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
root@ubuntuserv2402:~# mkfs.ext4 /dev/vg_root/lv_root
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2620416 4k blocks and 655360 inodes
Filesystem UUID: b563e02d-dd89-4f55-9971-779c924116e3
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@ubuntuserv2402:~# mount /dev/vg_root/lv_root /mnt/
root@ubuntuserv2402:~# rsync -avxHAX --progress / /mnt
total size is 4.934.762.732  speedup is 1,00
root@ubuntuserv2402:~# ls /mnt/
bin                boot   data       dev  home  lib64              lost+found  mnt  proc  run   sbin.usr-is-merged  srv       sys  usr
bin.usr-is-merged  cdrom  data-snap  etc  lib   lib.usr-is-merged  media       opt  root  sbin  snap swap.img  tmp  var

root@ubuntuserv2402:~# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

root@ubuntuserv2402:~# chroot /mnt
root@ubuntuserv2402:/# pwd
/
root@ubuntuserv2402:/# grub-mkconfig -o /boot/grub/grub.cfg
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-55-generic
Found initrd image: /boot/initrd.img-6.8.0-55-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
root@ubuntuserv2402:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.8.0-55-generic

user@ubuntuserv2402:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:2    0 11,5G  0 lvm  
sdc                         8:32   0   10G  0 disk 
└─vg_root-lv_root         252:0    0   10G  0 lvm  /

root@ubuntuserv2402:~# lvremove /dev/ubuntu-vg/ubuntu-lv 
Do you really want to remove and DISCARD active logical volume ubuntu-vg/ubuntu-lv? [y/n]: y
  Logical volume "ubuntu-lv" successfully removed.

root@ubuntuserv2402:~# lvcreate -n ubuntu-vg/ubuntu-lv -L 8G /dev/ubuntu-vg
WARNING: ext4 signature detected on /dev/ubuntu-vg/ubuntu-lv at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/ubuntu-vg/ubuntu-lv.
  Logical volume "ubuntu-lv" created.

root@ubuntuserv2402:~# mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv 
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2097152 4k blocks and 524288 inodes
Filesystem UUID: a6711cb3-1ee3-4fc2-b595-8357d483d127
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@ubuntuserv2402:~# mount /dev/ubuntu-vg/ubuntu-lv /mnt/
root@ubuntuserv2402:~# rsync -avxHAX --progress / /mnt

root@ubuntuserv2402:~# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
root@ubuntuserv2402:~# chroot /mnt/
root@ubuntuserv2402:/# grub-mkconfig -o /boot/grub/grub.cfg
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-55-generic
Found initrd image: /boot/initrd.img-6.8.0-55-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done

root@ubuntuserv2402:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.8.0-55-generic
W: Couldn't identify type of root file system for fsck hook

```

### Выделить том под /var в зеркало

```
root@ubuntuserv2402:/# pvcreate /dev/sdd /dev/sde 
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
root@ubuntuserv2402:/# vgcreate vg_var /dev/sdd /dev/sde
  Volume group "vg_var" successfully created
root@ubuntuserv2402:/# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952,00 MiB
  Logical volume "lv_var" created.

root@ubuntuserv2402:/# mkfs.ext4 /dev/vg_var/lv_var 
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 243712 4k blocks and 60928 inodes
Filesystem UUID: 4c90bb26-8362-4247-a2c2-d1ee5e89a5ce
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

root@ubuntuserv2402:/# mount /dev/vg_var/lv_var /mnt/
root@ubuntuserv2402:/# cp -aR /var/* /mnt/
root@ubuntuserv2402:/# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
root@ubuntuserv2402:/# umount /mnt/
root@ubuntuserv2402:/# mount /dev/vg_var/lv_var /var/
root@ubuntuserv2402:/# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

root@ubuntuserv2402:~# reboot

# Удаляем временные vg, lv
root@ubuntuserv2402:~# lvremove /dev/vg_root/lv_root 
Do you really want to remove and DISCARD active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed.
root@ubuntuserv2402:~# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
root@ubuntuserv2402:~# pvremove /dev/sdc 
  Labels on physical volume "/dev/sdc" successfully wiped.

```

## Выделить том под /home

```
root@ubuntuserv2402:~# lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg
  Logical volume "LogVol_Home" created.
root@ubuntuserv2402:~# mkfs.ext4 /dev/ubuntu-vg/LogVol_Home 
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: de833841-2040-487f-bd4c-022adacf36a8
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@ubuntuserv2402:~# mount /dev/ubuntu-vg/LogVol_Home /mnt/
root@ubuntuserv2402:~# cp -aR /home/* /mnt/
root@ubuntuserv2402:~# rm -rf /home/*
root@ubuntuserv2402:~# umount /mnt 
root@ubuntuserv2402:~# mount /dev/ubuntu-vg/LogVol_Home /home
root@ubuntuserv2402:~# echo "`blkid | grep Home | awk '{print $2}'` /home ext4 defaults 0 0" >> /etc/fstab
```
## Работа со снапшотами
```

root@ubuntuserv2402:~# touch /home/file{1..20}
root@ubuntuserv2402:~# ls /home/
file1  file10  file11  file12  file13  file14  file15  file16  file17  file18  file19  file2  file20  file3  file4  file5  file6  file7  file8  file9  lost+found  user
root@ubuntuserv2402:~# lvcreate -L 100MB -s -n home_snap /dev/ubuntu-vg/LogVol_Home
  Logical volume "home_snap" created.
root@ubuntuserv2402:~# rm -f /home/file{11..20}
root@ubuntuserv2402:~# ls /home/
file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  lost+found  user
root@ubuntuserv2402:~# umount /home

root@ubuntuserv2402:~# lvconvert --merge /dev/ubuntu-vg/home_snap 
  Merging of volume ubuntu-vg/home_snap started.
  ubuntu-vg/LogVol_Home: Merged: 100,00%

root@ubuntuserv2402:~# mount /dev/ubuntu-vg/LogVol_Home /home/
root@ubuntuserv2402:~# ls -al /home/
total 28
drwxr-xr-x  4 root root  4096 апр  2 15:34 .
drwxr-xr-x 25 root root  4096 апр  1 12:55 ..
-rw-r--r--  1 root root     0 апр  2 15:34 file1
-rw-r--r--  1 root root     0 апр  2 15:34 file10
-rw-r--r--  1 root root     0 апр  2 15:34 file11
-rw-r--r--  1 root root     0 апр  2 15:34 file12
-rw-r--r--  1 root root     0 апр  2 15:34 file13
-rw-r--r--  1 root root     0 апр  2 15:34 file14
-rw-r--r--  1 root root     0 апр  2 15:34 file15
-rw-r--r--  1 root root     0 апр  2 15:34 file16
-rw-r--r--  1 root root     0 апр  2 15:34 file17
-rw-r--r--  1 root root     0 апр  2 15:34 file18
-rw-r--r--  1 root root     0 апр  2 15:34 file19
-rw-r--r--  1 root root     0 апр  2 15:34 file2
-rw-r--r--  1 root root     0 апр  2 15:34 file20
-rw-r--r--  1 root root     0 апр  2 15:34 file3
-rw-r--r--  1 root root     0 апр  2 15:34 file4
-rw-r--r--  1 root root     0 апр  2 15:34 file5
-rw-r--r--  1 root root     0 апр  2 15:34 file6
-rw-r--r--  1 root root     0 апр  2 15:34 file7
-rw-r--r--  1 root root     0 апр  2 15:34 file8
-rw-r--r--  1 root root     0 апр  2 15:34 file9
drwx------  2 root root 16384 апр  1 15:27 lost+found
drwxr-x---  4 user user  4096 мар 23 13:40 user

```