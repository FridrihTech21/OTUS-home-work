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
![КАРТИНКА1](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_3/1.jpg)

Согласно выводу команды lsblk, успешно подключен дополнительный диск vdb объемом 20 ГБ к системе. Диск виден в системе, но пока не имеет файловой системы и точки монтирования.
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
Создал один основной (primary) раздел /dev/vdb1 размером 20 ГБ, который занимает весь доступный объем диск.
![КАРТИНКА2](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_3/2.jpg)

### 3.3 Отформатировал диск и смонтировал раздел:

Создал файловую систему ext4 на разделе /dev/vdb1. Создал точку монтирования /mnt/vdb1. Смонтировал раздел в эту точку. Выдал права на запись всем пользователям (chmod a+w).
Успешно подключил диск - как видно из lsblk, раздел /dev/vdb1 теперь смонтирован в /mnt/vdb1.

![КАРТИНКА3](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_3/3.jpg)

### 3.4 Создал каталог tmptblspc для tablespace:

Создал каталог для ТБС tmptblspc и привязал его к нему.
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
Наиболее гибкие форматы выходных файлов это «custom» (-Fc) и «directory» (-Fd). Они позволяют выбрать и изменить порядок объектов, поддерживают восстановление в несколько потоков, а также сжимаются по умолчанию. При этом только формат «directory» поддерживает выгрузку данных в несколько потоков.
Выгрузка базы данных в специальном формате:
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

Подключаемся к целеовой БД и восстанавливаем родительскую:

`postgres@tarasov-postgre-advance:/mnt/vdb1$ pg_restore -d postgres_unbackup /tmp/backup.dump`


### 4. Проверка целеостности данных - данные сохранились и доступны:
Проверим, что дамп успешно развернулся:

![КАРТИНКА4](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_3/4.jpg)

Но, если перезапустить машину, то при подключении к новой развернутой БД ошибка:
![КАРТИНКА6](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_3/6.jpg)

Заходим в логи PSQL и фильтруем вывод по FATAL:
![КАРТИНКА7](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_3/7.jpg)

Проверяем каталог для ТБС на наличие прав, кому принадлежит каталог и его существование:
```fvtarasov@tarasov-postgre-advance:~$ ls -la /mnt/vdb1/
total 8
drwxr-xr-x 2 root root 4096 Feb 12 08:56 .
drwxr-xr-x 3 root root 4096 Feb 12 08:56 ..
```

Каталог пустой, это свзано с тем, что раздел не смонтирован сейчас, что подтверждает пустое поле MOUNPOINTS раздела /vdb1:
```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   20G  0 disk
├─vda1  253:1    0 19.4G  0 part /
├─vda14 253:14   0    4M  0 part
└─vda15 253:15   0  600M  0 part /boot/efi
vdb     253:16   0   20G  0 disk
└─vdb1  253:17   0   20G  0 part
```

Примем действия для разрешения проблемы:
![КАРТИНКА8](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_3/8.jpg)
`sudo mount /dev/vdb1 /mnt/vdb1`: Успешно смонтировал раздел /dev/vdb1 в точку /mnt/vdb1. Теперь он доступен как обычный каталог.

`ls -la /mnt/vdb1/`: Показывает содержимое смонтированного диска.

Видны стандартные элементы: `lost+found (системная папка ext4)` и созданная ранее папка `tmptblspc` для табличного пространства PostgreSQL. Владельцы указаны корректно (postgres для tmptblspc).

`sudo blkid /dev/vdb1`: Получил UUID диска: `6b59de7a-fe90-422e-ada2-3d29d310ce6e`. Это более надёжный способ ссылаться на диск в /etc/fstab, чем имя устройства /dev/vdb1, которое может измениться.

`echo 'UUID=... /mnt/vdb1 ext4 defaults 0 2' | sudo tee -a /etc/fstab`: Добавил строку в файл /etc/fstab, которая указывает системе, что при загрузке нужно автоматически смонтировать диск с указанным UUID в /mnt/vdb1 как ext4.

`sudo mount -a`: Проверил синтаксис /etc/fstab. Команда выполнилась без ошибок, значит, строка добавлена правильно.

`df -h | grep vdb`1: Подтверждает, что раздел /dev/vdb1 в настоящее время смонтирован в /mnt/vdb1 и система видит его свободное место.

Теперь после перезапуска машины диск /dev/vdb1 будет автоматически смонтирован в /mnt/vdb1, и табличное пространство PostgreSQL tmptblspc, расположенное там, должно быть доступно. 

Проверим работу примонтированного устройства:
![КАРТИНКА9](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_3/9.jpg)

На картинке видно, что `uptime = 0 min`. Восстановленная БД доступна для использования, данные сохранены. 
