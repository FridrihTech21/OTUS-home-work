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
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/5/1.jpg)

На скриншоте сверху видно, что `WAL-G`, `PostgreSQL` установлены.

Далее создаем рабочее окружение для работы `WAL-G`:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/5/2.jpg)

Конфиг-файл `WAL-G`:
<details>
<summary>vi /opt/wal-g/.walg.json</summary>
  
```bash
{
    "WALG_FILE_PREFIX": "/opt/wal-g/backups/",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "WALG_UPLOAD_DISK_CONCURRENCY": "4",
    "PGDATA": "/usr/lib/postgresql/16/bin/",
    "PGHOST": "localhost"
}

```
</details>

`WALG_ DELTA_ MAX_ STEPS` — максимальное количество шагов `(delta steps)`, которые `WAL-G` может использовать для постепенно увеличивающихся (дельта) резервных копий. Позволяет экономить место, храня только изменения между резервными копиями.
`WALG_UPLOAD_DISK_CONCURRENCY` - Для настройки количества параллельных потоков, считывающих данные с диска во время `backup-push`, используется параметр `wal_concurrency`. По умолчанию, `WAL-G` использует 1 поток.

