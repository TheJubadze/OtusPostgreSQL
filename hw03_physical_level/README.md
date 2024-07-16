#### создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере
сделал в GCE

#### поставьте на нее PostgreSQL 15 через sudo apt
```
void@ubnt:~$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
OK
void@ubnt:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
void@ubnt:~$ sudo apt update

...

void@ubnt:~$ sudo apt install postgresql-16

...

void@ubnt:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select version();
                                                              version
-----------------------------------------------------------------------------------------------------------------------------------
 PostgreSQL 16.3 (Ubuntu 16.3-1.pgdg20.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit
(1 row)
```

#### проверьте что кластер запущен через sudo -u postgres pg_lsclusters
```
void@ubnt:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

#### зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
```
void@ubnt:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=#  insert into test values('1');
INSERT 0 1
postgres=# \q
void@ubnt:~$
```

#### остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
```
void@ubnt:~$ sudo -u postgres pg_ctlcluster 16 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@16-main
void@ubnt:~$ sudo systemctl stop postgresql@16-main
void@ubnt:~$
```

#### создайте новый диск к ВМ размером 10GB, добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk, проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
```
void@ubnt:~$ lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0     7:0    0    64M  1 loop /snap/core20/2318
loop1     7:1    0  91.9M  1 loop /snap/lxd/24061
loop2     7:2    0 352.9M  1 loop /snap/google-cloud-cli/247
loop3     7:3    0  38.8M  1 loop /snap/snapd/21759
sda       8:0    0    10G  0 disk
├─sda1    8:1    0   9.9G  0 part /
├─sda14   8:14   0     4M  0 part
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0    10G  0 disk
void@ubnt:~$ sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
mke2fs 1.45.5 (07-Jan-2020)
Discarding device blocks: done
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: 76e77fcf-8ec6-4866-b9c8-a80bb79062ec
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

void@ubnt:~$ sudo mkdir -p /mnt/disk-1
void@ubnt:~$ sudo mount -o discard,defaults /dev/sdb /mnt/disk-1
void@ubnt:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       9.6G  2.3G  7.3G  24% /
devtmpfs        473M     0  473M   0% /dev
tmpfs           477M     0  477M   0% /dev/shm
tmpfs            96M  960K   95M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           477M     0  477M   0% /sys/fs/cgroup
/dev/loop0       64M   64M     0 100% /snap/core20/2318
/dev/loop2      353M  353M     0 100% /snap/google-cloud-cli/247
/dev/loop1       92M   92M     0 100% /snap/lxd/24061
/dev/loop3       39M   39M     0 100% /snap/snapd/21759
/dev/sda15      105M  6.1M   99M   6% /boot/efi
tmpfs            96M     0   96M   0% /run/user/1002
/dev/sdb        9.8G   24K  9.8G   1% /mnt/disk-1
void@ubnt:~$
```

#### перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
после рестарта диск остается примонтированным:
```
void@ubnt:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       9.6G  2.3G  7.3G  24% /
devtmpfs        473M     0  473M   0% /dev
tmpfs           477M  1.1M  476M   1% /dev/shm
tmpfs            96M  976K   95M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           477M     0  477M   0% /sys/fs/cgroup
/dev/loop0       64M   64M     0 100% /snap/core20/2318
/dev/loop1      353M  353M     0 100% /snap/google-cloud-cli/247
/dev/loop2       92M   92M     0 100% /snap/lxd/24061
/dev/loop3       39M   39M     0 100% /snap/snapd/21759
/dev/sdb        9.8G   24K  9.8G   1% /mnt/disk-1
/dev/sda15      105M  6.1M   99M   6% /boot/efi
tmpfs            96M     0   96M   0% /run/user/1002
void@ubnt:~$
```

#### сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
```
void@ubnt:~$ sudo chown -R postgres:postgres /mnt/disk-1/
void@ubnt:~$ 
```

#### перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
```
void@ubnt:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
void@ubnt:~$ sudo systemctl stop postgresql@16-main
void@ubnt:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
void@ubnt:~$ sudo mv /var/lib/postgresql/16 /mnt/disk-1
```

#### попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
```
void@ubnt:~$ sudo systemctl start postgresql@16-main
Job for postgresql@16-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@16-main.service" and "journalctl -xe" for details.
void@ubnt:~$ systemctl status postgresql@16-main.service
● postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: failed (Result: protocol) since Tue 2024-07-16 08:03:48 UTC; 17s ago
    Process: 2464 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 16-main start (code=exited, status=1/FAILURE)

Jul 16 08:03:48 ubnt systemd[1]: Starting PostgreSQL Cluster 16-main...
Jul 16 08:03:48 ubnt postgresql@16-main[2464]: Error: /var/lib/postgresql/16/main is not accessible or does not exist
Jul 16 08:03:48 ubnt systemd[1]: postgresql@16-main.service: Can't open PID file /run/postgresql/16-main.pid (yet?) after start: Operation not permi>
Jul 16 08:03:48 ubnt systemd[1]: postgresql@16-main.service: Failed with result 'protocol'.
Jul 16 08:03:48 ubnt systemd[1]: Failed to start PostgreSQL Cluster 16-main.
lines 1-10/10 (END)
```

#### напишите получилось или нет и почему
Не получилось! Судя по логу, постгря не может найти перемещённый каталог `/var/lib/postgresql/16/main`

#### задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его
#### напишите что и почему поменяли
Внёс изменения в файл `postgresql.conf`:
Именил значение параметра `data_directory` с `/var/lib/postgresql/16` на `/mnt/disk-1/16/main`

#### попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
```
void@ubnt:~$ sudo systemctl start postgresql@16-main
void@ubnt:~$
```

#### напишите получилось или нет и почему
Судя по тому, что в выводе выше нет ошибок, получилось =)
Успешно полтому, что теперь постргря смотрит в правильное место, где лежат файлы базы - `/mnt/disk-1/16/main`

#### зайдите через через psql и проверьте содержимое ранее созданной таблицы
```
void@ubnt:~$ sudo -u postgres psql
psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=#
```

#### задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
- отсоединил диск от первой ВМ
- приаттачил этот же диск ко второй ВМ
- примонтировал созданный каталог на второй ВМ:
  ```
  void@ubnt2:~$ sudo mkdir -p /mnt/disk-1
  void@ubnt2:~$ sudo mount -o discard,defaults /dev/sdb /mnt/disk-1
  void@ubnt2:~$ df -h
  Filesystem      Size  Used Avail Use% Mounted on
  /dev/root       9.6G  2.3G  7.3G  24% /
  devtmpfs        473M     0  473M   0% /dev
  tmpfs           477M  1.1M  476M   1% /dev/shm
  tmpfs            96M  976K   95M   1% /run
  tmpfs           5.0M     0  5.0M   0% /run/lock
  tmpfs           477M     0  477M   0% /sys/fs/cgroup
  /dev/loop0       64M   64M     0 100% /snap/core20/2318
  /dev/loop1      353M  353M     0 100% /snap/google-cloud-cli/247
  /dev/loop2       92M   92M     0 100% /snap/lxd/24061
  /dev/loop3       39M   39M     0 100% /snap/snapd/21759
  /dev/sda15      105M  6.1M   99M   6% /boot/efi
  tmpfs            96M     0   96M   0% /run/user/1002
  /dev/sdb        9.8G   39M  9.7G   1% /mnt/disk-1
  void@ubnt2:~$
  ```
- остановил кластер на второй ВМ:
  ```
  void@ubnt2:~$ sudo systemctl stop postgresql@16-main
  void@ubnt2:~$
  ```
-  на второй ВМ внёс изменения в файл `/etc/postgresql/16/main/postgresql.conf`: именил значение параметра `data_directory` с `/var/lib/postgresql/16` на `/mnt/disk-1/16/main`
- стартанул кластер на второй ВМ:
  ```
  void@ubnt2:~$ sudo systemctl start postgresql@16-main
  void@ubnt2:~$
  ```
- на второй ВМ подключаемся к постгре и проверяем наличие наших данных на внешнем диске:
  ```
  void@ubnt2:~$ sudo -u postgres psql
  psql (16.3 (Ubuntu 16.3-1.pgdg20.04+1))
  Type "help" for help.

  postgres=# \dt
          List of relations
  Schema | Name | Type  |  Owner
  --------+------+-------+----------
  public | test | table | postgres
  (1 row)

  postgres=# select * from test
  postgres-# ;
  c1
  ----
  1
  (1 row)

  postgres=#
  ```
