# 1 НАСТРОЙКА СРЕДЫ И УСТАНОВКА
Перед началом работы с PostgreSQL нужно обновить/установить, соответсвующие пакеты: 

```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql
```

# 2 ВЫПОНЛЕНИЕ ЗАДАНИЯ
Для работы с PSQL будет использована утилита в командной строке системы `psql`. Для входа в БД запусти данную утилиту:
```
psql
```
Создаем БД для выполнения задания:
```
postgres=# create database task_1 with owner=postgres;
CREATE DATABASE
```
## 2.1 Первая часть задания
Открываем транзакцию №1 и №2:
##### TRANSACTION_1:
```
postgres=# \c task_1;
You are now connected to database "task_1" as user "postgres".
task_1=#
```
##### TRANSACTION_2:
```
fvtarasov@tarasov-postgre-advance:~$ sudo su - postgres
postgres@tarasov-postgre-advance:~$ psql -U postgres task_1
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.
```
Отключаем auto commit:
##### TRANSACTION_1 и TRANSACTION_2:
```
task_1=# \echo :AUTOCOMMIT
on
task_1=# \set AUTOCOMMIT off
task_1=# \echo :AUTOCOMMIT
off
```
Создаем таблицу и наполняем ее данными в транзакции №1:
##### TRANSACTION_1:
```
task_1=# create table table1(id int, amnt int);
CREATE TABLE
task_1=# insert into table1 (amnt) values (5),(10),(1000),(10);
INSERT 0 4
task_1=# select * from table1 commit;
 id | amnt
----+------
  1 |    5
  2 |   10
  3 | 1000
  4 |   10
(4 rows)
```
Просмотр текущего уровня изоляции в транзкации №1:
##### TRANSACTION_1:
```
task_1=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
Начало новой транзакции в обеих сессиях, добавление записи в таблицу транзакции №1, с поледующей выборкой записей в транзакции №2:
```
TRANSACTION_1:
task_1=# begin;
BEGIN
task_1=*# insert into table1 (amnt) values (66);
INSERT 0 1
task_1=*# select * from table1;
 id | amnt
----+------
  1 |    5
  2 |   10
  3 | 1000
  4 |   10
  5 |   66
(5 rows)

TRANSACTION_2:
task_1=# begin;
BEGIN
task_1=*# select * from table1;
 id | amnt
----+------
  1 |    5
  2 |   10
  3 | 1000
  4 |   10
(4 rows)
```
В транзакции №2 изменений нет, т.к. `read committed` видит снимок БД на момент открытия транзакции терминала №2, пока не завершится транзакция в термианале №1.

##### Подтверждение:
```
TRANSACTION_1:
task_1=*# commit;
COMMIT

TRANSACTION_2:
task_1=*# select * from table1;
 id | amnt
----+------
  1 |    5
  2 |   10
  3 | 1000
  4 |   10
  5 |   66
(5 rows)
```
Транзакция терминала №1 успешно завершилась, следовательно `SELECT` в терминале №2 отобразил новую запись

## 2.2 Вторая часть задания

Открываем сессии с уровнем изоляции `Repeatable read` в транзакциях:
```
TRANSACTION_2:
task_1=# begin;
BEGIN
task_1=*# set transaction isolation level repeatable read;
SET
task_1=*# show transaction isolation level;
 transaction_isolation
-----------------------
 repeatable read
(1 row)

TRANSACTION_2: 
task_1=*# select * from table1;
 id | amnt
----+------
  1 |    5
  2 |   10
  3 | 1000
  4 |   10
  5 |   66
(5 rows)
```
Запись не видна, т.к. TRANSACTION_1 не завершена. `Repeatable read` не позволяет видеть изминения в таблице после измения и подтверждения данных
```
TRANSACTION_1:
commit;

TRANSACTION_2:
task_1=*# rollback;
ROLLBACK
task_1=# select * from table1;
 id | amnt
----+------
  1 |    5
  2 |   10
  3 | 1000
  4 |   10
  5 |   66
  6 |  777
(6 rows)
```
Иизмнения видны, т.к. транзакция с `Repeatable read` завершена. После ее завершения применяется изоляция по умолчанию: `Read committed`, соответсвенно примененные измения будут отображены
