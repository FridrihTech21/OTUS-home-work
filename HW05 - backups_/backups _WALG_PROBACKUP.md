### 1. Настройте бэкапы PostgreSQL с использованием WAL-G, pg_probackup или любого другого аналогичного ПО для базы данных "Лояльность оптовиков".

Скачиваем бинарник `WAL-G`:
```
wget https://github.com/wal-g/wal-g/releases/download/v3.0.7/wal-g-pg-ubuntu-24.04-amd64.tar.gz
```
Далее распаковываем его и переносим в `/usr/local/bin/`:
```
tar -zxvf wal-g-pg-ubuntu-24.04-amd64.tar.gz
sudo mv wal-g /usr/local/bin/
```
Проверим версию `WAL-G`:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_5/1.jpg)

На скриншоте сверху видно, что `WAL-G`, `PostgreSQL` установлены.

Далее создаем рабочее окружение для работы `WAL-G`:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_5/2.jpg)

Конфиг-файл `WAL-G`:
<details>
<summary>vi /var/lib/postgresql/.walg.json</summary>
  
```bash
{
    "WALG_FILE_PREFIX": "/opt/wal-g/backups/",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "WALG_UPLOAD_DISK_CONCURRENCY": "4",
    "PGDATA": "/var/lib/postgresql/16/main",
    "PGHOST": "localhost"
}

```
</details>

`WALG_ DELTA_ MAX_ STEPS` — максимальное количество шагов `(delta steps)`, которые `WAL-G` может использовать для постепенно увеличивающихся (дельта) резервных копий. Позволяет экономить место, храня только изменения между резервными копиями.
`WALG_UPLOAD_DISK_CONCURRENCY` - Для настройки количества параллельных потоков, считывающих данные с диска во время `backup-push`, используется параметр `wal_concurrency`. По умолчанию, `WAL-G` использует 1 поток.

Создаем каталог для логов: `mkdir -p /opt/wal-g/log`

Настраиваем параметры конфигурации:
```
echo "wal_level=replica" >> /var/lib/postgresql/16/main/postgresql.auto.conf
echo "archive_mode=on" >> /var/lib/postgresql/16/main/postgresql.auto.conf
echo "archive_timeout=60" >> /var/lib/postgresql/16/main/postgresql.auto.conf 
echo "archive_command='wal-g wal-push \"%p\" >> /opt/wal-g/log/archive_command.log 2>&1' " >> /var/lib/postgresql/16/main/postgresql.auto.conf
echo "restore_command='wal-g wal-fetch \"%f\" \"%p\" >> /opt/wal-g/log/restore_command.log 2>&1' " >> /var/lib/postgresql/16/main/postgresql.auto.conf
cat /var/lib/postgresql/16/main/postgresql.auto.conf
vi /etc/postgresql/16/main/pg_hba.conf --->>> host all all 127.0.0.1/32 trust
sudo  systemctl restart postgresql
```
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_5/3.jpg)

После рестарта PostgreSQL проверим archive mode:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_5/4.jpg)

Создадим и наполним БД:
<details>
<summary>CREATE DB</summary>

 ```
create database loyalty_optoviks;
\c loyalty_optoviks;
create table optoviks (id serial primary key, company_name varchar(255) not null, contact_person varchar(255), email varchar(255), phone varchar(50));
create table products (id serial primary key, product_name varchar(255) not null, category varchar(100), price decimal(10, 2) check (price >= 0));
create table orders (id serial primary key, optovik_id integer references optoviks(id), order_date timestamp default now(), total_amount decimal(12, 2) check (total_amount >= 0));
create table loyalty_points (id serial primary key, optovik_id integer references optoviks(id), points integer check (points >= 0));
insert into optoviks (company_name, contact_person, email, phone) values
('Оптовик ООО "Гросс"', 'Иванов И.И.', 'ivanov@gross.ru', '+7 (495) 123-45-67'),
('Компания "Опт-Трейд"', 'Петрова П.П.', 'petrova@opt-trade.com', '+7 (495) 987-65-43');
insert into products (product_name, category, price) values
('Смартфон Model X', 'Электроника', 59999.99),
('Ноутбук ProBook Z', 'Компьютеры', 89999.00);
insert into orders (optovik_id, total_amount) values (1, 150000.00), (2, 75000.50);
insert into loyalty_points (optovik_id, points) values (1, 1500), (2, 750);
select * from optoviks;
select * from products;
select * from orders;
select * from loyalty_points;
 ```
</details>

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_5/5.jpg)

### 2. Восстановите данные на другом кластере, чтобы убедиться, что бэкапы работают.

Сначала снимем бекап:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_5/6.jpg)

Как видно, WAL-G успешно настроен. Создан первый полный бэкап базы данных PostgreSQL.

После чего, "тушим" наш экземпляр PostgreSQL: `sudo systemctl stop postgresql`
Создаем новый экземпляр, и выполняем рестор:
```
pg_createcluster -d /var/lib/postgresql/16/main2 16 main2
rm -rf /var/lib/postgresql/16/main2/*
wal-g backup-fetch /var/lib/postgresql/16/main2 LATEST
touch /var/lib/postgresql/16/main2/recovery.signal
sudo systemctl start postgresql@16-main2
```

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_5/7.jpg)


После рестора наглядно видно, что все данные в иходной БД перенеслись в развернутую при помощи бекапа.
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_5/8.jpg)
