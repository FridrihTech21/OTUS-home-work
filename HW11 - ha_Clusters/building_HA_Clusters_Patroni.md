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

# 4. Попробуем установить кластер Etcd, Patroni при помощи Ansible

Проводить установку будем с отдельной машины, например, с будущего балансера. Для этого установим Ansible:
```
sudo apt update && sudo apt install software-properties-common && sudo add-apt-repository --yes --update ppa:ansible/ansible && sudo apt install ansible
```

<img width="835" height="203" alt="image" src="https://github.com/user-attachments/assets/f41fca82-36a3-47cb-bcca-ceea8ffccae8" />

После чего, настроим сетевую связность к машинам второго кластера:
```
sudo tee -a /etc/hosts <<EOF
10.92.36.113 tarasov-test-otus-cluter-2-node-1.ru-central1.internal
10.92.36.50 tarasov-test-otus-cluter-2-node-2.ru-central1.internal
10.92.36.24 tarasov-test-otus-cluter-2-node-3.ru-central1.internal
10.92.5.112 tarasov-test-otus-balancer.ru-central1.internal
EOF
```

Создаем ключи и добавляем их на каждую машину:

```
ssh-keygen

sudo tee -a ~/.ssh/authorized_keys << EOF
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHDVA7NAl2EqlYv1sdyWOvBQDuLHRxUeAzqBPYlZQHSt tarasov-test-otus-balancer
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMW+LvQxHWfqMobyOsE5ygoVmosY4IhSMn8CwxbCiDAw tarasov-test-otus-cluter-2-node-1
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPzGyks1x6cagWgeZxbOjQlMDsmTQbVfghVjhwNmkrQh tarasov-test-otus-cluter-2-node-2
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIBCruOHXF1exFFJcSNelfi/sHSap5NV4MpsTluZuamT tarasov-test-otus-cluter-2-node-3
EOF
```

Не забываем добавить пользователя, подк отороым будет работать Ansible в `sudo`: `sudo usermod -aG sudo fvtarasov`

Подготовили `inventory`-файл и протестировали соединение:
<img width="1358" height="486" alt="image" src="https://github.com/user-attachments/assets/9cee7b1c-ba70-4e23-ab13-6e14bb1abcb5" />

<details>
<summary>inventory.ini</summary>
  
```yml
[servers]
tarasov-test-otus-cluter-2-node-1.ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519
tarasov-test-otus-cluter-2-node-2.ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519
tarasov-test-otus-cluter-2-node-3.ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519
```
</details>

Теперь добавим `playbook`, в котором обновим пакеты на целевых машинах:

<details>
<summary>setup.yml</summary>
  
```yml
tee -a setup.yml << EOF
---
- name:
  hosts: servers
  become: yes
  tasks:
    - name: Обновление списка пакетов
      ansible.builtin.apt:
        update_cache: yes
EOF
```
</details>

<img width="1358" height="461" alt="image" src="https://github.com/user-attachments/assets/92edee15-a073-47ff-a96f-464a8680a96e" />


## 4.1 Etcd

Приступим к установке Etcd:
<img width="1357" height="917" alt="image" src="https://github.com/user-attachments/assets/93cb9a31-d828-4c64-a00b-f6c1d93e0e09" />

Дерево объектов Ansible установки Etcd:
<img width="497" height="146" alt="image" src="https://github.com/user-attachments/assets/e9495e22-627d-4f79-83df-6200db748bf3" />

`inventory.ini` - файл, содержащий информацию об хостах
`setup.yml` - плейбук Ansible
`etcd.j2` - файл-шаблон для подстановки значений в конфиг-файл Etcd

<details>
<summary>setup.yml</summary>
  
```yml
tee -a setup.yml << EOF
---
- name: Configure etcd cluster
  hosts: servers 
  become: yes
  gather_facts: yes 
  vars:
    etcd_cluster_members: "{{ groups['servers'] | map('extract', hostvars, ['ansible_fqdn']) | list }}"
    etcd_initial_cluster_string: >-
      {{
        groups['servers']
        | map('extract', hostvars, ['ansible_fqdn'])
        | zip(['etcd1', 'etcd2', 'etcd3'])
        | map('reverse')
        | map('join', '=')
        | map('regex_replace', '^(.*)=(.*)$', '\\1=http://\\2:2380')
        | join(',')
      }}
    etcd_node_index: "{{ groups['servers'].index(inventory_hostname) }}"
    etcd_names_map: ['etcd1', 'etcd2', 'etcd3']
    etcd_node_name: "{{ etcd_names_map[etcd_node_index] }}"
  tasks:
    - name: Install etcd-server
      ansible.builtin.apt:
        name: etcd-server
        update_cache: yes
        state: present 
    - name: Install etcd-client
      ansible.builtin.apt:
        name: etcd-client
        update_cache: yes
        state: present 
    - name: Ensure /etc/default directory exists
      ansible.builtin.file:
        path: /etc/default
        state: directory
        mode: '0755'
    - name: Copy Etcd configuration template to /etc/default/etcd
      ansible.builtin.template:
        src: etcd.j2 
        dest: /etc/default/etcd
        owner: fvtarasov
        group: fvtarasov
        mode: '0644' 
    - name: Restart Etcd service
      ansible.builtin.systemd_service:
        name: etcd.service
        state: restarted
EOF
```
</details>

### 4.1.1 setup.yml

```yml
---
- name: Configure etcd cluster
  hosts: servers 
  become: yes
  gather_facts: yes 
  vars:
    etcd_cluster_members: "{{ groups['servers'] | map('extract', hostvars, ['ansible_fqdn']) | list }}"
    etcd_initial_cluster_string: >-
      {{
        groups['servers']
        | map('extract', hostvars, ['ansible_fqdn'])
        | zip(['etcd1', 'etcd2', 'etcd3'])
        | map('reverse')
        | map('join', '=')
        | map('regex_replace', '^(.*)=(.*)$', '\\1=http://\\2:2380')
        | join(',')
      }}
    etcd_node_index: "{{ groups['servers'].index(inventory_hostname) }}"
    etcd_names_map: ['etcd1', 'etcd2', 'etcd3']
    etcd_node_name: "{{ etcd_names_map[etcd_node_index] }}"
```
Плейбук, в процессе собирает информацию о хостах: IP, FQDN и т.д. Создает список всех хостов из инвентори, чтобы далее вставлять нужные значения в нужные переменные конфиг-файла Etcd.

После чего, выполняет основные пункты для установки ETCD:
```yml
 tasks:
    - name: Install etcd-server
      ansible.builtin.apt:
        name: etcd-server
        update_cache: yes
        state: present 
    - name: Install etcd-client
      ansible.builtin.apt:
        name: etcd-client
        update_cache: yes
        state: present 
    - name: Ensure /etc/default directory exists
      ansible.builtin.file:
        path: /etc/default
        state: directory
        mode: '0755'
    - name: Copy Etcd configuration template to /etc/default/etcd
      ansible.builtin.template:
        src: etcd.j2 
        dest: /etc/default/etcd
        owner: fvtarasov
        group: fvtarasov
        mode: '0644' 
    - name: Restart Etcd service
      ansible.builtin.systemd_service:
        name: etcd.service
        state: restarted
```

`- name: Copy Etcd configuration template to /etc/default/etcd` - данный таск вставляет шаблон для конфиг-файла:

```yml
ETCD_NAME="{{ etcd_node_name }}"
ETCD_LISTEN_CLIENT_URLS="http://{{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }}:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://{{ ansible_fqdn }}:2379"
ETCD_LISTEN_PEER_URLS="http://{{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }}:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://{{ ansible_fqdn }}:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="{{ etcd_initial_cluster_string }}"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```

`etcd_node_name` - одномерный массив, из которого по индексам "вытягивается" название для каждой отдельной ноды Etcd

`hostvars[inventory_hostname]['ansible_default_ipv4']['address']` - вытягиваем ip адрес из `hostvars`

`ansible_fqdn` - узнаем FQDN машины

`etcd_initial_cluster_string` - конкатенируем все FQDN и их порты для внесения в переменную: `ETCD_INITIAL_CLUSTER`

Проверим, что кластер Etcd из трех нод установлен:
<img width="1339" height="149" alt="image" src="https://github.com/user-attachments/assets/fab18445-15fd-4c42-bea3-44305da38dbe" />

## 4.2 PostgreSQL

Далее соберем `ansible-playbook` для установки PostgreSQL и немного подготвки к установке Patroni:
- `sudo apt -y install postgresql`;
- Проверка на пользователя replicator;
- Создать пользователя replicator;
- Добавить записи на разрешения коннекта к БД в pg_hba.conf;
- Отредактировать звапись listen_address, для прослушивания всех адресов;
- Рестарт PostgreSQL;
- Удаление содержимого PGDATA на нодах.

<details>
<summary>setup.yml</summary>
  
```yml

tee setup.yml << EOF
---
- name: Configure etcd cluster
  hosts: servers
  become: yes
  gather_facts: yes

  tasks:
    - name: Install python3-pip
      ansible.builtin.apt:
        name: python3-pip
        update_cache: yes
        state: present
    - name: Install python3-psycopg2
      ansible.builtin.apt:
        name: python3-psycopg2
        update_cache: yes
        state: present       
    - name: Install postgresql-server
      ansible.builtin.apt:
        name: postgresql
        update_cache: yes
        state: present
    - name: Grant all all from network 0.0.0.0/0 access 
      community.postgresql.postgresql_pg_hba:
        dest: /etc/postgresql/16/main/pg_hba.conf
        contype: host
        users: all
        source: 0.0.0.0/0
        databases: all
        method: scram-sha-256
    - name: Grant all postgres from local access 
      community.postgresql.postgresql_pg_hba:
        dest: /etc/postgresql/16/main/pg_hba.conf
        contype: local
        users: postgres
        databases: all
        method: trust
    - name: Grant replicator replicator from host access 
      community.postgresql.postgresql_pg_hba:
        dest: /etc/postgresql/16/main/pg_hba.conf
        contype: host
        users: replicator
        source: 0.0.0.0/0
        databases: replicator
        method: scram-sha-256
    - name: Replace a listen_addresses = '*'
      ansible.builtin.lineinfile:
        path: /etc/postgresql/16/main/postgresql.conf
        search_string: '#listen_addresses' 
        line: listen_addresses = '*'  
    - name: Restart PostgreSQL service
      ansible.builtin.systemd_service:
        name: postgresql.service
        state: restarted    
    - name: Delete if exist user replicator
      community.postgresql.postgresql_query:
        login_db: postgres
        query: drop user if exists replicator
    - name: Create user replicator
      community.postgresql.postgresql_query:
        login_db: postgres
        query: create user replicator login encrypted password 'password'
    - name: Stopped PostgreSQL service
      ansible.builtin.systemd_service:
        name: postgresql.service
        state: stopped   
    - name: Recursively remove directory
      ansible.builtin.file:
        path: /var/lib/postgresql/16/main/
        state: absent
      become: yes
EOF

```
</details>

<img width="1725" height="927" alt="image" src="https://github.com/user-attachments/assets/4f77653d-60eb-42e8-b525-655ad7e1c3ed" />

По картинке сверху видно, что все эшены плейбука выполнились успешно.

## 4.3 Patroni

Плэйбук с установкой Patroni содержит основные экшены:
- `Systemd patroni.service stop if exist` - проверка на запущенный Patroni
- `Install python3.12-venv`
- `Install acl package for proper file permissions when becoming unprivileged user` - установка acl для ansible
- `Create a directory /opt/patroni if it does not exist`
- `Creating virtual environment`
- `Install patroni[etcd3] into virtual environment, inheriting globally installed modules`
- `Install python3-psycopg2 into virtual environment, inheriting globally installed modules`
- `Create PGDATA directory /data/16 if it does not exist`
- `Create log directory /data/log/patroni if it does not exist`
- `Create etc patroni directory /etc/patroni if it does not exist`
- `Copy Patroni configuration template to /etc/patroni/conf.yml` - копирование шаблона конфига Patroni patroni.j2
- `Create sysd Patroni`
- `Systemd reload, enabled adn starting patroni.service`


<details>
<summary>setup.yml</summary>
  
```yml

tee setup.yml << EOF
---
- name: Configure etcd cluster
  hosts: servers
  become: yes
  gather_facts: yes
  vars:
    patroni_cluster_members: "{{ groups['servers'] | map('extract', hostvars, ['ansible_fqdn']) | list }}"
    patroni_initial_cluster_string: >-
      {{
        groups['servers']
        | map('extract', hostvars, ['ansible_fqdn'])
        | map('regex_replace', '^(.*?)(:[0-9]+)?$', '\1:2379')
        | join(',')
      }}
    patroni_node_index: "{{ groups['servers'].index(inventory_hostname) }}"
    patroni_names_map: ['patroni_node1', 'patroni_node2', 'patroni_node3']
    patroni_node: "{{ patroni_names_map[patroni_node_index] }}"
    patroni_replicator_ips: >-
      {{
        groups['servers']
        | map('extract', hostvars, ['ansible_default_ipv4', 'address'])
        | select('defined')
        | list
      }}

  tasks:
    - name: Systemd patroni.service stop if exist
      ansible.builtin.systemd_service:
        name: patroni.service
        state: stopped
    - name: Install python3.12-venv
      ansible.builtin.apt:
        name: python3.12-venv
        update_cache: yes
        state: present
    - name: Install acl package for proper file permissions when becoming unprivileged user
      ansible.builtin.apt:
        name: acl
        update_cache: yes
        state: present
    - name: Create a directory /opt/patroni if it does not exist
      ansible.builtin.file:
        path: /opt/patroni
        state: directory
        owner: postgres
        group: postgres
        mode: '0755'
    - name: Creating virtual environment
      ansible.builtin.command:
        cmd: python3 -m venv /opt/patroni/venv
      become: yes
      become_user: postgres
    - name: Install patroni[etcd3] into virtual environment, inheriting globally installed modules        
      ansible.builtin.pip:
        name: patroni[etcd3]
        virtualenv: /opt/patroni/venv
        virtualenv_site_packages: yes
    - name: Install python3-psycopg2 into virtual environment, inheriting globally installed modules
      ansible.builtin.pip:
        name: psycopg2-binary
        virtualenv: /opt/patroni/venv
        virtualenv_site_packages: yes
    - name: Create PGDATA directory /data/16 if it does not exist
      ansible.builtin.file:
        path: /data/16
        state: directory
        owner: postgres
        group: postgres
        mode: '0700'
    - name: Create log directory /data/log/patroni if it does not exist
      ansible.builtin.file:
        path: /data/log/patroni
        state: directory
        owner: postgres
        group: postgres
        mode: '0755'
    - name: Create etc patroni directory /etc/patroni if it does not exist
      ansible.builtin.file:
        path: /etc/patroni
        state: directory
        owner: postgres
        group: postgres
        mode: '0755'
    - name: Copy Patroni configuration template to /etc/patroni/conf.yml
      ansible.builtin.template:
        src: templates/patroni.j2
        dest: /etc/patroni/conf.yml
        owner: postgres
        group: postgres
        mode: '0644'
    - name: Create sysd Patroni
      ansible.builtin.template:
        src: templates/sysd_patroni.j2
        dest: /etc/systemd/system/patroni.service
    - name: Systemd reload, enabled adn starting patroni.service
      ansible.builtin.systemd_service:
        daemon_reload: true
        name: patroni.service
        enabled: true
        state: started
```
</details>

<details>
<summary>patroni.j2</summary>
  
```yml
scope: patroni_cluster
namespace: /patroni
name: {{ patroni_node }}
log:
  level: INFO
  dir: /data/log/patroni
  file_size: 50000000
  file_num: 10
restapi:
  listen: {{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }}:8008
  connect_address: {{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }}:8008

etcd:
  hosts: {{ patroni_initial_cluster_string }}

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

    {% for ip in patroni_replicator_ips %}

    - host replication replicator {{ ip }}/32 md5

    {% endfor %}
    - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: 'password'
      options:
        - CREATEDB
        - CREATEROLE

postgresql:
  listen: {{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }}:5432
  connect_address: {{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }}:5432
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
```
</details>


Нужно отдельно выделить данный кусок:

```yml
  pg_hba:
    - host replication replicator 127.0.0.1/8 md5

    {% for ip in patroni_replicator_ips %}

    - host replication replicator {{ ip }}/32 md5

    {% endfor %}
    - host all all 0.0.0.0/0 md5
```

Это цикл, который проходится по всем собранным адресам и добавляет их в строку.

Проверим, что все получилось:
<img width="1809" height="915" alt="image" src="https://github.com/user-attachments/assets/32688b4d-1ba5-4a65-aa1e-05a71e0df0d8" />
<img width="1784" height="163" alt="image" src="https://github.com/user-attachments/assets/727341dd-9194-4134-9cb4-638846fecb9d" />

Все работает и все взлетело! 

# 5. Репликация

Будем настраивать логическую репликацию. Т.к. ее настройка выполняется проще, и подойдет для тестовых целей целиком.
Прежде, удостоверимся, что кластера работают в норме:

Кластер-1:
<img width="1797" height="341" alt="image" src="https://github.com/user-attachments/assets/087d533c-8426-44f8-8ee5-a157e094fb8c" />
Кластер-2:
<img width="1793" height="337" alt="image" src="https://github.com/user-attachments/assets/d868e9cc-f651-4122-8fe4-917f752b12b0" />

Проверим, что `wal_level = replica`, `max_replication_slots = 10`, `max_wal_senders = 10` и что наш пользователь `replicator` существет с ему присущим одноименном параметре:

<img width="1428" height="608" alt="image" src="https://github.com/user-attachments/assets/6c0cce25-24da-444e-8ccd-9f7d07e2bfa1" />


## 5.1 Начало

### 5.1.1 Основные настройки Кластера-1

Создадим тестовую БД, наполним ее на стороне `primary` кластера.
Создадим публикацию и подписку, на кластере `primary` будет расположена публикация, на кластере `secondary` будет расположена подписка.

Конектимся к лидерам, например с удаленной машины при помощи установленного клиента psql:
```bash
psql -h 10.92.36.113 -d postgres -U postgres
psql -h 10.92.35.112 -d postgres -U postgres
```
<img width="1682" height="225" alt="image" src="https://github.com/user-attachments/assets/240c90cd-9af4-4362-9117-7d94df60df25" />

На кластере пубикации `cluter-1`(10.92.35.112) создадим тестовую БД:
```sql
postgres=# CREATE USER test_log_repl WITH SUPERUSER;
CREATE ROLE
postgres=# CREATE DATABASE repl_db WITH OWNER=test_log_repl;
CREATE DATABASE
postgres=# ALTER USER test_log_repl WITH PASSWORD 'qwerty@123';
ALTER ROLE 
postgres=#
```

<img width="806" height="361" alt="image" src="https://github.com/user-attachments/assets/02b04735-349c-445a-b8b4-817595696c8b" />

Наполним тестовые таблицы данными:
<img width="740" height="570" alt="image" src="https://github.com/user-attachments/assets/4af90a2b-21dc-4d16-b7f3-cb4889500518" />
<img width="825" height="286" alt="image" src="https://github.com/user-attachments/assets/62d3f27c-a349-42a7-83a6-56f6f0251107" />

Создадим слот репликации на Кластере-1:
```
SELECT pg_create_physical_replication_slot('standby_cluster_2_slot');        ---Создадим
SELECT slot_name, slot_type, active, restart_lsn FROM pg_replication_slots;  ---Проверим
```

<img width="785" height="339" alt="image" src="https://github.com/user-attachments/assets/038d617d-216e-48cd-adef-c9e508dd221d" />

Добавим следующий записи в `/data/16/pg_hba.conf` Кластера-1 через команду patronictl -c `/etc/patroni/conf.yml edit-config`:
```
host replication replicator 10.92.35.0/24 md5
host replication replicator 10.92.36.0/24 md5
```

<img width="776" height="466" alt="image" src="https://github.com/user-attachments/assets/bfaa3fd8-b204-4487-80a7-a38e82204be2" />

И перезапустим сллужбу Patroni Кластера-1: `sudo systemctl restart patroni.service`

### 5.1.1 Основные настройки Кластера-2

Схема включения Кластера-2 как репликатора Кластера состоит из:
- Остановка всех нод
- Правка конфига Patroni
- Ручной тест `pg_basebackup`
- Создание `/data/16/standby.signal`
- Прописать `primary_conninfo`
- Запуск ноды-лидера
- Запуск всех остальных нод Кластера-2

Останавливаем все ноды: 
```
sudo systemctl stop patroni.service
```

Правим конфиг `/etc/patroni/conf.yml`:
```
Было:
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
-----///-----

Стало:
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 30
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        max_replication_slots: 10
        max_wal_senders: 10
  standby_cluster:
    host: "10.92.35.112,10.92.35.52,10.92.35.162"
    port: 5432
    primary_slot_name: "standby_cluster_2_slot"
    create_replica_methods:
      - basebackup
```

`primary_slot_name: "standby_cluster_2_slot"` - наш добавленный слот репликации на Кластере-1

Запускаем ноду-лидера и проверим работу `pg_basebackup` c поднятой лидер-ноды Patroni Кластера-2:
```
sudo -u postgres /usr/lib/postgresql/16/bin/pg_basebackup \
  -h 10.92.35.112 \
  -p 5432 \
  -U replicator \
  -D /data/16 \
  -X stream \
  -P
```

<img width="806" height="821" alt="image" src="https://github.com/user-attachments/assets/c669af97-32c5-4bca-bb60-a751d735e042" />

Как видно, `pg_basebackup` работает!

Создаем сигнал-файл `standby.signal`, чтобы нода знала, что она является `standby`. Bносим в `postgresql.auto.conf` инфо о коннекте к `primary`:

<img width="821" height="201" alt="image" src="https://github.com/user-attachments/assets/4724f885-a815-466e-ac85-347897ed7458" />

