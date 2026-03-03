# Wordlists & Génération

## Wordlists Essentielles

```bash

/usr/share/wordlists/wfuzz/Injections/All_attack.txt

# SQL Injection
/usr/share/wordlists/wfuzz/Injections/SQL.txt
/usr/share/wordlists/seclists/Fuzzing/Databases/SQLi/

# Directories/Files
/usr/share/wordlists/dirb/common.txt
/usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
/usr/share/wordlists/seclists/Discovery/Web-Content/big.txt
/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt
/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files.txt
/usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt

# Passwords
/usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt
/usr/share/wordlists/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords.txt
/usr/share/wordlists/rockyou.txt
/usr/share/wordlists/seclists/Passwords/Common-Credentials/100k-most-used-passwords-NCSC.txt

# Usernames
/usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
/usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt
/usr/share/wordlists/seclists/Usernames/Names/names.txt

# XSS
/usr/share/wordlists/seclists/Fuzzing/XSS/
/usr/share/wordlists/wfuzz/Injections/XSS.txt

# LFI/RFI
/usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
/usr/share/wordlists/wfuzz/Injections/Traversal.txt

# Command Injection
/usr/share/wordlists/seclists/Fuzzing/command-injection-commix.txt

# Subdomains
/usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# API endpoints
/usr/share/wordlists/seclists/Discovery/Web-Content/api/
```

## 1. Lister Wordlists Kali

**Wordlists principales :**

```bash
ls -lh /usr/share/wordlists
ls -lh /usr/share/dirb/wordlists/
ls -lh /usr/share/seclists
ls -lh /usr/share/wfuzz/wordlist/
```

---

**Installer SecLists (si pas installé) :**

```bash
sudo apt update
sudo apt install seclists

# Ou depuis GitHub
git clone https://github.com/danielmiessler/SecLists.git
```

---

## 2. CeWL - Générer Wordlist Custom

**Génération basique :**

```bash
cewl --write output.txt --lowercase -m 5 http://enum-sandbox/
```

**Options :**

- `--write output.txt` : Fichier sortie
- `--lowercase` : Convertir en minuscules
- `-m 5` : Longueur minimale des mots (5 caractères)
- URL : Site à crawler

---

**Avec profondeur :**

```bash
cewl --write output.txt --lowercase -m 4 -d 2 http://enum-sandbox/manual
```

**Options :**

- `-d 2` : Profondeur crawl (suit liens jusqu'à 2 niveaux)

---

**Inclure metadata :**

```bash
cewl -a --write output.txt --lowercase -m 5 http://enum-sandbox/
```

**Options :**

- `-a` : Include meta data

---

**Inclure emails :**

```bash
cewl --email --write emails.txt http://enum-sandbox/
cewl -e -a --write output.txt http://enum-sandbox/
```

**Options :**

- `--email` ou `-e` : Extract email addresses
- `-a` : Include meta data (author, etc.)

---

**Inclure numéros :**

```bash
cewl --with-numbers --write output.txt http://enum-sandbox/
```

**Options :**

- `--with-numbers` : Accept words with numbers

---

**Compter occurrences :**

```bash
cewl -c --write output.txt http://enum-sandbox/
```

**Options :**

- `-c` : Show count of each word

---

**Output file for meta :**

```bash
cewl -a --meta_file meta.txt --write words.txt http://enum-sandbox/
```

**Options :**

- `--meta_file FILE` : Output file for meta data

---

**Verbose :**

```bash
cewl -v --write output.txt http://enum-sandbox/
```

**Options :**

- `-v` : Verbose

---

**User-Agent personnalisé :**

```bash
cewl --ua "Mozilla/5.0" --write output.txt http://enum-sandbox/
```

**Options :**

- `--ua STRING` : User agent

---

**Alternative à --write :**

```bash
cewl -w output.txt --lowercase -m 5 http://enum-sandbox/
```

**Options :**

- `-w FILE` : Write to file (synonyme de --write)

---

**Utilisation :**

- Crée wordlist basée sur contenu du site cible
- Utile pour : bruteforce passwords, découvrir endpoints

---

## 3. Créer Wordlist de Binaires

**Liste binaires système :**

```bash
ls /usr/bin | grep -v "/" > binaries.txt
```

**Explication :**

- `ls /usr/bin` : Liste contenu de /usr/bin
- `|` : Pipe (envoie résultat à commande suivante)
- `grep -v "/"` : Exclut lignes contenant `/` (garde que fichiers)
- `> binaries.txt` : Redirige dans fichier

**Utilisation :**

- Wordlist pour command injection
- Teste si binaires existent sur cible

---

**Créer wordlist custom :**

```bash
# Combinaisons d'admin
echo -e "admin\nadministrator\nroot\nsuperuser" > users.txt

# Passwords communs
echo -e "password\nPassword123\nadmin\nletmein" > passwords.txt
```

---
