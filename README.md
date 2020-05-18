## **ZFS**

При выполнении задания использовалось следующее ПО:

###**Хост**
** ОС - Linux Mint 19.3 **
** Гипервизор - VirtualBox 6.1.6**
** Средство для создания и конфигурирования виртуальной среды - Vagrant 2.2.7 **
** Создание образа виртуальной машины - Packer 1.5.5 **
** Система контроля версий -  Git 2.26.2 **

###**Виртуальная машина**
** ОС - CentOS Linux release 7.6.1810**
и **CentOS Linux release 8.0.1905**


### **Установка ПО на виртуальной машине**

При разворачивании конфигурации виртуальной машины  в секции `box.vm.provision` выполняется установка необходимых для выполнения задания пакетов

```
box.vm.provision "shell", inline: <<-SHELL
          yum install -y mdadm smartmontools hdparm gdisk lvm2 xfsdump
        SHELL
```

### **Создание пула ZFS**


Установим epel и zol репозитории:

```
[vagrant@zfs ~]$ sudo yum -y install epel-release
[vagrant@zfs ~]$ sudo yum -y localinstall http://download.zfsonlinux.org/epel/zfs-release.el7_6.noarch.rpm
[vagrant@zfs ~]$ yum-config-manager --enable zfs-kmod
[vagrant@zfs ~]$ sudo yum-config-manager --disable zfs
[vagrant@zfs ~]$ sudo yum install zfs
```

Для начала при помощи утилиты `lsblk`определяем имеющиеся в конфигурации машины диски

```
[vagrant@zfs examples]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0  250M  0 disk
sdc      8:32   0  250M  0 disk
sdd      8:48   0  250M  0 disk
sde      8:64   0  250M  0 disk
sdf      8:80   0  250M  0 disk
sdg      8:96   0  250M  0 disk
sdh      8:112  0  250M  0 disk
sdi      8:128  0  250M  0 disk
sdj      8:144  0  250M  0 disk
sdk      8:160  0  250M  0 disk
```

Создадим из дисков `sdb` и `sdc` с именем `poolmirror1` массив `raid1`, для этого:

```
[vagrant@zfs ~]$ sudo zpool create poolmirror1 mirror sdb sdc
```

Проверим результат командой:

```
[vagrant@zfs ~]$ zpool list
NAME          SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
poolmirror1   224M   104K   224M         -     1%     0%  1.00x  ONLINE
```

```
[vagrant@zfs examples]$ zpool status
  pool: poolmirror1
 state: ONLINE
  scan: none requested
config:

    NAME         STATE     READ WRITE CKSUM
    poolmirror1  ONLINE       0     0     0
      mirror-0   ONLINE       0     0     0
        sdb      ONLINE       0     0     0
        sdc      ONLINE       0     0     0

errors: No known data errors
```

И для удобства смонтируем его в директорию `mnt`

```
[vagrant@zfs ~]$ sudo zfs set mountpoint=/mnt/poolmirror1 poolmirror1
```

Проверяем результат

```
[vagrant@zfs ~]$ sudo zfs get mounted
NAME         PROPERTY  VALUE    SOURCE
poolmirror1  mounted   yes      -
```

```
[vagrant@zfs ~]$ ll -l /mnt/
total 1
drwxr-xr-x. 2 root root 2 May 16 13:55 poolmirror1
```


### **Тестирование алгоритмов сжатия ZFS**



Для удобства создадим для каждого алгоритма отдельный раздел и назовем его по названию алгоритма.

#### **lzjb**
Первый раздел  - `lzjb`, алгоритм для сжатия `lzjb`

```
[vagrant@zfs ~]$ sudo zfs create poolmirror1/lzjb
[vagrant@zfs ~]$ sudo zfs set compression=lzjb poolmirror1/lzjb
```
Проверяем применения алгоритма к разделу

```
[vagrant@zfs ~]$ zfs get compression,compressratio poolmirror1/lzjb
NAME              PROPERTY       VALUE     SOURCE
poolmirror1/lzjb  compression    lzjb      local
poolmirror1/lzjb  compressratio  1.00x     -
```

#### **gzip**
Второй раздел - `gzip`, алгоритм для сжатия `gzip`

```
[vagrant@zfs ~]$ sudo zfs create poolmirror1/gzip
[vagrant@zfs ~]$ sudo zfs set compression=gzip poolmirror1/gzip
```

Проверяем применения алгоритма к разделу

```
[vagrant@zfs ~]$ zfs get compression,compressratio poolmirror1/gzip
NAME              PROPERTY       VALUE     SOURCE
poolmirror1/gzip  compression    gzip      local
poolmirror1/gzip  compressratio  1.00x     -
```

#### **gzip-9**
Третий раздел `gzip-9`, алгоритм для сжатия `gzip-9`

```
[vagrant@zfs ~]$ sudo zfs create poolmirror1/gzip-9
[vagrant@zfs ~]$ sudo zfs set compression=gzip-9 poolmirror1/gzip-9
```

Проверяем применения алгоритма к разделу

```
[vagrant@zfs ~]$ zfs get compression,compressratio poolmirror1/gzip-9
NAME                PROPERTY       VALUE     SOURCE
poolmirror1/gzip-9  compression    gzip-9    local
poolmirror1/gzip-9  compressratio  1.00x     -
```

#### **zle**
Четвертый раздел `zle`, алгоритм для сжатия `zle`

```
[vagrant@zfs ~]$ sudo zfs create poolmirror1/zle
[vagrant@zfs ~]$ sudo zfs set compression=zle poolmirror1/zle
```

Проверяем применения алгоритма к разделу

```
[vagrant@zfs ~]$ zfs get compression,compressratio poolmirror1/zle
NAME             PROPERTY       VALUE     SOURCE
poolmirror1/zle  compression    zle       local
poolmirror1/zle  compressratio  1.00x     -
```

#### **lz4**
Четвертый раздел `zle`, алгоритм для сжатия `zle`

```
[vagrant@zfs ~]$ sudo zfs create poolmirror1/lz4
[vagrant@zfs ~]$ sudo zfs set compression=lz4 poolmirror1/lz4
```

Проверяем применения алгоритма к разделу

```
[vagrant@zfs ~]$ zfs get compression,compressratio poolmirror1/lz4
NAME             PROPERTY       VALUE     SOURCE
poolmirror1/lz4  compression    lz4       local
poolmirror1/lz4  compressratio  1.00x     -
```

Скопируем на каждый их разделов тестовый текстовый файл `War_And_Peace.txt`

```
[vagrant@zfs examples]$ sudo cp -v War_And_Peace.txt /mnt/poolmirror1/lzjb/ /mnt/poolmirror1/gzip /mnt/poolmirror1/gzip-9/ /mnt/poolmirror1/zle/ /mnt/poolmirror1/lz4/
```

Проверим результат эфективности сжатия

```
[vagrant@zfs examples]$ zfs get compression,compressratio poolmirror1/lzjb poolmirror1/gzip poolmirror1/gzip-9 poolmirror1/zle poolmirror1/lz4
NAME                PROPERTY       VALUE     SOURCE
poolmirror1/gzip    compression    gzip      local
poolmirror1/gzip    compressratio  2.69x     -
poolmirror1/gzip-9  compression    gzip-9    local
poolmirror1/gzip-9  compressratio  2.70x     -
poolmirror1/lz4     compression    lz4       local
poolmirror1/lz4     compressratio  1.64x     -
poolmirror1/lzjb    compression    lzjb      local
poolmirror1/lzjb    compressratio  1.37x     -
poolmirror1/zle     compression    zle       local
poolmirror1/zle     compressratio  1.03x     -
```

Эффективнее всего текст сжат на разделах с  алгоритмом `gzip`

### **Импорт диска ZFS**

При попытке импортировать диск в CentOS 7 выдается сообщение об ошибке и никакие дополнительные не дают успешно подмонтировать диск. Вероятно это связано с тем что диск был создан в zfs 0.8.x,  для Centos 7 последняя доступна официально версия - 0.7.13-1


```
[root@zfs examples]# zpool import -d zpoolexport
   pool: otus
     id: 6554193320433390805
  state: UNAVAIL
status: The pool can only be accessed in read-only mode on this system. It
    cannot be accessed in read-write mode because it uses the following
    feature(s) not supported on this system:
    org.zfsonlinux:project_quota (space/object accounting based on project ID.)
    com.delphix:spacemap_v2 (Space maps representing large segments are more efficient.)
action: The pool cannot be imported in read-write mode. Import the pool with
    "-o readonly=on", access the pool on a system that supports the
    required feature(s), or recreate the pool from backup.
 config:

    otus                                          UNAVAIL  unsupported feature(s)
      mirror-0                                    ONLINE
        /home/vagrant/examples/zpoolexport/filea  ONLINE
        /home/vagrant/examples/zpoolexport/fileb  ONLINE
```

В Centos 8 с версией zfs 0.8.2-1 успешно импортировался

Смотрим свойства пула

```
[vagrant@zfs examples]$ sudo zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

    otus                                          ONLINE
      mirror-0                                    ONLINE
        /home/vagrant/examples/zpoolexport/filea  ONLINE
        /home/vagrant/examples/zpoolexport/fileb  ONLINE
```
Импортируем пул

```
sudo zpool import -d /home/vagrant/examples/zpoolexport/ otus
```

Проверяем статус

Размер хранилища - `SIZE` - 480Mb

```
[vagrant@zfs ~]$ zpool list
NAME          SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus          480M  2.09M   478M        -         -     0%     0%  1.00x    ONLINE  -
poolmirror2   448M   118K   448M        -         -     2%     0%  1.00x    ONLINE  -
[vagrant@zfs ~]$
```

Тип пула - mirror

```
[vagrant@zfs ~]$ zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

    NAME                                          STATE     READ WRITE CKSUM
    otus                                          ONLINE       0     0     0
      mirror-0                                    ONLINE       0     0     0
        /home/vagrant/examples/zpoolexport/filea  ONLINE       0     0     0
        /home/vagrant/examples/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

```

Значение `recordsize` - 128K
Значение `checksum` - sha256
Значение `compression` - zle

```
[vagrant@zfs ~]$ zfs get all otus
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.06M                  -
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
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.03M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1.00M                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              off                    default
otus  redundant_metadata    all                    default
otus  overlay               off                    default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
```

### **Восстановление снапшота ZFS**


Восстанавливаем снапшот с удаленной директории

```
[vagrant@zfs examples]$ sudo zfs receive poolmirror2/snap001 < zfs_task2.file
```

Проверяем результат

```
[vagrant@zfs file_mess]$ ll -l /poolmirror2/snap001/
total 3471
-rw-r--r--. 1 root    root          0 May 15 06:46 10M.file
-rw-r--r--. 1 root    root     309987 May 15 06:39 Limbo.txt
-rw-r--r--. 1 root    root     509836 May 15 06:39 Moby_Dick.txt
-rw-r--r--. 1 root    root    1209374 May  6  2016 War_and_Peace.txt
-rw-r--r--. 1 root    root     727040 May 15 07:08 cinderella.tar
-rw-r--r--. 1 root    root         65 May 15 06:39 for_examaple.txt
-rw-r--r--. 1 root    root          0 May 15 06:39 homework4.txt
drwxr-xr-x. 3 vagrant vagrant       4 Dec 18  2017 task1
-rw-r--r--. 1 root    root     398635 May 15 06:45 world.sql
```

Я не смог найти секретного сообщения ни в файле, ни в самом репозитории( https://github.com/sindresorhus/awesome

```
[vagrant@zfs file_mess]$ cat secret_message
https://github.com/sindresorhus/awesome
```


