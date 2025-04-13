# otus_lvm
“Файловые системы и LVM-1” курса Administrator Linux.Professional

1.Уменьшить том под / до 8G.
Подготовка тома
 1.1 root@lvm:/home/vagrant# vgcreate vg_root /dev/sdb
  Physical volume "/dev/sdb" successfully created.
  Volume group "vg_root" successfully created
 1.2 root@lvm:/home/vagrant# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
 1.3 root@lvm:/home/vagrant# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Calculated size of logical volume is 0 extents. Needs to be larger.
Создание/монтирование файловой системы
 1.4 root@lvm:/home/vagrant# mkfs.ext4 /dev/vg_root/lv_root
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620416 4k blocks and 655360 inodes
Filesystem UUID: f4b5629d-0dbc-493b-8275-e1ee230b8011
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632
 1.5 mount /dev/vg_root/lv_root /mnt
Копируем все данные 
 1.6 rsync -avxHAX --progress / /mnt/
Проверяем 
 1.7 root@lvm:/home/vagrant# ls /mnt
bin   dev  home  lib32  libx32      media  opt   root  sbin  srv       sys  usr
boot  etc  lib   lib64  lost+found  mnt    proc  run   snap  swap.img  tmp  var
Имитируем root, создаем в него chroot и обновляем grub
 1.8 for i in /proc/ /sys/ /dev/ /run/ /boot/; \
 do mount --bind $i /mnt/$i; done
 1.9 chroot /mnt/
 1.10 grub-mkconfig -o /boot/grub/grub.cfg
Обновляем образ initrd
 1.11 update-initramfs -u
update-initramfs: Generating /boot/initrd.img-5.15.0-91-generic
 Перезагружаемся (reboot)
Проверяем:
 1.12 vagrant@lvm:~$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0  63.4M  1 loop /snap/core20/1974
loop1                       7:1    0 111.9M  1 loop /snap/lxd/24322
loop2                       7:2    0  89.4M  1 loop /snap/lxd/31333
loop3                       7:3    0  53.3M  1 loop /snap/snapd/19457
loop4                       7:4    0  44.4M  1 loop /snap/snapd/23771
sda                         8:0    0   128G  0 disk 
├─sda1                      8:1    0     1M  0 part 
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0   126G  0 part 
  └─ubuntu--vg-ubuntu--lv 253:1    0    63G  0 lvm  
sdb                         8:16   0    10G  0 disk 
└─vg_root-lv_root         253:0    0    10G  0 lvm  /
sdc                         8:32   0     2G  0 disk 
sdd                         8:48   0     1G  0 disk 
sde                         8:64   0     1G  0 disk 
Удаляем старый LV и создаем новый 
 1.13 root@lvm:/home/vagrant# lvremove /dev/ubuntu-vg/ubuntu-lv
Do you really want to remove and DISCARD active logical volume ubuntu-vg/ubuntu-lv? [y/n]: y
  Logical volume "ubuntu-lv" successfully removed
 1.14 root@lvm:/home/vagrant# lvcreate -n ubuntu-vg/ubuntu-lv -L 8G /dev/ubuntu-vg
WARNING: ext4 signature detected on /dev/ubuntu-vg/ubuntu-lv at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/ubuntu-vg/ubuntu-lv.
  Logical volume "ubuntu-lv" created.
Повторяем действия сделанные ранее:
mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv
mount /dev/ubuntu-vg/ubuntu-lv /mnt
rsync -avxHAX --progress / /mnt/
Конфигурируем grub:
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub-mkconfig -o /boot/grub/grub.cfg
root@lvm:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-5.15.0-91-generic
W: Couldn't identify type of root file system for fsck hook

2. Выделить том под /var - сделать в mirror.
На свободных дисках создаем зеркало:
 2.1 root@lvm:/# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
 2.2 root@lvm:/# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created
 2.3 root@lvm:/# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
Создаем ФС и перемещаем туда /var
 2.4 root@lvm:/# mkfs.ext4 /dev/vg_var/lv_var
 mke2fs 1.46.5 (30-Dec-2021)
 Writing superblocks and filesystem accounting information: done
 2.5 root@lvm:/# mount /dev/vg_var/lv_var /mnt
 2.6 root@lvm:/# cp -aR /var/* /mnt/
Сохраняем содержимое старого var
 2.7 root@lvm:/# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
Монтируем новый каталог var в каталог /var
 2.8 root@lvm:/# umount /mnt
 2.9 root@lvm:/# mount /dev/vg_var/lv_var /var
Правим fstab для автоматического монтирования /var
 2.10 root@lvm:/# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
Перезагружаемся в новый и удалять временную Volume Group:
 2.11 root@lvm:/# lvremove /dev/vg_root/lv_root
 2.12 vgremove /dev/vg_root
 2.13 root@lvm:/# pvremove /dev/sdb
 
3. Выделить том под /home.
Выделяем топ под /home 
 3.1 root@lvm:/#  lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg
  Logical volume "LogVol_Home" created.
 3.2 root@lvm:/# mkfs.ext4 /dev/ubuntu-vg/LogVol_Home
 3.4 root@lvm:/# mount /dev/ubuntu-vg/LogVol_Home /mnt/
 3.5 root@lvm:/# cp -aR /home/* /mnt/
 3.6 root@lvm:/# rm -rf /home/*
 3.7 root@lvm:/# umount /mnt
 3.8 root@lvm:/# mount /dev/ubuntu-vg/LogVol_Home /home/

4. /home - сделать том для снапшотов.
Генерируем файлы в /home
root@lvm:/# touch /home/file{1..20}

5. Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор).
root@lvm:/# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

6. Работа со снапшотами:
а) Генерируем файлы в /home
root@lvm:/# touch /home/file{1..20}

б) Cнимаем снапшот
root@lvm:/# lvcreate -L 100MB -s -n home_snap /dev/ubuntu-vg/LogVol_Home
  Logical volume "home_snap" created.

в) Удаляем часть файлов 
root@lvm:/# rm -f /home/file{11..20}

г) Восстановление из снапшота.
root@lvm:/# umount /home
root@lvm:/# lvconvert --merge /dev/ubuntu-vg/home_snap
  Merging of volume ubuntu-vg/home_snap started.
  ubuntu-vg/LogVol_Home: Merged: 100.00%
root@lvm:/# mount /dev/mapper/ubuntu--vg-LogVol_Home /home
root@lvm:/# ls -al /home
total 28
drwxr-xr-x  4 root    root     4096 Apr 13 20:03 .
drwxr-xr-x 19 root    root     4096 Jan 11  2024 ..
-rw-r--r--  1 root    root        0 Apr 13 20:03 file1
-rw-r--r--  1 root    root        0 Apr 13 20:03 file9
-rw-r--r--  1 root    root        0 Apr 13 20:03 file11
-rw-r--r--  1 root    root        0 Apr 13 20:03 file12
-rw-r--r--  1 root    root        0 Apr 13 20:03 file13
-rw-r--r--  1 root    root        0 Apr 13 20:03 file14
-rw-r--r--  1 root    root        0 Apr 13 20:03 file15
drwx------  2 root    root    16384 Apr 13 19:57 lost+found
drwxr-x---  4 vagrant vagrant  4096 Jan 11  2024 vagrant
