1. Включить стандартный, детализированный аудит Oracle, запись в базу.
Изменить в файле init.ora для соответствующей БД параметр audit_trail:
```sql
audit_trail=db,extended
SQL> show parameters audit_trail;
NAME                   TYPE              VALUE
---------------------- ----------------- ---------------------
audit_trail            string            DB, EXTENDED
```
2. Для пользователя "владелец приложения" из Lab2 аудировать:
а) все действия по созданию/изменению триггеров и представлений БД. Каждое изменение - отдельной записью.
```sql
SQL> AUDIT CREATE ANY TRIGGER BY APP_OWNER BY ACCESS;
Audit succeeded.

SQL> AUDIT ALTER ANY TRIGGER BY APP_OWNER BY ACCESS; 
Audit succeeded.

SQL> AUDIT DROP ANY TRIGGER BY APP_OWNER BY ACCESS; 
Audit succeeded.

SQL> AUDIT CREATE ANY VIEW BY APP_OWNER BY ACCESS;
Audit succeeded.

SQL> AUDIT DROP ANY VIEW BY APP_OWNER BY ACCESS; 
Audit succeeded.
```

б) фиксировать только неудачные попытки удаления из таблиц вашим пользователем. Одна запись на сессию.
```sql
AUDIT DELETE ANY TABLE BY APP_OWNER BY SESSION WHENEVER NOT SUCCESSFUL;
```
в) продеменострировать содержимое журнала аудита для стандартного аудита.
```sql
SELECT TO_CHAR(timestamp, 'HH24:MI:SS DD.MM.YYYY') time_stamp, username, action_name FROM dba_audit_trail ORDER BY timestamp;
```

3. VPD:
а) на таблицу протоколирования входов пользователя в БД (лаб 1.) переписывать запрос так, чтобы показывались только записи по тек. пользователю
```sql
CREATE TABLE MY_LOGON_TABLE (LOGON_TIME DATE, USER_NAME VARCHAR2(100));

CREATE OR REPLACE TRIGGER MY_LOGON_TRIGGER
    AFTER LOGON ON DATABASE
BEGIN
    INSERT INTO MY_LOGON_TABLE (LOGON_TIME, USER_NAME) VALUES (SYSDATE, USER);
END;

CREATE VIEW MY_LOGON_VIEW 
    AS (SELECT TO_CHAR(LOGON_TIME, 'HH24:MI:SS DD.MM.YYYY') "LOGON_TIME"
    FROM MY_LOGON_TABLE 
    WHERE USER_NAME = USER);
GRANT SELECT ON MY_LOGON_VIEW TO PUBLIC;
```
4. FGA  на HR.SALARIES
а) изменение зарплаты более чем на 5%.
```sql
begin
DBMS_FGA.ADD_POLICY(
object_schema => 'hr',
object_name   => 'employees',
policy_name   => 'chk_hr_sal',
audit_condition => ':new.salary > :old.salary * 1.05', 
audit_column => 'salary',
statement_types => 'update');
end;
/
```                                                                                                                                 
б) запрос фамилии,зарплаты по сотрудникам deptno (На выбор любой).
```sql
begin
DBMS_FGA.ADD_POLICY(
object_schema => 'hr',
object_name   => 'employees',
policy_name   => 'chk_hr_dep',
audit_condition => 'department_id = 10', 
audit_column => 'last_name,salary',
statement_types => 'select');    
end;
/         
```
в) продеменострировать содержимое журнала аудита детального аудита.
```sql
SELECT * FROM dba_fga_audit_trail;
```
5. Отчет по всем операциям в журналах аудита по выбранному пользователю за период. (sql запрос с параметром: дней истории от тек.даты)

Включаем вывод DBMS_OUTPUT:

```sql
SET SERVEROUTPUT ON;
```
```sql
CREATE OR REPLACE PROCEDURE GET_AUDIT_FOR_USER(USER IN VARCHAR2, DAYS_COUNT IN NUMBER) AS
  BEGIN
    FOR row IN (
    SELECT *
    FROM DBA_COMMON_AUDIT_TRAIL
    WHERE DB_USER = GET_AUDIT_FOR_USER.USER AND EXTENDED_TIMESTAMP > (
      SELECT (SYSDATE - GET_AUDIT_FOR_USER.DAYS_COUNT)
      FROM dual
    ))
    LOOP
      DBMS_OUTPUT.PUT_LINE('' || row.DB_USER || ' - ' || ROW.STATEMENT_TYPE || ' - ' || ROW.EXTENDED_TIMESTAMP);
    END LOOP;
  END GET_AUDIT_FOR_USER;
/
```
