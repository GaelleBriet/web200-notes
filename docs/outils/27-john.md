# John the Ripper — Cracking de Hashes

## 1. Concept

John the Ripper (john) est un outil de cracking de hashes orienté **CPU**, complémentaire de hashcat. Il excelle sur les hashes Unix, les fichiers protégés (ZIP, PDF, SSH...) et **identifie automatiquement** le type de hash dans la plupart des cas.

**Différence clé avec hashcat :**
- John → CPU, auto-détection du type, outils `*2john` pour extraire des hashes depuis des fichiers
- Hashcat → GPU, plus rapide sur les gros volumes, syntaxe de masque plus puissante

---

## 2. Syntaxe de base

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## 3. Commandes essentielles

### Attaque dictionnaire

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### Forcer un format spécifique

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=FORMAT
```

### Afficher les mots de passe crackés

```bash
john hash.txt --show
# ou avec format
john hash.txt --show --format=FORMAT
```

### Lister les formats supportés

```bash
john --list=formats
john --list=formats | grep -i md5
john --list=formats | grep -i jwt
```

### Avec règles (amplifier la wordlist)

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules=Jumbo
```

### Brute-force pur (incrémental)

```bash
john hash.txt --incremental
```

---

## 4. Formats courants (--format)

| Format        | Type de hash                      |
|---------------|-----------------------------------|
| `raw-md5`     | MD5 simple                        |
| `raw-sha1`    | SHA1 simple                       |
| `raw-sha256`  | SHA256 simple                     |
| `phpass`      | WordPress `$P$` / phpBB           |
| `sha512crypt` | Linux `/etc/shadow` `$6$`         |
| `md5crypt`    | Linux `/etc/shadow` `$1$`         |
| `bcrypt`      | bcrypt `$2y$`                     |
| `HMAC-SHA256` | JWT HS256                         |
| `NT`          | Hash NTLM Windows                 |
| `zip`         | Archives ZIP protégées            |
| `rar`         | Archives RAR protégées            |
| `ssh`         | Clés SSH protégées par passphrase |
| `pdf`         | PDF protégés                      |

---

## 5. Cas pratiques OSWA

### Hash MD5 simple

```bash
echo '5f4dcc3b5aa765d61d8327deb882cf99' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5
```

### Hash WordPress `$P$` (récupéré via SQLi)

```bash
echo '$P$Ghxmchgk.0YxEQutC7os3dZfBvqGIz/' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=phpass
```

### Hash `/etc/shadow` (auto-détection)

```bash
# Copier la ligne complète de /etc/shadow
echo 'root:$6$salt$longhashvalue...:18000:0:99999:7:::' > shadow.txt
john shadow.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### Cracker les deux fichiers passwd + shadow ensemble

```bash
# Unshadow : fusionner /etc/passwd et /etc/shadow
unshadow passwd.txt shadow.txt > combined.txt
john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### JWT HS256 (clé secrète)

```bash
echo 'eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiZ3Vlc3QifQ.SIGNATURE' > jwt.txt
john jwt.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256
```

---

## 6. Les outils `*2john` — Extraire des hashes depuis des fichiers

John fournit une suite d'utilitaires pour extraire le hash d'un fichier protégé et le mettre dans un format crackable.

### SSH — Clé privée protégée par passphrase

```bash
# Extraire le hash de la clé
ssh2john id_rsa > id_rsa.hash

# Cracker
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

### ZIP — Archive protégée

```bash
zip2john archive.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

### RAR — Archive protégée

```bash
rar2john archive.rar > rar.hash
john rar.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

### PDF — Fichier PDF protégé

```bash
pdf2john document.pdf > pdf.hash
john pdf.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

### Hash Windows depuis fichier NTDS/SAM

```bash
# Si ntds.dit ou SAM disponible
secretsdump.py → produit des hashes
john ntlm.hash --wordlist=/usr/share/wordlists/rockyou.txt --format=NT
```

---

## 7. Options utiles

| Option            | Description                             |
|-------------------|-----------------------------------------|
| `--show`          | Afficher les mots de passe déjà crackés |
| `--format=FORMAT` | Forcer le type de hash                  |
| `--rules`         | Appliquer des règles de mutation        |
| `--incremental`   | Brute-force pur                         |
| `--wordlist=FILE` | Wordlist personnalisée                  |
| `--list=formats`  | Lister tous les formats supportés       |
| `--pot=FILE`      | Fichier pot personnalisé (résultats)    |
| `--session=NOM`   | Nommer la session (reprendre plus tard) |
| `--restore=NOM`   | Reprendre une session interrompue       |

---

## 8. Hashcat vs John — Quand utiliser lequel ?

| Situation                  | Outil recommandé              |
|----------------------------|-------------------------------|
| Hash Unix (`$6$`, `$1$`)   | John (auto-détection)         |
| Hash WordPress `$P$`       | Les deux (john = plus simple) |
| JWT HS256                  | Hashcat (`-m 16500`)          |
| Clé SSH protégée           | **John** (`ssh2john`)         |
| ZIP/RAR/PDF protégés       | **John** (`zip2john`, etc.)   |
| Gros volume, GPU dispo     | Hashcat                       |
| Identification rapide type | John (auto) ou `hashid`       |
| Masques complexes          | Hashcat                       |

---

## 9. Workflow exam

```
1. Récupérer le hash ou le fichier protégé

2a. Si hash brut (MD5, SHA, $P$, $6$...) :
    echo 'HASH' > hash.txt
    john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

2b. Si fichier protégé (SSH, ZIP, RAR...) :
    ssh2john id_rsa > hash.txt    (ou zip2john, rar2john...)
    john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

3. Si type mal détecté → forcer le format
   john hash.txt --wordlist=... --format=FORMAT

4. Afficher le résultat
   john hash.txt --show

5. Si pas de résultat → essayer avec règles
   john hash.txt --wordlist=... --rules
```

!!! tip "Hash avec `$` dans le shell"
    Comme pour hashcat, utiliser des **guillemets simples** :  
    ```bash
    echo '$P$Ghxmchgk.0YxEQutC7os3dZfBvqGIz/' > hash.txt  # ✅
    ```

!!! tip "Vérifier les hashes déjà crackés"
    John mémorise les résultats dans `~/.john/john.pot`.   
    Si tu relances john sur le même hash sans `--show`, il dira "No password hashes loaded" → utiliser `--show` pour voir le résultat mis en cache.