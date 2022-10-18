
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
## посмотреть текущий уровень изоляции

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

> пример применения на транзакцию
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

```bash
iso=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```

## сделать select * from persons во второй сессии

```bash
iso=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
## видите ли вы новую запись и если да то почему?
- новую запись не видно
> текущий уровень изоляции read committed предпологает, что оператор может видеть только строки, зафиксированные до его начала.

## завершить первую транзакцию - commit;

```bash
iso=*# commit;
COMMIT
```
## сделать select * from persons во второй сессии

```bash
iso=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

## видите ли вы новую запись и если да то почему?
- новую запись видно
> неповторяющееся чтение (non-repeatable read). транзакция T1 изменила строку и зафиксировала изменения (COMMIT). При повторном чтении этой же строки транзакция T2 видит, что строка изменена.

## начать новые но уже repeatable read транзации

```bash
iso=# set transaction isolation level repeatable read;
SET
```

## в первой сессии добавить новую запись

```bash
insert into persons(first_name, second_name) values('sveta', 'svetova');
```

## сделать select * from persons во второй сессии

```bash
iso=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

## видите ли вы новую запись и если да то почему?
- новую запись не видно
> текущий уровень изоляции repeatable read предпологает, что оператор не увидит изменения не до не после commit в рамках текущей транзакции

## завершить первую транзакцию - commit;

```bash
iso=*# commit;
COMMIT
```

## сделать select * from persons во второй сессии
- новую запись не видно
> текущий уровень изоляции repeatable read предпологает, что все операторы текущей транзакции могут видеть только строки, зафиксированные до выполнения первого запроса или инструкции об изменение данных в этой транзакции.

```bash
iso=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
