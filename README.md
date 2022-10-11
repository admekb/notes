
# Работа с уровнями изоляции транзакции в PostgreSQL

## сделать в первой сессии новую таблицу и наполнить ее данными

```bash
iso=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
iso=*# insert into persons(first_name, second_name) values('ivan', 'ivanov');
INSERT 0 1
iso=*# insert into persons(first_name, second_name) values('petr', 'petrov');
INSERT 0 1
iso=*# commit;
```

## посмотреть текущий уровень изоляции

```bash
iso=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```

## в первой сессии добавить новую запись

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
