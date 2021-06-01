## Блокировки
### Разрешить блокировку 2мя способами:
Создаем блокировку (далее SQL1 - первая сессия, SQL2 - вторая сессия):
```sql
SQL> select * from practice3_block;

	ID	VALUE
---------- ----------
	 1	    1
	 2	    2
```
0) Создание блокировки:
```sql
SQL1> update practice3_block set value=3 where id=1;

1 row updated.
```

SQL2 зависает при выполнении команды:

![SQL2](1.png "SQL2")

1) Зафиксировать/откатить удерживающую транзакцию:
```sql
SQL1> commit;

Commit complete.

SQL1> select * from practice3_block;

	ID	VALUE
---------- ----------
	 1	    3
	 2	    2
```

либо

```sql
SQL1> rollback;

Rollback complete.

SQL2> select * from practice3_block;

	ID	VALUE
---------- ----------
	 1	    4
	 2	    2
```

2) Убить сессию:
Откатываем изменения таблицы
```sql
SQL> update practice3_block set value=1 where id=1;

1 row updated.

SQL> commit;

Commit complete.

```
И повторяем пункт 0 для создания блокировки


Получаем данные о сессии:
```sql
SQL1> SELECT s.inst_id, s.sid, s.serial# FROM gv$session s WHERE s.type != 'BACKGROUND';

   INST_ID	  SID	 SERIAL#
---------- ---------- ----------
	 1	   23	      15
	 1	  253	      75
```

Узнаем свой SID:
```sql
SQL1> SELECT DISTINCT sid FROM v$mystat;

       SID
----------
	23
```

Убиваем другую сессию:
```sql
SQL1> ALTER SYSTEM KILL SESSION '253,75';

System altered.
```

У SQL2 появляется ошибка:
```sql
SQL2> update practice3_block set value=4 where id =1;
update practice3_block set value=4 where id =1
       *
ERROR at line 1:
ORA-00028: your session has been killed
ORA-00028: your session has been killed
```

## Создать Deadlock на таблице:
1. Создать таблицу произвольной структуры с одним PRIMARY KEY
```sql
SQL> CREATE TABLE practice3_deadlock (id int not null, value int, CONSTRAINT p3dl_id_pk PRIMARY KEY (id));

Table created.

SQL> INSERT INTO practice3_deadlock VALUES (1,1);

1 row created.

SQL> INSERT INTO practice3_deadlock VALUES (2,2);

1 row created.

SQL> commit;

Commit complete.
```
2. Изменять данные в таблице из 2 параллельных сессий для получения deadlock
```sql
SQL1> UPDATE practice3_deadlock SET value=3 WHERE id=1;

1 row updated.

SQL1> UPDATE practice3_deadlock SET value=3 WHERE id=2;    

```

```sql
SQL2> UPDATE practice3_deadlock SET value=3 WHERE id=2;            

1 row updated.


SQL2> UPDATE practice3_deadlock SET value=4 WHERE id=1;

```
3. Продемонстрировать разрешение deadlock для одной из сессий (transaction fail)
```sql
SQL> UPDATE practice3_deadlock SET value=3 WHERE id=2;    
UPDATE practice3_deadlock SET value=3 WHERE id=2
                              *
ERROR at line 1:
ORA-00060: deadlock detected while waiting for resource
```
4. Выдержу из alert.log вашей системы

[oracleadm_ora_19348.trc](oracleadm_ora_19348.trc)

## Управление сегментами отмены
1. Совершить длительную транзакцию (10000 записей и более) и проанализировать статистику отмены (V$UNDOSTAT): количество использованных блоков сегментов Undo, максимальная длительность запросов.
```sql
SQL> CREATE TABLE practice3_undo(id int not null, value varchar(20), CONSTRAINT p3u_id_pk PRIMARY KEY (id));

Table created.

SQL> DECLARE
  2    rand_str varchar(20);
  3  BEGIN
  4    FOR i IN 1..100000 LOOP
  5      SELECT dbms_random.string('L', 18) INTO rand_str FROM dual;
  6      INSERT INTO practice3_undo VALUES (i, rand_str);
  7    END LOOP;
  8    COMMIT;
  9  END;
 10  
 11  /


PL/SQL procedure successfully completed.

SQL> SELECT TO_CHAR(BEGIN_TIME, 'HH24:MI:SS') "BEGIN_TIME", maxquerylen, undoblks FROM V$UNDOSTAT ORDER BY begin_time;

BEGIN_TI MAXQUERYLEN   UNDOBLKS
-------- ----------- ----------
15:38:33	 215	   1826
```

2. С использованием 1) вычислить размер табличного пространства отмены для поддержки 1-часового undo retention interval
Undo Size = Optimal Undo Retention * DB_BLOCK_SIZE * UNDO_BLOCK_REP_ESC = 60*60 * 16384 * 8.53 = 480 Мб
3. Продемонстрировать настроенные параметры для UNDO, аттрибуты табличного пространства для UNDO, установленные по-умолчанию для вашей системы
```sql
SQL> SHOW PARAMETER UNDO;

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
undo_management 		     string	 AUTO
undo_retention			     integer	 900
undo_tablespace 		     string	 UNDOTBS1

SQL> SELECT NAME, INCLUDED_IN_DATABASE_BACKUP, FLASHBACK_ON, BIGFILE FROM V$TABLESPACE WHERE NAME='UNDOTBS1';

NAME			       INC FLA BIG
------------------------------ --- --- ---
UNDOTBS1		       YES YES NO
```
4. Изменить настройки табличного пространства отмены для поддержки 1-часового гарантированного интервала хранения
/oracle/db/oradata/oracleadm/oracleadm.dbf
```sql
SQL> CREATE UNDO TABLESPACE undotbs2 DATAFILE '/oracle/db/oradata/oracleadm/oracleadm.dbf' SIZE 500M AUTOEXTEND ON NEXT 5M;

Tablespace created.

SQL> ALTER SYSTEM SET UNDO_TABLESPACE=UNDOTBS2 SCOPE=BOTH;

System altered.

SQL> ALTER SYSTEM SET UNDO_RETENTION=3600 SCOPE=BOTH;

System altered.
```
