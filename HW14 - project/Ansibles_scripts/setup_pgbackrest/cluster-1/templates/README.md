ОБЯЗАТЕЛЬНО В ШАБЛОНАХ .j2 ВСТАВИТЬ НЕОБХОДИМЫЕ КЛЮЧИ И СЕРТЫ!!!! ИНАЧЕ pgBackRest не будет работать!

Для теста их можно взять с S3

Если планируется централизованный запуск с отдельной машины(вне кластеров Patroni) пуша WAL в S3, тогда для этих машин стоит прописать следующее в /etc/pgbackrest/pgbackrest.conf:

```
[backup-cluster-1]
pg1-host=10.92.36.9
pg1-path=/data/16/
pg1-port=5432
pg1-host-user=postgres
pg1-database=test_db
pg1-socket-path=/data/16

pg2-host=10.92.36.64
pg2-path=/data/16/
pg2-port=5432
pg2-host-user=postgres
pg2-database=test_db
pg2-socket-path=/data/16

pg3-host=10.92.36.81
pg3-path=/data/16/
pg3-port=5432
pg3-host-user=postgres
pg3-database=test_db
pg3-socket-path=/data/16

[global]
repo1-retention-full=2 
repo1-type=s3
repo1-path=/backups-psql-16
repo1-s3-region=ru-central-1
repo1-s3-endpoint=https://tarasov-test-otus-proj-s3.ru-central1.internal
repo1-storage-port=9000
repo1-s3-bucket=backups
repo1-s3-key=admin
repo1-s3-key-secret=admin123
repo1-s3-uri-style=path
repo1-storage-verify-tls=n

tls-server-auth=client-cn=tarasov-test-otus-proj-s3.ru-central1.internal
tls-server-ca-file=/data/certs/ca.crt
tls-server-cert-file=/data/certs/server.crt
tls-server-key-file=/data/certs/server.key

backup-standby=y
start-fast=y
delta=y
compress-type=zst
compress-level=6

log-level-console=info
log-level-file=debug
```
