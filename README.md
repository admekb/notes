
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
