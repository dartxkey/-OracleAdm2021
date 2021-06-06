1. Включить стандартный, детализированный аудит Oracle, запись в базу.

В файле init.ora выставляем параметр audit_trail=db,extended
```SQL
SQL> show parameters audit_trail;

NAME                   TYPE              VALUE
---------------------- ------------	     ---------------------
audit_trail            string            DB, EXTENDED
```
2. Для пользователя "владелец приложения" из Lab2 аудировать:

- все действия по созданию/изменению триггеров и представлений БД. Каждое изменение - отдельной записью.
```SQL
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
- фиксировать только неудачные попытки удаления из таблиц вашим пользователем. Одна запись на сессию.
```SQL
SQL> AUDIT DELETE ANY TABLE BY APP_OWNER BY SESSION WHENEVER NOT SUCCESSFUL;

Audit succeeded.
```
- sql запрос: показать содержимое журнала аудита для стандартного аудита ( подсказка: надо совершить действия, на которые вы этот аудит поставили)

Создадим пользователем APP_OWNER представление таблицы
```SQL
SQL> CREATE VIEW test_table_ids AS SELECT id FROM test_table;

View created
```

```SQL
SQL> SELECT username, action_name FROM dba_audit_trail WHERE username = 'APP_OWNER';

USERNAME                 ACTION_NAME
------------------------ -----------------------
APP_OWNER                CREATE VIEW
```

