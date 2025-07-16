## ДЗ-04 Практические навыки работы с ZFS
Определить алгоритм с наилучшим сжатием:

Определить какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);  
создать 4 файловых системы на каждой применить свой алгоритм сжатия;  
для сжатия использовать либо текстовый файл, либо группу файлов.  
Определить настройки пула.  
С помощью команды zfs import собрать pool ZFS.  
Командами zfs определить настройки:  
    - размер хранилища;  
    - тип pool;  
    - значение recordsize;  
    - какое сжатие используется;  
    - какая контрольная сумма используется.  
Работа со снапшотами:  
скопировать файл из удаленной директории;  
восстановить файл локально. zfs receive;  
найти зашифрованное сообщение в файле secret_message.  

### Создаем виртуальную машину
```
root@u24srv04:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   20G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0 18.2G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0   10G  0 lvm  /
sdb                         8:16   0  512M  0 disk
sdc                         8:32   0  512M  0 disk
sdd                         8:48   0  512M  0 disk
sde                         8:64   0  512M  0 disk
sdf                         8:80   0  512M  0 disk
sdg                         8:96   0  512M  0 disk
sdh                         8:112  0  512M  0 disk
sdi                         8:128  0  512M  0 disk
sr0                        11:0    1 1024M  0 rom
root@u24srv04:~#
```
### Определение алгоритма с наилучшим сжатием
##### Установка пакета утилит ZFS 
```
root@u24srv04:~# apt install zfsutils-linux
...
```
##### Создаём четыре пула по два диска в режиме RAID 1
```
root@u24srv04:~# zpool create otus1 mirror /dev/sd{b,c}
root@u24srv04:~# zpool create otus2 mirror /dev/sd{d,e}
root@u24srv04:~# zpool create otus3 mirror /dev/sd{f,g}
root@u24srv04:~# zpool create otus4 mirror /dev/sd{h,i}
```
##### Смотрим информацию о пулах:
```
root@u24srv04:~# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M   116K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M   110K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M   106K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M   106K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
##### Добавим разные алгоритмы сжатия в каждую файловую систему:
```
root@u24srv04:~# zfs set compression=lzjb otus1
root@u24srv04:~# zfs set compression=lz4 otus2
root@u24srv04:~# zfs set compression=gzip-9 otus3
root@u24srv04:~# zfs set compression=zle otus4
```
##### Копируем один и тот же текстовый файл во все пулы:
```
wget -P /tmp https://gutenberg.org/cache/epub/2600/pg2600.converter.log
root@u24srv04:~# ls -l /tmp | grep pg2600
-rw-r--r-- 1 root root 41158891 Jul  2 07:31 pg2600.converter.log
root@u24srv04:~#
root@u24srv04:~# for i in {1..4}; do cp /tmp/pg2600.converter.log /otus$i; done
```
##### Проверим, что файл был скопирован во все пулы:
```
oot@u24srv04:~# ls -l /otus*
/otus1:
total 22109
-rw-r--r-- 1 root root 41158891 Jul 16 18:01 pg2600.converter.log

/otus2:
total 18011
-rw-r--r-- 1 root root 41158891 Jul 16 18:01 pg2600.converter.log

/otus3:
total 10968
-rw-r--r-- 1 root root 41158891 Jul 16 18:01 pg2600.converter.log

/otus4:
total 40224
-rw-r--r-- 1 root root 41158891 Jul 16 18:01 pg2600.converter.log
root@u24srv04:~#
```
##### Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:
```
root@u24srv04:~# zfs list
NAME    USED  AVAIL  REFER  MOUNTPOINT
otus1  21.7M   330M  21.6M  /otus1
otus2  17.7M   334M  17.6M  /otus2
otus3  10.9M   341M  10.7M  /otus3
otus4  39.4M   313M  39.3M  /otus4
root@u24srv04:~#
root@u24srv04:~# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.82x                  -
otus2  compressratio         2.23x                  -
otus3  compressratio         3.66x                  -
otus4  compressratio         1.00x                  -
root@u24srv04:~#
```
##### Вывод: алгоритм gzip-9 самый эффективный по сжатию.

### Определение настроек пула
##### Скачиваем архив в домашний каталог:
```
root@u24srv04:~# wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
```
##### Разархивируем его:
```
root@u24srv04:~# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
root@u24srv04:~#
```

##### Проверим, возможно ли импортировать данный каталог в пул
```
root@u24srv04:~# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                         ONLINE
          mirror-0                   ONLINE
            /root/zpoolexport/filea  ONLINE
            /root/zpoolexport/fileb  ONLINE
root@u24srv04:~#
```
##### Сделаем импорт данного пула к нам в ОС:
```
root@u24srv04:~# zpool import -d zpoolexport/ otus
root@u24srv04:~#
root@u24srv04:~# zpool status otus
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.
config:

        NAME                         STATE     READ WRITE CKSUM
        otus                         ONLINE       0     0     0
          mirror-0                   ONLINE       0     0     0
            /root/zpoolexport/filea  ONLINE       0     0     0
            /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
root@u24srv04:~#
```
##### Далее нам нужно определить настройки:
```
root@u24srv04:~# zpool get all otus
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupratio                     1.00x                          -
...
```
##### Запрос сразу всех параметром файловой системы:
```
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
...
```
##### C помощью команды grep можно уточнить конкретный параметр
```
root@u24srv04:~# zfs get all otus | grep read
otus  readonly              off                    default
root@u24srv04:~#
```
* разрешены чтение и запись  
##### Командой zfs get с указанием параметра получаем его значение
```
root@u24srv04:~# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
root@u24srv04:~#
```
##### Тип сжатия (или параметр отключения):
```
root@u24srv04:~# zfs get compression otus
NAME  PROPERTY     VALUE           SOURCE
otus  compression  zle             local
root@u24srv04:~#
```
##### Значение recordsize
```
root@u24srv04:~# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
root@u24srv04:~#
```
##### какая контрольная сумма используется
```
root@u24srv04:~# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
root@u24srv04:~#

```

### Работа со снапшотом, поиск сообщения от преподавателя
#### Скачаем файл, указанный в задании:  
```
root@u24srv04:~# wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download
[1] 3523
root@u24srv04:~#
Redirecting output to ‘wget-log’.

[1]+  Done                    wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI
root@u24srv04:~#
```
##### Восстановим файловую систему из снапшота:
```
root@u24srv04:~# zfs receive otus/test@today < otus_task2.file
root@u24srv04:~#
```
##### Далее, ищем в каталоге /otus/test файл с именем “secret_message”:
```
root@u24srv04:~# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
root@u24srv04:~#
```
##### Смотрим содержимое найденного файла:
```
root@u24srv04:~# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
```
### В файле ссылка на курс OTUS, задание выполнено






