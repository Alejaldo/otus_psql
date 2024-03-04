# Задание №3

## 1. Создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере
***Использую созданную виртуальную машину (ВМ) Yandex Cloud (4 GB RAM, 15 GB HDD, vCPU 2, Ubuntu 22.04 LTS) с привязкой ssh локальной машины, подключение делаю командой c терминала ноутбука***
```
ssh -i ~/.ssh/id_rsa chamonix@158.160.148.96
```
## 2. Поставьте на нее PostgreSQL 15 через sudo apt
***PostgreSQL 16 у меня уже установлена, проверяю это через проверку версии:***
```
$ psql --version
psql (PostgreSQL) 16.1 (Ubuntu 16.1-1.pgdg22.04+1)
```
## 3. Проверьте что кластер запущен через sudo -u postgres pg_lsclusters
***Проверяю:***
```
$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
## 4. Зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
***Создаю новую базу данных и таблице в ней с набором данных:***
```
$ sudo -u postgres psql
psql (16.1 (Ubuntu 16.1-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE DATABASE kntxt_db;
CREATE DATABASE
postgres=# \c kntxt_db
You are now connected to database "kntxt_db" as user "postgres".
kntxt_db=# CREATE TABLE techno_tracks (
    id SERIAL PRIMARY KEY,
    track_name TEXT,
    sub_genre TEXT,
    artist_name TEXT,
    planned_release_date DATE,
    beat_rate INT,
    label_name TEXT,
    sale_price DECIMAL,
    commission_rate DECIMAL
);
CREATE TABLE
kntxt_db=# INSERT INTO techno_tracks (track_name, sub_genre, artist_name, planned_release_date, beat_rate, label_name, sale_price, commission_rate) VALUES
('Abyssal Pulse', 'ACID techno', 'Charlotte de Witte', '2024-01-05', 128, 'KNTXT', 10.99, 15.0),
('Nocturnal Echoes', 'ACID techno', 'Charlotte de Witte', '2024-02-05', 129, 'Heldeep', 12.99, 15.0),
('Neon Shadows', 'Industrial techno', 'Hi-Lo', '2024-01-06', 130, 'Heldeep', 12.99, 12.0),
('Bassline Spectrum', 'Industrial techno', 'Hi-Lo', '2024-01-07', 131, 'Heldeep', 12.99, 12.0);
INSERT 0 4
kntxt_db=# SELECT * FROM techno_tracks;
 id |    track_name     |     sub_genre     |    artist_name     | planned_release_date | beat_rate | label_name | sale_price | commission_rate
----+-------------------+-------------------+--------------------+----------------------+-----------+------------+------------+-----------------
  1 | Abyssal Pulse     | ACID techno       | Charlotte de Witte | 2024-01-05           |       128 | KNTXT      |      10.99 |            15.0
  2 | Nocturnal Echoes  | ACID techno       | Charlotte de Witte | 2024-02-05           |       129 | Heldeep    |      12.99 |            15.0
  3 | Neon Shadows      | Industrial techno | Hi-Lo              | 2024-01-06           |       130 | Heldeep    |      12.99 |            12.0
  4 | Bassline Spectrum | Industrial techno | Hi-Lo              | 2024-01-07           |       131 | Heldeep    |      12.99 |            12.0
(4 rows)
kntxt_db=# \q
chamonix@chamonix:~$
```
## 5. Остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
***Делаю остановку, в своем случае через 16 main stop:***
```
$ sudo -u postgres pg_ctlcluster 16 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@16-main
$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
## 6. Создайте новый диск к ВМ размером 10GB + 7. Добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
***Пункты 6 и 7 реализую через функционал кнопки Attach disk в разделе Compute Cloud / Virtual machines / chamonix / Disks - где в модальном окне создаю диск (кнопка Create new disk) и нажимаю кнопку Attach***
## 8. Проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
***Делаю по инструкции монтирование диска:***
```
chamonix@chamonix:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 63.9M  1 loop /snap/core20/2105
loop2    7:2    0 49.8M  1 loop /snap/snapd/18357
loop3    7:3    0 40.4M  1 loop /snap/snapd/20671
loop4    7:4    0 63.9M  1 loop /snap/core20/2182
loop5    7:5    0   87M  1 loop /snap/lxd/27037
loop6    7:6    0   87M  1 loop /snap/lxd/27428
vda    252:0    0   15G  0 disk
├─vda1 252:1    0    1M  0 part
└─vda2 252:2    0   15G  0 part /
vdb    252:16   0   10G  0 disk

chamonix@chamonix:~$ sudo apt update

chamonix@chamonix:~$ sudo parted -l
Error: /dev/vdb: unrecognised disk label
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 10.7GB
Sector size (logical/physical): 512B/4096B
Partition Table: unknown
Disk Flags:

Model: Virtio Block Device (virtblk)
Disk /dev/vda: 16.1GB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  16.1GB  16.1GB  ext4


chamonix@chamonix:~$ sudo parted -l | grep Error
Error: /dev/vdb: unrecognised disk label
chamonix@chamonix:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 63.9M  1 loop /snap/core20/2105
loop2    7:2    0 49.8M  1 loop /snap/snapd/18357
loop3    7:3    0 40.4M  1 loop /snap/snapd/20671
loop4    7:4    0 63.9M  1 loop /snap/core20/2182
loop5    7:5    0   87M  1 loop /snap/lxd/27037
loop6    7:6    0   87M  1 loop /snap/lxd/27428
vda    252:0    0   15G  0 disk
├─vda1 252:1    0    1M  0 part
└─vda2 252:2    0   15G  0 part /
vdb    252:16   0   10G  0 disk
chamonix@chamonix:~$ sudo parted /dev/sda mklabel gpt
Error: Could not stat device /dev/sda - No such file or directory.
Retry/Cancel? cancel
chamonix@chamonix:~$ sudo parted /dev/vdb mklabel gpt
Information: You may need to update /etc/fstab.

chamonix@chamonix:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.

chamonix@chamonix:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 63.9M  1 loop /snap/core20/2105
loop2    7:2    0 49.8M  1 loop /snap/snapd/18357
loop3    7:3    0 40.4M  1 loop /snap/snapd/20671
loop4    7:4    0 63.9M  1 loop /snap/core20/2182
loop5    7:5    0   87M  1 loop /snap/lxd/27037
loop6    7:6    0   87M  1 loop /snap/lxd/27428
vda    252:0    0   15G  0 disk
├─vda1 252:1    0    1M  0 part
└─vda2 252:2    0   15G  0 part /
vdb    252:16   0   10G  0 disk
└─vdb1 252:17   0   10G  0 part
chamonix@chamonix:~$ sudo mkfs.ext4 -L datapartition /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: 9aa3aa84-d248-4d9e-8f34-50a1c570d872
Superblock backups stored on blocks:
  32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

chamonix@chamonix:~$ sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
NAME   FSTYPE LABEL         UUID                                 MOUNTPOINT
loop0                                                            /snap/core20/2105
loop2                                                            /snap/snapd/18357
loop3                                                            /snap/snapd/20671
loop4                                                            /snap/core20/2182
loop5                                                            /snap/lxd/27037
loop6                                                            /snap/lxd/27428
vda
├─vda1
└─vda2 ext4                 ed465c6e-049a-41c6-8e0b-c8da348a3577 /
vdb
└─vdb1 ext4   datapartition 9aa3aa84-d248-4d9e-8f34-50a1c570d872
chamonix@chamonix:~$ sudo mkdir -p /mnt/data
chamonix@chamonix:~$ sudo mount -o defaults /dev/vdb1 /mnt/data
chamonix@chamonix:~$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        15G  5.3G  8.8G  38% /
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
chamonix@chamonix:~$ echo "success" | sudo tee /mnt/data/test_file
success
chamonix@chamonix:~$ cat /mnt/data/test_file
success

```
## 9. Перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
```
chamonix@chamonix:~$ sudo reboot
Connection to 158.160.148.96 closed by remote host.
Connection to 158.160.148.96 closed.
[~]$ ssh -i ~/.ssh/id_rsa chamonix@158.160.148.96
chamonix@chamonix:~$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        15G  5.3G  8.8G  38% /
```
***Вижу что нет нового диска, поэтому проверяю информацию о UUID нового диска***
```
chamonix@chamonix:~$ sudo lsblk --fs
NAME   FSTYPE   FSVER LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                            0   100% /snap/core20/2105
loop1  squashfs 4.0                                                            0   100% /snap/lxd/27428
loop2  squashfs 4.0                                                            0   100% /snap/snapd/18357
loop3  squashfs 4.0                                                            0   100% /snap/snapd/20671
loop4  squashfs 4.0                                                            0   100% /snap/core20/2182
loop5  squashfs 4.0                                                            0   100% /snap/lxd/27037
vda
├─vda1
└─vda2 ext4     1.0                 ed465c6e-049a-41c6-8e0b-c8da348a3577    8.8G    36% /
vdb
└─vdb1 ext4     1.0   datapartition 9aa3aa84-d248-4d9e-8f34-50a1c570d872
```
***а затем добавляю новую строку в файл "/etc/fstab" (в моем случае это последняя строка)***
```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/ed465c6e-049a-41c6-8e0b-c8da348a3577 / ext4 defaults 0 1
UUID=9aa3aa84-d248-4d9e-8f34-50a1c570d872 /mnt/data ext4 defaults 0 2
```
***проверяю заново:***
```
chamonix@chamonix:~$ sudo reboot
Connection to 158.160.148.96 closed by remote host.
Connection to 158.160.148.96 closed.
[~]$ ssh -i ~/.ssh/id_rsa chamonix@158.160.148.96
chamonix@chamonix:~$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        15G  5.3G  8.8G  38% /
/dev/vdb1       9.8G   28K  9.3G   1% /mnt/data
```
***и теперб вижу что новый диск отображается***
## 10. Сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
***Даю права касательного нового диска***
```
sudo chown -R postgres:postgres /mnt/data
```
## 11. Перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
***Осуществляю перенос:***
```
sudo mv /var/lib/postgresql/16 /mnt/data/
```
## 12. Попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
***Дела попытку запуска кластера:***
```
$ sudo -u postgres pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
```
## 13. Напишите получилось или нет и почему
***Запуск не удался, потому что до этого физически файлы postrges были перенесены с диска на диск, но при этом в конфигурациях не поменял рабочий путь***
## 14. Задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его + 15. Напишите что и почему поменяли
***Мне нужен файл `postgresql.conf`, который находится в папке `/etc/postgresql/16/main/`, открываю его в nano редакторе и меняю параметр `data_directory`:***
```
data_directory = '/mnt/data/16/main'
```
***Это искомый параметр который теперь будет искать данные potsgres по новому адресу, в данном случае на новом диске***
## 16. Попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start + 17. Напишите получилось или нет и почему
***Новая попытка запуска успешна***
```
chamonix@chamonix:~$ sudo -u postgres pg_ctlcluster 16 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@16-main
Removed stale pid file.
chamonix@chamonix:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
16  main    5432 online postgres /mnt/data/16/main /var/log/postgresql/postgresql-16-main.log
```
***Теперь путь указанный в конфигурационном файле указан верно, поскольку кластер запустился***
## 18. Зайдите через через psql и проверьте содержимое ранее созданной таблицы
***Проверяю свои данные в базе данных***
```
chamonix@chamonix:~$ sudo -u postgres psql -d kntxt_db
psql (16.1 (Ubuntu 16.1-1.pgdg22.04+1))
Type "help" for help.

kntxt_db=# SELECT * FROM techno_tracks;
 id |    track_name     |     sub_genre     |    artist_name     | planned_release_date | beat_rate | label_name | sale_price | commission_rate
----+-------------------+-------------------+--------------------+----------------------+-----------+------------+------------+-----------------
  1 | Abyssal Pulse     | ACID techno       | Charlotte de Witte | 2024-01-05           |       128 | KNTXT      |      10.99 |            15.0
  2 | Nocturnal Echoes  | ACID techno       | Charlotte de Witte | 2024-02-05           |       129 | Heldeep    |      12.99 |            15.0
  3 | Neon Shadows      | Industrial techno | Hi-Lo              | 2024-01-06           |       130 | Heldeep    |      12.99 |            12.0
  4 | Bassline Spectrum | Industrial techno | Hi-Lo              | 2024-01-07           |       131 | Heldeep    |      12.99 |            12.0
(4 rows)
```
***все на месте, база данных и таблица в базе содержат изначально созданные данные***
## 19. Не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
***Создаю вторую ВМ с параметрами: (4 GB RAM, 10 GB HDD, vCPU 2, Ubuntu 22.04 LTS) также в YandexCloud и в той же зоне ru-central1-d для возможности переподключить диск***
***Захожу в новом окне терминала в новую ВМ и устанавливаю Postges как и на первой ВМ:***
```
[~]$ ssh -i ~/.ssh/id_rsa kntxt-user@158.160.142.238
kntxt-user@kntxt-vm:~$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc
kntxt-user@kntxt-vm:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
***перед удалением останавлю postgres, затем уже удалю папку с данными***
```
kntxt-user@kntxt-vm:~$ sudo pg_ctlcluster 16 main stop
kntxt-user@kntxt-vm:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
kntxt-user@kntxt-vm:~$ sudo rm -r /var/lib/postgresql/16/main
kntxt-user@kntxt-vm:~$ sudo ls -la /var/lib/postgresql/16/
total 8
drwxr-xr-x 2 postgres postgres 4096 Mar  4 19:31 .
drwxr-xr-x 3 postgres postgres 4096 Mar  4 19:28 ..
kntxt-user@kntxt-vm:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 63.3M  1 loop /snap/core20/1822
loop2    7:2    0 63.9M  1 loop /snap/core20/2182
loop3    7:3    0   87M  1 loop /snap/lxd/27037
loop4    7:4    0 49.8M  1 loop /snap/snapd/18357
loop5    7:5    0 40.4M  1 loop /snap/snapd/20671
loop6    7:6    0   87M  1 loop /snap/lxd/27428
vda    252:0    0   10G  0 disk
├─vda1 252:1    0    1M  0 part
└─vda2 252:2    0   10G  0 part /
vdb    252:16   0   10G  0 disk
└─vdb1 252:17   0   10G  0 part
kntxt-user@kntxt-vm:~$ sudo mkdir -p /mnt/data
kntxt-user@kntxt-vm:~$ sudo mount /dev/vdb1 /mnt/data
kntxt-user@kntxt-vm:~$ sudo blkid
/dev/vda2: UUID="ed465c6e-049a-41c6-8e0b-c8da348a3577" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="8c548af8-0682-43e1-9da4-2db0a9f4935b"
/dev/loop6: TYPE="squashfs"
/dev/vdb1: LABEL="datapartition" UUID="9aa3aa84-d248-4d9e-8f34-50a1c570d872" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="primary" PARTUUID="413b6a39-3f15-451d-8009-f1b148fbf4b5"
/dev/loop4: TYPE="squashfs"
/dev/loop2: TYPE="squashfs"
/dev/loop0: TYPE="squashfs"
/dev/loop5: TYPE="squashfs"
/dev/vda1: PARTUUID="c66f751c-f027-41a2-ba00-3342a74eb0cb"
/dev/loop3: TYPE="squashfs"
kntxt-user@kntxt-vm:~$ sudo nano /etc/fstab
```
***теперь в файл `/etc/fstab` записываю строку и сохраняю файл***
```
UUID=9aa3aa84-d248-4d9e-8f34-50a1c570d872 /mnt/data ext4 defaults 0 2
```
***проверяю что все смонтировано корректно после перезагрузки:***
```
kntxt-user@kntxt-vm:~$ sudo reboot
kntxt-user@kntxt-vm:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 63.3M  1 loop /snap/core20/1822
loop1    7:1    0 63.9M  1 loop /snap/core20/2182
loop2    7:2    0   87M  1 loop /snap/lxd/27037
loop3    7:3    0 49.8M  1 loop /snap/snapd/18357
loop4    7:4    0 40.4M  1 loop /snap/snapd/20671
loop5    7:5    0   87M  1 loop /snap/lxd/27428
vda    252:0    0   10G  0 disk
├─vda1 252:1    0    1M  0 part
└─vda2 252:2    0   10G  0 part /
vdb    252:16   0   10G  0 disk
└─vdb1 252:17   0   10G  0 part /mnt/data

```
***то есть все корректно было смонитровано***

***Теперь настраиваю корректный путь в конфигурационном файле:***
```
kntxt-user@kntxt-vm:~$ sudo nano /etc/postgresql/16/main/postgresql.conf
```
***искомое значение переменной:***
```
data_directory = '/mnt/data/16/main'
```
***и затем запускаю кластер postgres и проверяю сохранилась ли ранее созданная база данных и таблица с данными в ней:***
```
kntxt-user@kntxt-vm:~$ sudo -u postgres pg_ctlcluster 16 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@16-main
Removed stale pid file.
kntxt-user@kntxt-vm:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
16  main    5432 online postgres /mnt/data/16/main /var/log/postgresql/postgresql-16-main.log
kntxt-user@kntxt-vm:~$ sudo -u postgres psql -d kntxt_db
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
Type "help" for help.

kntxt_db=# SELECT * FROM techno_tracks;
 id |    track_name     |     sub_genre     |    artist_name     | planned_release_date | beat_rate | label_name | sale_price | commission_rate
----+-------------------+-------------------+--------------------+----------------------+-----------+------------+------------+-----------------
  1 | Abyssal Pulse     | ACID techno       | Charlotte de Witte | 2024-01-05           |       128 | KNTXT      |      10.99 |            15.0
  2 | Nocturnal Echoes  | ACID techno       | Charlotte de Witte | 2024-02-05           |       129 | Heldeep    |      12.99 |            15.0
  3 | Neon Shadows      | Industrial techno | Hi-Lo              | 2024-01-06           |       130 | Heldeep    |      12.99 |            12.0
  4 | Bassline Spectrum | Industrial techno | Hi-Lo              | 2024-01-07           |       131 | Heldeep    |      12.99 |            12.0
(4 rows)
```
***Таким образом данные успешно были сохранены***
