# Вариант 1. Кластер своими руками
Выбрал данный вариант, т.к. интерактив очень интересный и веселый =)

Статья будет разделена на несколько разделов и подразделов, соотвествующим своим разделам. 
Попробуем решить задание со "звездочкой", т.е. посторим "мастер" кластер и "стэндбай". Развернуты кластеры будут в разных зонах доступности одного ЯОблака. Попытаемся настроить `failover` между двумя кластерами, репликацию, отказоустйчивый DNS(балансировщики нам в помощь). И в заключение проверим отказоустойчивость одного из кластеров, а также, как поведет построенная система, если одина из зон выйдет из строя.

Наша система будет устроена по "спартански" в тестовых целях. Каждый кластер будет включать в себя по три машины, также будет выделена отдельная машина в отдельной зоне доступности. 
Каждый кластер включает в себя etcd и Patroni.

Далее будут приведены инструкции по установке и нстройке типового тестовог кластера Etcd+Patroni. Данные действия выполняем на каждом кластере, в т.ч. установка Etcd & Patroni

# 1. Установка Etcd

Обновляем пакеты: `sudo apt update`

Прорисываем на каждом хосте в /etc/hosts:
```
sudo tee -a /etc/hosts <<EOF
10.92.35.52 tarasov-test-otus-cluter-1-node-1.ru-central1.internal
10.92.35.112 tarasov-test-otus-cluter-1-node-2.ru-central1.internal
10.92.35.162 tarasov-test-otus-cluter-1-node-3.ru-central1.internal
EOF
```

Устанавливаем ETCD на каждой ноде `sudo apt -y install etcd-server && sudo apt -y install etcd-client`:
<img width="861" height="235" alt="image" src="https://github.com/user-attachments/assets/c2f1c096-fe7c-4853-9fee-a3045e1dedd0" />

Редактируем конфиг-файл на каждой ноде etcd:
```
vi /etc/default/etcd
```

<details>
<summary>/etc/default/etcd</summary>
  
  ```bash
#ETCD_1:
ETCD_NAME="etcd1"
ETCD_LISTEN_CLIENT_URLS="http://10.92.35.52:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://tarasov-test-otus-cluter-1-node-1.ru-central1.internal:2379"
ETCD_LISTEN_PEER_URLS="http://10.92.35.52:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://tarasov-test-otus-cluter-1-node-1.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://tarasov-test-otus-cluter-1-node-1.ru-central1.internal:2380,etcd2=http://tarasov-test-otus-cluter-1-node-2.ru-central1.internal:2380,etcd3=http://tarasov-test-otus-cluter-1-node-3.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"

#ETCD_2:
ETCD_NAME="etcd2"
ETCD_LISTEN_CLIENT_URLS="http://10.92.35.112:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://tarasov-test-otus-cluter-1-node-2.ru-central1.internal:2379"
ETCD_LISTEN_PEER_URLS="http://10.92.35.112:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://tarasov-test-otus-cluter-1-node-2.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://tarasov-test-otus-cluter-1-node-1.ru-central1.internal:2380,etcd2=http://tarasov-test-otus-cluter-1-node-2.ru-central1.internal:2380,etcd3=http://tarasov-test-otus-cluter-1-node-3.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"

#ETCD_3:
ETCD_NAME="etcd3"
ETCD_LISTEN_CLIENT_URLS="http://10.92.35.162:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://tarasov-test-otus-cluter-1-node-3.ru-central1.internal:2379"
ETCD_LISTEN_PEER_URLS="http://10.92.35.162:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://tarasov-test-otus-cluter-1-node-3.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://tarasov-test-otus-cluter-1-node-1.ru-central1.internal:2380,etcd2=http://tarasov-test-otus-cluter-1-node-2.ru-central1.internal:2380,etcd3=http://tarasov-test-otus-cluter-1-node-3.ru-central1.internal:2380"
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
<img width="1354" height="195" alt="image" src="https://github.com/user-attachments/assets/7b351425-7422-48bd-b7dc-3505a9c98ff7" />

# 2. Установка PostgreSQL

Устанавливаем с помощью команды:
```
sudo apt -y install postgresql
```

После чего, требуется провести настройки и некоторые приготовления для установки Patroni:
- Создать пользователя replicator;
- Добавить записи на разрешения коннекта к БД в pg_hba.conf;
- Отредактировать звапись listen_address, для прослушивания всех адресов;
- Рестарт PostgreSQL;
- Удаление содержимого PGDATA на 2-й и 3-ей ноде.

Выпонление данных пунктов показано на картинке ниже:
<img width="817" height="735" alt="image" src="https://github.com/user-attachments/assets/70de4fc9-f108-4aed-bd21-5587aa2f65f5" />


# 3. Установка Patroni
Patroni будет крутиться в изолированной среде Python для того, чтобы не нарушать систему, контролировать зависимости. Создавать виртуальное окружение Patroni будем при помощи python3.12-venv. Для того, чтобы воспользоваться виртуальным окружением, требуется каталог для него, владелец каталога должен быть postgres. Также незабываем установить Patroni и драйвер для работы Patroni c PostgreSQL. Все это продемонстрировано на рисунках ниже:
<img width="863" height="704" alt="image" src="https://github.com/user-attachments/assets/3e5893ca-5116-46dc-9f90-16631195fe6f" />
<img width="702" height="163" alt="image" src="https://github.com/user-attachments/assets/ea8ebb32-ab81-4f1b-b272-33aabee6a142" />
<img width="966" height="801" alt="image" src="https://github.com/user-attachments/assets/6362ae7c-cc5d-4171-b4ef-d03717f271c0" />
<img width="836" height="148" alt="image" src="https://github.com/user-attachments/assets/a7586ad2-3937-4f3d-9d8a-d4da5af21a2d" />

Не забываем создать PGDATA, директорию для логов Patroni и конфига:`sudo mkdir -p /data/16` || `sudo mkdir -p /data/log/patroni` || `sudo mkdir -p /etc/patroni/`

И назначить овнера postgres:`sudo chown -R postgres:postgres /data` || `sudo chown -R postgres:postgres /data/log/patroni` || `sudo chown -R postgres:postgres /etc/patroni/`

Назначаем полные права владльцу PGDATA: `sudo chmod 700 /data/16`

<img width="1325" height="314" alt="image" src="https://github.com/user-attachments/assets/bd07b821-419c-46d8-a8e2-0a48f7cad5d1" />

Далее настроем конфигурацию Patroni в конфиг-файле `sudo vi /etc/patroni/conf.yml`:
<details>
<summary>/etc/patroni/conf.yml</summary>
  
```bash
sudo tee /etc/patroni/conf.yml << EOF
scope: patroni_cluster
namespace: /patroni
name: patroni_node3
log:
  level: INFO
  dir: /data/log/patroni
  file_size: 50000000
  file_num: 10
restapi:
  listen: 10.92.35.162:8008
  connect_address: 10.92.35.162:8008

etcd:
  hosts: tarasov-test-otus-cluter-1-node-1.ru-central1.internal:2379,tarasov-test-otus-cluter-1-node-2.ru-central1.internal:2379,tarasov-test-otus-cluter-1-node-3.ru-central1.internal:2379

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
    - host replication replicator 10.92.35.52/32 md5
    - host replication replicator 10.92.35.112/32 md5
    - host replication replicator 10.92.35.162/32 md5
    - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: 'password'
      options:
        - CREATEDB
        - CREATEROLE

postgresql:
  listen: 127.0.0.1,10.92.35.162:5432
  connect_address: 10.92.35.162:5432
  data_dir: /data/16
  bin_dir: /usr/lib/postgresql/16/bin
  authentication:
    replication:
      username: replicator
      password: 'password'
    superuser:
      username: postgres
      password: 'password'
    rewind:
      username: rewind_user
      password: 'password'
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
EOF
```
</details>

Создаем Service-Unit `sudo vi /etc/systemd/system/patroni.service`:
<details>
<summary>/etc/systemd/system/patroni.service</summary>
  
```bash
sudo tee /etc/systemd/system/patroni.service << EOF
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
EOF
```
</details>

Следующим шагом будет застваить Systemd перечитать daemon-reload и запустить Patroni:
```
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
```

Проверим, что все запустилось:

<img width="758" height="206" alt="image" src="https://github.com/user-attachments/assets/5f5e353e-1fd0-4b9f-a2fd-ccca3f8d5f21" />

