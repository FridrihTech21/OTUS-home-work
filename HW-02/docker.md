# 1. Создайте инстанс с Ubuntu 20.04 в Яндекс.Облаке или аналогах.

# 2. Установите Docker Engine.
Перед первой установкой Docker Engine на новом компьютере необходимо настроить репозиторий Docker apt. После этого вы можете установить и обновить Docker из репозитория.
### 2.1 Настройте репозиторий apt в Docker.
Добавлен официальный GPG-ключ Docker:

		sudo tee /etc/apt/sources.list.d/docker.sources <<EOF   
		Types: deb
		URIs: https://download.docker.com/linux/ubuntu
		Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
		Components: stable
		Signed-By: /etc/apt/keyrings/docker.asc
		EOF

Добавлен репозиторий в Apt sources:

		sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
		Types: deb
		URIs: https://download.docker.com/linux/ubuntu
		Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
		Components: stable
		Signed-By: /etc/apt/keyrings/docker.asc
		EOF

		sudo apt update
	
### 2.2 Установка пакетов Docker.
		sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

Служба Docker запускается автоматически после установки. Убедимся, что Docker запущен:
		
		fvtarasov@tarasov-postgre-advance:~$ sudo systemctl status docker
		● docker.service - Docker Application Container Engine
			 Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
			 Active: active (running) since Tue 2026-01-13 20:27:29 UTC; 15s ago
		TriggeredBy: ● docker.socket
			   Docs: https://docs.docker.com
		   Main PID: 2925 (dockerd)
			  Tasks: 9
			 Memory: 21.6M
			 CGroup: /system.slice/docker.service
					 └─2925 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

		Jan 13 20:27:28 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:28.288953096Z" level=info msg="Creating a containerd client" address=/run/containerd/containerd.sock timeout=1m0s
		Jan 13 20:27:28 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:28.377642298Z" level=info msg="Loading containers: start."
		Jan 13 20:27:29 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:29.022778556Z" level=info msg="Loading containers: done."
		Jan 13 20:27:29 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:29.114814297Z" level=warning msg="WARNING: No swap limit support"
		Jan 13 20:27:29 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:29.114874590Z" level=info msg="Docker daemon" commit=01f442b containerd-snapshotter=false storage-driver=overlay2 version=28.1.1
		Jan 13 20:27:29 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:29.114967081Z" level=info msg="Initializing buildkit"
		Jan 13 20:27:29 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:29.160404750Z" level=info msg="Completed buildkit initialization"
		Jan 13 20:27:29 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:29.165455834Z" level=info msg="Daemon has completed initialization"
		Jan 13 20:27:29 tarasov-postgre-advance dockerd[2925]: time="2026-01-13T20:27:29.165677657Z" level=info msg="API listen on /run/docker.sock"
		Jan 13 20:27:29 tarasov-postgre-advance systemd[1]: Started Docker Application Container Engine.

    
# 3. Создайте каталог /var/lib/postgres для хранения данных.
Каталог создан:
  
		fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ pwd
		/var/lib/postgres

# 4. Разверните контейнер с PostgreSQL 14, смонтировав в него /var/lib/postgres.
	fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ sudo docker run --name server_postgres_14 -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password -e POSTGRES_USER=postgres -e POSTGRES_DB=postgres -d postgres:14
	05318abac1396ae75a07fea6d6b61da1300d9789092aa1028cbf171d84b70c6d
	fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ sudo docker ps -a
	CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                         NAMES
	05318abac139   postgres:14   "docker-entrypoint.s…"   11 seconds ago   Up 11 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   server_postgres_14

# 5. Разверните контейнер с клиентом PostgreSQL.
	fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ sudo docker run -d --name postgres-client --network host postgres:14 
	5da8f5436c8997d683c691e6316cdbfe7dcb6d87dc11c8ccd71138fa64712ac6
	
	fvtarasov@tarasov-postgre-advance:~$ sudo docker ps -a
	CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                         NAMES
	5da8f5436c89   postgres:14   "docker-entrypoint.s…"   3 seconds ago   Up 2 seconds                                                 postgres-client
	05318abac139   postgres:14   "docker-entrypoint.s…"   6 minutes ago   Up 6 minutes   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   server_postgres_14
	
# 6. Подключитесь из контейнера с клиентом к контейнеру с сервером и создайте таблицу с данными о перевозках.
	create table shipments(id serial, product_name text, quantity int, destination text);

	insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
	insert into shipments(product_name, quantity, destination) values('bananas', 1500, 'Asia');
	insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');
	insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
	insert into shipments(product_name, quantity, destination) values('coffee', 700, 'Canada');
	insert into shipments(product_name, quantity, destination) values('coffee', 300, 'Japan');
	insert into shipments(product_name, quantity, destination) values('sugar', 1000, 'Europe');
	insert into shipments(product_name, quantity, destination) values('sugar', 800, 'Asia');
	insert into shipments(product_name, quantity, destination) values('sugar', 600, 'Africa');
	insert into shipments(product_name, quantity, destination) values('sugar', 400, 'USA');

Подключение через контейнер клиента:

	fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ sudo docker exec -it postgres-client /bin/bash
	root@tarasov-postgre-advance:/# su - postgres
	postgres@tarasov-postgre-advance:~$ psql -h localhost -p 5432 -U postgres
	Password for user postgres:
	psql (14.20 (Debian 14.20-1.pgdg13+1))
	Type "help" for help.

Выполнение DDL и DML:

	postgres=#
	postgres=# create table shipments(id serial, product_name text, quantity int, destination text);
	CREATE TABLE
	postgres=# insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
	s(product_name, quantity, destination) values('bananINSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('bananas', 1500, 'Asia');
	me, quantity, destination) values('bananas', 2000, 'AINSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');
	, destination) values('coffee', 500, 'USA');
	insert into shipments(product_name, quantity, destination)INSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
	 values('coffee', 700, 'Canada');
	insert into shipmenINSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('coffee', 700, 'Canada');
	INSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('coffee', 300, 'Japan');
	ope');
	insert into shipments(product_name, quantity, INSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('sugar', 1000, 'Europe');
	destination) values('sugar', 800, 'Asia');
	insert into shipments(product_name, quantity, destination) valINSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('sugar', 800, 'Asia');
	roduct_name, quantity, destination) values('sugar', 4INSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('sugar', 600, 'Africa');
	INSERT 0 1
	postgres=# insert into shipments(product_name, quantity, destination) values('sugar', 400, 'USA');
	INSERT 0 1
	postgres=#
	postgres=# select * from shipments;
	 id | product_name | quantity | destination
	----+--------------+----------+-------------
	  1 | bananas      |     1000 | Europe
	  2 | bananas      |     1500 | Asia
	  3 | bananas      |     2000 | Africa
	  4 | coffee       |      500 | USA
	  5 | coffee       |      700 | Canada
	  6 | coffee       |      300 | Japan
	  7 | sugar        |     1000 | Europe
	  8 | sugar        |      800 | Asia
	  9 | sugar        |      600 | Africa
	 10 | sugar        |      400 | USA
	(10 rows)

# 7. Подключитесь к контейнеру с сервером с ноутбука или компьютера.

На картинке видно, что подключение прошло успешно. 

![Подключение из DBeaver к контейнеру сервера PostgreSQL14](project/lab_2/Задание_2_подключение_с_ноутбука.jpg)
	
# 8. Удалите контейнер с сервером и создайте его заново.

	fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ sudo docker stop server_postgres_14
	server_postgres_14
	fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ sudo docker rm server_postgres_14
	server_postgres_14
	fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ sudo docker run --name server_postgres_14 -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password -e POSTGRES_USER=postgres -e POSTGRES_DB=postgres -d postgres:14
	65be9f47405395dd75a6a1d6d37f3ccdad8cc39b4ba35476f535a3008d23a347

# 9. Проверьте, что данные остались на месте.

Проверка показала, что данные остались на месте, так как volume:/var/lib/postgres:/var/lib/postgresql/data сохранен за пределами файловой системы docker
	
	fvtarasov@tarasov-postgre-advance:/var/lib/postgres$ sudo docker exec -it postgres-client /bin/bash
	root@tarasov-postgre-advance:/# su - postgres
	postgres@tarasov-postgre-advance:~$ psql -h localhost -p 5432 -U postgres
	Password for user postgres:
	psql (14.20 (Debian 14.20-1.pgdg13+1))
	Type "help" for help.

	postgres=# select * from shipments;
	 id | product_name | quantity | destination
	----+--------------+----------+-------------
	  1 | bananas      |     1000 | Europe
	  2 | bananas      |     1500 | Asia
	  3 | bananas      |     2000 | Africa
	  4 | coffee       |      500 | USA
	  5 | coffee       |      700 | Canada
	  6 | coffee       |      300 | Japan
	  7 | sugar        |     1000 | Europe
	  8 | sugar        |      800 | Asia
	  9 | sugar        |      600 | Africa
	 10 | sugar        |      400 | USA
	(10 rows)

  # Вывод
  Домашняя работа продемонстрировала эффективность использования Docker для разворачивания и изоляции PostgreSQL-сервера, а также важность volumes для сохранения данных. Был реализован замкнутый цикл: создание инфраструктуры, подключение, манипуляция данными, проверка сохранности.
