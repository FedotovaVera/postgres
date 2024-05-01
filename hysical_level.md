Я создала виртуальную машину, установила postgres , убедилась, что кластер запущен, зашла под пользователем postgres в psql и создала таблицу с произвольным содержимым.
Остановила postgres.
Создаю новый диск:
```
vera@DESKTOP-Q6JLRHI:~$ yc compute disk create \
>     --name new-disk \
>     --type network-hdd \
>     --size 5 \
>     --description "second disk for otus-vm"
done (6s)
id: fhmusodl566gslduqfui
folder_id: b1gaaj0i5bs0h43veitl
created_at: "2024-04-26T20:39:03Z"
name: new-disk
description: second disk for otus-vm
type_id: network-hdd
zone_id: ru-central1-a
size: "5368709120"
block_size: "4096"
status: READY
disk_placement_policy: {}
```

Проверяю, что создался успешно:
```
vera@DESKTOP-Q6JLRHI:~$ yc compute disk list
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
|          ID          |   NAME   |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP |       DESCRIPTION       |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
| fhmqgfcsvgih9fij333v |          | 21474836480 | ru-central1-a | READY  | fhmmsrusraph0ei5qui9 |                 |                         |
| fhmusodl566gslduqfui | new-disk |  5368709120 | ru-central1-a | READY  |                      |                 | second disk for otus-vm |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
```

Подключаю новый диск к ВМ:
```
vera@DESKTOP-Q6JLRHI:~$ yc compute instance attach-disk otus-db-pg-vm-1 \
>     --disk-name new-disk \
>     --mode rw \
>     --auto-delete
done (10s)
id: fhmmsrusraph0ei5qui9
folder_id: b1gaaj0i5bs0h43veitl
created_at: "2024-04-26T19:50:36Z"
name: otus-db-pg-vm-1
zone_id: ru-central1-a
platform_id: standard-v3
resources:
  memory: "2147483648"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhmqgfcsvgih9fij333v
  auto_delete: true
  disk_id: fhmqgfcsvgih9fij333v
secondary_disks:
  - mode: READ_WRITE
    device_name: fhmusodl566gslduqfui
    auto_delete: true
    disk_id: fhmusodl566gslduqfui
network_interfaces:
  - index: "0"
    mac_address: d0:0d:16:e6:fd:cd
    subnet_id: e9b09tsm3ilc707agsen
    primary_v4_address:
      address: 10.128.0.26
      one_to_one_nat:
        address: 62.84.125.252
        ip_version: IPV4
    security_group_ids:
      - enpebhgrlav2rjpjhick
serial_port_settings:
  ssh_authorization: INSTANCE_METADATA
gpu_settings: {}
fqdn: otus-db-pg-vm-1.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```

Убеждаюсь, что успешно:

```
vera@DESKTOP-Q6JLRHI:~$ yc compute disk list
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
|          ID          |   NAME   |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP |       DESCRIPTION       |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
| fhmqgfcsvgih9fij333v |          | 21474836480 | ru-central1-a | READY  | fhmmsrusraph0ei5qui9 |                 |                         |
| fhmusodl566gslduqfui | new-disk |  5368709120 | ru-central1-a | READY  | fhmmsrusraph0ei5qui9 |                 | second disk for otus-vm |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
```
Создаю раздел и проверяю: 
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo fdisk /dev/vdb

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xe2f5ab61.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-10485759, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-10485759, default 10485759):

Created a new partition 1 of type 'Linux' and of size 5 GiB.

Command (m for help): p
Disk /dev/vdb: 5 GiB, 5368709120 bytes, 10485760 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0xe2f5ab61

Device     Boot Start      End  Sectors Size Id Type
/dev/vdb1        2048 10485759 10483712   5G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

dmitrydergunov95@otus-db-pg-vm-1:~$ w
 21:11:37 up  1:20,  2 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
dmitryde pts/0    87.117.185.149   20:07   47:56   0.04s  0.04s -bash
dmitryde pts/1    87.117.185.149   21:08    1.00s  0.01s  0.00s w
```
Форматирую диск, монтирую раздел и меняю права - даю разрешение на запись всем пользователям:
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo mkfs.ext4 /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 1310464 4k blocks and 327680 inodes
Filesystem UUID: c60293a2-cb41-495e-8abd-29c4c6e33b39
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

dmitrydergunov95@otus-db-pg-vm-1:~$ sudo mkdir /mnt/vdb1
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo mount /dev/vdb1 /mnt/vdb1
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo chmod a+w /mnt/vdb1
```
Пробую создать табличное пространство, но получаю ошибку:
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo su postgres
postgres@otus-db-pg-vm-1:/home/dmitrydergunov95$ cd /mnt/vdb1
postgres@otus-db-pg-vm-1:/mnt/vdb1$ mkdir tmptblspc
postgres@otus-db-pg-vm-1:/mnt/vdb1$ create tablespace my_ts location '/mnt/vdb1/tmptblspc';
Command 'create' not found, did you mean:
  command 'pcreate' from deb pbuilder-scripts (22)
Try: apt install <deb name>
```

