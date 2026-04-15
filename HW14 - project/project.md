ТЕМА: 
Развертывание двух реплицируемых географически распределенных кластеров PostgreSQL в режиме высокой доступности и отказоустойчивости на базе Patroni и резервным копированием в MINIO S3

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

## Добавить скрин с остальными хостами

<img width="1684" height="465" alt="image" src="https://github.com/user-attachments/assets/c7de4ed8-5a7a-4bea-9b16-fd51f953b6d4" />

Теперь добавим playbook, в котором обновим пакеты на целевых машинах:

<details>
<summary>setup.yml</summary>
  
```yml
tee -a setup.yml << EOF
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

<img width="1691" height="481" alt="image" src="https://github.com/user-attachments/assets/1fe2e458-6178-4b1b-8872-819257bf24c5" />



1) Patroni(Primary & Standby)
2) Репликация(Primary & Standby)
3) Keepalived & HAproxy
4) pgBackRest & S3 MINIO


4) Резервное копирование в S3 MINIO

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
