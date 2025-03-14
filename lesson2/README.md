## Занятие 2. Работа с mdadm

**Домашнее задание**
### 1. Добавить в виртуальную машину несколько дисков
 Добавил 5 дисков по 1 ГБ.

```
[user@localhost ~]$ lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdb               8:16   0    1G  0 disk
sdc               8:32   0    1G  0 disk
sdd               8:48   0    1G  0 disk
sde               8:64   0    1G  0 disk
sdf               8:80   0    1G  0 disk

# Занулим на всякий случай суперблоки
[root@mdadm ~]$ mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
mdadm: Unrecognised md component device - /dev/sdb
...
```

## 2. Собрать RAID-0/1/5/10 на выбор
```
[user@localhost ~]$ sudo mdadm --create /dev/md1 --level=10 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md1 started.
[user@localhost ~]$ cat /proc/mdstat
Personalities : [raid10]
md1 : active raid10 sde[3] sdd[2] sdc[1] sdb[0]
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]

unused devices: <none>

[user@localhost ~]$ sudo mdadm -D /dev/md1
/dev/md1:
        Raid Level : raid10
      Raid Devices : 4
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       1       8       32        1      active sync set-B   /dev/sdc
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

```

## 3. Сломать и починить RAID

```
# маркируем устройство как faulty
[user@localhost ~]$ sudo mdadm /dev/md1 --fail /dev/sdb
mdadm: set /dev/sdb faulty in /dev/md1
[user@localhost ~]$ cat /proc/mdstat
Personalities : [raid10]
md1 : active raid10 sde[3] sdd[2] sdc[1] sdb[0](F)
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/3] [_UUU]

unused devices: <none>
[user@localhost ~]$ sudo mdadm -D /dev/md1
       0       8       16        -      faulty   /dev/sdb

# удаляем сбойное устройство
[root@localhost user]# mdadm /dev/md0 --remove /dev/sdb
mdadm: hot removed /dev/sdb from /dev/md1

# добавляем новый диск в raid
[root@localhost user]# mdadm /dev/md1 --add /dev/sdf

# видим что raid уже перестроился
[root@localhost user]# cat /proc/mdstat
Personalities : [raid10]
md1 : active raid10 sdf[4] sde[3] sdd[2] sdc[1]
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]

unused devices: <none>
```

## 4. Создать GPT таблицу, пять разделов и смонтировать их в системе.

```
[user@localhost ~]$ sudo gdisk /dev/md1
GPT fdisk (gdisk) version 1.0.10

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): n
Partition number (1-128, default 1):
First sector (34-4186078, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-4186078, default = 4184063) or {+-}size{KMGTP}: +1G
Partition number (2-128, default 2):
First sector (34-4186078, default = 2099200) or {+-}size{KMGTP}:
Last sector (2099200-4186078, default = 4184063) or {+-}size{KMGTP}: +500M
Partition number (3-128, default 3):
First sector (34-4186078, default = 3123200) or {+-}size{KMGTP}:
Last sector (3123200-4186078, default = 4184063) or {+-}size{KMGTP}: +500M
Partition number (4-128, default 4):
First sector (34-4186078, default = 4147200) or {+-}size{KMGTP}:
Last sector (4147200-4186078, default = 4184063) or {+-}size{KMGTP}: +500M
Last sector (4147200-4186078, default = 4184063) or {+-}size{KMGTP}:

Command (? for help): p
Disk /dev/md1: 4186112 sectors, 2.0 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 68699214-970C-4416-87C6-27708AD454D6
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 4186078
Partitions will be aligned on 2048-sector boundaries
Total free space is 4029 sectors (2.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         2099199   1024.0 MiB  8300  Linux filesystem
   2         2099200         3123199   500.0 MiB   8300  Linux filesystem
   3         3123200         4147199   500.0 MiB   8300  Linux filesystem
   4         4147200         4184063   18.0 MiB    8300  Linux filesystem

Command (? for help): w
Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/md1.
The operation has completed successfully.


[user@localhost ~]$ for i in $(seq 1 4); do sudo mkfs.ext4 /dev/md1p$i; done
[root@localhost user]# mkdir -p /mnt/part{1,2,3,4,}
[root@localhost user]# for i in $(seq 1 4); do mount /dev/md1p$i /mnt/part$i; done

[root@localhost user]# lsblk -f
NAME            FSTYPE            FSVER    LABEL                   UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sdc             linux_raid_member 1.2      localhost.localdomain:1 10f7fb03-f8f8-5ac9-62af-e035061e0162
└─md1
  ├─md1p1       ext4              1.0                              060687cb-f17c-4321-aa61-f64a82ec4c3a    905,9M     0% /mnt/part1
  ├─md1p2       ext4              1.0                              c5153154-db1a-49c6-9e30-569cd9630ff8    429,2M     0% /mnt/part2
  ├─md1p3       ext4              1.0                              f4008951-47c0-435e-b2cd-ad9e588eabb0    429,2M     0% /mnt/part3
  └─md1p4       ext4              1.0                              43832c41-c694-48a3-ba6c-0d2e7661be89     14,3M     0% /mnt/part4
```



