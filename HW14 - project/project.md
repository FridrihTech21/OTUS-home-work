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
sudo tee -a /etc/hosts <<EOF
10.92.5.83 tarasov-test-otus-proj-balancer tarasov-test-otus-proj-balancer.ru-central1.internal
10.92.5.173 tarasov-test-otus-proj-s3 tarasov-test-otus-proj-s3.ru-central1.internal

ВПИСАТЬ ОСТАЛЬНЫЕ ХОСТs

EOF

sudo tee -a .ssh/authorized_keys <<EOF
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILe9OR8QM0BUoSrEpsf0syR+6f7Ll0+hkYg2E3FqI00i fvtarasov@tarasov-test-otus-proj-s3
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIM7ZdqiyXBzEHM15yXSSd+PQp0+PzDBk7SZ+zpNCCWRE fvtarasov@tarasov-test-otus-proj-balancer

ВПИСАТЬ ОСТАЛЬНЫЕ ССШ ХОСТОВ

EOF
```

Не забываем добавить пользователя, под которым будет работать Ansible в `sudo`: `sudo usermod -aG sudo fvtarasov`

Далее пишем inventory.ini для проверки сетевой связности и в путь. 

<details>
<summary>inventory.ini</summary>
  
```yml
tee inventory.ini << EOF
[servers]
tarasov-test-otus-proj-balancer.ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519
tarasov-test-otus-proj-s3.ru-central1.internal ansible_user=fvtarasov ansible_ssh_private_key_file=~/.ssh/id_ed25519

Добавить остальные хосты

EOF
```
</details>

## Добавить скрин с остальными хостами

<img width="1684" height="465" alt="image" src="https://github.com/user-attachments/assets/c7de4ed8-5a7a-4bea-9b16-fd51f953b6d4" />

Связность имеется, продолжим!

1) Patroni(Primary & Standby)
2) Репликация(Primary & Standby)
3) Keepalived & HAproxy
4) pgBackRest & S3 MINIO


4) Резервное копирование в S3 MINIO

4.1) Установка 

4.2) Настройка
Внутренняя настройка S3

4.3) Осуществление резервного копирования с primary в S3(В этом поможет HAproxy)
