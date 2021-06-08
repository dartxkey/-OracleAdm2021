1. Включить стандартный, детализированный аудит Oracle, запись в базу.
Изменить в файле init.ora для соответствующей БД параметр audit_trail:
audit_trail=db,extended
SQL> show parameters audit_trail;
NAME                   TYPE              VALUE
---------------------- ----------------- ---------------------
audit_trail            string            DB, EXTENDED

2. Для пользователя "владелец приложения" из Lab2 аудировать:
а) все действия по созданию/изменению триггеров и представлений БД. Каждое изменение - отдельной записью.
AUDIT CREATE ANY TRIGGER BY APP_OWNER BY ACCESS;
AUDIT ALTER ANY TRIGGER BY APP_OWNER BY ACCESS;
AUDIT DROP ANY TRIGGER BY APP_OWNER BY ACCESS;
AUDIT CREATE ANY VIEW BY APP_OWNER BY ACCESS;
AUDIT DROP ANY VIEW BY APP_OWNER BY ACCESS;

б) фиксировать только неудачные попытки удаления из таблиц вашим пользователем. Одна запись на сессию.
AUDIT DELETE ANY TABLE BY APP_OWNER BY SESSION WHENEVER NOT SUCCESSFUL;

в) продеменострировать содержимое журнала аудита для стандартного аудита.
SELECT TO_CHAR(timestamp, 'HH24:MI:SS DD.MM.YYYY') time_stamp, username, action_name FROM dba_audit_trail ORDER BY timestamp;

3. VPD:
а) на таблицу протоколирования входов пользователя в БД (лаб 1.) переписывать запрос так, чтобы показывались только записи по тек. пользователю
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

4. FGA  на HR.SALARIES
а) изменение зарплаты более чем на 5%.
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
                                                                                                                                 
б) запрос фамилии,зарплаты по сотрудникам deptno (На выбор любой).
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

в) продеменострировать содержимое журнала аудита детального аудита.
SELECT TO_CHAR(timestamp, 'HH24:MI:SS DD.MM.YYYY') time_stamp, db_user, os_user, object_schema, object_name, sql_text FROM dba_fga_audit_trail ORDER BY timestamp;
