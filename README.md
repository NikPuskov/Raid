# Raid
Возьмем за основу Vagrantfile https://github.com/erlong15/otus-linux/blob/master/Vagrantfile

Изменим этот Vagrantfile добавив туда 2 дополнительных диска (Vagrantfile приложен)

Поднимем виртуалку на Vagrant

RAID

1) Проверим какие диски собираем в RAID

vagrant@ubuntu-jammy:~$ sudo lshw -short | grep disk

/0/100/d/0         /dev/sdc   disk           262MB VBOX HARDDISK

/0/100/d/1         /dev/sdd   disk           262MB VBOX HARDDISK

/0/100/d/2         /dev/sde   disk           262MB VBOX HARDDISK

/0/100/d/3         /dev/sdf   disk           262MB VBOX HARDDISK

/0/100/d/4         /dev/sdg   disk           262MB VBOX HARDDISK

/0/100/d/5         /dev/sdh   disk           262MB VBOX HARDDISK

/0/100/14/0.0.0    /dev/sda   disk           42GB HARDDISK

/0/100/14/0.1.0    /dev/sdb   disk           10MB HARDDISK

2) Собираем RAID

sudo mdadm --create --verbose /dev/md0 -l 10 -n 6 /dev/sd{c,d,e,f,g,h}

3) Проверим собранный RAID

vagrant@ubuntu-jammy:~$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Wed Nov 20 14:16:11 2024
        Raid Level : raid10
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 6
     Total Devices : 6
       Persistence : Superblock is persistent

       Update Time : Wed Nov 20 14:16:15 2024
             State : clean
    Active Devices : 6
   Working Devices : 6
    Failed Devices : 0
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : ubuntu-jammy:0  (local to host ubuntu-jammy)
              UUID : 884e8de1:7973356f:4effa52c:524778de
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       32        0      active sync set-A   /dev/sdc
       1       8       48        1      active sync set-B   /dev/sdd
       2       8       64        2      active sync set-A   /dev/sde
       3       8       80        3      active sync set-B   /dev/sdf
       4       8       96        4      active sync set-A   /dev/sdg
       5       8      112        5      active sync set-B   /dev/sdh

4) Создадим mdadm.conf

   echo "DEVICE partitions" > /etc/mdadm/mdadm.conf

   mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

   (mdadm.conf приложен)

5) Сломаем RAID

root@ubuntu-jammy:~# mdadm /dev/md0 --fail /dev/sde

Проверим

root@ubuntu-jammy:~# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md0 : active raid10 sdh[5] sdg[4] sdf[3] sde[2](F) sdd[1] sdc[0]
      761856 blocks super 1.2 512K chunks 2 near-copies [6/5] [UU_UUU]

root@ubuntu-jammy:~# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Wed Nov 20 14:16:11 2024
        Raid Level : raid10
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 6
     Total Devices : 6
       Persistence : Superblock is persistent

       Update Time : Wed Nov 20 14:47:35 2024
             State : clean, degraded
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 1
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : ubuntu-jammy:0  (local to host ubuntu-jammy)
              UUID : 884e8de1:7973356f:4effa52c:524778de
            Events : 19

    Number   Major   Minor   RaidDevice State
       0       8       32        0      active sync set-A   /dev/sdc
       1       8       48        1      active sync set-B   /dev/sdd
       -       0        0        2      removed
       3       8       80        3      active sync set-B   /dev/sdf
       4       8       96        4      active sync set-A   /dev/sdg
       5       8      112        5      active sync set-B   /dev/sdh

       2       8       64        -      faulty   /dev/sde

Удалим сломанный диск 

root@ubuntu-jammy:~# mdadm /dev/md0 --remove /dev/sde

6) Починим RAID

root@ubuntu-jammy:~# mdadm /dev/md0 --add /dev/sde

root@ubuntu-jammy:~# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Wed Nov 20 14:16:11 2024
        Raid Level : raid10
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 6
     Total Devices : 6
       Persistence : Superblock is persistent

       Update Time : Wed Nov 20 14:52:36 2024
             State : clean
    Active Devices : 6
   Working Devices : 6
    Failed Devices : 0
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : ubuntu-jammy:0  (local to host ubuntu-jammy)
              UUID : 884e8de1:7973356f:4effa52c:524778de
            Events : 39

    Number   Major   Minor   RaidDevice State
       0       8       32        0      active sync set-A   /dev/sdc
       1       8       48        1      active sync set-B   /dev/sdd
       6       8       64        2      active sync set-A   /dev/sde
       3       8       80        3      active sync set-B   /dev/sdf
       4       8       96        4      active sync set-A   /dev/sdg
       5       8      112        5      active sync set-B   /dev/sdh

7) Создать GPT раздел, пять разделов и смонтировать их на диск

parted -s /dev/md0 mklabel gpt

parted /dev/md0 mkpart primary ext4 0% 20%

parted /dev/md0 mkpart primary ext4 20% 40%

parted /dev/md0 mkpart primary ext4 40% 60%

parted /dev/md0 mkpart primary ext4 60% 80%

parted /dev/md0 mkpart primary ext4 80% 100%

for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done

mkdir -p /raid/part{1,2,3,4,5}

for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done

В итоге получаем

root@ubuntu-jammy:~# lsblk

NAME      MAJ:MIN RM   SIZE RO TYPE   MOUNTPOINTS

loop0       7:0    0 111.9M  1 loop   /snap/lxd/24322

loop1       7:1    0  63.3M  1 loop   /snap/core20/1879

loop2       7:2    0  53.2M  1 loop   /snap/snapd/19122

sda         8:0    0    40G  0 disk

└─sda1      8:1    0    40G  0 part   /

sdb         8:16   0    10M  0 disk

sdc         8:32   0   250M  0 disk

└─md0       9:0    0   744M  0 raid10

  ├─md0p1 259:1    0   147M  0 part   /raid/part1

  ├─md0p2 259:4    0 148.5M  0 part   /raid/part2

  ├─md0p3 259:5    0   150M  0 part   /raid/part3

  ├─md0p4 259:8    0 148.5M  0 part   /raid/part4

  └─md0p5 259:9    0   147M  0 part   /raid/part5

sdd         8:48   0   250M  0 disk

└─md0       9:0    0   744M  0 raid10

  ├─md0p1 259:1    0   147M  0 part   /raid/part1

  ├─md0p2 259:4    0 148.5M  0 part   /raid/part2

  ├─md0p3 259:5    0   150M  0 part   /raid/part3

  ├─md0p4 259:8    0 148.5M  0 part   /raid/part4

  └─md0p5 259:9    0   147M  0 part   /raid/part5

sde         8:64   0   250M  0 disk

└─md0       9:0    0   744M  0 raid10

  ├─md0p1 259:1    0   147M  0 part   /raid/part1

  ├─md0p2 259:4    0 148.5M  0 part   /raid/part2

  ├─md0p3 259:5    0   150M  0 part   /raid/part3

  ├─md0p4 259:8    0 148.5M  0 part   /raid/part4

  └─md0p5 259:9    0   147M  0 part   /raid/part5

sdf         8:80   0   250M  0 disk

└─md0       9:0    0   744M  0 raid10

  ├─md0p1 259:1    0   147M  0 part   /raid/part1

  ├─md0p2 259:4    0 148.5M  0 part   /raid/part2

  ├─md0p3 259:5    0   150M  0 part   /raid/part3

  ├─md0p4 259:8    0 148.5M  0 part   /raid/part4

  └─md0p5 259:9    0   147M  0 part   /raid/part5

sdg         8:96   0   250M  0 disk

└─md0       9:0    0   744M  0 raid10

  ├─md0p1 259:1    0   147M  0 part   /raid/part1

  ├─md0p2 259:4    0 148.5M  0 part   /raid/part2

  ├─md0p3 259:5    0   150M  0 part   /raid/part3

  ├─md0p4 259:8    0 148.5M  0 part   /raid/part4

  └─md0p5 259:9    0   147M  0 part   /raid/part5

sdh         8:112  0   250M  0 disk

└─md0       9:0    0   744M  0 raid10

  ├─md0p1 259:1    0   147M  0 part   /raid/part1

  ├─md0p2 259:4    0 148.5M  0 part   /raid/part2

  ├─md0p3 259:5    0   150M  0 part   /raid/part3

  ├─md0p4 259:8    0 148.5M  0 part   /raid/part4

  └─md0p5 259:9    0   147M  0 part   /raid/part5


   


       
