3. Шаблон БД и скрипты создания

[Скрипты](scripts.zip)

4. Вывод из словаря данных V$:
![Вывод V](https://user-images.githubusercontent.com/70812795/121181911-f3198280-c87b-11eb-82c2-7bd8087628a1.png)

![image](https://user-images.githubusercontent.com/70812795/121182028-180df580-c87c-11eb-9cad-97cea49add33.png)

5. Создаем SPFILE 

[SPFILE](spfileoracleadm.ora)

6. Изменяем параметр системы:
```SQL
SQL> ALTER SYSTEM SET open_cursors=400 SCOPE=MEMORY;
```

![image](https://user-images.githubusercontent.com/70812795/121182450-881c7b80-c87c-11eb-8818-bc0f42c8a1ae.png)

7. Варианты монтирования и остановки БД:
```SQL
SQL> SHUTDOWN NORMAL;
SQL> STARTUP MOUNT;
SQL> ALTER DATABASE OPEN READ ONLY;
```
```SQL
SQL> ALTER SYSTEM ENABLE RESTRICTED SESSION;
SQL> ALTER SYSTEM DISABLE RESTRICTED SESSION;
```
```SQL
SQL> SHUTDOWN NORMAL;
SQL> SHUTDOWN TRANSACTIONAL;
SQL> SHUTDOWN IMMEDIATE;


SQL> SHUTDOWN ABORT;
SQL> STARTUP MOUNT;
SQL> RECOVER DATABASE;
```

8. В SUSPEND и обратно

![Suspend](https://user-images.githubusercontent.com/70812795/121182902-f3fee400-c87c-11eb-9b05-a57c9da8da9c.png)

9. Изменение параметров
```SQL
SQL> alter system set optimizer_mode = rule scope=spfile;
select value from v$parameter where name='optimizer_mode'
union all
select value from v$spparameter where name='optimizer_mode';
```

![image](https://user-images.githubusercontent.com/70812795/121183271-53f58a80-c87d-11eb-845b-41dbc9095047.png)

Поскольку значение измененного параметра не было сохранено в spfile, они действительны только для текущего экземпляра.

10 - 12. Создать профили, создать пользователей и применить к ним профили:
```SQL
/* a.	владелец приложения: без квоты на создание объектов, без ограничений по времени сессии,  количеству сессий, роли для просмотра словаря данных и динамических представлений */
SQL> CREATE PROFILE app_owner_prof LIMIT SESSIONS_PER_USER UNLIMITED CONNECT_TIME UNLIMITED;
SQL> CREATE USER APP_OWNER IDENTIFIED BY 12345678;
SQL> ALTER USER APP_OWNER PROFILE app_owner_prof;
SQL> GRANT UNLIMITED TABLESPACE TO APP_OWNER;
SQL> GRANT SELECT_CATALOG_ROLE TO APP_OWNER;
```

```SQL
/* b.	ограниченный: квота 50М, 15 мин простоя сессии, макс 2 сессии, без доступа к словарю данных */
SQL> CREATE PROFILE limited_prof LIMIT SESSIONS_PER_USER 2 IDLE_TIME 900;
SQL> CREATE USER LIMITED IDENTIFIED BY 12345678;
SQL> ALTER USER LIMITED PROFILE limited_prof;
SQL> ALTER USER LIMITED QUOTA 50M ON USERS;
```

12. продемонстрировать пользователя в dba_users, выборка параметров профиля для пользователя

![image](https://user-images.githubusercontent.com/70812795/121183605-b2bb0400-c87d-11eb-909f-f29cff35a2ac.png)

![image](https://user-images.githubusercontent.com/70812795/121183682-c6666a80-c87d-11eb-9e36-14aca2ddbf16.png)

13. Установить профиль b) как значение По-умолчанию для всех вновь создаваемых пользователей
```SQL
SQL> ALTER PROFILE DEFAULT LIMIT SESSIONS_PER_USER 2 IDLE_TIME 900;
```

![Profile](https://i.imgur.com/6rp6Nql.png)

14. Создать  нового пользователя, убедиться, что профиль для него подключен, назначенный в п.13.

```SQL
SQL> CREATE USER TEST_USER IDENTIFIED BY 12345678;
SQL> SELECT profile FROM dba_users WHERE username='TEST_USER';
PROFILE
--------------------------------------------------------------------------------
DEFAULT
```
