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
Создавать виртуальное окружение Patroni будем при помощи `python3.12-venv`. Для того, чтобы воспользоваться виртуальным окружением, требуется каталог для него, владелец каталога должен быть `postgres`. Также незабываем установить Patroni и драйвер для работы Patroni c PostgreSQL. Все это продемонстрированно на рисунках ниже:
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

