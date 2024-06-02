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
| fhm7k24ffqdlfha6oasr | new-disk |  5368709120 | ru-central1-a | READY  |                      |                 | second disk for otus-vm |
| fhmusm9m6cmokl597nna |          | 21474836480 | ru-central1-a | READY  | fhma79ff5l089bmha17s |                 |                         |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
```

Подключаю новый диск к ВМ:
```
vera@DESKTOP-Q6JLRHI:~$ yc compute instance attach-disk otus-db-pg-vm-1 \
>     --disk-name new-disk \
>     --mode rw \
>     --auto-delete
done (11s)
id: fhma79ff5l089bmha17s
folder_id: b1gaaj0i5bs0h43veitl
created_at: "2024-05-16T17:45:36Z"
name: test-vm
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
  device_name: fhmusm9m6cmokl597nna
  auto_delete: true
  disk_id: fhmusm9m6cmokl597nna
secondary_disks:
  - mode: READ_WRITE
    device_name: fhm7k24ffqdlfha6oasr
    auto_delete: true
    disk_id: fhm7k24ffqdlfha6oasr
network_interfaces:
  - index: "0"
    mac_address: d0:0d:a3:a5:ef:2d
    subnet_id: e9b09tsm3ilc707agsen
    primary_v4_address:
      address: 10.128.0.19
      one_to_one_nat:
        address: 51.250.87.207
        ip_version: IPV4
    security_group_ids:
      - enpebhgrlav2rjpjhick
serial_port_settings:
  ssh_authorization: INSTANCE_METADATA
gpu_settings: {}
fqdn: test-vm.ru-central1.internal
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
| fhm7k24ffqdlfha6oasr | new-disk |  5368709120 | ru-central1-a | READY  | fhma79ff5l089bmha17s |                 | second disk for otus-vm |
| fhmusm9m6cmokl597nna |          | 21474836480 | ru-central1-a | READY  | fhma79ff5l089bmha17s |                 |                         |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
```
Создаю раздел и проверяю: 
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo fdisk /dev/vdb

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x3a86b08b.

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
Disk identifier: 0x3a86b08b

Device     Boot Start      End  Sectors Size Id Type
/dev/vdb1        2048 10485759 10483712   5G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

dmitrydergunov95@otus-db-pg-vm-1:~$ w
 18:18:03 up 31 min,  1 user,  load average: 0.00, 0.00, 0.02
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
dmitryde pts/0    87.117.189.48    18:14    0.00s  0.02s  0.00s w
```
Форматирую диск, монтирую раздел и меняю права - даю разрешение на запись всем пользователям:
```
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo mkfs.ext4 /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 1310464 4k blocks and 327680 inodes
Filesystem UUID: ce2d4706-aeb7-4737-9fed-8ee6b2ad8cb2
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
Вышла и подняла кластер. Зашла снова и выполняю команду:
```
dmitrydergunov95@test-vm:~$ sudo su postgres
postgres@test-vm:/home/dmitrydergunov95$ cd /mnt/vdb1
postgres@test-vm:/mnt/vdb1$ psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# create tablespace my_ts location '/mnt/vdb1/tmptblspc';
CREATE TABLESPACE
postgres=# \db
             List of tablespaces
    Name    |  Owner   |      Location
------------+----------+---------------------
 my_ts      | postgres | /mnt/vdb1/tmptblspc
 pg_default | postgres |
 pg_global  | postgres |
(3 rows)
dmitrydergunov95@test-vm:~$ sudo chown postgres:postgres /mnt/vdb1
```
```
dmitrydergunov95@test-vm:~$ sudo pg_ctlcluster 15 main stop
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
dmitrydergunov95@test-vm:~$ mv /var/lib/postgresql/15 /mnt/vdb1
mv: cannot access '/var/lib/postgresql/15/main': Permission denied
dmitrydergunov95@test-vm:~$ sudo su postgres
postgres@test-vm:/home/dmitrydergunov95$ mv /var/lib/postgresql/15 /mnt/vdb1
mv: replace '/mnt/vdb1/15', overriding mode 0755 (rwxr-xr-x)? yes
mv: inter-device move failed: '/var/lib/postgresql/15' to '/mnt/vdb1/15'; unable to remove target: Directory not empty
```
Не получается. Гуглю, пробую по другому:
```
dmitrydergunov95@test-vm:~$ sudo pg_ctlcluster 15 main stop
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
dmitrydergunov95@test-vm:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Thu 2024-05-16 18:03:08 UTC; 1h 37min ago
   Main PID: 4599 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 16 18:03:08 test-vm systemd[1]: Starting PostgreSQL RDBMS...
May 16 18:03:08 test-vm systemd[1]: Finished PostgreSQL RDBMS.
dmitrydergunov95@test-vm:~$ sudo systemctl stop postgresql
dmitrydergunov95@test-vm:~$ sudo systemctl status postgresql
○ postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Thu 2024-05-16 19:40:41 UTC; 2s ago
   Main PID: 4599 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 16 18:03:08 test-vm systemd[1]: Starting PostgreSQL RDBMS...
May 16 18:03:08 test-vm systemd[1]: Finished PostgreSQL RDBMS.
May 16 19:40:41 test-vm systemd[1]: postgresql.service: Deactivated successfully.
May 16 19:40:41 test-vm systemd[1]: Stopped PostgreSQL RDBMS.
dmitrydergunov95@test-vm:~$ sudo rsync -av /var/lib/postgresql/15 /mnt/vdb1
sending incremental file list
15/
15/main/
15/main/PG_VERSION
15/main/postgresql.auto.conf
15/main/postmaster.opts
15/main/base/
15/main/base/1/
15/main/base/1/112
15/main/base/1/113
15/main/base/1/1247
15/main/base/1/1247_fsm
15/main/base/1/1247_vm
15/main/base/1/1249
15/main/base/1/1249_fsm
15/main/base/1/1249_vm
15/main/base/1/1255
15/main/base/1/1255_fsm
15/main/base/1/1255_vm
15/main/base/1/1259
15/main/base/1/1259_fsm
15/main/base/1/1259_vm
15/main/base/1/13374
15/main/base/1/13374_fsm
15/main/base/1/13374_vm
15/main/base/1/13377
15/main/base/1/13378
15/main/base/1/13379
15/main/base/1/13379_fsm
15/main/base/1/13379_vm
15/main/base/1/13382
15/main/base/1/13383
15/main/base/1/13384
15/main/base/1/13384_fsm
15/main/base/1/13384_vm
15/main/base/1/13387
15/main/base/1/13388
15/main/base/1/13389
15/main/base/1/13389_fsm
15/main/base/1/13389_vm
15/main/base/1/13392
15/main/base/1/13393
15/main/base/1/1417
15/main/base/1/1418
15/main/base/1/174
15/main/base/1/175
15/main/base/1/2187
15/main/base/1/2224
15/main/base/1/2228
15/main/base/1/2328
15/main/base/1/2336
15/main/base/1/2337
15/main/base/1/2579
15/main/base/1/2600
15/main/base/1/2600_fsm
15/main/base/1/2600_vm
15/main/base/1/2601
15/main/base/1/2601_fsm
15/main/base/1/2601_vm
15/main/base/1/2602
15/main/base/1/2602_fsm
15/main/base/1/2602_vm
15/main/base/1/2603
15/main/base/1/2603_fsm
15/main/base/1/2603_vm
15/main/base/1/2604
15/main/base/1/2605
15/main/base/1/2605_fsm
15/main/base/1/2605_vm
15/main/base/1/2606
15/main/base/1/2606_fsm
15/main/base/1/2606_vm
15/main/base/1/2607
15/main/base/1/2607_fsm
15/main/base/1/2607_vm
15/main/base/1/2608
15/main/base/1/2608_fsm
15/main/base/1/2608_vm
15/main/base/1/2609
15/main/base/1/2609_fsm
15/main/base/1/2609_vm
15/main/base/1/2610
15/main/base/1/2610_fsm
15/main/base/1/2610_vm
15/main/base/1/2611
15/main/base/1/2612
15/main/base/1/2612_fsm
15/main/base/1/2612_vm
15/main/base/1/2613
15/main/base/1/2615
15/main/base/1/2615_fsm
15/main/base/1/2615_vm
15/main/base/1/2616
15/main/base/1/2616_fsm
15/main/base/1/2616_vm
15/main/base/1/2617
15/main/base/1/2617_fsm
15/main/base/1/2617_vm
15/main/base/1/2618
15/main/base/1/2618_fsm
15/main/base/1/2618_vm
15/main/base/1/2619
15/main/base/1/2619_fsm
15/main/base/1/2619_vm
15/main/base/1/2620
15/main/base/1/2650
15/main/base/1/2651
15/main/base/1/2652
15/main/base/1/2653
15/main/base/1/2654
15/main/base/1/2655
15/main/base/1/2656
15/main/base/1/2657
15/main/base/1/2658
15/main/base/1/2659
15/main/base/1/2660
15/main/base/1/2661
15/main/base/1/2662
15/main/base/1/2663
15/main/base/1/2664
15/main/base/1/2665
15/main/base/1/2666
15/main/base/1/2667
15/main/base/1/2668
15/main/base/1/2669
15/main/base/1/2670
15/main/base/1/2673
15/main/base/1/2674
15/main/base/1/2675
15/main/base/1/2678
15/main/base/1/2679
15/main/base/1/2680
15/main/base/1/2681
15/main/base/1/2682
15/main/base/1/2683
15/main/base/1/2684
15/main/base/1/2685
15/main/base/1/2686
15/main/base/1/2687
15/main/base/1/2688
15/main/base/1/2689
15/main/base/1/2690
15/main/base/1/2691
15/main/base/1/2692
15/main/base/1/2693
15/main/base/1/2696
15/main/base/1/2699
15/main/base/1/2701
15/main/base/1/2702
15/main/base/1/2703
15/main/base/1/2704
15/main/base/1/2753
15/main/base/1/2753_fsm
15/main/base/1/2753_vm
15/main/base/1/2754
15/main/base/1/2755
15/main/base/1/2756
15/main/base/1/2757
15/main/base/1/2830
15/main/base/1/2831
15/main/base/1/2832
15/main/base/1/2833
15/main/base/1/2834
15/main/base/1/2835
15/main/base/1/2836
15/main/base/1/2836_fsm
15/main/base/1/2836_vm
15/main/base/1/2837
15/main/base/1/2838
15/main/base/1/2838_fsm
15/main/base/1/2838_vm
15/main/base/1/2839
15/main/base/1/2840
15/main/base/1/2840_fsm
15/main/base/1/2840_vm
15/main/base/1/2841
15/main/base/1/2995
15/main/base/1/2996
15/main/base/1/3079
15/main/base/1/3079_fsm
15/main/base/1/3079_vm
15/main/base/1/3080
15/main/base/1/3081
15/main/base/1/3085
15/main/base/1/3118
15/main/base/1/3119
15/main/base/1/3164
15/main/base/1/3256
15/main/base/1/3257
15/main/base/1/3258
15/main/base/1/3350
15/main/base/1/3351
15/main/base/1/3379
15/main/base/1/3380
15/main/base/1/3381
15/main/base/1/3394
15/main/base/1/3394_fsm
15/main/base/1/3394_vm
15/main/base/1/3395
15/main/base/1/3429
15/main/base/1/3430
15/main/base/1/3431
15/main/base/1/3433
15/main/base/1/3439
15/main/base/1/3440
15/main/base/1/3455
15/main/base/1/3456
15/main/base/1/3456_fsm
15/main/base/1/3456_vm
15/main/base/1/3466
15/main/base/1/3467
15/main/base/1/3468
15/main/base/1/3501
15/main/base/1/3502
15/main/base/1/3503
15/main/base/1/3534
15/main/base/1/3541
15/main/base/1/3541_fsm
15/main/base/1/3541_vm
15/main/base/1/3542
15/main/base/1/3574
15/main/base/1/3575
15/main/base/1/3576
15/main/base/1/3596
15/main/base/1/3597
15/main/base/1/3598
15/main/base/1/3599
15/main/base/1/3600
15/main/base/1/3600_fsm
15/main/base/1/3600_vm
15/main/base/1/3601
15/main/base/1/3601_fsm
15/main/base/1/3601_vm
15/main/base/1/3602
15/main/base/1/3602_fsm
15/main/base/1/3602_vm
15/main/base/1/3603
15/main/base/1/3603_fsm
15/main/base/1/3603_vm
15/main/base/1/3604
15/main/base/1/3605
15/main/base/1/3606
15/main/base/1/3607
15/main/base/1/3608
15/main/base/1/3609
15/main/base/1/3712
15/main/base/1/3764
15/main/base/1/3764_fsm
15/main/base/1/3764_vm
15/main/base/1/3766
15/main/base/1/3767
15/main/base/1/3997
15/main/base/1/4143
15/main/base/1/4144
15/main/base/1/4145
15/main/base/1/4146
15/main/base/1/4147
15/main/base/1/4148
15/main/base/1/4149
15/main/base/1/4150
15/main/base/1/4151
15/main/base/1/4152
15/main/base/1/4153
15/main/base/1/4154
15/main/base/1/4155
15/main/base/1/4156
15/main/base/1/4157
15/main/base/1/4158
15/main/base/1/4159
15/main/base/1/4160
15/main/base/1/4163
15/main/base/1/4164
15/main/base/1/4165
15/main/base/1/4166
15/main/base/1/4167
15/main/base/1/4168
15/main/base/1/4169
15/main/base/1/4170
15/main/base/1/4171
15/main/base/1/4172
15/main/base/1/4173
15/main/base/1/4174
15/main/base/1/5002
15/main/base/1/548
15/main/base/1/549
15/main/base/1/6102
15/main/base/1/6104
15/main/base/1/6106
15/main/base/1/6110
15/main/base/1/6111
15/main/base/1/6112
15/main/base/1/6113
15/main/base/1/6116
15/main/base/1/6117
15/main/base/1/6175
15/main/base/1/6176
15/main/base/1/6228
15/main/base/1/6229
15/main/base/1/6237
15/main/base/1/6238
15/main/base/1/6239
15/main/base/1/826
15/main/base/1/827
15/main/base/1/828
15/main/base/1/PG_VERSION
15/main/base/1/pg_filenode.map
15/main/base/1/pg_internal.init
15/main/base/4/
15/main/base/4/112
15/main/base/4/113
15/main/base/4/1247
15/main/base/4/1247_fsm
15/main/base/4/1247_vm
15/main/base/4/1249
15/main/base/4/1249_fsm
15/main/base/4/1249_vm
15/main/base/4/1255
15/main/base/4/1255_fsm
15/main/base/4/1255_vm
15/main/base/4/1259
15/main/base/4/1259_fsm
15/main/base/4/1259_vm
15/main/base/4/13374
15/main/base/4/13374_fsm
15/main/base/4/13374_vm
15/main/base/4/13377
15/main/base/4/13378
15/main/base/4/13379
15/main/base/4/13379_fsm
15/main/base/4/13379_vm
15/main/base/4/13382
15/main/base/4/13383
15/main/base/4/13384
15/main/base/4/13384_fsm
15/main/base/4/13384_vm
15/main/base/4/13387
15/main/base/4/13388
15/main/base/4/13389
15/main/base/4/13389_fsm
15/main/base/4/13389_vm
15/main/base/4/13392
15/main/base/4/13393
15/main/base/4/1417
15/main/base/4/1418
15/main/base/4/174
15/main/base/4/175
15/main/base/4/2187
15/main/base/4/2224
15/main/base/4/2228
15/main/base/4/2328
15/main/base/4/2336
15/main/base/4/2337
15/main/base/4/2579
15/main/base/4/2600
15/main/base/4/2600_fsm
15/main/base/4/2600_vm
15/main/base/4/2601
15/main/base/4/2601_fsm
15/main/base/4/2601_vm
15/main/base/4/2602
15/main/base/4/2602_fsm
15/main/base/4/2602_vm
15/main/base/4/2603
15/main/base/4/2603_fsm
15/main/base/4/2603_vm
15/main/base/4/2604
15/main/base/4/2605
15/main/base/4/2605_fsm
15/main/base/4/2605_vm
15/main/base/4/2606
15/main/base/4/2606_fsm
15/main/base/4/2606_vm
15/main/base/4/2607
15/main/base/4/2607_fsm
15/main/base/4/2607_vm
15/main/base/4/2608
15/main/base/4/2608_fsm
15/main/base/4/2608_vm
15/main/base/4/2609
15/main/base/4/2609_fsm
15/main/base/4/2609_vm
15/main/base/4/2610
15/main/base/4/2610_fsm
15/main/base/4/2610_vm
15/main/base/4/2611
15/main/base/4/2612
15/main/base/4/2612_fsm
15/main/base/4/2612_vm
15/main/base/4/2613
15/main/base/4/2615
15/main/base/4/2615_fsm
15/main/base/4/2615_vm
15/main/base/4/2616
15/main/base/4/2616_fsm
15/main/base/4/2616_vm
15/main/base/4/2617
15/main/base/4/2617_fsm
15/main/base/4/2617_vm
15/main/base/4/2618
15/main/base/4/2618_fsm
15/main/base/4/2618_vm
15/main/base/4/2619
15/main/base/4/2619_fsm
15/main/base/4/2619_vm
15/main/base/4/2620
15/main/base/4/2650
15/main/base/4/2651
15/main/base/4/2652
15/main/base/4/2653
15/main/base/4/2654
15/main/base/4/2655
15/main/base/4/2656
15/main/base/4/2657
15/main/base/4/2658
15/main/base/4/2659
15/main/base/4/2660
15/main/base/4/2661
15/main/base/4/2662
15/main/base/4/2663
15/main/base/4/2664
15/main/base/4/2665
15/main/base/4/2666
15/main/base/4/2667
15/main/base/4/2668
15/main/base/4/2669
15/main/base/4/2670
15/main/base/4/2673
15/main/base/4/2674
15/main/base/4/2675
15/main/base/4/2678
15/main/base/4/2679
15/main/base/4/2680
15/main/base/4/2681
15/main/base/4/2682
15/main/base/4/2683
15/main/base/4/2684
15/main/base/4/2685
15/main/base/4/2686
15/main/base/4/2687
15/main/base/4/2688
15/main/base/4/2689
15/main/base/4/2690
15/main/base/4/2691
15/main/base/4/2692
15/main/base/4/2693
15/main/base/4/2696
15/main/base/4/2699
15/main/base/4/2701
15/main/base/4/2702
15/main/base/4/2703
15/main/base/4/2704
15/main/base/4/2753
15/main/base/4/2753_fsm
15/main/base/4/2753_vm
15/main/base/4/2754
15/main/base/4/2755
15/main/base/4/2756
15/main/base/4/2757
15/main/base/4/2830
15/main/base/4/2831
15/main/base/4/2832
15/main/base/4/2833
15/main/base/4/2834
15/main/base/4/2835
15/main/base/4/2836
15/main/base/4/2836_fsm
15/main/base/4/2836_vm
15/main/base/4/2837
15/main/base/4/2838
15/main/base/4/2838_fsm
15/main/base/4/2838_vm
15/main/base/4/2839
15/main/base/4/2840
15/main/base/4/2840_fsm
15/main/base/4/2840_vm
15/main/base/4/2841
15/main/base/4/2995
15/main/base/4/2996
15/main/base/4/3079
15/main/base/4/3079_fsm
15/main/base/4/3079_vm
15/main/base/4/3080
15/main/base/4/3081
15/main/base/4/3085
15/main/base/4/3118
15/main/base/4/3119
15/main/base/4/3164
15/main/base/4/3256
15/main/base/4/3257
15/main/base/4/3258
15/main/base/4/3350
15/main/base/4/3351
15/main/base/4/3379
15/main/base/4/3380
15/main/base/4/3381
15/main/base/4/3394
15/main/base/4/3394_fsm
15/main/base/4/3394_vm
15/main/base/4/3395
15/main/base/4/3429
15/main/base/4/3430
15/main/base/4/3431
15/main/base/4/3433
15/main/base/4/3439
15/main/base/4/3440
15/main/base/4/3455
15/main/base/4/3456
15/main/base/4/3456_fsm
15/main/base/4/3456_vm
15/main/base/4/3466
15/main/base/4/3467
15/main/base/4/3468
15/main/base/4/3501
15/main/base/4/3502
15/main/base/4/3503
15/main/base/4/3534
15/main/base/4/3541
15/main/base/4/3541_fsm
15/main/base/4/3541_vm
15/main/base/4/3542
15/main/base/4/3574
15/main/base/4/3575
15/main/base/4/3576
15/main/base/4/3596
15/main/base/4/3597
15/main/base/4/3598
15/main/base/4/3599
15/main/base/4/3600
15/main/base/4/3600_fsm
15/main/base/4/3600_vm
15/main/base/4/3601
15/main/base/4/3601_fsm
15/main/base/4/3601_vm
15/main/base/4/3602
15/main/base/4/3602_fsm
15/main/base/4/3602_vm
15/main/base/4/3603
15/main/base/4/3603_fsm
15/main/base/4/3603_vm
15/main/base/4/3604
15/main/base/4/3605
15/main/base/4/3606
15/main/base/4/3607
15/main/base/4/3608
15/main/base/4/3609
15/main/base/4/3712
15/main/base/4/3764
15/main/base/4/3764_fsm
15/main/base/4/3764_vm
15/main/base/4/3766
15/main/base/4/3767
15/main/base/4/3997
15/main/base/4/4143
15/main/base/4/4144
15/main/base/4/4145
15/main/base/4/4146
15/main/base/4/4147
15/main/base/4/4148
15/main/base/4/4149
15/main/base/4/4150
15/main/base/4/4151
15/main/base/4/4152
15/main/base/4/4153
15/main/base/4/4154
15/main/base/4/4155
15/main/base/4/4156
15/main/base/4/4157
15/main/base/4/4158
15/main/base/4/4159
15/main/base/4/4160
15/main/base/4/4163
15/main/base/4/4164
15/main/base/4/4165
15/main/base/4/4166
15/main/base/4/4167
15/main/base/4/4168
15/main/base/4/4169
15/main/base/4/4170
15/main/base/4/4171
15/main/base/4/4172
15/main/base/4/4173
15/main/base/4/4174
15/main/base/4/5002
15/main/base/4/548
15/main/base/4/549
15/main/base/4/6102
15/main/base/4/6104
15/main/base/4/6106
15/main/base/4/6110
15/main/base/4/6111
15/main/base/4/6112
15/main/base/4/6113
15/main/base/4/6116
15/main/base/4/6117
15/main/base/4/6175
15/main/base/4/6176
15/main/base/4/6228
15/main/base/4/6229
15/main/base/4/6237
15/main/base/4/6238
15/main/base/4/6239
15/main/base/4/826
15/main/base/4/827
15/main/base/4/828
15/main/base/4/PG_VERSION
15/main/base/4/pg_filenode.map
15/main/base/5/
15/main/base/5/112
15/main/base/5/113
15/main/base/5/1247
15/main/base/5/1247_fsm
15/main/base/5/1247_vm
15/main/base/5/1249
15/main/base/5/1249_fsm
15/main/base/5/1249_vm
15/main/base/5/1255
15/main/base/5/1255_fsm
15/main/base/5/1255_vm
15/main/base/5/1259
15/main/base/5/1259_fsm
15/main/base/5/1259_vm
15/main/base/5/13374
15/main/base/5/13374_fsm
15/main/base/5/13374_vm
15/main/base/5/13377
15/main/base/5/13378
15/main/base/5/13379
15/main/base/5/13379_fsm
15/main/base/5/13379_vm
15/main/base/5/13382
15/main/base/5/13383
15/main/base/5/13384
15/main/base/5/13384_fsm
15/main/base/5/13384_vm
15/main/base/5/13387
15/main/base/5/13388
15/main/base/5/13389
15/main/base/5/13389_fsm
15/main/base/5/13389_vm
15/main/base/5/13392
15/main/base/5/13393
15/main/base/5/1417
15/main/base/5/1418
15/main/base/5/16388
15/main/base/5/174
15/main/base/5/175
15/main/base/5/2187
15/main/base/5/2224
15/main/base/5/2228
15/main/base/5/2328
15/main/base/5/2336
15/main/base/5/2337
15/main/base/5/2579
15/main/base/5/2600
15/main/base/5/2600_fsm
15/main/base/5/2600_vm
15/main/base/5/2601
15/main/base/5/2601_fsm
15/main/base/5/2601_vm
15/main/base/5/2602
15/main/base/5/2602_fsm
15/main/base/5/2602_vm
15/main/base/5/2603
15/main/base/5/2603_fsm
15/main/base/5/2603_vm
15/main/base/5/2604
15/main/base/5/2605
15/main/base/5/2605_fsm
15/main/base/5/2605_vm
15/main/base/5/2606
15/main/base/5/2606_fsm
15/main/base/5/2606_vm
15/main/base/5/2607
15/main/base/5/2607_fsm
15/main/base/5/2607_vm
15/main/base/5/2608
15/main/base/5/2608_fsm
15/main/base/5/2608_vm
15/main/base/5/2609
15/main/base/5/2609_fsm
15/main/base/5/2609_vm
15/main/base/5/2610
15/main/base/5/2610_fsm
15/main/base/5/2610_vm
15/main/base/5/2611
15/main/base/5/2612
15/main/base/5/2612_fsm
15/main/base/5/2612_vm
15/main/base/5/2613
15/main/base/5/2615
15/main/base/5/2615_fsm
15/main/base/5/2615_vm
15/main/base/5/2616
15/main/base/5/2616_fsm
15/main/base/5/2616_vm
15/main/base/5/2617
15/main/base/5/2617_fsm
15/main/base/5/2617_vm
15/main/base/5/2618
15/main/base/5/2618_fsm
15/main/base/5/2618_vm
15/main/base/5/2619
15/main/base/5/2619_fsm
15/main/base/5/2619_vm
15/main/base/5/2620
15/main/base/5/2650
15/main/base/5/2651
15/main/base/5/2652
15/main/base/5/2653
15/main/base/5/2654
15/main/base/5/2655
15/main/base/5/2656
15/main/base/5/2657
15/main/base/5/2658
15/main/base/5/2659
15/main/base/5/2660
15/main/base/5/2661
15/main/base/5/2662
15/main/base/5/2663
15/main/base/5/2664
15/main/base/5/2665
15/main/base/5/2666
15/main/base/5/2667
15/main/base/5/2668
15/main/base/5/2669
15/main/base/5/2670
15/main/base/5/2673
15/main/base/5/2674
15/main/base/5/2675
15/main/base/5/2678
15/main/base/5/2679
15/main/base/5/2680
15/main/base/5/2681
15/main/base/5/2682
15/main/base/5/2683
15/main/base/5/2684
15/main/base/5/2685
15/main/base/5/2686
15/main/base/5/2687
15/main/base/5/2688
15/main/base/5/2689
15/main/base/5/2690
15/main/base/5/2691
15/main/base/5/2692
15/main/base/5/2693
15/main/base/5/2696
15/main/base/5/2699
15/main/base/5/2701
15/main/base/5/2702
15/main/base/5/2703
15/main/base/5/2704
15/main/base/5/2753
15/main/base/5/2753_fsm
15/main/base/5/2753_vm
15/main/base/5/2754
15/main/base/5/2755
15/main/base/5/2756
15/main/base/5/2757
15/main/base/5/2830
15/main/base/5/2831
15/main/base/5/2832
15/main/base/5/2833
15/main/base/5/2834
15/main/base/5/2835
15/main/base/5/2836
15/main/base/5/2836_fsm
15/main/base/5/2836_vm
15/main/base/5/2837
15/main/base/5/2838
15/main/base/5/2838_fsm
15/main/base/5/2838_vm
15/main/base/5/2839
15/main/base/5/2840
15/main/base/5/2840_fsm
15/main/base/5/2840_vm
15/main/base/5/2841
15/main/base/5/2995
15/main/base/5/2996
15/main/base/5/3079
15/main/base/5/3079_fsm
15/main/base/5/3079_vm
15/main/base/5/3080
15/main/base/5/3081
15/main/base/5/3085
15/main/base/5/3118
15/main/base/5/3119
15/main/base/5/3164
15/main/base/5/3256
15/main/base/5/3257
15/main/base/5/3258
15/main/base/5/3350
15/main/base/5/3351
15/main/base/5/3379
15/main/base/5/3380
15/main/base/5/3381
15/main/base/5/3394
15/main/base/5/3394_fsm
15/main/base/5/3394_vm
15/main/base/5/3395
15/main/base/5/3429
15/main/base/5/3430
15/main/base/5/3431
15/main/base/5/3433
15/main/base/5/3439
15/main/base/5/3440
15/main/base/5/3455
15/main/base/5/3456
15/main/base/5/3456_fsm
15/main/base/5/3456_vm
15/main/base/5/3466
15/main/base/5/3467
15/main/base/5/3468
15/main/base/5/3501
15/main/base/5/3502
15/main/base/5/3503
15/main/base/5/3534
15/main/base/5/3541
15/main/base/5/3541_fsm
15/main/base/5/3541_vm
15/main/base/5/3542
15/main/base/5/3574
15/main/base/5/3575
15/main/base/5/3576
15/main/base/5/3596
15/main/base/5/3597
15/main/base/5/3598
15/main/base/5/3599
15/main/base/5/3600
15/main/base/5/3600_fsm
15/main/base/5/3600_vm
15/main/base/5/3601
15/main/base/5/3601_fsm
15/main/base/5/3601_vm
15/main/base/5/3602
15/main/base/5/3602_fsm
15/main/base/5/3602_vm
15/main/base/5/3603
15/main/base/5/3603_fsm
15/main/base/5/3603_vm
15/main/base/5/3604
15/main/base/5/3605
15/main/base/5/3606
15/main/base/5/3607
15/main/base/5/3608
15/main/base/5/3609
15/main/base/5/3712
15/main/base/5/3764
15/main/base/5/3764_fsm
15/main/base/5/3764_vm
15/main/base/5/3766
15/main/base/5/3767
15/main/base/5/3997
15/main/base/5/4143
15/main/base/5/4144
15/main/base/5/4145
15/main/base/5/4146
15/main/base/5/4147
15/main/base/5/4148
15/main/base/5/4149
15/main/base/5/4150
15/main/base/5/4151
15/main/base/5/4152
15/main/base/5/4153
15/main/base/5/4154
15/main/base/5/4155
15/main/base/5/4156
15/main/base/5/4157
15/main/base/5/4158
15/main/base/5/4159
15/main/base/5/4160
15/main/base/5/4163
15/main/base/5/4164
15/main/base/5/4165
15/main/base/5/4166
15/main/base/5/4167
15/main/base/5/4168
15/main/base/5/4169
15/main/base/5/4170
15/main/base/5/4171
15/main/base/5/4172
15/main/base/5/4173
15/main/base/5/4174
15/main/base/5/5002
15/main/base/5/548
15/main/base/5/549
15/main/base/5/6102
15/main/base/5/6104
15/main/base/5/6106
15/main/base/5/6110
15/main/base/5/6111
15/main/base/5/6112
15/main/base/5/6113
15/main/base/5/6116
15/main/base/5/6117
15/main/base/5/6175
15/main/base/5/6176
15/main/base/5/6228
15/main/base/5/6229
15/main/base/5/6237
15/main/base/5/6238
15/main/base/5/6239
15/main/base/5/826
15/main/base/5/827
15/main/base/5/828
15/main/base/5/PG_VERSION
15/main/base/5/pg_filenode.map
15/main/base/5/pg_internal.init
15/main/global/
15/main/global/1213
15/main/global/1213_fsm
15/main/global/1213_vm
15/main/global/1214
15/main/global/1232
15/main/global/1233
15/main/global/1260
15/main/global/1260_fsm
15/main/global/1260_vm
15/main/global/1261
15/main/global/1261_fsm
15/main/global/1261_vm
15/main/global/1262
15/main/global/1262_fsm
15/main/global/1262_vm
15/main/global/2396
15/main/global/2396_fsm
15/main/global/2396_vm
15/main/global/2397
15/main/global/2671
15/main/global/2672
15/main/global/2676
15/main/global/2677
15/main/global/2694
15/main/global/2695
15/main/global/2697
15/main/global/2698
15/main/global/2846
15/main/global/2847
15/main/global/2964
15/main/global/2965
15/main/global/2966
15/main/global/2967
15/main/global/3592
15/main/global/3593
15/main/global/4060
15/main/global/4061
15/main/global/4175
15/main/global/4176
15/main/global/4177
15/main/global/4178
15/main/global/4181
15/main/global/4182
15/main/global/4183
15/main/global/4184
15/main/global/4185
15/main/global/4186
15/main/global/6000
15/main/global/6001
15/main/global/6002
15/main/global/6100
15/main/global/6114
15/main/global/6115
15/main/global/6243
15/main/global/6244
15/main/global/6245
15/main/global/6246
15/main/global/6247
15/main/global/pg_control
15/main/global/pg_filenode.map
15/main/global/pg_internal.init
15/main/pg_commit_ts/
15/main/pg_dynshmem/
15/main/pg_logical/
15/main/pg_logical/replorigin_checkpoint
15/main/pg_logical/mappings/
15/main/pg_logical/snapshots/
15/main/pg_multixact/
15/main/pg_multixact/members/
15/main/pg_multixact/members/0000
15/main/pg_multixact/offsets/
15/main/pg_multixact/offsets/0000
15/main/pg_notify/
15/main/pg_replslot/
15/main/pg_serial/
15/main/pg_snapshots/
15/main/pg_stat/
15/main/pg_stat/pgstat.stat
15/main/pg_stat_tmp/
15/main/pg_subtrans/
15/main/pg_subtrans/0000
15/main/pg_tblspc/
15/main/pg_tblspc/16391 -> /mnt/vdb1/tmptblspc
15/main/pg_twophase/
15/main/pg_wal/
15/main/pg_wal/000000010000000000000001
15/main/pg_wal/archive_status/
15/main/pg_xact/
15/main/pg_xact/0000

sent 40,294,650 bytes  received 18,559 bytes  80,626,418.00 bytes/sec
total size is 40,228,690  speedup is 1.00
dmitrydergunov95@test-vm:~$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
dmitrydergunov95@test-vm:~$ sudo systemctl start postgresql
dmitrydergunov95@test-vm:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Thu 2024-05-16 19:43:42 UTC; 4s ago
    Process: 6627 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 6627 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 16 19:43:42 test-vm systemd[1]: Starting PostgreSQL RDBMS...
May 16 19:43:42 test-vm systemd[1]: Finished PostgreSQL RDBMS.
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
dmitrydergunov95@test-vm:~$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Cluster is already running.
dmitrydergunov95@test-vm:~$ sudo pg_ctlcluster 15 main start
Job for postgresql@15-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@15-main.service" and "journalctl -xeu postgresql@15-main.service" for details.
```
Запустить не получается, т.к. не сменили старый путь к папке на новый. Думаю что делать дальше. Меняю:
```
dmitrydergunov95@test-vm:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
```
Что я поменяла:
```
data_directory = '/mnt/vdb1/15/main'            # use data in another directory
```
Поехали подключаться: 
```
dmitrydergunov95@test-vm:~$ sudo systemctl start postgresql
dmitrydergunov95@test-vm:~$ sudo systemctl stop postgresql
dmitrydergunov95@test-vm:~$ sudo systemctl start postgresql
dmitrydergunov95@test-vm:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Thu 2024-05-16 19:52:42 UTC; 5s ago
    Process: 6726 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 6726 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 16 19:52:42 test-vm systemd[1]: Starting PostgreSQL RDBMS...
May 16 19:52:42 test-vm systemd[1]: Finished PostgreSQL RDBMS.
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 down   postgres /mnt/vdb1/15/main /var/log/postgresql/postgresql-15-main.log
dmitrydergunov95@test-vm:~$ sudo pg_ctlcluster 15 main start
Job for postgresql@15-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@15-main.service" and "journalctl -xeu postgresql@15-main.service" for details.
dmitrydergunov95@test-vm:~$ sudo reboot
Connection to 51.250.87.207 closed by remote host.
Connection to 51.250.87.207 closed.
ERROR: exit status 255


client-trace-id: 778e1ace-86d9-433b-bf10-a011bf73653a

Use client-trace-id for investigation of issues in cloud support
If you are going to ask for help of cloud support, please send the following trace file: /home/vera/.config/yandex-cloud/logs/2024-05-16T22-39-32.117-yc_compute_ssh.txt
vera@DESKTOP-Q6JLRHI:~$ yc compute ssh --name test-vm --folder-id b1gaaj0i5bs0h43veitl
ssh: connect to host 51.250.87.207 port 22: Connection refused
ERROR: exit status 255


client-trace-id: c1fe023d-6122-437d-80ee-56b2ba3cc6f7

Use client-trace-id for investigation of issues in cloud support
If you are going to ask for help of cloud support, please send the following trace file: /home/vera/.config/yandex-cloud/logs/2024-05-16T22-55-10.507-yc_compute_ssh.txt
vera@DESKTOP-Q6JLRHI:~$ yc compute ssh --name test-vm --folder-id b1gaaj0i5bs0h43veitl
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-107-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu May 16 07:39:49 PM UTC 2024

  System load:  0.0                Processes:             148
  Usage of /:   24.2% of 19.59GB   Users logged in:       1
  Memory usage: 16%                IPv4 address for eth0: 10.128.0.19
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Thu May 16 19:39:50 2024 from 87.117.189.48
dmitrydergunov95@test-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner     Data directory    Log file
15  main    5432 down   <unknown> /mnt/vdb1/15/main /var/log/postgresql/postgresql-15-main.log
dmitrydergunov95@test-vm:~$ sudo pg_ctlcluster 15 main start
Error: /mnt/vdb1/15/main is not accessible or does not exist
dmitrydergunov95@test-vm:~$ ls /mnt/vdb1/15/main
ls: cannot access '/mnt/vdb1/15/main': No such file or directory
dmitrydergunov95@test-vm:~$ ls /mnt/vdb1/15
ls: cannot access '/mnt/vdb1/15': No such file or directory
dmitrydergunov95@test-vm:~$ ls /mnt/vdb1
dmitrydergunov95@test-vm:~$ ls /var/lib/postgresql/15
main
```
Если я правильно поняла - у меня не скопировались данные. и поэтому кластер не может стартовать. Что я сделала не так?
**Новая попытка**
```
dmitrydergunov95@otus-db-pg-vm-1:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0    87M  1 loop /snap/lxd/28373
loop1    7:1    0 111.9M  1 loop /snap/lxd/24322
loop2    7:2    0  63.3M  1 loop /snap/core20/1822
loop3    7:3    0  63.9M  1 loop /snap/core20/2318
loop4    7:4    0  38.7M  1 loop /snap/snapd/21465
loop5    7:5    0  38.8M  1 loop /snap/snapd/21759
vda    252:0    0    20G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0    20G  0 part /
vdb    252:16   0    10G  0 disk
└─vdb1 252:17   0     5G  0 part
dmitrydergunov95@otus-db-pg-vm-1:~$ lsblk --fs
NAME   FSTYPE   FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                    0   100% /snap/lxd/28373
loop1  squashfs 4.0                                                    0   100% /snap/lxd/24322
loop2  squashfs 4.0                                                    0   100% /snap/core20/1822
loop3  squashfs 4.0                                                    0   100% /snap/core20/2318
loop4  squashfs 4.0                                                    0   100% /snap/snapd/21465
loop5  squashfs 4.0                                                    0   100% /snap/snapd/21759
vda
├─vda1
└─vda2 ext4     1.0         ed465c6e-049a-41c6-8e0b-c8da348a3577   14.5G    22% /
vdb
└─vdb1 ext4     1.0         4133f77e-96f4-4d1c-9549-3766d2e1a498
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo mkdir -p /mnt//mnt/vdb1
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo mount -o defaults /dev/vdb1 /mnt/vdb1
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo chmod a+w /mnt/vdb1
dmitrydergunov95@otus-db-pg-vm-1:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1.1M  196M   1% /run
/dev/vda2        20G  4.3G   15G  23% /
tmpfs           982M     0  982M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           197M  4.0K  197M   1% /run/user/748239047
/dev/vdb1       4.9G   39M  4.6G   1% /mnt/vdb1
dmitrydergunov95@otus-db-pg-vm-1:~$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/ed465c6e-049a-41c6-8e0b-c8da348a3577 / ext4 defaults 0 1
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo mount -a
dmitrydergunov95@otus-db-pg-vm-1:~$ lsblk -fs
NAME  FSTYPE   FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0 squashfs 4.0                                                    0   100% /snap/lxd/28373
loop1 squashfs 4.0                                                    0   100% /snap/lxd/24322
loop2 squashfs 4.0                                                    0   100% /snap/core20/1822
loop3 squashfs 4.0                                                    0   100% /snap/core20/2318
loop4 squashfs 4.0                                                    0   100% /snap/snapd/21465
loop5 squashfs 4.0                                                    0   100% /snap/snapd/21759
vda1
└─vda
vda2  ext4     1.0         ed465c6e-049a-41c6-8e0b-c8da348a3577   14.5G    22% /
└─vda
vdb1  ext4     1.0         4133f77e-96f4-4d1c-9549-3766d2e1a498    4.5G     1% /mnt/vdb1
└─vdb
dmitrydergunov95@otus-db-pg-vm-1:~$ df -h -x tmpfs -x devtmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        20G  4.3G   15G  23% /
/dev/vdb1       4.9G   39M  4.6G   1% /mnt/vdb1
dmitrydergunov95@otus-db-pg-vm-1:~$ ls -l /mnt/vdb1
total 24
drwxr-xr-x 3 postgres postgres  4096 Jun  2 10:09 15
drwx------ 2 root     root     16384 Jun  2 10:19 lost+found
drwx------ 3 postgres postgres  4096 Jun  2 10:23 tmptblspc

dmitrydergunov95@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 down   postgres /mnt/vdb1/15/main /var/log/postgresql/postgresql-15-main.log
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main start
dmitrydergunov95@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/vdb1/15/main /var/log/postgresql/postgresql-15-main.log
dmitrydergunov95@otus-db-pg-vm-1:~$ sudo -u postgres psql
psql (15.7 (Ubuntu 15.7-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
 id
----
  1
  2
  3
(3 rows)

```
