
# Настройка postgres
Причины торможения postgres:
* 80% связано либо с кривыми запросами либо отсутствием индексов
* 20% не оптимально настроенный postgres
Настройка возможна:
* прописать их в запуске службы postgres -c name=value
* прописать в файлах с параметрами (действует постоянно)
* непосредственно в сесси (действует до конца транзакции/сессии)

## postgresql.conf
*посмотреть расположение*
```sql
show config_file
```
> расположение по умолчанию
```console
/var/lib/pgsql/data/postgresql.conf - Centos
/etc/postgresql/14/main/postgresql.conf - Ubuntu
/user/local/var/postgres/postgres.conf - MacOS
```
> полный путь передается аргументом к postmaster
```console
config_file=/etc/postgresql/14/main/postgresql.conf
```
> пути к остальным файлам с параметрами задаются в нем же
```console
hba_file = '/etc/postgresql/14/main/pg_hba.conf'
ident_file = '/etc/postgresql/14/main/pg_ident.conf'
```
## postgresql.auto.conf
*имеет выше приоритет чем postgresql.conf, изменения вносятся через alter system set*

## pg_settings
* источник информации о параметрах и значениях
* а так же откуда эти значения берутся
* и как могут изменяться и применяться
* расширенное представление show
```sql
select * from pg_settings;
```
```sql
alter system set max_connections=200;
```
```sql
select * from pg_settings where name=max_connections=200;
```
```sql
select pg_reload_conf();
```
* internal – изменить нельзя, задано при установке
* postmaster – перезапуск сервера
* sighup – повторное считывание файлов конфигурации •
* superuser – во время выполнения, только суперпользователь
* user – во время выполнения, любой пользователь
* backend - при запуске нового сеанса

## Варианты на что можно применить параметры
> на базу данных
```sql
alter database test set work_mem='128 MB';
```
> на пользователя
```sql
alter user test set work_mem='128 MB';
```
> на транзакцию
```sql
begin;
set work_mem = '128MB';
show work_mem;
rollback;
```
* Признак не правильной настройки work_mem переполненный каталог /var/lib/pgsql/14/base/pgsql_temp

## pg_file_settings
* отображение раскомментированных параметров
```sql
select * from pg_file_settings;
```
## Что делать если повредились wal файлы
* откат на последний примененный чекпоинт
```console
pg_resetwal -f /var/lib/pgsql/14/data/
```
***
## shared_buffers
***
Используется для кэширования данных. По умолчанию низкое значение (для поддержки как можно большего кол-ва ОС). Начать стоит с его изменения. Согласно документации, рекомендуемое значение для данного параметра - 25% от общей оперативной памяти на сервере. PostgreSQL использует 2 кэша - свой (изменяется **shared_buffers**) и ОС. Редко значение больше, чем 40% окажет влияние на производительность.
***
## max_connections
***
Максимальное количество соединений. Для изменения данного параметра придётся перезапускать сервер. Если планируется использование PostgreSQL как DWH, то большое количество соединений не нужно. Данный параметр тесно связан с **work_mem**. Поэтому будьте пределено аккуратны с ним
***
## effective_cache_size
***
Служит подсказкой для планировщика, сколько ОП у него в запасе. Можно определить как **shared_buffers** + ОП системы - ОП используемое самой ОС и другими приложениями. За счёт данного параметра планировщик может чаще использовать индексы, строить hash таблицы. Наиболее часто используемое значение 75% ОП от общей на сервере. 
***
## work_mem
***
Используется для сортировок, построения hash таблиц. Это позволяет выполнять данные операции в памяти, что гораздо быстрее обращения к диску. В рамках одного запроса данный параметр может быть использован множество раз. Если ваш запрос содержит 5 операций сортировки, то память, которая потребуется для его выполнения уже как минимум **work_mem** * 5. Т.к. скорее-всего на сервере вы не одни и сессий много, то каждая из них может использовать этот параметр по нескольку раз, поэтому не рекомендуется делать его слишком большим. Можно выставить небольшое значение для глобального параметра в конфиге и потом, в случае сложных запросов, менять этот параметр локально (для текущей сессии)
Сущестуют различные варианты определения этого параметра в качестве начального:
Total RAM * 0.25 / max_connections
work_mem = RAM/32..64

***
## maintenance_work_mem
***
Определяет максимальное количество ОП для операций типа VACUUM, CREATE INDEX, CREATE FOREIGN KEY. Увеличение этого параметра позволит быстрее выполнять эти операции. Не связано с **work_mem** поэтому можно ставить в разы больше, чем **work_mem**
***
## wal_buffers
***
Объём разделяемой памяти, который будет использоваться для буферизации данных WAL, ещё не записанных на диск. Если у вас большое количество одновременных подключений, увеличение параметра улучшит производительность. По умолчанию -1, определяется автоматически, как 1/32 от **shared_buffers**, но не больше, чем 16 МБ (в ручную можно задавать большие значения). Обычно ставят 16 МБ
***
## max_wal_size
***
Максимальный размер, до которого может вырастать WAL между автоматическими контрольными точками в WAL. Значение по умолчанию — 1 ГБ. Увеличение этого параметра может привести к увеличению времени, которое потребуется для восстановления после сбоя, но позволяет реже выполнять операцию сбрасывания на диск. Так же сбрасывание может выполниться и при достижении нужного времени, определённого параметром **checkpoint_timeout**
***
## checkpoint_timeout
***
Чем реже происходит сбрасывание, тем дольше будет восстановление БД после сбоя. Значение по умолчанию 5 минут, рекомендуемое - от 30 минут до часа. 
***
Необходимо "синхронизировать" два этих параметра. Для этого можно поставить **checkpoint_timeout** в выбранный промежуток, включить параметр **log_checkpoints** и по нему отследить, сколько было записано буферов. После чего подогнать параметр **max_wal_size**

## effective_io_concurrency
Допустимое число одновременных операций ввода/вывода. Для жестких дисков указывается по количеству шпинделей, для массивов RAID5/6 следует исключить диски четности. Для SATA SSD это значение рекомендуется указывать равным 200, а для быстрых NVMe дисков его можно увеличить до 500-1000. При этом следует понимать, что высокие значения в сочетании с медленными дисками сделают обратный эффект.

#Установка postgres в GCE
Посомтрим какая версия доступна из коробки
sudo apt-cache search postgresql | grep postgresql

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'И ставим postgres14
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add 
sudo apt -y update
sudo apt-cache search postgresql | grep postgresql
sudo apt -y install postgresql-14

Далее настраиваем listen adresses и pg_hba, делаем рестарт и проверяем через дата грип коннект (не забываем про vpc network -> firewalld)



Установка и настройка postgres в ЯО
Как воспользоваться пробным периодом
https://cloud.yandex.ru/docs/free-trial
Рассчитать стоимость
https://cloud.yandex.ru/prices
Yandex Managed Service for PostgreSQL
https://cloud.yandex.ru/docs/managed-postgresql
Установка Yandex.Cloud (CLI)
https://cloud.yandex.ru/docs/cli/quickstart

Непосредственно консоль
https://console.cloud.yandex.ru/folders/b1gm6apq5aonai4olued/compute/instances

Установка postgres
sudo yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&\
sudo yum -y update &&\
yum install -y postgresql14-server postgresql14 &&\
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb &&\
systemctl status postgresql-14
systemctl start postgresql-14 

Прописываем в .bash_profile строку
export PATH="/usr/pgsql-14/bin/:$PATH"

Далее настраиваем listen adresses и pg_hba, делаем рестарт и проверяем через дата грип коннект



#Посмотрим расположение конфиг файла через psql и idle
show config_file;


#Так же посмоттреть через функцию:
select current_setting('config_file');


#Далее смотрим структуру файла postgresql.conf (комменты, единицы измерения и т.д)
vi postgresql.conf

смотрим системное представление 
select * from pg_settings;


Далее рассмторим параметры которые требуют рестарт сервера

select * from pg_settings where context = 'postmaster';

И изменим параметры max_connections через конфиг файл и проверим;

select * from pg_settings where name='max_connections';

Смотрим pending_restart

select pg_reload_conf();


Смотрим по параметрам вьюху
select count(*) from pg_settings;
select unit, count(*) from pg_settings group by unit order by 2 desc;
select category, count(*) from pg_settings group by category order by 2 desc;
select context, count(*) from pg_settings group by context order by 2 desc;
select source, count(*) from pg_settings group by source order by 2 desc;

select * from pg_settings where source = 'override';


Переходим ко вью pg_file_settings;
select count(*) from pg_file_settings;
select sourcefile, count(*) from pg_file_settings group by sourcefile;

select * from pg_file_settings;

Далее пробуем преминить параметр с ошибкой, смотри что их этого получается
select * from pg_file_settings where name='work_mem';

Смотрим проблему с единицами измерения

select setting || ' x ' || coalesce(unit, 'units')
from pg_settings
where name = 'work_mem';

select setting || ' x ' || coalesce(unit, 'units')
from pg_settings
where name = 'max_connections';


Далее говорим о том как задать параметр с помощью alter system

alter system set work_mem = '16 MB';
select * from pg_file_settings where name='work_mem';

Сбросить параметр
ALTER SYSTEM RESET work_mem;


Далее говорим про set config в рамках транзакции

Установка параметров во время исполнения
Для изменения параметров во время сеанса можно использовать команду SET:

=> SET work_mem TO '24MB';
SET
Или функцию set_config:

=> SELECT set_config('work_mem', '32MB', false);
 set_config 
------------
 32MB
(1 row)

Третий параметр функции говорит о том, нужно ли устанавливать значение только для текущей транзакции (true)
или до конца работы сеанса (false). Это важно при работе приложения через пул соединений, когда в одном сеансе
могут выполняться транзакции разных пользователей.


И для конкретных пользователей и бд
create database test;
alter database test set work_mem='8 MB';

create user test with login password 'test';
alter user test set work_mem='16 MB';

SELECT coalesce(role.rolname, 'database wide') as role,
       coalesce(db.datname, 'cluster wide') as database,
       setconfig as what_changed
FROM pg_db_role_setting role_setting
LEFT JOIN pg_roles role ON role.oid = role_setting.setrole
LEFT JOIN pg_database db ON db.oid = role_setting.setdatabase;


Так же можно добавить свой параметр:


Далее превреям работу pgbench. Инициализируем необходимые нам таблицы в бд

/usr/pgsql-14/bin/pgbench -i test
/usr/pgsql-14/bin/pgbench -c 50 -j 2 -P 10 -T 30 test

/usr/pgsql-14/bin/pgbench -c 50 -C -j 2 -P 10 -T 30 -M extended test


Удалить wal и чтобы восстановить
pg_resetwal -f /var/lib/pgsql/14/data/


Далее генерируем необходимые параметры в pgtune
И вставляем их в папку conf.d заранее прописав ее в параметры


# Репликация postgres
Физическая репликация:
* Записи WAL передаются на реплику и применяются:
> поток данных только в одну сторону
> реплицируется кластер целиком, выборочная реплика невозможна
* Реплика точная копия мастера:
> одна и та же основаная версия сервера
> полностью совместимые архитектуры и платформы
* Реплика доступна только для чтения

Использование физической репликации:
* Горячий резерв для высокой доступности
* Балансировка OLTP-нагрузки
* Реплика для отчетов
* Несколько реплик и каскадная репликация
* Отложенная репликация

*Процесс wal sender передает файлы на процесс wal receiver на реплике гле применяються изменения*

Возможности реплики
* запросы на чтение данных (select, copy to, курсоры)
* установка параметров сервера (set, reset)
* управление транзакциями (begin, commit, rollback)
* создание резервной копии (pg_basebackup)

*При включенном параметре hot_standby_feedback реплика заставляет мастера ждать пока она вычитывает данные из таблицы и не дает запустить vacuum*

# На реплике нет процессов
* Wal writer
* Autovacuum

## Асинхронная репликация - последовательное выполнение транзакций на всех репликах.
* Хорошая пропускная способность
* Появляется время отлика в течении которого отдельные реплики могут быть фактически неидентичными.

```sql
synchronous_commit=off
```
## Синхронная репликация - коммит на реплике должен быть доступен до того, как он будет доступен на мастере.
* Плюсы - абсолютно идентисная копия
* Минусы - мастер ждет ответа от реплики об удачном коммите. В случае сбоя реплики работа мастера останавливается.

## synchronous_standby_names = (список реплик)

## synchronous_commit варианты
* remote_write - дожидаемся только ответа о том, что информация дошла до реплики, но не факт, что произошла запись на диск
* on - подтверждает, что произошла запись на диск в WAL файл
* remote_apply - дает подтверждение тому, что запись применена в базе

## Проверить применение wal файлов
*на мастере*
* Представление pg_current_wal_lsn()
* Остальные pg_stat_replication
> Реплика передает мастеру статус репликации при каждой записи на диск, но как минимум раз в wal_receiver_status_interval секунд (по умолчанию 10).
* Если используется слот репликации, то информацию о нем можно получить из представления pg_replication_slots

*на реплике*
* в представлении pg_stat_wal_receiver и с помощью функций pg_last_wal_receive_lsn() и pg_last_wal_replay_lsn()

## Параметры репликации
* max_worker_processes -  должно быть таким же или большим, чем на местере. Иначе нельзя будет делать запросы на реплику.
* wal_level - replica или logical в зависимости от типа репликации
* archive_mode - on для передачи wal файлов
* wal_receiver_status_interval - частота передачи сигнала от реплику мастеру (10 сек по умолчанию)
* wal_retrieve_retry_interval - время ожидания реплики, прежде чем повторить попытку получить WAL (5 сек по умолчанию)
* recovery_min_apply_delay - время задержки на восстановление (в случае истечения времени, реплика отваливается)
* wal_keep_segments = 32 (можно поставить 256)	Сколько хранить сегментов. Нужно количество надо выбрать такое, чтобы резервный сервер успевал все забирать и обрабатывать. Я поставил 256 чтобы за сутки wal-файлы не затирались
* hot_standby = on -	добавляет информацию, необходимую для запуска только для чтения запросов на резервный сервер
* checkpoint_segments = 3	checkpoint_segments = 16	Можно увеличить количество сегментов в WAL-логе

## Создание физической реплики
Начиная с версии 10, все необходимые настройки уже присутствуют по умолчанию: 
* wal_level = replica;
* max_wal_senders - число резервных серверов, которое может подключится к основному серверу
* разрешение на подключение в pg_hba.conf по протоколу репликации.
Создадим 2 кластер
$ pg_createcluster -d /var/lib/postgresql/12/main2 12 main2
Удалим оттуда файлы
$ rm -rf /var/lib/postgresql/12/main2
Сделаем бэкап нашей БД. Ключ -R создаст заготовку управляющего файла recovery.conf (запуск на вторичном сервере, если другой хост то –h)
$ pg_basebackup -p 5432 -R -D /var/lib/postgresql/12/main2
Зададим другой порт (до версии 10)
-- $ echo 'port = 5433' >> /var/lib/postgresql/10/main2/postgresql.auto.conf
Добавим параметр горячего резерва, чтобы реплика принимала запросы на чтение (до версии 10) --$ echo 'hot_standby = on' >> /var/lib/postgresql/10/main2/postgresql.auto.conf
Стартуем кластер
$ pg_ctlcluster 12 main2 start
Смотрим как стартовал
Настройки не копируются с primary
$ pg_lsclusters
* max_replication_slots - количество слотов

## Перевод реплики в состояние мастера
```console
pg_ctlcluster 12 main2 promote
```
> получим два независимых сервера

pg_lsclusters 	- наличие кластера Постгреса

sudo pg_ctlcluster 12 main start		- запуск Постгреса

sudo -u postgres psql	-d otus	- переход в оболочку psql


create database name;	создать базу
\c name			перейти в нее

create table student as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;

select * from student;

show wal_level;
alter system set wal_level = replica;

	/etc/postgresql/12/main/pg_hba.conf
host	replication		имя_пользователя	IP_адрес		md5

	/etc/postgresql/12/main/postgresql.conf
listen_addresses = 'localhost, IP_адрес'

Создадим 2 кластер
sudo pg_createcluster -d /var/lib/postgresql/12/main2 12 main2

	/etc/postgresql/12/main2/pg_hba.conf
host	replication		имя_пользователя	IP_адрес		md5

	/etc/postgresql/12/main2/postgresql.conf
listen_addresses = 'localhost, IP_адрес'

Удалим оттуда файлы
sudo rm -rf /var/lib/postgresql/12/main2

Сделаем бэкап нашей БД. Ключ -R создаст заготовку управляющего файла recovery.conf.
sudo -u postgres pg_basebackup -p 5432 -R -D /var/lib/postgresql/12/main2

Зададим другой порт (задан по умолчанию)
-- echo 'port = 5433' >> /var/lib/postgresql/10/main2/postgresql.auto.conf

Добавим параметр горячего резерва, чтобы реплика принимала запросы на чтение 
(задан по умолчанию ключом D)
--sudo echo 'hot_standby = on' >> /var/lib/postgresql/12/main2/postgresql.auto.conf

Стартуем кластер
sudo pg_ctlcluster 12 main2 start

Смотрим как стартовал
pg_lsclusters


Проверим состояние репликации:
на мастере
SELECT * FROM pg_stat_replication \gx
SELECT * FROM pg_current_wal_lsn();


sudo -u postgres psql -p 5434

на реплике
select * from pg_stat_wal_receiver \gx
select pg_last_wal_receive_lsn();
select pg_last_wal_replay_lsn();

Перевод в состояние мастера
sudo pg_ctlcluster 12 main2 promote

SELECT * FROM pg_stat_replication \gx


Логическая репликация
у первого сервера
ALTER SYSTEM SET wal_level = logical;

Рестартуем кластер
sudo pg_ctlcluster 12 main restart

sudo -u postgres psql -p 5432 -d otus
На первом сервере создаем публикацию:
CREATE PUBLICATION test_pub FOR TABLE student;

Просмотр созданной публикации
\dRp+

Задать пароль
\password 
pas123


sudo pg_ctlcluster 14 main start
sudo -u postgres psql -p 5433
select version();

создадим подписку на втором экземпляре
--создадим подписку к БД по Порту с Юзером и Паролем и Копированием данных=false
CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=localhost port=5432 user=postgres password=pas123 dbname=otus' 
PUBLICATION test_pub WITH (copy_data = true);

\dRs
SELECT * FROM pg_stat_subscription \gx


--добавим одинаковые данные
---добавить индекс
create unique index on student(id);
\dS+ test

добавить одиноковые значения
удалить на втором экземпляре конфликтные записи

SELECT * FROM pg_stat_subscription \gx	просмотр состояния (при конфликте пусто)

просмотр логов
$ tail /var/log/postgresql/postgresql-14-main.log


drop publication test_pub;
drop subscription test_sub;

Удаление кластера
sudo pg_dropcluster 12 main2

----------


## Логическая репликация
* wal_level = logical;
* может быть двухнаправленной
* нет мастера и реплики
* на сервере создается публикация, на которую другие будут серверы могут подписываться
* подписчику передается информация об изменениях строк в таблицах
* публикующий сервер выдает изменения данных построчно в порядке их фиксации (реплицируются команды insert, update, delete), в 11 версии и TRUNCATE
* возможна начальная синхронизация
* DDL команды не передаются, таблица приемники нужно создавать вручную
* применение происходит без применения команд SQL
* возможны конфликты с локальными данными
* назначение: консолидация и общие спровочиник, обновление основной версии сервера, мастер-мастер в будущем

## изменяем тип репликации на логическую

```console
postgres=# show wal_level;
 wal_level
-----------
 replica
(1 row)

postgres=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM
postgres=# \q
root@admekb:~# pg_ctlcluster 14 main restart
root@admekb:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.5 (Ubuntu 14.5-2.pgdg22.04+2))
Type "help" for help.

postgres=# show wal_level;
 wal_level
-----------
 logical
(1 row)
```
> полный путь передается аргументом к postmaster
```console
config_file=/etc/postgresql/14/main/postgresql.conf
```
> пути к остальным файлам с параметрами задаются в нем же
```console
hba_file = '/etc/postgresql/14/main/pg_hba.conf'
ident_file = '/etc/postgresql/14/main/pg_ident.conf'
```
