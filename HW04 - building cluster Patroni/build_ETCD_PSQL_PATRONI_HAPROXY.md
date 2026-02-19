### 1. Создайте 3 виртуальные машины для etcd и 3 виртуальные машины для Patroni.

Созданные машины:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/0_5.jpg)

### 2. Разверните HA-кластер PostgreSQL с использованием Patroni.

Сетевой стэк будет таким:
```
10.92.36.60 tarasov-postgre-advance-etcd-1
10.92.36.131 tarasov-postgre-advance-etcd-2
10.92.36.28 tarasov-postgre-advance-etcd-3
10.92.36.93  tarasov-postgre-advance-1
10.92.36.99  tarasov-postgre-advance-2
10.92.36.12  tarasov-postgre-advance-3
10.92.36.120 tarasov-postgre-advance-ha-proxy
```
Для наглядности на просторах сети Интернет была найдена диаграмма построения HA-кластера, включающего в себя: `ETCD`, `PostgreSQL`, `Patroni`, `HAProxy`.
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/0.webp)

Дак развернем же его!

Обновляем пакеты: `sudo apt update`

Прорисываем на каждом хосте в /etc/hosts:
```
sudo tee -a /etc/hosts <<EOF
10.92.36.60 tarasov-postgre-advance-etcd-1
10.92.36.131 tarasov-postgre-advance-etcd-2
10.92.36.28 tarasov-postgre-advance-etcd-3
10.92.36.93 tarasov-postgre-advance-1
10.92.36.99 tarasov-postgre-advance-2
10.92.36.12 tarasov-postgre-advance-3
10.92.36.120 tarasov-postgre-advance-ha-proxy
EOF
```
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/1.jpg)

## 2.1 Шаг 1. Установка и настройка ETCD

Устанавливаем ETCD:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/2.jpg)


Редактируем конфиг-файл на каждой ноде etcd:

```
vi /etc/default/etcd
```
<details>
<summary>/etc/default/etcd</summary>
  
  ```bash
#ETCD_1:
ETCD_NAME="etcd1"
ETCD_LISTEN_CLIENT_URLS="http://10.92.36.60:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://tarasov-postgre-advance-etcd-1:2379"
ETCD_LISTEN_PEER_URLS="http://10.92.36.60:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://tarasov-postgre-advance-etcd-1:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://tarasov-postgre-advance-etcd-1:2380,etcd2=http://tarasov-postgre-advance-etcd-2:2380,etcd3=http://tarasov-postgre-advance-etcd-3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"

#ETCD_2:
ETCD_NAME="etcd2"
ETCD_LISTEN_CLIENT_URLS="http://10.92.36.131:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://tarasov-postgre-advance-etcd-2:2379"
ETCD_LISTEN_PEER_URLS="http://10.92.36.131:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://tarasov-postgre-advance-etcd-2:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://tarasov-postgre-advance-etcd-1:2380,etcd2=http://tarasov-postgre-advance-etcd-2:2380,etcd3=http://tarasov-postgre-advance-etcd-3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"


#ETCD_3:
ETCD_NAME="etcd3"
ETCD_LISTEN_CLIENT_URLS="http://10.92.36.28:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://tarasov-postgre-advance-etcd-3:2379"
ETCD_LISTEN_PEER_URLS="http://10.92.36.28:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://tarasov-postgre-advance-etcd-3:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://tarasov-postgre-advance-etcd-1:2380,etcd2=http://tarasov-postgre-advance-etcd-2:2380,etcd3=http://tarasov-postgre-advance-etcd-3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```
</details>


После сохранения конфиг-файла ETCD рестартуем сервис, и смотрим статус по кластеру ETCD:
```
etcdctl endpoint status --cluster -w table
```
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/3.jpg)

## 2.2 Шаг 2. Установка и настройка PostgreSQL

Устанавливаем с помощью команды:
```
sudo apt -y install postgresql
```

После чего, требуется провести настройки и некоторые приготовления для установки Patroni:
1) Создать пользователя `replicator`;
2) Добавить записи на разрешения коннекта к БД в pg_hba.conf;
3) Отредактировать звапись `listen_address`, для прослушивания всех адресов;
4) Рестарт PostgreSQL;
5) Удаление содержимого PGDATA на 2-й и 3-ей ноде.
   
Выпонление данных пунктов показано на картинке ниже:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/4.jpg)

## 2.3 Шаг 3. Установка и настройка кластера Patroni

Patroni будет крутиться в изолированной среде Python для того, чтобы не нарушать систему, контролировать зависимости.
Создавать виртуальное окружение Patroni будем при помощи `python3.12-venv`. Для того, чтобы воспользоваться виртуальным окружением, требуется каталог для него, владелец каталога должен быть `postgres`. Также незабываем установить Patroni и драйвер для работы Patroni c PostgreSQL. Все это продемонстрировано на рисунках ниже:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/5.jpg)
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/6.jpg)
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/7.jpg)
***
Не забываем создать PGDATA, директорию для логов Patroni и конфига:`sudo mkdir -p /data/16` || `sudo mkdir -p /data/log/patroni` || `sudo mkdir -p /etc/patroni/`

И назначить овнера postgres:`sudo chown -R postgres:postgres /data` || `sudo chown -R postgres:postgres /data/log/patroni` || `sudo chown -R postgres:postgres /etc/patroni/`

Назначаем полные права владльцу PGDATA: `sudo chmod 700 /data/16`
***

Далее настроем конфигурацию Patroni в конфиг-файле `sudo vi /etc/patroni/conf.yml`:
<details>
<summary>/etc/patroni/conf.yml</summary>
  
```bash
scope: patroni_cluster
namespace: /patroni
name: patroni_node2
log:
  level: INFO
  dir: /data/log/patroni
  file_size: 50000000
  file_num: 10
restapi:
  listen: 10.92.36.99:8008
  connect_address: 10.92.36.99:8008

etcd:
  hosts: tarasov-postgre-advance-etcd-1:2379,tarasov-postgre-advance-etcd-2:2379,tarasov-postgre-advance-etcd-3:2379

bootstrap:
  dcs:
    failsafe_mode: true
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: true
    synchronous_mode_strict: true
    synchronous_mode_count: 1
    master_start_timeout: 30
    slots:
      prod_replica1:
        type: physical

  initdb:
    - encoding: UTF8

  pg_hba:
    - host replication replicator 127.0.0.1/8 md5
    - host replication replicator 10.92.36.93/32 md5
    - host replication replicator 10.92.36.99/32 md5
    - host replication replicator 10.92.36.12/32 md5
    - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: 'qwerty@123'
      options:
        - CREATEDB
        - CREATEROLE

postgresql:
  listen: 127.0.0.1,10.92.36.99:5432
  connect_address: 10.92.36.99:5432
  data_dir: /data/16
  bin_dir: /usr/lib/postgresql/16/bin
  authentication:
    replication:
      username: replicator
      password: 'qwerty@123'
    superuser:
      username: postgres
      password: 'qwerty@123'
    rewind:
      username: rewind_user
      password: 'qwerty@123'
  parameters:
    unix_socket_directories: '.'
  create_replica_methods: ["basebackup"]
  basebackup:
    max-rate: 100M
    checkpoint: fast

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```
</details>

Создаем Service-Unit `sudo vi /etc/systemd/system/patroni.service`:
<details>
<summary>/etc/systemd/system/patroni.service</summary>
  
```bash
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target
[Service]
Type=simple:
User=postgres
Group=postgres
ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni/conf.yml
KillMode=process
TimeoutSec=30
Restart=no
[Install]
WantedBy=multi-user.target
```
</details>

Следующим шагом будет застваить Systemd перечитать daemon-reload и запустить Patroni:
```
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
```

Запуск Patroni показан на рисунке:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/8.jpg)

Теперь проверим, что манипуляции были не бессмыслены, и кластер Patroni поднят:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/9.jpg)


## 2.4 Проверим работу Patroni 

Для проверки БД выполним простой тест с `tarasov-postgre-advance-1(лидером)` и `tarasov-postgre-advance-2(стендбаем = реплика)`. Цель: показать показать различие между лидером и репликой Patroni:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/10.jpg)
На рисунке видно, что была создана таблица `test_1`, а также наполнена лидером. Можно заметить, что на реплике при обновлении записи вызвана ошибка: `ERROR:  cannot execute UPDATE in a read-only transaction`. 
Данная ошибка валидна для реплики, т.к. реплики использются для чтения записей БД. Это сделано с целью разгрузить потоки запросов к кластеру. 
Соответсвенно, лидер может принимать запросы как на чтение так и на запись.

Также, можно заглянуть в логи. В них виден аудит по таблице `test_1`:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/11.jpg)

Теперь к самому интересному: `failover`. Просто перезапустим нынешнего лидера, после чего помотрим кто будет назначен лидером из 2-х реплик:

Со скриншота видно, что лидером является `patroni_node1`:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/12.jpg)

После рестарта службы на `patroni_node1` лидером становиться `patroni_node2`:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/13.jpg)
Почему именно `patroni_node2`??? Все просто: `patroni_node2` был в приоритете на назначение его лидером в кластере. 

### 3. Настройте HAProxy для балансировки нагрузки.

## 3.1 Шаг №1. Установка HAProxy

Устанавливаем HAProxy через:
```
sudo apt-get update
sudo apt-get install -y haproxy
```
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/14.jpg)

## 3.2 Шаг №2. Настройка HAProxy

Редактируем кофиг-файл HAProxy: `sudo vi /etc/haproxy/haproxy.cfg`
<details>
<summary>haproxy.cfg</summary>
  
```bash
global
   maxconn 100
   log 127.0.0.1 local2

defaults
   log global
   mode tcp
   retries 2
   timeout client 30m
   timeout connect 4s
   timeout server 30m
   timeout check 5s

listen stats
   mode http
   bind *:7000
   stats enable
   stats uri /

listen production
   bind *:5000
   option httpchk GET/master
   http-check expect status 200
   default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
   server tarasov-postgre-advance-1 10.92.36.93:5432 maxconn 100 check port 8008
   server tarasov-postgre-advance-2 10.92.36.99:5432 maxconn 100 check port 8008
   server tarasov-postgre-advance-3 10.92.36.12:5432 maxconn 100 check port 8008

listen standby
   bind *:5001
   option httpchk GET/replica
   http-check expect status 200
   default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
   server tarasov-postgre-advance-1 10.92.36.93:5432 maxconn 100 check port 8008
   server tarasov-postgre-advance-2 10.92.36.99:5432 maxconn 100 check port 8008
   server tarasov-postgre-advance-3 10.92.36.12:5432 maxconn 100 check port 8008
```
</details>

Как видно из скринов ниже, HAProxy запущен:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/15.jpg)
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/16.jpg)

Теперь попробуем подсоеденится к БД для чтения и записи. 
На скриншоте видно, что подключение по 5000 приведет на на мастер Patroni, а подключение по 5001 порту приведет нас к одной из реплик.
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/17.jpg)

### 4. Проверьте отказоустойчивость кластера, имитируя сбой на одном из узлов.

Имитриуем сбой кластера путем остановки сервиса `Patroni` на лидере кластера. 
Ождаемое поведение:
1) При выходе из строя лидера одна из реплик, являбщаяся `stanby` становиться новым лидером;
2) Кластер продалжает функционировать, значит должен принимать запросы на запись и чтение;
3) Проверка `HAProxy`. Запросы перенаправляются на новый лидер;
4) Проверка `HAProxy`. Запросы на чтение перенаправляются едиснтвенной оставшейся реплике;
5) После восстановления бывший лидер становится репликой, нынешняя реплика должны быть `Sync Stanby`.

Исходные данные:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/18.jpg)
`patroni_node1` является лидером кластера.
`patroni_node2` является репликой кластера.
`patroni_node3` является стэндбаем, соотвественно данная нода должна взять на себя роль лидера, если `patroni_node1` откажет.

Статистика HAProxy на момент нормальной работы кластера, на ней видно перечисленные ноды Patroni, какие роли занимают.
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/19.jpg)

## 4.1 Остановка лидера Patroni

Останавливаем лидер Patroni `sudo systemctl stop patroni`:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/20.jpg)

После просмотра статуса кластера видно, что `patroni_node2` не запустилась как реплика. 
Откроем логи: ` tail -n 50 /data/log/postgresql-2026-02-19_06.log`
<details>
<summary>postgresql-2026-02-19_06.log</summary>
  
```bash
2026-02-19 06:34:41 UTC [4604]: [3-1] user=,db=,app=,client= LOG:  starting PostgreSQL 16.11 (Ubuntu 16.11-0ubuntu0.24.04.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, 64-bit
2026-02-19 06:34:41 UTC [4604]: [4-1] user=,db=,app=,client= LOG:  listening on IPv4 address "127.0.0.1", port 5432
2026-02-19 06:34:41 UTC [4604]: [5-1] user=,db=,app=,client= LOG:  listening on IPv4 address "10.92.36.99", port 5432
2026-02-19 06:34:41 UTC [4604]: [6-1] user=,db=,app=,client= LOG:  listening on Unix socket "./.s.PGSQL.5432"
2026-02-19 06:34:41 UTC [4609]: [1-1] user=,db=,app=,client= LOG:  database system was shut down in recovery at 2026-02-19 06:28:27 UTC
2026-02-19 06:34:41 UTC [4609]: [2-1] user=,db=,app=,client= LOG:  entering standby mode
2026-02-19 06:34:41 UTC [4609]: [3-1] user=,db=,app=,client= FATAL:  requested timeline 9 is not a child of this server's history
2026-02-19 06:34:41 UTC [4609]: [4-1] user=,db=,app=,client= DETAIL:  Latest checkpoint is at 0/6000028 on timeline 8, but in the history of the requested timeline, the server forked off from that timeline at 0/50A8F20.
2026-02-19 06:34:41 UTC [4610]: [1-1] user=[unknown],db=[unknown],app=[unknown],client=127.0.0.1 LOG:  connection received: host=127.0.0.1 port=47334
2026-02-19 06:34:41 UTC [4610]: [2-1] user=postgres,db=postgres,app=[unknown],client=127.0.0.1 FATAL:  the database system is starting up
2026-02-19 06:34:41 UTC [4604]: [7-1] user=,db=,app=,client= LOG:  startup process (PID 4609) exited with exit code 1
2026-02-19 06:34:41 UTC [4604]: [8-1] user=,db=,app=,client= LOG:  aborting startup due to startup process failure
2026-02-19 06:34:41 UTC [4604]: [9-1] user=,db=,app=,client= LOG:  database system is shut down
```
</details>


Проблема: 
```
2026-02-19 06:34:41 UTC [4609]: [3-1] user=,db=,app=,client= FATAL:  requested timeline 9 is not a child of this server's history
2026-02-19 06:34:41 UTC [4609]: [4-1] user=,db=,app=,client= DETAIL:  Latest checkpoint is at 0/6000028 on timeline 8, but in the history of the requested timeline, the server forked off from that timeline at 0/50A8F20.
```
Данная ошибка свидетельствует о несоответсвии временной шкалы, что можно было наблюдать на скриншоте в начале раздела №4(скриншот статуса кластера). На нем видно, что поле TL(timeline) разное у `patroni_node2` с `patroni_node1` и `patroni_node3`. В PostgreSQL TL используется для отслежевания точек восстановления после восстановления или как в нашем случае `failover`.
После `failover`, когда `patroni_node3` стал лидером, он, создал новую временную шкалу `(timeline 9)`, начав запись `WAL` с того места, где остановился предыдущий лидер. Узел `patroni_node2`, который был отключен до `failover`, оставался на старой временной шкале `(timeline 8)`. Когда `Patroni` на `node2` попытался запустить `PostgreSQL` и синхронизироваться с кластером, `PostgreSQL` обнаружил, что ему предлагается восстановиться на `timeline 9`, которая не является продолжением его текущей истории `(timeline 8)`. Соответственно, `patroni_node2` не может синхронизироваться, так как его данные устарели и принадлежат к "мертвой" ветке истории `WAL`. Для восстановления узла в кластер требуется процедура `reinit`, которая пересоздаст его данные с помощью `pg_basebackup` с текущего лидера, приведя его в актуальное состояние на `timeline 9`:
```
patronictl -c /etc/patroni/conf.yml reinit patroni_cluster patroni_node2
```

Как видно, `node2` восстановлена. Также, можно заметить, что теперь её роль `Sync Standby`:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/21.jpg)

На картинке ниже видна собранная статистика `HAProxy` по кластеру `Patroni`, и видно, что ее показания "бъются" с реальной картиной:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/22.jpg)

## 4.2 Проверка работоспособонсти кластера

После "аварийной" ситуации соответствующий трафик должен быть направлен на соответсвующие ноды кластера. Проверим, подключившись для этого к узлу `HAProxy` по портам `5000(Leader)` и `5001(Replica)`:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/23.jpg)

Как видно со скриншота, перенаправление подключений соответсвующему трафику работает. Доказывает это запрос `select pg_is_in_recovery();` на двух нодах, где `t` - реплика; `f` - лидер. А также проделанные операции и их результаты.

## 4.3 Восстановление "упавшей" ноды

Запустим службу Patroni на `node1`:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/24.jpg)

Видно, что `node1` запущена, работает и синхронизирована с остальными нодами. Посмотрим на статистику `HAProxy`:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/4_v2/25.jpg)

На ней также видно, что `node1` запущена, и работает исправно.

## 4.4 Итог

Мы успешно имитрировали аварийную ситуацию, а также произошла непредвиденная ошибка по рассинхронизации TL, которая была исправлена. 
После восстановления кластера, ождиаемые роли были заняты соответствующими нодами:
1) `patroni_node1` является репликой кластера;
2) `patroni_node2` является стэндбаем кластера;
3) `patroni_node3` является лидером кластера.
