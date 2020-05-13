# PostgreSQL
## Задание
PostgreSQL  
- Настроить hot_standby репликацию с использованием слотов
- Настроить правильное резервное копирование
Для сдачи работы присылаем ссылку на репозиторий, в котором должны  
обязательно быть Vagranfile и плейбук Ansible, конфигурационные  
файлы postgresql.conf, pg_hba.conf и recovery.conf, а так же конфиг barman,  
либо скрипт резервного копирования. Команда "vagrant up" должна поднимать  
машины с настроенной репликацией и резервным копированием. Рекомендуется в  
README.md файл вложить результаты (текст или скриншоты) проверки  
работы репликации и резервного копирования.  
## Выполнение
Стенд из 3 ВМ - *server*, *replica*, *backup*  
настраивается автоматически плэйбуком [pg.yml](pg.yml)

server - основная ВМ, с которой будет проходить  
репликация на replica и резервное копирование на backup  

###
На все ВМ устаналивается пакет
postgresql11-server из PGDG CentOS repositories

На Server
barman-cli из 2ndquadrant-dl-default-release-pg11

На backup
barman из 2ndquadrant-dl-default-release-pg11
### Конфиг server
[pg_hba.conf](files/server/pg_hba.conf)  
нужно указать с каких адресов будут подключатся replica и backup  
```bash
host    all             all             192.168.27.152/32       md5
host    replication     all             192.168.27.151/32       md5
host	replication     all	        192.168.27.152/32       md5
```
[postgresql.conf](files/server/postgresql.conf)
включить режим репликации и подключение к barman  
```bash
listen_addresses = 'localhost,192.168.27.150'
wal_level = replica
archive_mode = on
archive_command = 'barman-wal-archive 192.168.27.152 server %p'
```
для демонстрации в СУБД загружается демо база postgrespro  
```bash
psql -f /tmp/demo_small.sql
```
создать пользователей replica и backup  
```bash
$createuser -s -P barman
$createuser -l --replication -P replica
```
создается слот репликации для replica  
```bash
psql -c "SELECT pg_create_physical_replication_slot('replica');"
```
### Конфиг replica
перед запуском репликации делается резервное копирование основного сервера  
```bash
pg_basebackup -h 192.168.27.150 -D /var/lib/pgsql/11/data -U replica
```
[recovery.conf](files/replica/recovery.conf) 
подключение к серверу  
```bash
standby_mode = 'on'
primary_conninfo = 'user=replica host=192.168.27.150'
primary_slot_name = 'replica'
```
[postgresql.conf](files/replica/postgresql.conf)  
```bash
listen_addresses = 'localhost,192.168.27.151'
hot_standby = on
```
### Конфиг backup
для работы barman выполняется обмен ключами с основным сервером  
в crontab создается задание для запуска barman  

[server.conf](files/backup/server.conf)  
подключение к основному серверу  
политика хранения резервных копий  
префикс пути к исполняемым файлам postrgesql 
```bash
[server]
description = "main server"
conninfo = host=192.168.27.150 user=barman dbname=demo
backup_method = postgres
streaming_conninfo = host=192.168.27.150 user=barman dbname=demo
streaming_archiver = on
slot_name = barman
create_slot = auto
archiver = on
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 1 MONTHS
wal_retention_policy = main
path_prefix="/usr/pgsql-11/bin"
```

## Проверка
### Основной сервер
```bash
[root@centos7 linux-dz-27]# vagrant ssh server
Last login: Wed May 13 08:44:30 2020 from 10.0.2.2
[vagrant@server ~]$ sudo -i
[root@server ~]# su postgres
bash-4.2$ psql
could not change directory to "/root": Permission denied
psql (11.7)
Type "help" for help.

postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 6838
usesysid         | 16385
usename          | replica
application_name | walreceiver
client_addr      | 192.168.27.151
client_hostname  | 
client_port      | 42336
backend_start    | 2020-05-13 08:44:20.85507+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/12001358
write_lsn        | 0/12001358
flush_lsn        | 0/12001358
replay_lsn       | 0/12001358
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
-[ RECORD 2 ]----+------------------------------
pid              | 7114
usesysid         | 16384
usename          | barman
application_name | barman_receive_wal
client_addr      | 192.168.27.152
client_hostname  | 
client_port      | 59732
backend_start    | 2020-05-13 08:45:03.530018+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/12001358
write_lsn        | 0/12001358
flush_lsn        | 0/12000000
replay_lsn       | 
write_lag        | 00:00:00.000539
flush_lag        | 00:12:02.798423
replay_lag       | 00:12:33.637873
sync_priority    | 0
sync_state       | async

postgres=# 
```

### Сервер резервного копирования
```bash
[root@backup ~]# barman check server
Server server:
	PostgreSQL: OK
	is_superuser: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: OK (no last_backup_maximum_age provided)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: OK (have 0 backups, expected at least 0)
	pg_basebackup: OK
	pg_basebackup compatible: OK
	pg_basebackup supports tablespaces mapping: OK
	systemid coherence: OK (no system Id stored on disk)
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archive_mode: OK
	archive_command: OK
	archiver errors: OK
```
