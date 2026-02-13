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
etcd1: 10.92.36.48
etcd2: 10.92.36.31
etcd3: 10.92.36.28
node1: 10.92.36.82
node2: 10.92.36.60
node3: 10.92.36.103
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

### Шаг 6. Проверка работы кластера etcd

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

