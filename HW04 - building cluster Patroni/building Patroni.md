# 1. Создайте 3 виртуальные машины для etcd и 3 виртуальные машины для Patroni.
Машины для проведения лабараторной работы созданы: 
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_4/1.jpg)

# 2. Разверните HA-кластер PostgreSQL с использованием Patroni.
Данный этап разделим на три части:
```
2.1 Установка и настройка etcd;
2.2 Установка PostgreSQL 17;
2.3 Установка и настройка Patroni.
```

ETCD выступает в роли свидетеля, наблюдателя. Для обеспечения HA будет поднято 3 ноды etcd.
Patroni выступает инструментом для управления высокодоступными кластерами PostgreSQL. Он упрощает настройку и управление репликацией благодаря автоматическому переключению на резервные узлы и восстановлению после сбоев.

Сетевая структура следующая:
```
etcd1: 10.92.36.60
etcd2: 10.92.36.131
etcd3: 10.92.36.28
node1: 10.92.36.93
node2: 10.92.36.99
node3: 10.92.36.12
ha-proxy: 10.92.36.120
```

## 2.1 Установка и настройка etcd

### 2.1.1 Шаг №1
Скачиваю пакет etcd с оффициального гита, распаковываю архив и перемещаю файлы:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_4/2.jpg)

### 2.1.2 Шаг №2
Требуется создать соответствующих пользователей и директории.

Добавляем системную группу `etcd`, системного пользователя без права входа, добавив его в группу `etcd`:
```
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
```

Добавляем директрии и права на них соответсвующее системному пользователю `etcd`:
```
sudo mkdir -p /var/lib/etcd /etc/default/etcd/.tls
sudo chown -R etcd:etcd /var/lib/etcd /etc/default/etcd
```

Не забываем вписать информацию о хостах в /etc/hosts:
```
# etcd cluster nodes
10.92.36.48 etcd1
10.92.36.31 etcd2
10.92.36.28 etcd3

# Patroni nodes
10.92.36.82 node1
10.92.36.60 node2
10.92.36.103 node3
```

### 2.1.3 Шаг №3 Генерация сертификатов
Для тестовых стендов можно взять самоподписнные TLS сертификаты, для автоматизации данной рутинной задачи можно вопользоваться следующим .sh сценарием:
<details>
<summary>create_self_signed_tls_cert.sh</summary>

  ```bash
  #!/bin/bash
  
  # Директория для сертификатов:
  CERT_DIR="/etc/default/etcd/.tls"
  mkdir -p ${CERT_DIR}
  cd ${CERT_DIR}
  
  # Создание CA-сертификата:
  openssl genrsa -out ca.key 4096
  openssl req -x509 -new -key ca.key -days 10000 -out ca.crt -subj "/C=RU/ST=Moscow Region/L=Moscow/O=MyOrg/OU=MyUnit/CN=myorg.com"
  
  # Функция для генерации сертификатов для нод:
  generate_cert() {
      NODE_NAME=$1
      NODE_IP=$2
  
      cat <<EOF > ${CERT_DIR}/${NODE_NAME}.san.conf
  [ req ]
  default_bits       = 4096
  distinguished_name = req_distinguished_name
  req_extensions     = req_ext
  [ req_distinguished_name ]
  countryName                 = RU
  stateOrProvinceName         = Moscow Region
  localityName                = Moscow
  organizationName            = MyOrg
  commonName                  = ${NODE_NAME}
  [ req_ext ]
  subjectAltName = @alt_names
  [ alt_names ]
  DNS.1   = ${NODE_NAME}
  IP.1    = ${NODE_IP}
  IP.2    = 127.0.0.1
  EOF
  
  openssl genrsa -out ${NODE_NAME}.key 4096
  openssl req -config ${NODE_NAME}.san.conf -new -key ${NODE_NAME}.key -out ${NODE_NAME}.csr -subj "/C=RU/ST=Moscow Region/L=Moscow/O=MyOrg/CN=${NODE_NAME}"
      openssl x509 -extfile ${NODE_NAME}.san.conf -extensions req_ext -req -in ${NODE_NAME}.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ${NODE_NAME}.crt -days 10000
  
      rm -f ${NODE_NAME}.san.conf ${NODE_NAME}.csr
  }
  
  # Список нод etcd и их IP-адресов:
  ETCD_NODES=("etcd1" "etcd2" "etcd3")
  ETCD_IPS=("10.92.36.48" "10.92.36.31" "10.92.36.28")
  
  # Список нод Patroni и их IP-адресов:
  PATRONI_NODES=("node1" "node2" "node3")
  PATRONI_IPS=("10.92.36.82" "10.92.36.60" "10.92.36.103")
  
  
  # Генерация сертификатов для нод etcd:
  for i in "${!ETCD_NODES[@]}"; do
  generate_cert "${ETCD_NODES[$i]}" "${ETCD_IPS[$i]}"
  done
  
  # Генерация сертификатов для нод Patroni:
  for i in "${!PATRONI_NODES[@]}"; do
  generate_cert "${PATRONI_NODES[$i]}" "${PATRONI_IPS[$i]}"
  done
  
  
  chown -R etcd:etcd ${CERT_DIR}
  chmod 600 ${CERT_DIR}/*.key
  chmod 644 ${CERT_DIR}/*.crt
  
  echo "Сертификаты успешно сгенерированы и сохранены в ${CERT_DIR}"
</details>
```
</details>


Запуск скрипта:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_4/3.jpg)

Перенос сертификатов на все остальные хосты:
```
scp ca.crt etcd1.crt etcd2.crt etcd3.crt node1.crt node2.crt node3.crt ca.key etcd2.key fvtarasov@10.92.36.31:/home/fvtarasov/.tls
scp ca.crt etcd1.crt etcd2.crt etcd3.crt node1.crt node2.crt node3.crt ca.key etcd3.key fvtarasov@10.92.36.28:/home/fvtarasov/.tls
scp ca.crt etcd1.crt etcd2.crt etcd3.crt node1.crt node2.crt node3.crt ca.key node1.key fvtarasov@10.92.36.82:/home/fvtarasov/.tls
scp ca.crt etcd1.crt etcd2.crt etcd3.crt node1.crt node2.crt node3.crt ca.key node2.key fvtarasov@10.92.36.60:/home/fvtarasov/.tls
scp ca.crt etcd1.crt etcd2.crt etcd3.crt node1.crt node2.crt node3.crt ca.key node3.key fvtarasov@10.92.36.103:/home/fvtarasov/.tls
```


Смена прав на всех узлах:
`
sudo chown -R etcd:etcd /etc/default/etcd/.tls && sudo chmod -R 744 /etc/default/etcd/.tls && sudo chmod 600 /etc/default/etcd/.tls/*.key
`

### 2.1.4 Шаг 4. Конфигурация etcd

Создаем конфигурационный файл /etc/etcd/etcd.conf.yml на каждой ноде etcd:
<details>
<summary>create_self_signed_tls_cert.sh</summary>

  ```bash
# /etc/etcd/etcd.conf.yml
name: etcd3 # Требуется изменить на других нодах-хостах
data-dir: /var/lib/etcd/default
listen-peer-urls: https://0.0.0.0:2380
listen-client-urls: https://0.0.0.0:2379
advertise-client-urls: https://etcd3:2379 # Требуется изменить на других нодах-хостах
initial-advertise-peer-urls: https://etcd3:2380 # Требуется изменить на других нодах-хостах
initial-cluster-token: etcd_scope
initial-cluster: etcd1=https://etcd1:2380,etcd2=https://etcd2:2380,etcd3=https://etcd3:2380
initial-cluster-state: new
election-timeout: 5000
heartbeat-interval: 500
 
client-transport-security:
  cert-file: /etc/default/etcd/.tls/etcd3.crt # Требуется изменить на других нодах-хостах
  key-file: /etc/default/etcd/.tls/etcd3.key
  client-cert-auth: true
  trusted-ca-file: /etc/default/etcd/.tls/ca.crt
 
peer-transport-security:
  cert-file: /etc/default/etcd/.tls/etcd3.crt # Требуется изменить на других нодах-хостах
  key-file: /etc/default/etcd/.tls/etcd3.key
  client-cert-auth: true
  trusted-ca-file: /etc/default/etcd/.tls/ca.crt
</details>
```
</details>

Этот файл описывает, как именно должен запуститься и работать экземпляр etcd на узлах. Важно, чтобы настройки были согласованы между всеми тремя узлами кластера (etcd1, etcd2, etcd3), иначе кластер не соберётся.

### 2.1.5 Шаг 5. Создаем Service-Unit для запуска etcd

Создаем на каждой ноде etcd сервис-юнит по пути: `/etc/systemd/system/etcd.service`

<details>
<summary>etcd.service</summary>

  ```bash
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io/docs/
Wants=network-online.target
After=network-online.target
 
[Service]
User=etcd
Type=notify
ExecStart=/usr/bin/etcd --config-file=/etc/etcd/etcd.conf.yml
Restart=always
RestartSec=5
LimitNOFILE=40000
 
[Install]
WantedBy=multi-user.target

```
</details>

Не забываем выполнить на каждой ноде:
```
sudo systemctl daemon-reload
sudo systemctl enable etcd
```
`sudo systemctl daemon-reload` — обновляет кеш юнитов systemd, чтобы он узнал о новом файле `/etc/systemd/system/etcd.service`.
`sudo systemctl enable etcd` — включает автозапуск службы etcd при загрузке системы.

После чего запускаем службу: `sudo systemctl start etcd`

### 2.1.6 Шаг 6. Проверка работы кластера etcd

Проверяем статус кластера etcd:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_4/4.jpg)

Для более удобного вывода информации можно создать алиас. Добавляем alias в `~/.bashrc`:
```
echo 'alias ectl-status="sudo etcdctl --cacert=/etc/default/etcd/.tls/ca.crt --cert=/etc/default/etcd/.tls/etcd1.crt --key=/etc/default/etcd/.tls/etcd1.key --endpoints=https://etcd1:2379,https://etcd2:2379,https://etcd3:2379 endpoint status --cluster -w table"' >> ~/.bashrc
```

Далее указываем параметр со значением `ETCD_INITIAL_CLUSTER_STATE=existing` для `/etc/systemd/system/etcd.service`. Не забываем перечитать файл серви-юнита и отправляем в перезапуск.
```
sudo systemctl daemon-reload
sudo systemctl restart etcd
```
`ETCD_INITIAL_CLUSTER_STATE="existing"` — позволяет узлу подключиться к уже работающему кластеру, а не пытаться создать новый. Обязательно использовать после первого запуска, чтобы кластер продолжал работать при перезапуске узлов.

Утираем потный пот и идем далее! =)

## 2.2 Установка PostgreSQL 17

Воспользуемся:
```
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install -y postgresql-17
```
`postgresql-common` — обязательный пакет для работы с PostgreSQL в системах на базе Debian/Ubuntu.
`apt.postgresql.org.sh` — скрипт, который добавляет официальный репозиторий PostgreSQL, чтобы установить актуальную версию СУБД.

В итоге имеем три PostgreSQL:
```
fvtarasov@tarasov-postgre-advance-1:~/.tls$ sudo su - postgres
postgres@tarasov-postgre-advance-1:~$ psql --version
psql (PostgreSQL) 17.8 (Ubuntu 17.8-1.pgdg24.04+1)
---
fvtarasov@tarasov-postgre-advance-2:~/.tls$ sudo su - postgres
postgres@tarasov-postgre-advance-2:~$ psql --version
psql (PostgreSQL) 17.8 (Ubuntu 17.8-1.pgdg24.04+1)
---
fvtarasov@tarasov-postgre-advance-3:~/.tls$ sudo su - postgres
postgres@tarasov-postgre-advance-3:~$ psql --version
psql (PostgreSQL) 17.8 (Ubuntu 17.8-1.pgdg24.04+1)
```

Добавляем записи в pg_hba.conf:
```
host all all 0.0.0.0/0 scram-sha-256
host replication replicator 0.0.0.0/0 scram-sha-256
```

Создаем каталог для TLS Patroni, и перемещаем в них необходимые сертификаты, которые ранее при помощи скрипта собирали:
```
mkdir -p /opt/patroni/.tls

sudo cp /etc/default/etcd/.tls/ca.crt /etc/default/etcd/.tls/ca.key /etc/default/etcd/.tls/etcd1.crt /etc/default/etcd/.tls/etcd2.crt /etc/default/etcd/.tls/etcd3.crt /etc/default/etcd/.tls/node1.crt /etc/default/etcd/.tls/node2.crt /etc/default/etcd/.tls/node3.crt /etc/default/etcd/.tls/node3.key /opt/patroni/.tls

sudo cp /etc/default/etcd/.tls/ca.crt /etc/default/etcd/.tls/ca.key /etc/default/etcd/.tls/etcd1.crt /etc/default/etcd/.tls/etcd2.crt /etc/default/etcd/.tls/etcd3.crt /etc/default/etcd/.tls/node1.crt /etc/default/etcd/.tls/node2.crt /etc/default/etcd/.tls/node2.key /etc/default/etcd/.tls/node3.crt /opt/patroni/.tls
...

chmod -R 744 /opt/patroni/.tls
chmod 600 /opt/patroni/.tls/*.key
```

## 2.3 Установка и настройка Patroni

Patroni будет крутиться в изолированной среде Python для того, чтобы не нарушать систему, контролировать зависимости. Установка происходит из локальных репозиториев, все организовано в директории `/opt/patroni`.

### 2.3.1 Шаг 1. Установка Python

Устанваливаем Python на машинах:
```
sudo apt update
sudo apt install build-essential libssl-dev libffi-dev bzip2

wget https://www.python.org/ftp/python/3.12.4/Python-3.12.4.tar.xz
tar -xf Python-3.12.4.tar.xz
cd Python-3.12.4/
./configure --enable-optimizations
make altinstall
sudo ln -sf /usr/local/bin/python3.12 /usr/bin/python3
sudo ln -sf /usr/local/bin/pip3.12 /usr/bin/pip3
```
Таким образом при установке Patroni можно вычеркнуть риск повреждения системы. Плюс получим удобный каталог с бин-файлами. 

### 2.3.2 Шаг 2. Создаем виртуальную среду 
Выполняем команды:
```
sudo su - postgres
mkdir -p /opt/patroni
python3.12 -m venv /opt/patroni --without-pip
source /opt/patroni/bin/activate
curl https://bootstrap.pypa.io/get-pip.py -o /opt/patroni/bin/get-pip.py
python /opt/patroni/bin/get-pip.py
mkdir –p /opt/patroni/packages
mkdir /data/log/patroni
chown -R postgres:postgres /data/log/patroni
sudo mkdir -p /home/postgres/
sudo chown -R postgres:postgres /home/postgres/
```

### 2.3.3 Шаг 3. Установка пакетов Patroni

Примерный список пакетов для скачивания, которые нужно положить в `/opt/patroni/packages`:
```
click-8.1.7-py3-none-any.whl
dnspython-2.6.1-py3-none-any.whl
patroni-3.3.2-py3-none-any.whl
prettytable-3.10.2-py3-none-any.whl
psutil-6.0.0-cp36-abi3-manylinux_2_12_x86_64.manylinux2010_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl
python_dateutil-2.9.0.post0-py2.py3-none-any.whl
python-etcd-0.4.5.tar.gz
PyYAML-6.0.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
setuptools-72.1.0-py3-none-any.whl
six-1.16.0-py2.py3-none-any.whl
urllib3-2.2.2-py3-none-any.whl
wcwidth-0.2.13-py2.py3-none-any.whl
ydiff-1.3.tar.gz
psycopg2_binary-2.9.9-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
```

### 2.3.4 Шаг 4. Установка Patroni при помощи скачанных пакетов

Далее устанавливаем Patroni без доступа к интернету, используя заранее скачанные пакеты, и устанавливаем драйвер для подключения:
```
pip3 install --no-index --find-links=/opt/patroni/packages patroni[etcd3]
pip3 install --no-index --find-links=/opt/patroni/packages psycopg2-binary
chown -R postgres:postgres /opt/patroni
```

Теперь нужно добавить в профайл PgSQL параметры:
```
export PG_CONFIG=/usr/lib/postgresql/17/bin/pg_config
export PATRONI_CONFIG_FILE=/etc/patroni/config.yml
```

Активируем виртуальную среду: `source /opt/patroni/bin/activate`
Создаем конфиг-файл Patroni для каждой ноды:`vi /etc/patroni/config.yml`

<details>
<summary>Содержимое Service-Unit для Patroni</summary>

```bash
patroni
scope: patroni_cluster
namespace: /patroni
name: patroni_node3 # Изменить на 2 ноде
log:
  level: INFO
  dir: /data/log/patroni
  file_size: 50000000
  file_num: 10
restapi:
  listen: 0.0.0.0:8008
  connect_address: node3:8008 # Изменить на ноде
  verify_client: optional
  cafile: /opt/patroni/.tls/ca.crt
  certfile: /opt/patroni/.tls/node3.crt # Не забыть изменить сертификаты на ноде
  keyfile: /opt/patroni/.tls/node3.key
ctl:
  cacert: /opt/patroni/.tls/ca.crt # Не забыть изменить сертификаты на 2 ноде
  certfile: /opt/patroni/.tls/node3.crt
  keyfile: /opt/patroni/.tls/node3.key
etcd3:
  hosts: ["etcd1:2379", "etcd2:2379", "etcd3:2379"]
  protocol: https
  cacert: /opt/patroni/.tls/ca.crt
  cert: /opt/patroni/.tls/node3.crt # Не забыть изменить сертификаты на 2 ноде
  key: /opt/patroni/.tls/node3.key
watchdog:
  mode: off # Если настроен, можно включить
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
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        shared_buffers: '512MB'
        wal_level: 'replica'
        wal_keep_size: '512MB'
        max_connections: 100
        effective_cache_size: '1GB'
        maintenance_work_mem: '256MB'
        max_wal_senders: 5
        max_replication_slots: 5
        checkpoint_completion_target: 0.7
        log_connections: 'on'
        log_disconnections: 'on'
        log_statement: 'ddl'
        log_line_prefix: '%m [%p] %q%u@%d '
        logging_collector: 'on'
        log_destination: 'stderr'
        log_directory: '/data/log'
        log_filename: 'postgresql-%Y-%m-%d.log'
        log_rotation_size: '100MB'
        log_rotation_age: '1d'
        log_min_duration_statement: -1
        log_min_error_statement: 'error'
        log_min_messages: 'warning'
        log_error_verbosity: 'verbose'
        log_hostname: 'off'
        log_duration: 'off'
        log_timezone: 'Europe/Moscow'
        timezone: 'Europe/Moscow'
        lc_messages: 'C.UTF-8'
        password_encryption: 'scram-sha-256'
        debug_print_parse: 'off'
        debug_print_rewritten: 'off'
        debug_print_plan: 'off'
        superuser_reserved_connections: 3
        synchronous_commit: 'on'
        synchronous_standby_names: '*'
        hot_standby: 'on'
        compute_query_id: 'on'
      pg_hba:
        - local all all peer
        - host all all 127.0.0.1/32 scram-sha-256
        - host all all 0.0.0.0/0 md5
        - host replication replicator 127.0.0.1/32 scram-sha-256
        - host replication replicator 10.92.36.103/24 scram-sha-256   # не забыть заменить на ноде
  pg_hba:
    - local all all peer
    - host all all 127.0.0.1/32 scram-sha-256
    - host all all 0.0.0.0/0 md5
    - host replication replicator 127.0.0.1/32 scram-sha-256
    - host replication replicator 10.92.36.103/24 scram-sha-256   # не забыть заменить на ноде
  initdb: ["encoding=UTF8", "data-checksums", "username=postgres", "auth=scram-sha-256"]
  users:
    admin:
      password: 'new_secure_password1'
      options: ["createdb"]
postgresql:
  listen: 0.0.0.0
  connect_address: 10.92.36.103:5432 # не забываем заменить адрес на ноде.
  use_unix_socket: true
  data_dir: /data/17
  config_dir: /data/17
  bin_dir: /usr/bin
  pgpass: /home/postgres/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: 'new_repl_password'
    superuser:
      username: postgres
      password: 'new_superuser_password'
    rewind:
      username: postgres
      password: 'new_superuser_password'
  parameters:
    unix_socket_directories: "/var/run/postgresql"
  create_replica_methods: ["basebackup"]
  basebackup:
    max-rate: 100M
    checkpoint: fast
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
  
  create user replicator replication login encrypted password 'new_repl_password';
```
</details>

Для пущего спокойствия проверяем конфиг-файл:
```
patroni --validate-config /etc/patroni/config.yml
```
Если ничего не выводит, то все ок и идем дальше.

Выделим Service-Unit для Patroni на каждом хосте: `sudo vi /etc/systemd/system/patroni.service`
<details>
<summary>Содержимое Service-Unit для Patroni</summary>

```bash
service patroni
[Unit]
Description=Patroni high-availability PostgreSQL
After=network.target
 
[Service]
User=postgres
Type=simple
ExecStart=/opt/patroni/bin/patroni /etc/patroni/config.yml
Restart=always
RestartSec=5
LimitNOFILE=1024
 
[Install]
WantedBy=multi-user.target
```
</details>


Далее нужно перезапустить Systemd и запустить Patroni:
```
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
```

Как видно на картинке Patroni запустился:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_4/6.jpg)

В заверешнии установки и настройки Patroni, нужно его ноды синхронизировать, для этого выполним:
1) Останавливаем Patroni: `sudo systemctl stop patroni`
2) Теперь благополучно удаляем содержимое директории PGDATA: `sudo rm -rf /data/17/*`
3) Запускаем Patroni: `sudo systemctl start patroni`

И наслаждаемся синхронностью нод:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_4/7.jpg)

Что видно? То что node2 и node3 являются репликами node1, соответсвенно на нее подписаны, и что видно на скриншоте выше. 

И в заверешнии установки Patroni создадим `alias`, для более удобного просмотрета состояния кластера:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_4/8.jpg)

### 2.3.5 Автоматический failover

Для демонстрации автоназначения лидера, т.е. failover будем использовать: `systemctl stop patroni`
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_4/9.jpg)
На картинке видно, что имеются 3 ноды: patroni_node1(Leader), patroni_node2(Replica), patroni_node3(Sync Standby).
Лидер принимает подключение для добавления/изменения данных в БД. Стендюай приоритетный кандидат на позицию лидера, в момент занимаемой роли, стендбай принимает подключения только на чтение. И реплика просто принимает подключения на чтение из БД.
После остановки лидера, стендбай принимает роль лидера на себя. А реплика теперь является стендбаем. Запуск patroni_node1 инициализирует его как реплику. 
Итого: patroni_node1(Replica), patroni_node2(Sync Standby), patroni_node3(Leader).

# 3. Настройте HAProxy для балансировки нагрузки.

Устанвилваем HAProxy на отдельную машину:
```
sudo apt-get install haproxy
```

После установке HAProxy, требуется изменить его основной конфигурационный файл:
<details>
<summary>Содержимое Service-Unit для Patroni</summary>

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
   bind *:5432
   option httpchk GET/master
   http-check expect status 200
   default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
   server node1 node1:5432 maxconn 100 check port 8008
   server node2 node2:5432 maxconn 100 check port 8008
   server node3 node3:5432 maxconn 100 check port 8008

listen standby
   bind *:5433
   option httpchk GET/replica
   http-check expect status 200
   default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
   server node1 node1:5432 maxconn 100 check port 8008
   server node2 node2:5432 maxconn 100 check port 8008
   server node3 node3:5432 maxconn 100 check port 8008

```
</details>

listen production - собирает статус для лидера Patroni
listen standby - собирает статус реплик Patroni
