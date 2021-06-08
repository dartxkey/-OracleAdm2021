1. Создать отдельное табличное пространство UNDOTS_G (AUM)
a) Установить гарантированный UNDO_RETENTION 15 минут.
```sql
CREATE UNDO TABLESPACE UNDOTS_G DATAFILE 'undots_ga.dbf' SIZE 10M AUTOEXTEND ON RETENTION GUARANTEE;
ALTER SYSTEM SET UNDO_RETENTION = 900 SCOPE = BOTH;
```
2. Используя таблицу HR.EMPLOYEES (или созданную вами)
- изменить + удалить запись, использую Flashback Versions показать историю изменений
```sql
SELECT to_char(versions_starttime, 'HH24:MI:SS DD.MM.YYYY') START_TIME, to_char(versions_endtime, 'HH24:MI:SS DD.MM.YYYY') END_TIME, versions_operation OP, id, word FROM WORDS VERSIONS BETWEEN TIMESTAMP MINVALUE AND MAXVALUE ORDER BY versions_starttime;
START_TIME                   END_TIME                  OP  ID   WORD
---------------------------- ------------------------- --- ---- ------------------- 
16:56:55 23.04.2018                                    I   3    baba
16:56:55 23.04.2018          16:58:07 23.04.2018       I   2    abba
16:56:55 23.04.2018          16:58:07 23.04.2018       I   1    aaaa
16:58:07 23.04.2018                                    D   2    abba
16:58:07 23.04.2018                                    U   1    keks
```
- восстановить запись  через flasback query
```sql
SELECT * FROM words;
ID   WORD
---- --------------------
1    keks
3    baba

SELECT * FROM words AS OF TIMESTAMP TO_TIMESTAMP ('16:58:07 23.04.2018', 'HH24:MI:SS DD.MM.YYYY');
ID   WORD
---- --------------------
1    aaaa
1    abba
3    baba
```
- выполнить добавление записей в hr.departments
```sql
INSERT INTO words VALUES(4, 'pop');
INSERT INTO words VALUES(5, 'cop');
INSERT INTO words VALUES(6, 'pab');
COMMIT;

SELECT * FROM words;
ID   WORD
---- --------------------
1    keks
3    baba
4    pop
5    cop
6    pab
```
- используя Flashback Table сделать выборку из таблицы до вставки
```sql
FLASHBACK TABLE WORDS TO TIMESTAMP TO_TIMESTAMP ('17:20:05 23.04.2018', 'HH24:MI:SS DD.MM.YYYY');

SELECT * FROM words;
ID   WORD
---- --------------------
1    keks
3    baba
```
- откатить последнюю вставку в hr.departments через Flashback Transaction Backout
```sql
ALTER DATABSE ADD supplemental log data;

INSERT INTO words VALUES(4, 'pop');
COMMIT;

SELECT * FROM words;
ID   WORD
---- --------------------
1    keks
3    baba
4    pop

SELECT versions_xid XID, versions_startscn START_SCN, versions_endscn END_SCN, versions_operation OP, id, word FROM WORDS VERSIONS BETWEEN TIMESTAMP MINVALUE AND MAXVALUE ORDER BY versions_startscn;
Узнаем, что:
XID              START_SCN  END_SCN    OP  ID    WORD
---------------- ---------- ---------- --- ----- ------------------
06003E008E010000 998428                I   4     pop

ALTER SYSTEM switch logfile;

declare
    v_txid sys.xid_array;
begin
    v_txid := sys.xid_array('06003E008E010000'); 
    dbms_flashback.transaction_backout(1, v_txid, dbms_flashback.cascade);
end;
/

SELECT * FROM words;
ID   WORD
---- --------------------
1    keks
3    baba
```
3. Удостоверится, что БД рабоатет в ARCHIVELOG
```sql
SQL> SELECT LOG_MODE FROM SYS.V$DATABASE;
LOG_MODE
------------------------------------
ARCHIVELOG

SQL> archive log list;
Database log mode	       Archive Mode
Automatic archival	       Enabled
Archive destination	       /home/oracle/app/oracle/oradata/pana2/arch
Oldest online log sequence     86
Next log sequence to archive   88
Current log sequence	       88

/home/oracle/app/oracle/oradata/pana2/control01.ctl
```
4. Мультиплексировать Control Files До 4 экземпляров в разных локациях.
Скопируем Control Files из директории /home/oracle/app/oracle/oradata/pana2/ в другие локации. После чего в spfile добавим пути до копий:
```sql
control_files=("/home/oracle/app/oracle/oradata/pana2/control01.ctl", 
"/home/oracle/app/oracle/oradata/pana2/control02.ctl",
"/home/oracle/cntrl/control01.ctl",
"/home/oracle/cntrl/control02.ctl",
"/home/oracle/Music/control01.ctl",
"/home/oracle/Music/control02.ctl",
"/home/oracle/Documents/control01.ctl",
"/home/oracle/Documents/control02.ctl")
```
