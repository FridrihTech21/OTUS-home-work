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

1) Patroni(Primary & Standby)
2) Репликация(Primary & Standby)
3) Keepalived & HAproxy
4) pgBackRest & S3 MINIO


4) Резервное копирование в S3 MINIO

4.1) Установка 

4.2) Настройка
Внутренняя настройка S3

4.3) Осуществление резервного копирования с primary в S3(В этом поможет HAproxy)
