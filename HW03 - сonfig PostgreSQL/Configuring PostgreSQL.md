# 1. Создайте виртуальную машину с Ubuntu 20.04 и установите PostgreSQL 15 или выше:
```
postgres@tarasov-postgre-advance:/home/fvtarasov$ psql --version
psql (PostgreSQL) 17.7 (Ubuntu 17.7-3.pgdg24.04+1)
postgres@tarasov-postgre-advance:/home/fvtarasov$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

# 2. Создайте таблицу с данными о перевозках:
```
postgres@tarasov-postgre-advance:/home/fvtarasov$ psql postgres
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
Type "help" for help.

postgres=# CREATE TABLE shipments (
postgres(#     id SERIAL,
postgres(#     product_name TEXT,
postgres(#     quantity INT,
postgres(#     destination TEXT
postgres(# );
CREATE TABLE
postgres=# INSERT INTO shipments (product_name, quantity, destination) VALUES
postgres-# ('bananas', 1000, 'Europe'),
postgres-# ('bananas', 1500, 'Asia'),
postgres-# ('bananas', 2000, 'Africa'),
postgres-# ('coffee', 500, 'USA'),
postgres-# ('coffee', 700, 'Canada'),
postgres-# ('coffee', 300, 'Japan'),
postgres-# ('sugar', 1000, 'Europe'),
postgres-# ('sugar', 800, 'Asia'),
postgres-# ('sugar', 600, 'Africa'),
postgres-# ('sugar', 400, 'USA'),
postgres-# ('rice', 1200, 'China'),
postgres-# ('rice', 900, 'India'),
postgres-# ('tea', 300, 'Russia'),
postgres-# ('tea', 450, 'UK'),
postgres-# ('tea', 200, 'Germany'),
postgres-# ('cotton', 2500, 'Turkey'),
postgres-# ('cotton', 3000, 'USA'),
postgres-# ('cocoa', 600, 'Belgium'),
postgres-# ('cocoa', 800, 'Netherlands'),
postgres-# ('spices', 150, 'Italy');
INSERT 0 20
```

# 3. Добавьте внешний диск к виртуальной машине и перенесите туда базу данных:

### 3.1 Подключил диск tarasov-otus-disk:
![КАРТИНКА1](/OTUS-home-work/project/lab_3/1.jpg)

```
fvtarasov@tarasov-postgre-advance:~$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME    FSTYPE  SIZE MOUNTPOIN LABEL
vda              20G
├─vda1  ext4   19.4G /         cloudimg-rootfs
├─vda14           4M
└─vda15 vfat    600M /boot/efi UEFI
vdb              20G
```

### 3.2 Создал раздел:
![КАРТИНКА2](project/lab_3/2.jpg)

### 3.3 Отформатировал диск и смонтировал раздел:
![КАРТИНКА3](project/lab_3/3.jpg)

### 3.4 Создал каталог tmptblspc для tablespace:
```
fvtarasov@tarasov-postgre-advance:~$ sudo su - postgres
postgres@tarasov-postgre-advance:~$ cd /mnt/vdb1 && mkdir tmptblspc
postgres@tarasov-postgre-advance:/mnt/vdb1$ psql
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
Type "help" for help.

postgres=# CREATE TABLESPACE tmptblspc LOCATION '/mnt/vdb1/tmptblspc';
CREATE TABLESPACE
```

### 3.5 Сделал дамп БД:
```
postgres@tarasov-postgre-advance:/mnt/vdb1$ pg_dump -Fc postgres > /tmp/backup.dump
postgres@tarasov-postgre-advance:/mnt/vdb1$ ls -la /tmp
total 56
drwxrwxrwt 12 root     root     4096 Jan 17 18:10 .
drwxr-xr-x 21 root     root     4096 Jan 17 16:54 ..
drwxrwxrwt  2 root     root     4096 Jan 17 16:54 .ICE-unix
drwxrwxrwt  2 root     root     4096 Jan 17 16:54 .X11-unix
drwxrwxrwt  2 root     root     4096 Jan 17 16:54 .XIM-unix
drwxrwxrwt  2 root     root     4096 Jan 17 16:54 .font-unix
-rw-rw-r--  1 postgres postgres 5564 Jan 17 18:10 backup.dump
```

### 3.6 Создал целевую БД:
```
postgres@tarasov-postgre-advance:/mnt/vdb1$ psql
psql (17.7 (Ubuntu 17.7-3.pgdg24.04+1))
Type "help" for help.

postgres=# create database postgres_unbackup tablespace tmptblspc;
CREATE DATABASE
```

### 3.7 Восстановил из дампа БД:
`postgres@tarasov-postgre-advance:/mnt/vdb1$ pg_restore -d postgres_unbackup /tmp/backup.dump`


### 4. Проверка целеостности данных - данные сохранились и доступны:
![КАРТИНКА4](project/lab_3/3.jpg)
