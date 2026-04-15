# ТЕМА: Развертывание двух реплицируемых географически распределенных кластеров PostgreSQL в режиме высокой доступности и отказоустойчивости на базе Patroni и резервным копированием в MINIO S3

`tarasov-test-otus-proj-balancer` - HAproxy, keepalived

`tarasov-test-otus-proj-s3` - S3 MINIO

`tarasov-test-otus-proj-cluster-1-node-1` - нода изначального Primary

`tarasov-test-otus-proj-cluster-1-node-2` - нода изначального Primary

`tarasov-test-otus-proj-cluster-1-node-3` - нода изначального Primary

`tarasov-test-otus-proj-cluster-2-node-1` - нода изначального Standby

`tarasov-test-otus-proj-cluster-2-node-2` - нода изначального Standby

`tarasov-test-otus-proj-cluster-2-node-3` - нода изначального Standby

Развертывать будем при помощи Ansible:
```
sudo apt update && sudo apt install software-properties-common && sudo add-apt-repository --yes --update ppa:ansible/ansible && sudo apt install ansible
```
<img width="1106" height="222" alt="image" src="https://github.com/user-attachments/assets/aeb57607-715c-4d83-8239-b750e67a5097" />

Настроим сетевую свзяность и SSH:
```
sudo tee -a /etc/hosts << EOF
10.92.36.9 tarasov-test-otus-proj-cluster-1-node-1 tarasov-test-otus-proj-cluster-1-node-1.ru-central1.internal
10.92.36.33 tarasov-test-otus-proj-cluster-1-node-2 tarasov-test-otus-proj-cluster-1-node-2.ru-central1.internal
10.92.36.81 tarasov-test-otus-proj-cluster-1-node-3 tarasov-test-otus-proj-cluster-1-node-3.ru-central1.internal
10.92.35.60 tarasov-test-otus-proj-cluster-2-node-1 tarasov-test-otus-proj-cluster-2-node-1.ru-central1.internal
10.92.35.117 tarasov-test-otus-proj-cluster-2-node-2 tarasov-test-otus-proj-cluster-2-node-2.ru-central1.internal
10.92.35.12 tarasov-test-otus-proj-cluster-2-node-3 tarasov-test-otus-proj-cluster-2-node-3.ru-central1.internal
10.92.5.173 tarasov-test-otus-proj-s3 tarasov-test-otus-proj-s3.ru-central1.internal
10.92.5.83 tarasov-test-otus-proj-balancer tarasov-test-otus-proj-balancer.ru-central1.internal
EOF


tee -a ~/.ssh/authorized_keys << EOF
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIM7ZdqiyXBzEHM15yXSSd+PQp0+PzDBk7SZ+zpNCCWRE fvtarasov@tarasov-test-otus-proj-balancer
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKBrpWnLATvYhvYI204+LojR/PHBKmUbJGkQWbcZgCFh fvtarasov@tarasov-test-otus-proj-cluster-1-node-1
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAxJ7KQCgGo5fj7CI7KrlGcDwiY48KAhFxj8eVcs/4ky fvtarasov@tarasov-test-otus-proj-cluster-1-node-2
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBPuthMgI36dY6TkRa+Y4jdd4yJJz1DXIhaxKOY10gfi fvtarasov@tarasov-test-otus-proj-cluster-1-node-3
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKEbhoiioC+5GlUiXZtf6Ra6nCfwqdDZJ41w5G5ONozE fvtarasov@tarasov-test-otus-proj-cluster-2-node-1
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIX0yy3WI4hK2ebTrnIokcOpxDOfInSh1vd3WMO7BKMH fvtarasov@tarasov-test-otus-proj-cluster-2-node-2
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH5qfXnm91mi4qe1HMinJyhtRBEPzVofpc2JL4vpkXXs fvtarasov@tarasov-test-otus-proj-cluster-2-node-3
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILe9OR8QM0BUoSrEpsf0syR+6f7Ll0+hkYg2E3FqI00i fvtarasov@tarasov-test-otus-proj-s3
EOF
```

Не забываем добавить пользователя, под которым будет работать Ansible в `sudo`: `sudo usermod -aG sudo fvtarasov`

Далее пишем inventory.ini для проверки сетевой связности и в путь. 

<details>
<summary>inventory.ini</summary>
  
```yml
tee inventory.ini << EOF
[servers]
tarasov-test-otus-proj-cluster-[num_of_cluster]-node-[num_node].ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519
tarasov-test-otus-proj-cluster-[num_of_cluster]-node-[num_node].ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519
tarasov-test-otus-proj-cluster-[num_of_cluster]-node-[num_node].ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519
EOF
```
</details>

<img width="1379" height="714" alt="image" src="https://github.com/user-attachments/assets/89a93e5e-4968-45c8-b6fa-46e19fc0b0bf" />


Теперь добавим playbook, в котором обновим пакеты на целевых машинах:

<details>
<summary>setup.yml</summary>
  
```yml
tee -a update_packs.yml << EOF
---
- name:
  hosts: servers
  become: yes
  tasks:
    - name: Updating lists packages
      ansible.builtin.apt:
        update_cache: yes
EOF
```
</details>

<img width="1379" height="645" alt="image" src="https://github.com/user-attachments/assets/54a4aeeb-8dee-4390-8416-abc3359e8084" />

Далее будем идти по плану:
1) Установка кластеров Patroni
   1.1) Установка ETCD
   1.2) Установка PostgreSQL
   1.3) Установка Patroni
2) Репликация(Primary & Standby)
   2.1) Настройка Primary
   2.2) Настройка Standby
   2.3) Проверка репликации  
4) Keepalived & HAproxy
5) pgBackRest & S3 MINIO

# 1. Установка кластеров Patroni

Установка будет проведена средствами Ansible. 

## 1.1 Установка ETCD

Дерево каталога:
<img width="777" height="182" alt="image" src="https://github.com/user-attachments/assets/02409b6b-3d40-4d6b-b876-5eee584ecece" />

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
    - name: Change cloud.cfg
      become: true
      ansible.builtin.replace:
        path: /etc/cloud/cloud.cfg
        regexp: '- update_etc_hosts'
        replace: '#  - update_etc_hosts'
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

<details>
<summary>inventory.ini</summary>
  
```yml
tee -a inventory.ini << EOF
---
- name:
  hosts: servers
  become: yes
  tasks:
    - name: Updating lists packages
      ansible.builtin.apt:
        update_cache: yes
EOF
```
</details>

<details>
<summary>etcd.j2</summary>
  
```yml
tee -a etcd.j2 << EOF
---
- name:
  hosts: servers
  become: yes
  tasks:
    - name: Updating lists packages
      ansible.builtin.apt:
        update_cache: yes
EOF
```
</details>

<details>
<summary>setup.yml</summary>
  
```yml
tee -a update_packs.yml << EOF
---
- name:
  hosts: servers
  become: yes
  tasks:
    - name: Updating lists packages
      ansible.builtin.apt:
        update_cache: yes
EOF
```
</details>

<img width="1765" height="224" alt="image" src="https://github.com/user-attachments/assets/7100515b-9801-4146-9458-8d15a2938faf" />

ETCD установлен.

## 1.2 Установка PostgreSQL

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

<img width="1771" height="1008" alt="image" src="https://github.com/user-attachments/assets/6394e908-424b-4dac-887b-79d17373807b" />
<img width="800" height="181" alt="image" src="https://github.com/user-attachments/assets/115d2c07-14de-4014-84a3-f202af6a5f11" />

## 1.3 Установка Patroni

<details>
<summary>setup.yml</summary>
  
```yml
sudo tee setup.yml << EOF
---
- name: Install and Configure Patroni cluster
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
    - name: Get service facts from host
      ansible.builtin.service_facts:
    - name: Systemd patroni.service stop if exist
      ansible.builtin.systemd_service:
        name: patroni.service
        state: stopped
      when: "'patroni.service' in ansible_facts.services"
      ignore_errors: true
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
EOF
```
</details>

<details>
<summary>patroni.j2</summary>
  
```yml
sudo tee templates/patroni.j2 << EOF
scope: patroni_cluster_1
namespace: /patroni_1
name: {{ patroni_node }}_cluster_1
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
EOF
```
</details>

<details>
<summary>sysd_patroni.j2</summary>
  
```yml
sudo tee templates/sysd_patroni.j2 << EOF
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

<img width="1091" height="180" alt="image" src="https://github.com/user-attachments/assets/9d3af9df-fee6-4173-a9d4-3529395ff613" />
<img width="1097" height="178" alt="image" src="https://github.com/user-attachments/assets/a51ecd3b-70f5-406e-80e2-32e7587a5f6d" />

Patroni установлен.

# 2 Репликация(Primary & Standby)

## 2.1 Настройка Primary

## 2.2 Настройка Standby

## 2.3 Проверка репликации  





# 4) Резервное копирование в S3 MINIO

4.1) Установка 

Установка проходит при помощи Ansible:

<details>
<summary>inventory.ini</summary>
  
```yml
[server_S3]
tarasov-test-otus-proj-s3.ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519
```
</details>

<details>
<summary>setup.yml</summary>
  
```yml
tee setup.yml << EOF
---
- name:
  hosts: server_S3
  become: yes
  tasks:
    - name: Set timezone to
      become: true
      community.general.timezone:
        name: Europe/Moscow
    - name: Install chrony
      become: true
      ansible.builtin.apt:
        name: chrony
        update_cache: yes
        state: present
    - name: IPTABLES. Insert rule in the beginning of INPUT chain for PORT=9000
      become: true
      ansible.builtin.command:
        cmd: iptables -I INPUT -p tcp --dport 9000 -j ACCEPT
    - name: IPTABLES. Insert rule in the beginning of INPUT chain for PORT=9001
      become: true
      ansible.builtin.command:
        cmd: iptables -I INPUT -p tcp --dport 9001 -j ACCEPT
    - name: IPTABLES. Install persistent
      become: true
      ansible.builtin.apt:
        name: iptables-persistent
        update_cache: yes
        state: present
    - name: IPTABLES. Save netfilter
      become: true
      ansible.builtin.command:
        cmd: netfilter-persistent save
    - name: Updating lists packages
      ansible.builtin.apt:
        update_cache: yes   
    - name: Download MINIO
      ansible.builtin.get_url:
        url: https://dl.min.io/server/minio/release/linux-amd64/archive/minio.RELEASE.2025-04-22T22-12-26Z
        dest: /tmp/minio
        mode: '0775'
    - name: Move minio-file to /usr/local/bin/minio
      become: true
      ansible.builtin.command:
        cmd: mv /tmp/minio /usr/local/bin/minio
    - name: Add user minio without home_dir
      become: true
      ansible.builtin.user:
        name: minio
        shell: /bin/bash
    - name: Create /var/lib/minio/data
      become: true
      ansible.builtin.file:
        path: /var/lib/minio/data
        state: directory
        owner: minio
        group: minio
        mode: '0775'
    - name: Download mc
      ansible.builtin.get_url:
        url: https://dl.min.io/client/mc/release/linux-amd64/mc
        dest: /tmp/mc
        mode: '0775'
    - name: Move mc-file to /usr/local/bin/
      become: true
      ansible.builtin.command:
        cmd: mv /tmp/mc /usr/local/bin/
    - name: Change owner and group for /usr/local/bin/mc
      become: true
      ansible.builtin.file:
        path: /usr/local/bin/mc
        owner: minio
        group: minio
    - name: Download minio.service
      ansible.builtin.get_url:
        url: https://raw.githubusercontent.com/minio/minio-service/master/linux-systemd/minio.service
        dest: /tmp/minio.service
        mode: '0775'
    - name: Move minio.service to  /etc/systemd/system/
      become: true
      ansible.builtin.command:
        cmd: mv /tmp/minio.service  /etc/systemd/system/
    - name: Type=notify change to Type=simple in /etc/systemd/system/minio.service
      become: true
      ansible.builtin.replace:
        path: /etc/systemd/system/minio.service
        regexp: 'Type=notify'
        replace: 'Type=simple' 
    - name: User=minio-user change to User=minio in /etc/systemd/system/minio.service
      become: true
      ansible.builtin.replace:
        path: /etc/systemd/system/minio.service
        regexp: 'User=minio-user'
        replace: 'User=minio' 
    - name: Group=minio-user change to Group=minio in /etc/systemd/system/minio.service
      become: true
      ansible.builtin.replace:
        path: /etc/systemd/system/minio.service
        regexp: 'Group=minio-user'
        replace: 'Group=minio' 
    - name: Copy EnvironmentFile for minio.service
      become: true
      ansible.builtin.template:
        src: EnvironmentFile.j2
        dest: /etc/default/minio
    - name: Enabled, Reload and Start sys.d 
      ansible.builtin.systemd_service:
        name: minio.service
        daemon_reload: true
        state: started
        enabled: true
EOF
```
</details>

<img width="1895" height="998" alt="image" src="https://github.com/user-attachments/assets/7b14a5cc-262b-47ff-bfeb-7c4d3a91e4ac" />

4.2) Настройка
Внутренняя настройка S3

4.3) Осуществление резервного копирования с primary в S3(В этом поможет HAproxy)
