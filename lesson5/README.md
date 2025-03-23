
1. Определить алгоритм с наилучшим сжатием:

- Определить какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);
- создать 4 файловых системы на каждой применить свой алгоритм сжатия;
- для сжатия использовать либо текстовый файл, либо группу файлов.

```
apt install zfsutils-linux
root@ubuntuserv2402:/home/user# lsblk
sdb                         8:16   0    4G  0 disk
├─sdb1                      8:17   0  100M  0 part
├─sdb2                      8:18   0  100M  0 part
├─sdb3                      8:19   0  100M  0 part
├─sdb4                      8:20   0  100M  0 part
├─sdb5                      8:21   0  100M  0 part
├─sdb6                      8:22   0  100M  0 part
├─sdb7                      8:23   0  100M  0 part
└─sdb8                      8:24   0  100M  0 part

# Создаём 4 пула из двух дисков в режиме RAID 1
zpool create otus1 mirror /dev/sdb1 /dev/sdb2
zpool create otus2 mirror /dev/sdb3 /dev/sdb4
zpool create otus3 mirror /dev/sdb5 /dev/sdb6
zpool create otus4 mirror /dev/sdb7 /dev/sdb8

root@ubuntuserv2402:/home/user# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1    80M    96K  79.9M        -         -     3%     0%  1.00x    ONLINE  -
otus2    80M   106K  79.9M        -         -     3%     0%  1.00x    ONLINE  -
otus3    80M   106K  79.9M        -         -     3%     0%  1.00x    ONLINE  -
otus4    80M   532K  79.5M        -         -     3%     0%  1.00x    ONLINE  -

# Получаем все доступные алгоритмы сжатия
zfs get
compression     YES      YES   on | off | lzjb | gzip | gzip-[1-9] | zle | lz4 | zstd | zstd-[1-19] | zstd-fast | zstd-fast-[1-10,20,30,40,50,60,70,80,90,100,500,1000]

# Добавим разные алгоритмы сжатия в каждую файловую систему
root@ubuntuserv2402:/home/user# zfs set compression=lzjb otus1
root@ubuntuserv2402:/home/user# zfs set compression=lz4 otus2
root@ubuntuserv2402:/home/user# zfs set compression=gzip-9 otus3
root@ubuntuserv2402:/home/user# zfs set compression=zle otus4

root@ubuntuserv2402:/home/user# zfs get all | grep compression
otus1  compression           lzjb                       local
otus2  compression           lz4                        local
otus3  compression           gzip-9                     local
otus4  compression           zle                        local

# Скачаем один и тот же текстовый файл во все пулы
for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
root@ubuntuserv2402:/home/user# ls -l /otus*
/otus1:
total 22101
-rw-r--r-- 1 root root 41130189 мар  2 08:31 pg2600.converter.log
/otus2:
total 18010
-rw-r--r-- 1 root root 41130189 мар  2 08:31 pg2600.converter.log
/otus3:
total 10968
-rw-r--r-- 1 root root 41130189 мар  2 08:31 pg2600.converter.log
/otus4:
total 40201
-rw-r--r-- 1 root root 41130189 мар  2 08:31 pg2600.converter.log

# Проверим, сколько места занимает один и тот же файл в разных пулах
root@ubuntuserv2402:/home/user# zfs list
NAME    USED  AVAIL  REFER  MOUNTPOINT
otus1  21.9M  18.1M  21.6M  /otus1
otus2  17.7M  22.3M  17.6M  /otus2
otus3  10.8M  29.2M  10.7M  /otus3
otus4  39.9M   104K  39.3M  /otus4

# проверим степень сжатия файлов
root@ubuntuserv2402:/home/user# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.81x                      -
otus2  compressratio         2.23x                      -
otus3  compressratio         3.66x                      -
otus4  compressratio         1.00x                      -

Таким образом, у нас получается, что алгоритм **gzip-9** самый эффективный по сжатию.
```

2. Определить настройки пула.  
    С помощью команды zfs import собрать pool ZFS.  
    Командами zfs определить настройки:  
         
    - размер хранилища;  
          
    - тип pool;  
          
    - значение recordsize;  
         
    - какое сжатие используется;  
         
    - какая контрольная сумма используется.

```
wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
tar -xzvf archive.tar.gz

root@ubuntuserv2402:/home/user# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                              ONLINE
          mirror-0                        ONLINE
            /home/user/zpoolexport/filea  ONLINE
            /home/user/zpoolexport/fileb  ONLINE

# Импортируем пул
root@ubuntuserv2402:/home/user# zpool import -d zpoolexport/ otus
root@ubuntuserv2402:/home/user# zpool status
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.
config:

        NAME                              STATE     READ WRITE CKSUM
        otus                              ONLINE       0     0     0
          mirror-0                        ONLINE       0     0     0
            /home/user/zpoolexport/filea  ONLINE       0     0     0
            /home/user/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

# Получаем параметры пула
root@ubuntuserv2402:/home/user# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -

root@ubuntuserv2402:/home/user# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default

root@ubuntuserv2402:/home/user# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local

root@ubuntuserv2402:/home/user# zfs get compression otus
NAME  PROPERTY     VALUE           SOURCE
otus  compression  zle             local

root@ubuntuserv2402:/home/user# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local

```

3. Работа со снапшотами:

- скопировать файл из удаленной директории;
- восстановить файл локально. zfs receive;
- найти зашифрованное сообщение в файле secret_message.

```
wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download

# Восстановим файловую систему из снапшота
zfs receive otus/test@today < otus_task2.file

# Ищем файл
root@ubuntuserv2402:/otus/test# find /otus/test/ -name "secret_message"
/otus/test/task1/file_mess/secret_message

# Вывод содержимого
root@ubuntuserv2402:/otus/test# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
```