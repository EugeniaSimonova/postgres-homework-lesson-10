# Блокировки

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.


В скрипте урока приводится параметр `deadlock_timeout` ("Другой способ состоит в том, чтобы включить параметр log_lock_waits. В этом случае в журнал сообщений сервера будет попадать информация, если транзакция ждала дольше, чем deadlock_timeout (несмотря на то, что используется параметр для взаимоблокировок, речь идет об обычных ожиданиях"). Я начала с того, что выставила настройку `log_min_duration_statement` и получила такой же результат, блокировки оказались в логах.
```
ALTER SYSTEM SET log_lock_waits = on;

ALTER SYSTEM SET deadlock_timeout = 200; 
SELECT pg_reload_conf();

# session1

BEGIN;
INSERT INTO accounts VALUES (14,4000.00), (15,2000.00), (16,3000.00);

# начать вторую сессию и подождать пока блокировка попадёт в журнал

COMMIT;

# session2

BEGIN;
ALTER TABLE accounts ADD COLUMN status varchar(30) DEFAULT 'old';
COMMIT;

```

```
postgres@ees-09:~$ tail -n 5 /var/log/postgresql/postgresql-14-main.log
#...
2023-03-05 08:22:25.843 UTC [58238] postgres@postgres LOG:  process 58238 acquired AccessExclusiveLock on relation 16393 of database 13761 after 53348.496 ms
2023-03-05 08:22:25.843 UTC [58238] postgres@postgres STATEMENT:  ALTER TABLE accounts ADD COLUMN status varchar(30) DEFAULT 'old';

```

2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

```
# session1

BEGIN;

UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 1;

SELECT txid_current(), pg_backend_pid();

 txid_current | pg_backend_pid 
--------------+----------------
          753 |           1367

```

```
# session2

BEGIN;

SELECT txid_current(), pg_backend_pid();

 txid_current | pg_backend_pid 
--------------+----------------
          754 |           1466

UPDATE accounts SET amount = amount + 110.00 WHERE acc_no = 1;

```

```
# session3

BEGIN;

SELECT txid_current(), pg_backend_pid();

 txid_current | pg_backend_pid 
--------------+----------------
          755 |           1723


UPDATE accounts SET amount = amount - 50.00 WHERE acc_no = 1;

```
```
 pid  |   locktype    |   lockid   |       mode       | granted 
------+---------------+------------+------------------+---------
 1723 | relation      | accounts   | RowExclusiveLock | t   -- Построчная блокировка таблицы третьей транзакцией
 1466 | relation      | accounts   | RowExclusiveLock | t   -- Построчная блокировка таблицы второй транзакцией
 1367 | relation      | accounts   | RowExclusiveLock | t   -- Построчная блокировка таблицы первой транзакцией
 1466 | transactionid | 753        | ShareLock        | f   -- Блокировка ShareLock, запрашиваемая второй транзакцией, но так как первая транзакция наложила блокировку ExclusiveLock, эта not granted
 1466 | transactionid | 754        | ExclusiveLock    | t   -- Блокировка ExclusiveLock второй транзакции, которая ждет завершения первой
 1723 | tuple         | accounts:6 | ExclusiveLock    | f   -- Третья транзакция ссылается на tuple второй 
 1723 | transactionid | 756        | ExclusiveLock    | t   -- Блокировка ExclusiveLock третьей транзакции, которая ждет завершения второй
 1367 | transactionid | 753        | ExclusiveLock    | t   -- Установлена эксклюзивная блокировка транзакцией первой сессии
 1466 | tuple         | accounts:6 | ExclusiveLock    | t   -- tuple второй транзакции, который ждет завершения первой




SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
FROM pg_locks WHERE relation = 'accounts'::regclass;

 locktype |       mode       | granted | pid  | wait_for 
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 1723 | {1466} // третья транзакция ждет завершения второй
 relation | RowExclusiveLock | t       | 1466 | {1367} // вторая транзакция ждет завершения первой
 relation | RowExclusiveLock | t       | 1367 | {}     // первая транзакция наложила блокировку RowExclusiveLock на строку и выполняется
 tuple    | ExclusiveLock    | f       | 1723 | {1466}  // tuple второй транзакции на который ссылается третья
 tuple    | ExclusiveLock    | t       | 1466 | {1367}  // tuple, созданный второй транзакцией, ждет завершения первой транзакции
 

```



3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?


```
# session1
begin;
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 1;
UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
```

```
# session2
begin;
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;
UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 3;

```

```
# session3
begin;
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 3;
UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;
```

Разобраться можно, операции внесены в журнал

```
2023-03-05 13:29:38.702 UTC [1367] postgres@postgres ERROR:  deadlock detected
2023-03-05 13:29:38.702 UTC [1367] postgres@postgres DETAIL:  Process 1367 waits for ShareLock on transaction 761; blocked by process 1466.
	Process 1466 waits for ShareLock on transaction 762; blocked by process 1723.
	Process 1723 waits for ShareLock on transaction 760; blocked by process 1367.
	Process 1367: UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
	Process 1466: UPDATE accounts SET amount = amount + 20.00 WHERE acc_no = 3;
	Process 1723: UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;
2023-03-05 13:29:38.702 UTC [1367] postgres@postgres HINT:  See server log for query details.
2023-03-05 13:29:38.702 UTC [1367] postgres@postgres CONTEXT:  while updating tuple (0,21) in relation "accounts"
2023-03-05 13:29:38.702 UTC [1367] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
2023-03-05 13:30:30.851 UTC [1466] postgres@postgres LOG:  process 1466 still waiting for ShareLock on transaction 762 after 120000.120 ms
2023-03-05 13:30:30.851 UTC [1466] postgres@postgres DETAIL:  Process holding the lock: 1723. Wait queue: 1466.
2023-03-05 13:30:30.851 UTC [1466] postgres@postgres CONTEXT:  while updating tuple (0,20) in relation "accounts"
2023-03-05 13:30:30.851 UTC [1466] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 20.00 WHERE acc_no = 3;
```


4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

да
