## Домашнее задание - Файловая система и LVM
### Клонирование и запуск Vagrant
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/lvm-hw-3.git`
В текущей директории появится папка с именем репозитория. В данном случае hw-1. Ознакомимся с содержимым:
```
cd lvm-hw-3
ls -l
README.md
Vagrantfile
```
Здесь:
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ, выводим список дисков:
```
vagrant up
sudo -i
[root@lvm ~]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```
### Уменьшить том под / до 8G
Подготовляем временный том и монтируем его:
```
pvcreate /dev/sdb
vgcreate vg_root /dev/sdb
lvcreate -n lv_root -l+100%free /dev/vg_root
mkfs.xfs /dev/vg_root/lv_root
mount /dev/vg_root/lv_root /mnt
```
Для переноса данных будем использовать утилиту xfsdump, с помощью её копируем все данные во временный том:
```
yum install xfsdump -y
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
Затем переконфигурируем grub длā того, чтобý при старте перейти в новýй /
Сымитируем текущий root -> сделаем в него chroot и обновим grub:
```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
```
Обновим образ initrd:
```
cd /boot
for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done
```
Редактируем /boot/grub2/grub.cfg, заменим rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root и перезагружаем:
```
vi /boot/grub2/grub.cfg
exit
reboot
```
Перезагружаемся и убеждаемся, что мы загрузились новым рутом командой lsblk:
```
vagrant ssh
Last login: Wed Mar 15 07:38:48 2023 from 10.0.2.2
[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:1    0 37.5G  0 lvm  
  └─VolGroup00-LogVol01 253:2    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```
Уменьшаем размер LV до 8Г, путем удаления и создания занова:
```
lvremove /dev/VolGroup00/LogVol00
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
```
И выполняем те же действия, что и с временным томом:
```
mkfs.xfs /dev/VolGroup00/LogVol00
mount /dev/VolGroup00/LogVol00 /mnt
xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
cd /boot
for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done
```
Пока не перезагружаемя выполняем следующие задания

### Выделить том под /var в зеркало
На свободных дисках создаем зеркало:
```
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var
```
Создаем на нем ФС и перемещаем туда /var:
```
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
umount /mnt
mount /dev/vg_var/lv_var /var
```
Правим fstab длā автоматического монтированиā /var:
```
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
После чего можно успешно перезагружаться и проверить загрузку ОС в новый (уменьшенный root) командой lsblk:
```
exit
reboot
Connection to 127.0.0.1 closed by remote host.
altynbek@lab-serv01:~/Otus-homeworks/lvm-hw-3$ vagrant ssh
Last login: Wed Mar 15 07:52:31 2023 from 10.0.2.2
[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk 
└─vg_root-lv_root        253:2    0   10G  0 lvm  
sdc                        8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm  
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm  
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk 
```
Удаляем временную Volume Group:
```
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb
```
### Выделить том под /home
Выделяем том под /home:
```
lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol_Home
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/
```
Правим /etc/fstab:
```
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
### /home - сделать том для снапшотов
Сгенерируем файлы в /home/:
```
touch /home/file{1..20}
```
Снимаем снапшот:
```
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
```
Удаляем часть файлов:
```
rm -f /home/file{11..20}
[root@lvm ~]# ls /home
file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  vagrant
```
Восстанавливаемся со снапшота:
```
umount /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home
[root@lvm ~]# ls /home/
file1  file10  file11  file12  file13  file14  file15  file16  file17  file18  file19  file2  file20  file3  file4  file5  file6  file7  file8  file9  vagrant
```
------------

# Задание со *, на нашей куче дисков попробовать поставить btrfs/zfs: с кешем и снэпшотами, разметить здесь каталог /opt
Устанавливаем последнию версию ядра и доп пакетов ядра и перезагружаемся:
```
yum install kernel kernel-devel kernel-headers
reboot
```
Устанавливаем пакеты zfs:
```
yum install https://zfsonlinux.org/epel/zfs-release-2-2.el7.noarch.rpm epel-release -y
yum install -y zfs
echo zfs > /etc/modules-load.d/zfs.conf
reboot
```
После перезагрузки проверяем подключены ли модули zfs в ядро:
```
[root@lvm ~]# lsmod | grep zfs
zfs                  4265838  6 
zunicode              331170  1 zfs
zzstd                 468972  1 zfs
zlua                  155622  1 zfs
zcommon                94285  1 zfs
znvpair                94388  2 zfs,zcommon
zavl                   15698  1 zfs
icp                   305871  1 zfs
spl                   100846  6 icp,zfs,zavl,zzstd,zcommon,znvpair
```
Размечаем диск /dev/sdb для zfs:
```
fdisk /dev/sdb << EOF
o
n
p
1

+1G
t
bf
n
p
2

+1G
t

bf
w
EOF
```
Создаем диск zfs с поддержкой кэширования и проверяем:
```
zpool create opt -f mirror /dev/sdb1 /dev/sdb2 cache /dev/sde
zpool status
  pool: opt
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	opt         ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb1    ONLINE       0     0     0
	    sdb2    ONLINE       0     0     0
	cache
	  sde       ONLINE       0     0     0

errors: No known data errors
```